# Architecture

Memorae is a personal memory and productivity assistant: a single Express
API backing a React web/PWA client, a Telegram bot, and a set of Google
integrations — all reading and writing the same database through the same
service layer. This document explains how the pieces fit together and the
design principles that shaped them.

## The one rule everything else follows

**There is exactly one memory system, one command parser, one search
index, and one chat engine — regardless of which channel a request comes
from.** The web app's `POST /api/chat`, the Telegram bot, and the mobile
PWA's quick-capture sheet all call the same
`services/chatEngine/processChatMessage()` function. There is no
Telegram-specific memory table, no mobile-specific task table, no
duplicate business logic anywhere. If you're looking for "where does X
actually happen," it's in `backend/src/services/`, never in a route
handler or a channel-specific adapter.

## High-level layout

```
Memorae Copy/
├── backend/                 Express + TypeScript API, Prisma/SQLite
│   ├── prisma/schema.prisma  the entire data model (33 models)
│   ├── src/
│   │   ├── routes/           thin HTTP layer — auth, validation, calls services
│   │   ├── services/         all actual logic lives here (see below)
│   │   ├── middleware/       auth, rate limiting, error handling
│   │   ├── config/           env.ts (secrets/URLs), models.ts (local model choices)
│   │   └── __tests__/        one file per feature area, real DB + real local models
│   └── scripts/backup.sh|ps1 safe SQLite backup (see BACKUPS.md)
└── frontend/                 React 19 + TypeScript + Vite, installable PWA
    └── src/
        ├── pages/             one file per dashboard section
        ├── components/        layout chrome, UI primitives, capture sheet
        ├── context/           auth, theme, toast, PWA install state, capture sheet
        ├── api/                thin fetch wrapper + typed endpoint bindings
        └── sw.ts               custom service worker (precache + push + offline cache)
```

## Backend service map

Each directory under `backend/src/services/` owns one capability and is
called from routes, from the chat engine, from Telegram, and from
background jobs interchangeably — the same function, not a copy per
caller.

| Service | Owns |
|---|---|
| `memory/` | Memory CRUD, duplicate/conflict detection, merge, history, timeline |
| `memory/reliability.ts` | Duplicate/conflict/outdated/low-confidence/stale-temporary scans |
| `memory/explain.ts` | Deterministic "why did you remember this" provenance |
| `documents/` | Upload, extraction (PDF/DOCX/TXT/MD/CSV/images), chunking |
| `embeddings/` | Local MiniLM embeddings (semantic search) |
| `stt/` | Local Whisper speech-to-text |
| `ocr/` | Local Tesseract OCR |
| `search/` | Hybrid (keyword + semantic) search across 6 source types |
| `extraction/` | AI-driven "is this worth remembering" classification from chat |
| `tasks/`, `reminders/`, `lists/` | The productivity engine |
| `nlp/dates.ts`, `nlp/recurrence.ts` | Natural-language date/recurrence parsing |
| `timezone/` | IANA timezone conversion (DST-correct) |
| `commands/` | Deterministic NL command parser — the thing that turns "remind me tomorrow at 7pm to call Ali" into a real reminder without an AI call |
| `chatEngine/` | The one chat entry point every channel calls (RAG + command parsing + extraction) |
| `ai/` | The `AIProvider` abstraction — OpenAI or local Ollama, interchangeable, response-cached |
| `capture/` | Voice/image upload → transcribe/OCR → structured entity extraction → memory/task/reminder |
| `multimodal/` | Entity-extraction prompts for voice and image content |
| `proactive/` | Daily briefing, weekly review, ad-hoc "what's important" queries, suggestion engine |
| `sharing/` | `requireAccess()` — the single authorization gate for memory/list/task/project sharing |
| `connectors/` | Provider-agnostic OAuth framework; `connectors/google/` is the first (only) implementation |
| `telegram/` | Bot API client, account linking, long-polling/webhook, rate limiting |
| `push/` | Web Push (VAPID) subscriptions and delivery |
| `notifications/scheduler.ts` | Background job: fires a push notification once per due reminder |
| `activity/` | Read-time merge of memory/task/reminder/capture/document/suggestion events for the Activity feed |

## Request flow: a chat message

1. `POST /api/chat` → `routes/chat.ts` → `services/chatEngine/processChatMessage()`.
2. Deterministic command parser (`services/commands`) tries first — task/
   reminder/list creation, status queries, confirmations. No AI call
   needed; this is why "remind me tomorrow at 7pm to call Ali" works with
   zero API keys configured.
3. If nothing matched, `services/search` runs hybrid search (keyword +
   local-embedding cosine similarity) across the user's own memories,
   documents, tasks, reminders, lists, and Drive files — top-K only, never
   the whole database — and builds a source-attributed context block.
4. The configured `AIProvider` (OpenAI or Ollama) generates a reply
   grounded in that context, with an explicit instruction not to claim
   things beyond what's provided.
5. Best-effort, non-blocking: `services/extraction` asks the AI provider
   to classify anything in the exchange worth remembering long-term, and
   persists it as a new `Memory` via the exact same `createMemory()` used
   everywhere else (duplicate detection included).

The Telegram bot and the mobile PWA's "Note" quick-capture both call step
1 directly — there is no separate code path.

## Frontend architecture

React Router SPA, no server-side rendering. Context providers (nested in
`App.tsx`): `ThemeProvider` → `ToastProvider` → `PWAProvider` →
`AuthProvider` → `CaptureProvider`. Pages live under `src/pages/`, one per
dashboard section; `AppShell.tsx` renders the desktop sidebar/topbar or
the mobile topbar/bottom-nav based on a CSS breakpoint (767px), not a JS
media-query branch — both chrome variants are always mounted, only one is
visible at a time, which keeps the mobile bottom-nav's raised Capture
button and the desktop topbar's Capture button wired to the exact same
`useCapture()` hook.

The PWA layer (`sw.ts`, `context/PWAContext.tsx`,
`components/pwa/InstallBanner.tsx`) is additive: precached app shell for
offline loading, a short-lived cache of read-only API GETs, and Web Push
handling. It never introduces a second data path — every mutation goes
through the same `fetch` calls as the desktop build.

## Design principles worth knowing before you change something

- **Local-first, cloud-optional.** Embeddings, speech-to-text, OCR, and
  document parsing are 100% local and always on. AI (OpenAI or Ollama),
  Google, Telegram, and Web Push are each independently optional and the
  app is fully documented to degrade gracefully — never crash — when any
  one of them is unconfigured. See ENVIRONMENT.md and the cost audit in
  README.md.
- **Never claim a source that doesn't exist.** `memory/explain.ts` only
  ever quotes `sourceExcerpt`, captured verbatim at creation time. If none
  was captured, it says so explicitly rather than paraphrasing a summary
  as if it were the user's own words.
- **Never send anything on the user's behalf.** Gmail integration is
  structurally incapable of sending — the OAuth scope granted
  (`gmail.compose`) doesn't include `gmail.send`, so this isn't just an
  app-level guardrail, Google enforces it at the token level too.
- **Never permanently delete automatically.** "Forget" is always a soft
  delete (`status: "deleted"`, recoverable). Permanent purge is a
  separate, explicit, user-only action no automated flow ever calls.
- **Isolation is enforced once, not per-feature.** `requireAccess()` in
  `services/sharing/access.ts` is the single gate every shareable
  resource's routes call; a stranger with no relationship to a resource
  gets `404` (indistinguishable from "doesn't exist"), never `403`
  (which would confirm the resource exists). See SECURITY.md.
