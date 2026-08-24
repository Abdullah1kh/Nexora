# Memorae — Phase 1 + Phase 2 + Phase 3 + Phase 4 + Phase 5 + Phase 6 + Phase 7 + Phase 8 + Phase 9 + Phase 10 + Phase 11 + Phase 12

A personal memory and productivity assistant: store memories, search them,
upload documents, manage tasks/reminders/lists, capture voice notes and
photos/screenshots, chat with an AI that answers using retrieved context
(RAG) and explains where each answer came from, do all of that from
Telegram too, connect your Google account for Calendar/Gmail/Drive, and now
get a daily briefing, a weekly review, and configurable, anti-spam
proactive suggestions — including plain-English commands like "remind me
tomorrow at 7pm to call Ali" and "what should I prioritize?"

- **Phase 1**: project foundation, auth, memory CRUD, basic keyword search, a
  memory-grounded chat interface, dashboard.
- **Phase 2**: a real long-term memory engine (types, importance, confidence,
  source tracking, temporal validity, relationships, duplicate detection,
  merging, correction, archive/restore/forget, history, timeline) with
  automatic AI-driven extraction from conversations, plus a document
  ingestion and hybrid (full-text + semantic) search system across memories
  and documents.
- **Phase 3**: a productivity engine — tasks (with subtasks, projects,
  priorities, deadlines, statuses, recurrence, dependencies, attachments,
  notes), reminders (one-time/recurring, snooze, reschedule, cancel,
  complete, priority), and lists — all understood through natural language
  in chat, with reliable IANA timezone handling and a confirmation step
  before any destructive action.
- **Phase 4**: multimodal capture — upload a voice note and get it
  transcribed locally (Whisper, CPU-only) and turned into a memory/task/
  reminder; upload a photo or screenshot and get it OCR'd locally
  (tesseract.js) and turned into tasks/reminders/memories for any deadlines,
  appointments, contacts, or addresses found in it.
- **Phase 5**: a Telegram bot — text, voice notes, photos, and documents
  sent to the bot are handled by the exact same backend services as the web
  app (same chat engine, same capture pipeline, same document indexer, same
  memory/task/reminder/list tables). No separate bot-side memory or
  database exists. Secure account linking, duplicate-delivery protection,
  and rate limiting included.
- **Phase 6**: a reusable OAuth connector framework, with Google Calendar
  (read/create/update/delete events, conflict detection, timezone-aware),
  Gmail (search, classify, summarize, extract tasks/deadlines/events,
  connect emails to memories, draft-only replies — sending is not just
  disabled, it's not implemented), and Google Drive (discover, index,
  connect to memories, search) as the first connector. OAuth tokens are
  encrypted at rest; the framework is built so another provider can be
  added without touching the token-storage/refresh/CSRF plumbing.
- **Phase 7** (this update): proactive intelligence — a daily briefing and
  weekly review combining calendar/tasks/reminders/emails/projects/
  memories; deterministic answers to "what am I forgetting?", "what's
  important today?", and "what should I prioritize?"; and a rule-based
  proactive suggestion engine (missing reminders, deadline clustering,
  calendar conflicts, unanswered important emails, memory clusters worth
  organizing) with real anti-spam controls — dedup, daily caps, dismissal
  cooldowns, and configurable/disable-able quiet hours.
- **Phase 8**: the complete dashboard — an original, responsive,
  dark/light-mode UI covering all fourteen sections (Today, Calendar, Tasks,
  Reminders, Memories, Inbox, Documents, Lists, Projects, Email, Search,
  Shared Memory, Activity, Settings), a global command palette (⌘K) backed
  by real hybrid search, and a memory-sharing feature (read-only links) and
  unified activity feed built specifically to back this UI. Every screen is
  wired to the real API — nothing here is mock data. See "Phase 8: the
  dashboard" below.
- **Phase 9**: real user-to-user collaboration — shared memories, lists,
  tasks, and projects with three permission levels (view, edit, manage),
  invitation/accept/decline, permission changes, revoke (including
  self-revoke/"leave"), and an append-only audit log. Strict user-level
  data isolation is enforced everywhere: every existing `:id`-addressed
  memory/list/task/project endpoint now runs through a single
  authorization gate before touching the database, and unauthorized
  requests are indistinguishable from a resource simply not existing. See
  "Phase 9: shared memory & collaboration" below.
- **Phase 10**: memory reliability — duplicate, conflicting, outdated,
  low-confidence, and stale-temporary memory detection surfaced as
  reviewable suggestions (KEEP / MERGE / ARCHIVE / FORGET / CORRECT) on a
  new Memory Review screen, reusing the Phase 7 suggestion engine rather
  than a parallel system. Full explainability: "Why did you remember
  this?" / "Where did this come from?" / "Why did you create this
  reminder?" are answered deterministically from real stored data — the
  original excerpt, source type, and (for AI-extracted memories) the
  model's own stated reasoning — never a fabricated quote. See "Phase 10:
  memory reliability & explainability" below.
- **Phase 11**: a full PWA optimized for iPhone —
  installable (manifest + service worker), a redesigned 5-tab mobile nav
  with a raised center "Capture" action, a one-tap voice/camera/quick-note
  bottom sheet reachable from anywhere, offline app-shell caching plus
  short-lived caching of read-only API data, and real Web Push
  notifications for due reminders (VAPID + a background due-reminder
  scheduler) — all built on the exact same backend, account, memory,
  tasks, and integrations as desktop; nothing is duplicated. See "Phase
  11: PWA & mobile" below.
- **Cost audit** (this update): every external service audited for real
  $0 operation — a full local/free-alternative table, a new fully local
  Ollama `AIProvider` alongside OpenAI (pick via `AI_PROVIDER`, defaults
  unchanged), an exact-match AI response cache, Gmail processing batched
  from 3 AI calls to 1, and OCR's implicit CDN dependency closed in favor
  of an explicit local file. See "Cost audit: running Memorae for $0"
  below.

No other external integrations (calendar providers besides Google, other
email providers, etc.) are implemented yet — that's explicitly out of scope
for this phase.

## Architecture

- `backend/` — Express + TypeScript API, Prisma ORM, JWT auth
- `frontend/` — React + TypeScript (Vite), React Router, an original CSS
  design system (`src/styles/`, CSS custom properties, no UI framework/
  component library). Covers every phase's endpoints — see "Phase 8: the
  dashboard" below.
- Database — SQLite for local dev (zero setup), via Prisma. Swappable to
  Postgres for staging/production — only the Prisma datasource provider and
  `DATABASE_URL` change (see Phase 1 notes below).
- AI provider — an `AIProvider` interface (`backend/src/services/ai/`) with
  one implementation, `OpenAIProvider`, selected via `AI_PROVIDER`. Used for
  both chat replies and memory extraction.
- Embeddings — a **local** `EmbeddingProvider`
  (`backend/src/services/embeddings/`) built on `@xenova/transformers`
  running `Xenova/all-MiniLM-L6-v2` fully in-process. No API key, no network
  call at inference time (model weights are cached on disk after the first
  run). This is what powers semantic search — it's real vector search, not a
  keyword hack pretending to be one.
- Documents are stored on local disk only (`backend/uploads/<userId>/…`).
  There is no cloud storage integration in this phase, so "stay local by
  default" is satisfied structurally — there's nowhere else for them to go
  yet.

### Why SQLite instead of Postgres for local dev

Postgres and Docker are not installed on this machine, so local dev uses
SQLite through Prisma. The schema only uses types supported by both engines.
To run against Postgres instead: switch `provider = "sqlite"` to
`"postgresql"` in `backend/prisma/schema.prisma`, set `DATABASE_URL` to a
Postgres connection string, and run `npx prisma migrate dev` for a fresh
Postgres migration.

## Local development setup

Requires Node.js 18+.

```bash
cd backend
cp .env.example .env
npm install
npx prisma migrate dev
npm run dev              # http://localhost:4000
```

```bash
cd frontend
cp .env.example .env
npm install
npm run dev               # http://localhost:5173
```

The first request that touches embeddings (memory create/search/chat) will
download and cache the ~90MB MiniLM model — this needs internet access once;
after that it's fully offline.

To enable chat + automatic memory extraction, set `OPENAI_API_KEY` in
`backend/.env`. Without it, everything except chat still works fully — the
chat endpoint returns a clear 503 explaining the key is missing instead of
faking a reply.

## Environment variables

**backend/.env** (unchanged from Phase 1 — no new required variables)
| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Prisma connection string |
| `PORT` | API port (default 4000) |
| `JWT_SECRET` | Secret used to sign auth tokens |
| `JWT_EXPIRES_IN` | Token lifetime (default `7d`) |
| `AI_PROVIDER` | Which AI provider to use (`openai`) — also used for extraction |
| `OPENAI_API_KEY` | Required for chat replies and memory extraction |
| `OPENAI_MODEL` | Model name (default `gpt-4o-mini`) |
| `CORS_ORIGIN` | Allowed frontend origin |

**frontend/.env**: `VITE_API_URL` (unchanged).

## Phase 2: the memory engine

Every memory now has: `type`, `category`, `tags`, `importance` (1–5),
`confidence` (0–1), `status` (`active`/`archived`/`deleted`), `sourceType`
(`manual`/`chat`/`document`/`extraction`) + `sourceId`/`sourceExcerpt`,
`validFrom`/`validUntil`, and a real embedding for semantic matching.

`type` is one of `temporary | context | long_term | task | reminder | event |
list_item | project_info`. `temporary` is a signal, not a storage bucket —
the extractor uses it to mean "don't keep this," so nothing ever gets
persisted with that type (see extraction below).

### Endpoints (`/api/memories`)

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | List, filterable by `q`, `status`, `type`, `category`, `tag` |
| GET | `/timeline` | Chronological view, filterable by `from`/`to`/`type` |
| POST | `/` | Create — returns `409` with candidate duplicates unless `?force=true` |
| GET | `/:id` | Fetch one |
| PUT | `/:id` | Update (recomputes embedding, logs history as `updated`) |
| POST | `/:id/correct` | Same as update but logged as `corrected`, with an optional `reason` |
| GET | `/:id/history` | Full audit trail (every create/update/correct/merge/archive/restore/delete/purge, with a snapshot of the prior state) |
| GET | `/:id/related` | Explicit relations (duplicate/related/conflicts/supersedes/part_of) plus semantically similar memories |
| GET | `/:id/duplicates` | Near-duplicate candidates by embedding similarity |
| POST | `/:id/archive` / `/:id/restore` | Soft state transitions |
| DELETE | `/:id` | "Forget" — soft delete (`status=deleted`, recoverable). Add `?permanent=true` to hard-delete (purge) |
| POST | `/merge` | `{ primaryId, duplicateIds[] }` — merges content/tags into the primary, archives the rest, links them via a `supersedes` relation |

### How duplicate detection and conflict detection work

- **Duplicate**: cosine similarity of the new memory's embedding against all
  active memories ≥ 0.93 (near-identical content). Creation is blocked (409)
  unless `force=true`; the caller can then merge instead of creating a
  near-copy.
- **Conflict**: similarity in the 0.55–0.93 band *and* meaningful tag/category
  overlap — i.e. plausibly the same fact, but the text differs (a value
  changed). These are recorded as `conflicts` relations in both directions
  and surfaced via `/related` and in the `create` response's `conflicts`
  field — they are flagged for the user, not silently auto-resolved, since
  deciding which version is correct isn't something the system can safely
  guess.

### Automatic extraction from conversations

After every chat exchange, `services/extraction` sends the single
user/assistant turn to the AI provider with a strict system prompt asking it
to classify discrete facts by type and emit structured JSON — or an empty
array if nothing in the exchange is worth remembering. Small talk, one-off
context, and anything the model marks `temporary` never reach the database:
the extractor drops `temporary` items outright and only persists candidates
that clear a confidence threshold (0.5). This is the mechanism that keeps the
system from blindly turning every conversation into permanent memory. Each
persisted memory carries `sourceType: "extraction"`, `sourceId` (the chat
message it came from), and `sourceExcerpt` (the triggering user message), so
you can always trace a memory back to the conversation that produced it.
Extraction runs best-effort — if the AI call fails or returns malformed
output, it degrades to "nothing extracted" rather than breaking the chat
response.

## Phase 2: documents and hybrid search

### Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/documents` | List, with chunk counts |
| POST | `/api/documents` | Upload (`multipart/form-data`, field `file`, 20MB limit) |
| GET | `/api/documents/:id` | Detail, including extracted text |
| POST | `/api/documents/:id/reindex` | Re-run extraction + chunking + embedding from the stored file |
| DELETE | `/api/documents/:id` | Delete the document, its chunks, and the file on disk |
| GET | `/api/search?q=…` | Hybrid search — see below |

Supported formats: **TXT, Markdown, CSV** (read directly), **PDF** (via
`pdf-parse`, which wraps `pdf.js`), **DOCX** (via `mammoth`), and
**images/screenshots** (PNG/JPEG/WebP/BMP via `tesseract.js` OCR — all
local, no cloud OCR API). Duplicate uploads are detected by SHA-256 content
hash per user — re-uploading the same file returns the existing indexed
document (`200` + `duplicate: true`) instead of indexing it twice.

Text is chunked (~800 chars, ~120 char overlap, breaking on whitespace) and
each chunk gets its own embedding, so search and retrieval work at
paragraph granularity rather than whole-document granularity.

### Search: `GET /api/search`

Query params: `q`, `sources` (`memory,document`), `category`, `tag`, `type`
(memory type), `sourceType`, `dateFrom`/`dateTo` (ISO datetimes), `limit`.

Each result is scored as `0.6 × semanticScore + 0.4 × keywordScore` (both
0–1), where `semanticScore` is cosine similarity between the query embedding
and the memory/chunk embedding, and `keywordScore` is the fraction of query
words found verbatim. Results below both thresholds are dropped; the rest
are sorted by combined score. Every result carries a `source` object
(`sourceKind`, id, and for documents the filename + chunk index) so results
are always attributable, never anonymous.

### RAG in chat, not "send the whole database"

`POST /api/chat` calls the same hybrid `search()` function, capped at the
top 5 results, and only those snippets are injected into the AI prompt as
context (with `[source:memory:<id>]` / `[source:document:<id>]` tags so the
model can cite them). The rest of the user's memories/documents are never
sent. The chat response includes a `sources` array (kind, id, title, score)
so the frontend/API consumer can show exactly what informed the answer.

## Phase 3: the productivity engine

Tasks, reminders, and lists, each with full REST CRUD (`/api/tasks`,
`/api/projects`, `/api/reminders`, `/api/lists`) **and** natural-language
control through the same `POST /api/chat` endpoint used for everything else.

### Timezone handling

Every user has a `timezone` (IANA name, e.g. `America/New_York`, set at
registration or via `PATCH /api/auth/me`, defaults to `UTC`). All dates are
stored in UTC; `backend/src/services/timezone/` converts between UTC and a
user's wall-clock time using only `Intl` (no bundled tz database) — verified
against real DST transitions in the test suite (EST vs. EDT in New York,
and a non-DST zone). Natural-language phrases ("tomorrow at 7pm", "next
Friday", "in 30 minutes") are parsed with `chrono-node` against a
timezone-correct reference "now" (a documented fake-local-time trick — see
comments in `dates.ts`), so the same phrase produces different, *correct*
absolute times for users in different timezones.

### Natural-language commands (deterministic, no AI key required)

`backend/src/services/commands/` recognizes command shapes like the ones in
the spec and executes them directly — no AI call, so this works offline and
is exact rather than guessed:

- `"Remind me tomorrow at 7 PM to call Ali."` → one-time reminder
- `"Every Monday remind me to review finances."` → recurring reminder
  (defaults to 9am local when no time is given)
- `"Add milk to my grocery list."` → creates the list if it doesn't exist,
  adds the item (supports multiple items: "add milk and eggs to...")
- `"Create a task to finish the website by Friday."` → task with a due date
  (defaults to end-of-day when no time is given)
- `"Show my overdue tasks."` / `"What's on my grocery list?"` → answered
  directly from the database, no AI/hallucination risk
- Also handled: complete/cancel task or reminder by title, snooze/reschedule
  a reminder, list reminders/tasks

Anything that doesn't match a recognized shape falls through to the normal
RAG+AI chat flow unchanged, so open-ended questions still work when an AI
key is configured (and that flow's retrieval now also searches tasks,
reminders, and list items — see "integration" below).

### Confirmation before destructive actions

Cancelling or deleting a task/reminder/list/list-item via chat **never
executes immediately**. It creates a `PendingConfirmation` row and asks
`Are you sure you want to cancel "X"? Reply "yes" to confirm.` The action
only runs if the very next message is affirmative; any other reply (or a
"no") discards it with no changes made. Confirmations expire after 5
minutes. Non-destructive, reversible actions (create, complete, snooze,
reschedule) execute immediately — only actual data loss is gated.

### Recurring tasks and reminders

Recurrence is parsed from phrases like "every Monday", "daily", "every 2
weeks" into a small JSON spec, and next-occurrence math is delegated to the
`rrule` library (anchored to the user's timezone). Completing a recurring
*reminder* advances it to the next occurrence and keeps it pending — it
never just disappears. Completing a recurring *task* marks that instance
done and spawns a new task row for the next occurrence, linked via
`recurrenceRootId`.

### Tasks: subtasks, dependencies, attachments, notes

Tasks can have `parentTaskId` (subtasks), `projectId` (grouping), a
`TaskDependency` graph (blocked-by relationships, cycle-checked on
creation — `GET /api/tasks/:id` returns a computed `blocked` flag), append-only
`TaskNote`s, and file `TaskAttachment`s (stored locally, same pattern as
documents).

### Integration with the memory system, chat, and dashboard

- Tasks, reminders, and list items are indexed by the same `search()`
  service as memories/documents (`sources=task,reminder,list_item`), so
  asking about them in normal chat retrieves them via the same RAG path.
- `GET /api/dashboard/summary` now also returns `taskCount`,
  `overdueTaskCount`, `reminderCount`, `listCount`, `upcomingTasks`, and
  `upcomingReminders` alongside the Phase 1/2 memory stats — one summary,
  one source of truth, whether you look via chat, the dashboard endpoint, or
  direct REST calls.

## Phase 4: multimodal capture

Upload an audio file or a photo/screenshot (`POST /api/capture/voice` or
`POST /api/capture/image`, `multipart/form-data`, field `file`) and get back
a `Capture` record plus whatever memories/tasks/reminders were created from
it. Everything is `GET`-able afterwards at `/api/capture` and
`/api/capture/:id`.

### Model choices — picked for a Mac M2 (16GB) and a GTX 1660 SUPER (6GB)

Both local models here run **on CPU only** (via ONNX Runtime through
`transformers.js`, and a small native binary for OCR) — no GPU/CUDA setup
required at all. That was a deliberate choice, not a limitation: it means
identical behavior on both of your machines, no VRAM budget to manage, and
no risk of picking a model that's "impractical for your hardware" because
CPU inference for these model sizes is fast and low-memory regardless of
what GPU is or isn't present.

| Capability | Model | Size | Why |
|---|---|---|---|
| Speech-to-text | `Xenova/whisper-base.en` (default) | ~145MB | Good accuracy/speed balance; transcribes a few seconds of speech in ~1-2s on CPU |
| Speech-to-text (lighter option) | `Xenova/whisper-tiny.en` | ~75MB | Swap in via `STT_MODEL` if you want faster/smaller at the cost of some accuracy |
| Speech-to-text (heavier option) | `Xenova/whisper-small.en` | ~485MB | Swap in via `STT_MODEL` for noticeably better accuracy; still CPU-friendly, just slower |
| OCR | tesseract.js (`eng` by default) | a few MB | CPU-only, no GPU needed, mature and accurate for printed/screenshot text |
| Semantic embeddings | `Xenova/all-MiniLM-L6-v2` (Phase 2, reused here) | ~90MB | Already in use for memory/document search |
| Structured extraction | your configured `AIProvider` (optional) | — | Turning a transcript/OCR text into person/amount/task/reminder entities needs real language understanding; this reuses the same pluggable `AIProvider` abstraction as chat and Phase 2 memory extraction rather than bundling a separate local LLM (a local LLM capable of reliable structured extraction — even a small 1-3B model — is multiple GB and meaningfully slower on this hardware than the sub-500MB models above; not a good tradeoff for this step) |

**Explicitly avoided**: Whisper `medium`/`large` (multi-GB, too slow on
CPU for this hardware) and any local vision-language model for "true" image
understanding beyond OCR (the smallest usable ones are still 1.5GB+ and
meaningfully slower than tesseract.js; OCR + text-based structured
extraction covers everything the spec asks for — text, dates, deadlines,
appointments, tasks, addresses, contacts — without that cost).

**Model configuration** — every model is swappable without touching code,
via `backend/.env`:

| Variable | Default | Purpose |
|---|---|---|
| `STT_MODEL` | `Xenova/whisper-base.en` | Any `Xenova/whisper-*` transformers.js model |
| `OCR_LANGUAGE` | `eng` | Any tesseract.js language code |
| `EMBEDDING_MODEL` | `Xenova/all-MiniLM-L6-v2` | Any transformers.js feature-extraction model |
| `AI_PROVIDER` / `OPENAI_MODEL` | `openai` / `gpt-4o-mini` | Powers the structured-extraction step (shared with chat/Phase 2) |

See `backend/src/config/models.ts` for the single place these are read from.

### Voice: transcription + extraction

Any audio format ffmpeg can read (webm, m4a, ogg, wav, aiff, mp3, ...) is
accepted — it's transcoded to 16kHz mono WAV (via a bundled `ffmpeg-static`
binary, no system ffmpeg install required) before being fed to Whisper. The
transcript is always saved, even if the extraction step below can't run.

If an AI provider is configured, the transcript is sent through
`services/multimodal/voiceExtraction.ts` to pull out exactly the fields the
spec asks for:

```
"I need to remember that Ahmed owes me 15,000 and remind me next Friday."
  → person: "Ahmed"
  → memory: "Ahmed owes me 15,000"
  → amount: 15000
  → task: null
  → reminder: { content: "...", when: "next Friday" }
```

`person`/`memory` become a Memory (`sourceType: "voice"`), `task` becomes a
Task, and `reminder` becomes a Reminder — `when` is resolved to an absolute,
timezone-correct date using the same natural-language date parser from
Phase 3. Every created record's `sourceId` points back at the `Capture`, so
you can always trace it to the voice note it came from.

### Images/screenshots: OCR + extraction

`services/multimodal/imageExtraction.ts` takes the raw OCR text and (again,
only if an AI provider is configured) extracts `dates`, `deadlines`,
`appointments`, `tasks`, `addresses`, `contacts`, and `usefulInfo` as
arrays. Each is turned into the appropriate record: deadlines → Tasks
(with a due date if one parses), appointments → Reminders (if a date/time
parses), tasks → Tasks, contacts/addresses/useful info → Memories
(`sourceType: "image"`, category `contact`/`address`/`general`). Memory
creation goes through the same Phase 2 duplicate-detection path, so
re-processing the same screenshot twice doesn't create duplicate memories.

### What still works with no AI key configured

OCR and transcription are 100% local and always run, regardless of AI
configuration — you get a `Capture` record with the real transcript/OCR
text back either way. Only the "turn this into memory/task/reminder
records" step is skipped when no `OPENAI_API_KEY` is set (same honest
degrade-gracefully behavior as chat and Phase 2 extraction — verified live
in this session, see below).

## Phase 5: Telegram integration

The core design goal: **the bot is not a second app.** Every message it
handles is routed through the exact same services the REST API and web app
use — there is no Telegram-specific memory store, no separate command
parser, no duplicate database. Concretely:

- Text messages call `services/chatEngine::processChatMessage()` — the same
  function `POST /api/chat` calls. Same command parser, same RAG search,
  same AI provider, same extraction pipeline, same `ChatMessage` history
  table.
- Voice notes call `services/capture::createVoiceCapture()` — the same
  function `POST /api/capture/voice` calls. Same local Whisper transcription,
  same entity extraction, same `Capture` table.
- Photos call `services/capture::createImageCapture()` — same OCR pipeline
  as `POST /api/capture/image`.
- Non-image documents call `services/documents::createDocument()` — same
  chunking/embedding/duplicate-detection as `POST /api/documents`.

The `routes/chat.ts`, `routes/capture.ts`, and `routes/documents.ts` REST
handlers are now themselves thin wrappers around these same services —
Phase 5 didn't just reuse them, it made the web API and the bot two callers
of one shared core (see `backend/src/services/chatEngine/index.ts`).

### Secure account linking

1. A logged-in web user calls `POST /api/telegram/link-code` and gets back
   an 8-character single-use code, valid for 10 minutes (ambiguous
   characters like `0`/`O` and `1`/`I` excluded so it's easy to type).
2. They send `/link CODE` (or open a `t.me/YourBot?start=CODE` deep link,
   which Telegram delivers to the bot as `/start CODE`) from their Telegram
   account.
3. The bot atomically marks the code used and creates a `TelegramAccount`
   row binding that Telegram user id to the Memorae user id. The code
   cannot be replayed, and if a different Memorae account tries to claim a
   Telegram account that's already linked elsewhere, it's rejected.
4. From then on, every message from that Telegram chat resolves to that
   user id before anything else happens. An unlinked chat gets linking
   instructions and nothing else — no memory access, no AI calls.

### Duplicate messages and rate limits

- **Duplicates**: Telegram's delivery guarantee is "at least once" — webhook
  retries and overlapping poll windows can redeliver the same update.
  Every `update_id` is claimed exactly once via a unique-constraint insert
  (`TelegramProcessedUpdate`) before any processing happens; redeliveries
  are silently skipped.
- **Incoming rate limit**: a chat is capped at 20 processed messages/minute;
  beyond that it gets a "slow down" reply instead of triggering more
  AI/transcription/OCR work.
- **Outgoing rate limit**: replies to a given chat are spaced at least 1
  second apart in-process, matching Telegram's own per-chat limit, so the
  bot doesn't get throttled when it needs to send several messages in a row
  (e.g. "transcribing..." then the transcript).
- Both limiters are in-memory and per-process — correct for the single-process
  deployment this app runs as; a multi-instance deployment would need to
  move them to a shared store (e.g. Redis), noted here rather than built
  since it's out of scope at this stage.

### Errors

Every update is processed inside a try/catch; any failure (a bad file
download, a transcription error, an AI provider error) results in a plain
"Something went wrong processing that — please try again" reply rather than
a crash or silence. The underlying error is logged server-side for
debugging.

### Running the bot

```bash
# In backend/.env:
TELEGRAM_BOT_TOKEN="<token from @BotFather>"
TELEGRAM_POLLING_ENABLED="true"
```

Then `npm run dev` — the poller starts automatically (a no-op if the token
isn't set) and long-polls Telegram, so no public URL is needed for local
use. For production, set `TELEGRAM_WEBHOOK_SECRET` instead and point
Telegram's `setWebhook` at `POST /api/telegram/webhook` (helper functions in
`services/telegram/client.ts`); the endpoint verifies Telegram's secret
header before processing anything.

### What it understands (same examples as the spec, all working end-to-end)

- `"Remember that my passport expires in March 2028."` → memory (via the AI
  extraction path, same as any chat message)
- `"Remind me tomorrow at 6."` → reminder (deterministic command parser)
- `"What's on my schedule?"` → AI+RAG path over tasks/reminders/memories
- `"Add eggs to groceries."` → list item (deterministic command parser)
- `"Find everything I said about Project X."` → AI+RAG search over memories
- A destructive command like `"cancel task X"` → asks for confirmation and
  waits for "yes"/"no", exactly like the web chat — because it *is* the web
  chat's confirmation system (`PendingConfirmation` is keyed by user id, not
  by channel)

## Phase 6: Google connectors (Calendar, Gmail, Drive)

### The reusable connector framework

`backend/src/services/connectors/` is provider-agnostic:

- `types.ts` — the `OAuthProviderConfig` shape any provider implements
  (authorize/token/revoke URLs, scopes, client credentials)
- `oauthFlow.ts` — generic authorize-URL building, code exchange, refresh,
  and revoke, parameterized by an `OAuthProviderConfig` — none of this code
  mentions Google
- `connection.ts` — generic encrypted token storage and
  `getValidAccessToken()`, keyed by `(userId, provider)` — this is what
  every API call (Calendar/Gmail/Drive today) goes through to get a fresh
  token, transparently refreshing when needed
- `crypto.ts` — AES-256-GCM encryption for tokens at rest

Everything under `services/connectors/google/` (config, API clients for
Calendar/Gmail/Drive) is the Google-specific *implementation* of that
framework. Adding a second provider (e.g. Microsoft/Outlook) means writing
an equivalent `services/connectors/microsoft/` directory and one more
`OAuthProviderConfig` — the CSRF state handling, encrypted storage, and
refresh logic are already done.

### Secure OAuth authentication

- `GET /api/connectors/google/connect` (authenticated) issues a signed,
  single-use, 10-minute CSRF `state` (`OAuthState` table) and returns
  Google's consent URL (`access_type=offline`, `prompt=consent` — so a
  refresh token is always issued).
- `GET /api/connectors/google/callback` (Google redirects the browser here
  — no app JWT is available at this point, which is why the CSRF state
  carries the user identity) validates the state, exchanges the code,
  fetches the Google profile, and stores the tokens.
- **Tokens are never stored in plaintext.** `accessTokenEnc`/
  `refreshTokenEnc` are AES-256-GCM ciphertext; decryption only happens
  in-memory, per-call, inside `getValidAccessToken()`. Verified directly in
  tests: the stored ciphertext does not contain the plaintext token as a
  substring, and decrypting it recovers the original value.
- `GET /api/connectors/google/status` and `POST /api/connectors/google/disconnect`
  round out the flow — disconnect revokes both tokens with Google
  (best-effort) and deletes the local `Connection` row.

### Google Calendar

`backend/src/services/connectors/google/calendar.ts` — `listEvents`,
`createEvent`, `updateEvent`, `deleteEvent`, `getEvent`.

- **Conflict detection**: before creating or moving an event, existing
  events in that time window are fetched and checked for overlap. A
  conflict returns `409` with the colliding events (mirroring the Phase 2
  memory-duplicate UX) unless `?force=true`.
- **Timezone handling**: event start/end are sent to Google as an absolute
  UTC instant (`dateTime`) plus an explicit IANA `timeZone` for
  display/recurrence — the same reliable timezone approach as Phase 3, not
  a second implementation.

### Gmail

`backend/src/services/connectors/google/gmail.ts` (API access) +
`emailIntelligence.ts` (AI-backed analysis):

- **Search**: `GET /api/gmail/search?q=...` — real Gmail search syntax,
  results cached locally in `EmailRecord` (not a mailbox mirror — only
  messages you've looked at get a row).
- **Classify / summarize / extract**: `POST /api/gmail/messages/:id/process`
  runs all three via the same `AIProvider` abstraction used everywhere else
  in the app — classification into task/receipt/newsletter/personal/
  important/spam/other, a one-line summary, and structured
  tasks/deadlines/events extraction (same JSON-extraction pattern as Phase
  2/4's memory and voice/image extraction).
- **Connect emails to memories**: `POST /api/gmail/messages/:id/link-memory`
  creates a Memory with `sourceType: "email"`, going through the same
  Phase 2 duplicate-detection path as any other memory.
- **Draft replies — never send**: `POST /api/gmail/messages/:id/draft-reply`
  generates reply text via AI and creates a **Gmail draft** via the API's
  `drafts.create`. This is enforced two ways, not one: (1) there is no
  send code path anywhere in this app, and (2) the OAuth scope requested is
  `gmail.compose`, which Google does not grant send permission under at
  all — even a bug in this codebase couldn't send an email, because the
  access token itself isn't authorized to. Verified in tests by asserting
  no request to any Gmail `/send` endpoint is ever made.

### Google Drive

`backend/src/services/connectors/google/drive.ts`:

- **Discover**: `GET /api/drive/files` lists Drive files (metadata only).
- **Index**: `POST /api/drive/files/:id/index` downloads (or, for Google
  Docs/Sheets/Slides, exports as plain text/CSV) the file, then reuses the
  exact same `extractText`/`chunkText`/`embedText` pipeline as local
  document uploads (Phase 2) — no separate extraction logic for Drive
  files. Unsupported types (video, etc.) are marked `unsupported`, not
  treated as an error.
- **Connect to memories**: `POST /api/drive/files/:id/link-memory` — same
  pattern as Gmail.
- **Search**: Drive file chunks are a new `drive_file` source in the same
  central hybrid `search()` used by memories/documents/tasks/reminders/
  lists — one search endpoint, one ranking algorithm, across everything.

## Phase 7: proactive intelligence

### Daily briefing and weekly review

`GET /api/proactive/daily-briefing` and `GET /api/proactive/weekly-review`
(also reachable in chat/Telegram as `"daily briefing"` / `"weekly review"`)
combine, via `services/proactive/context.ts`:

- Calendar (if Google is connected — today's events / this week's load)
- Tasks: overdue, due today (daily) or completed/unfinished/overdue/
  upcoming deadlines (weekly)
- Reminders: due today (daily) or upcoming (weekly)
- Important emails (cached, classified `EmailRecord`s from Phase 6)
- Active projects
- Relevant memories (high-importance, recently updated)

The structured data (`sections`) is always complete and correct on its own;
a `narrative` field additionally asks the AI provider to turn it into a few
sentences of prose, and — like every other AI-optional feature in this
app — falls back to a plain concatenation of the facts if no AI provider is
configured, rather than failing. Both are logged to `BriefingLog` for
history.

**Timezone note**: gathering "today"/"this week" now correctly uses the
user's IANA timezone (via the same `services/timezone` used since Phase 3)
rather than the server process's local time — this also fixed a
Phase-3-era bug where `Task`'s `dueToday` filter used server-local day
boundaries; it now accepts an optional `timeZone` and defaults to UTC when
not given.

### Ad-hoc queries

`"What am I forgetting?"`, `"What's important today?"`, and `"What should I
prioritize?"` are recognized as deterministic commands in
`services/commands/index.ts` (same dispatcher as Phase 3's task/reminder/
list commands) and answered directly from the gathered context — ranked by
overdue-first then priority for "prioritize", combining overdue
tasks/reminders and open suggestions for "forgetting". No AI call is
required for these three, so they work offline and can't hallucinate.

### Proactive suggestions

`services/proactive/suggestions.ts` runs five rules on demand
(`POST /api/proactive/suggestions/generate` — call this periodically, e.g.
once when the user opens the app, or wire it to a scheduler; no background
job runner is part of this phase):

| Rule | Example |
|---|---|
| `missing_reminder` | A memory typed `task`/`reminder` (e.g. from voice/chat extraction) with no matching Task/Reminder title → "You mentioned 'renew passport' but haven't created a reminder or task for it." |
| `deadline_cluster` | 3+ open tasks due the same day → "You have 3 deadlines on Friday." |
| `calendar_conflict` | Two Google Calendar events overlap in the next 7 days → "You have a calendar conflict." |
| `unanswered_email` | An `important`/`task`-classified email, >24h old, with no drafted reply → "You have an unanswered important email." |
| `memory_cluster` | 5+ active memories share a tag → "You have several memories about Project X. Would you like me to organize them?" |

`GET /api/proactive/suggestions` returns pending ones;
`POST .../:id/dismiss` and `POST .../:id/act` resolve them (acting on a
`missing_reminder` suggestion creates the actual reminder in one step).

### Configurability and anti-spam (the actual requirement, not an afterthought)

`GET`/`PUT /api/proactive/settings` (`ProactiveSettings`, one row per user):

- `enabled` — a single master switch; off means zero suggestion generation
  *and* zero surfacing, not just a hidden UI toggle
- `maxSuggestionsPerDay` (default 5) — a hard cap on new suggestions
  created per rolling 24h window, enforced while rules run, not just when
  displaying results, so a burst of qualifying data can't flood the table
- `enabledRuleTypes` — pick which of the five rules run at all
- `quietHoursStart`/`quietHoursEnd` (optional local hours, wraps past
  midnight) — suggestions still generate quietly, but `GET /suggestions`
  returns nothing during quiet hours; nothing is ever pushed, so "quiet"
  just means "not surfaced right now"

Anti-spam mechanics, layered:

1. **Dedup** — every candidate has a stable `dedupeKey` derived from the
   underlying fact (e.g. the memory id, or the sorted pair of conflicting
   event ids). A `@@unique([userId, dedupeKey])` constraint means the same
   fact can never produce two pending rows.
2. **Dismissal cooldown** — dismissing a suggestion suppresses that exact
   `dedupeKey` for 7 days before it can be regenerated.
3. **Daily cap** — independent of the above, no more than
   `maxSuggestionsPerDay` new rows are created per user per rolling 24h,
   even if every rule fires simultaneously.

## Phase 8: the dashboard

### Design system, not a copy of anything

`frontend/src/styles/theme.css` defines an original set of design tokens —
a signature indigo/violet primary (`#5b4bdb` light / `#8b7cf6` dark) paired
with a warm amber accent for attention states, warm-neutral (not pure gray)
surfaces, and independently-tuned light and dark palettes — plus spacing,
radius, and shadow scales used consistently across every page. Icons are a
small original inline-SVG set (`components/ui/Icon.tsx`) drawn for this
project rather than an imported icon library. No Memorae branding, assets,
or source from elsewhere was copied; this is a from-scratch original UI in
the spirit of modern AI productivity apps (command palette, keyboard-first,
minimal chrome), not a clone of one.

### The fourteen sections

All fourteen requested sections exist as real, functional pages wired to
the actual API (`frontend/src/pages/`) — **Today** (daily briefing +
suggestions + upcoming tasks/reminders/memories, the landing page after
login), **Calendar** (agenda view, create with conflict detection, delete),
**Tasks** (filter by open/overdue/done/all, create with priority/due date,
complete/cancel/delete), **Reminders** (create, snooze, complete, cancel),
**Memories** (grid explorer with type filter and search, detail modal with
related memories, edit/archive/delete/share), **Inbox** (Gmail messages
classified important/actionable, draft-reply, link-to-memory), **Documents**
(upload + status, plus Drive discovery/indexing side by side), **Lists**
(two-pane list/detail, add/complete/remove items), **Projects** (create,
task counts, archive), **Email** (full Gmail search with
classify/summarize/extract/draft/link actions), **Search** (the same hybrid
search as the command palette, with a source-type filter, as a full page),
**Shared Memory** (manage active share links; the actual public,
unauthenticated share view lives at `/shared/:token`), **Activity** (a
day-grouped timeline of memory edits, completed tasks/reminders, processed
captures, indexed documents, and resolved suggestions), and **Settings**
(profile/timezone, appearance, Google connector status/connect/disconnect,
Telegram linking, and full proactive-suggestion configuration).

### Cross-cutting features

- **Global search** — the topbar's "Search everything" trigger (or ⌘K/Ctrl+K
  from anywhere) opens a command palette that queries the real
  `GET /api/search` hybrid endpoint as you type, plus fuzzy-matches
  section names for quick navigation. A full `/search` page offers the same
  search with explicit source/date filtering.
- **Command/chat interface** — `/chat` (also reachable via the sidebar's
  "Ask AI" button) is the same `POST /api/chat` used since Phase 1,
  restyled — it's the one place natural-language commands, AI Q&A, and
  everything the command parser understands (Phase 3-7) are available from
  the UI. Verified live: typing "Add bread to my groceries list." in the
  chat UI created a real list, visible immediately on the Lists page.
- **Timeline** — the Activity page's day-grouped feed, and Memories' related
  memories.
- **Task views** — filterable list view (open/overdue/done/all) with
  priority badges and blocked-task indicators.
- **Calendar** — an agenda (day-grouped list) view rather than a month
  grid — deliberate: it reads Google Calendar data directly and stays
  simple/fast rather than reimplementing a calendar-grid widget.
- **Memory explorer** — card grid with type/category badges, importance
  stars, tag chips, and a detail view showing related memories and full
  content.
- **Document explorer** — local documents and Drive files shown together,
  with status badges (processing/indexed/failed/unsupported) and one-click
  Drive indexing.
- **AI suggestions** — surfaced on Today (act/dismiss inline) and fully
  configurable on Settings.
- **Activity history** — see above.

### New backend built specifically to support this dashboard

Two small, genuinely functional features exist because the dashboard's
spec named them as sections, not because they were part of an earlier
phase:

- **Shared Memory** (`services/sharing/`, `Memory Share` table) — a
  read-only, revocable share link for a single memory
  (`POST /api/sharing/memories/:id`), resolved publicly and
  unauthenticated at `GET /api/shared/:token` (a share token *is* the
  credential — no separate multi-user permissions model exists, by design,
  since none was built in any earlier phase). Verified live: created a
  share link, opened it in a fresh unauthenticated view, saw the memory
  read-only; revoking it correctly returns 410 afterward.
- **Activity feed** (`services/activity/`) — a read-time merge across
  `MemoryHistory`, completed `Task`/`Reminder` rows, `Capture`, `Document`,
  `Suggestion`, and `BriefingLog` — deliberately *not* a new denormalized
  log table, so it can never drift from the records it summarizes.

### Responsive and accessible

- **Desktop** (≥768px): fixed sidebar with all 14 sections, sticky topbar
  with search trigger and "Ask AI" button.
- **Mobile** (<768px, tested at 375×812): a compact top bar (hamburger +
  page title + search icon) opening a slide-in drawer with the full nav,
  and a 4-item bottom tab bar (Today/Tasks/Search/Chat) for the most-used
  sections — verified live via the browser's mobile viewport preset,
  including the drawer opening correctly with the full nav list and user
  footer.
- Focus-visible outlines on every interactive element, semantic
  `<nav aria-label>` landmarks, `aria-label`s on icon-only buttons, and
  color choices checked for contrast in both themes.
- **Dark/light/system** theme, persisted to `localStorage`, verified live
  in both directions (screenshotted the same Today page in dark and light).

## Phase 9: shared memory & collaboration

### The core problem: sharing without weakening isolation

Every route that reads or writes a single memory/list/task/project used to
do a raw `where: { id, userId: req.userId! }` ownership check — correct for
"my own data only," but with no concept of "someone else's data I've been
given access to." Phase 9 needed to add that without reopening the
isolation guarantees every earlier phase depends on.

The fix is one authorization gate, `requireAccess()`
(`backend/src/services/sharing/access.ts`), that every `:id`-addressed
memory/list/task/project route now calls first:

1. Look up the resource's real owner (a small per-type lookup: `Memory`,
   `List`, `Task`, `Project` all have a `userId` column).
2. If the caller *is* the owner, access is `"owner"` — full access, no
   further lookup needed.
3. Otherwise, look for a `ShareGrant` row for `(resourceType, resourceId,
   granteeUserId: caller, status: "accepted")`. If one exists, access is
   that grant's permission (`view` | `edit` | `manage`).
4. If neither, the caller has no relationship to the resource at all.

The route then calls the *existing, already-tested* service function
(`memoryService.getMemory`, `taskService.updateTask`, etc.) using the
**resolved owner's ID**, not the caller's ID — so a collaborator's request
transparently succeeds against service-layer code that was never rewritten
to understand sharing, while the route layer independently enforces (and
audits) the permission check before that call happens.

### Anti-enumeration: 404 vs. 403 is a deliberate security choice, not an accident

`requireAccess(userId, resourceType, resourceId, minPermission)`:

- **No relationship to the resource at all → `404`.** This is
  indistinguishable from "this resource doesn't exist," which is exactly
  the point — a stranger probing random IDs cannot tell the difference
  between "wrong ID" and "someone else's private memory," so they learn
  nothing by probing.
- **A real but insufficient grant → `403`.** Once a user has *any* accepted
  grant on a resource, its existence is already known to them, so there's
  nothing left to hide — returning a clear "you don't have edit access"
  is more useful than another 404, and every 403 in this path also writes
  an `access.denied` audit entry.

This rule is applied uniformly across memories, lists, tasks, and projects
— see `routes/memories.ts`, `routes/lists.ts`, `routes/tasks.ts`,
`routes/projects.ts`.

### Permission levels

Three ranked levels (`view < edit < manage`), enforced per-endpoint:

- **view** — read the resource (`GET /:id` and read-only sub-routes like
  `/history`, `/related`).
- **edit** — read + modify content (`PUT /:id`, add/complete list items,
  change task status, add task notes/attachments/dependencies, correct/
  archive/restore a memory).
- **manage** — edit + control sharing itself: invite/uninvite other
  collaborators, change anyone's permission level, and delete the resource
  (`DELETE /:id`). Deletion requires `manage`, not just `edit`, so an
  editor can't unilaterally destroy something they were only given
  content-editing rights over.

The owner implicitly has every permission and does not need a `ShareGrant`
row.

### Invitations, acceptance, and revoke (`services/sharing/collaboration.ts`)

- **`POST /api/collaboration/invite`** — requires `manage` on the resource.
  The invitee must already have a registered Memorae account (looked up by
  email); inviting an unregistered email returns `404` rather than
  creating a dangling invitation for someone who may never sign up.
  Inviting the resource's own owner is rejected (`400`). Grants are
  upserted on `(resourceType, resourceId, granteeEmail)`, so re-inviting
  the same person just updates the existing grant.
- **Pending ≠ accepted** — a fresh invitation has `status: "pending"` and
  grants **no access at all** until the invitee explicitly accepts it
  (`resolveAccess` only honors `status: "accepted"` grants). This means a
  user can be invited to something without silently gaining read access
  before they've agreed to collaborate.
- **`POST /invitations/:id/accept` / `/decline`** — only the actual
  invitee (`grant.granteeUserId === callingUserId`) can act on their own
  invitation; anyone else gets `404`. A declined (or later revoked)
  invitation cannot be re-accepted — accept only succeeds while
  `status === "pending"`.
- **`PUT /grants/:id`** (change permission) and **`DELETE /grants/:id`**
  (revoke) both require `manage` on the underlying resource — *except*
  revoke also allows a grantee to revoke their own grant ("leave a share")
  even if they only had `view` access, since leaving something you were
  given access to shouldn't require elevated permission over it.
- **`GET /shared-with-me`** / **`GET /shared-by-me`** — the two directions
  of "what's shared," each scoped strictly to the calling user.

### Audit log (`services/sharing/audit.ts`, `AuditLogEntry` table)

Every sharing-relevant action is logged: `share.invite`, `share.accept`,
`share.decline`, `share.permission_change` (with from/to permission in
`metadata`), `share.revoke`, `share.leave`, and `access.denied` (written
whenever `requireAccess` throws a `403`, i.e. only for real-but-insufficient
grants, never for the 404 case — logging every failed probe against
resources a user has zero relationship to would itself leak information
through log-visibility). Logging is append-only and **never throws** into
the calling request — a logging failure degrades to "no audit row," never
"broken feature."

`GET /api/collaboration/audit-log` returns entries where the caller is the
actor, the target, or the owner of the referenced resource — never entries
belonging entirely to other users' private data.

## Phase 10: memory reliability & explainability

### Detection, reusing the Phase 7 suggestion engine

Five new detectors live in `services/memory/reliability.ts`:

- **`scanDuplicateMemories`** — an all-pairs embedding-similarity scan
  across a user's active memories at the existing
  `DUPLICATE_SIMILARITY_THRESHOLD` (0.93).
- **`scanConflictingMemories`** — reads `MemoryRelation(type: "conflicts")`
  rows already written at creation time (Phase 2), filtered to pairs where
  both sides are still active.
- **`findOutdatedMemories`** — active memories whose `validUntil` has
  passed.
- **`findLowConfidenceMemories`** — active memories below
  `LOW_CONFIDENCE_THRESHOLD` (0.5).
- **`findStaleTemporaryMemories`** — `type: "temporary"` memories still
  active more than `STALE_TEMPORARY_DAYS` (3) after creation — the type
  exists specifically to mark something as short-lived, so one still
  around past that window is worth a second look.

Rather than a parallel notification system, each detector is registered as
a new rule (`duplicate_memory`, `conflicting_memory`, `outdated_memory`,
`low_confidence_memory`, `stale_temporary_memory`) in the exact
`Suggestion`-table machinery Phase 7 built — same dedup keys, same 7-day
dismissal cooldown, same daily cap, same per-rule-type settings toggle
(`services/proactive/settings.ts`, `SettingsPage.tsx`). `POST
/api/memory-review/scan` runs just these five rules on demand (still
respecting the user's enabled-rule-type settings — an explicit scan
doesn't bypass a rule the user turned off).

### The Memory Review screen and its five actions

A new page (`/memory-review`, `pages/MemoryReviewPage.tsx`) lists pending
memory-reliability suggestions with the memory (or pair of memories, for
duplicate/conflict) shown inline, and up to five actions per suggestion —
`services/memory/review.ts` maps each to real, already-existing memory
operations:

- **KEEP** → dismisses the suggestion (same 7-day cooldown as any other
  dismissed suggestion) — the memory itself is untouched.
- **MERGE** → calls the existing `mergeMemories` (Phase 2): the duplicate
  is archived, not deleted, and its content is folded into the primary.
- **ARCHIVE** → `setStatus(..., "archived")` — reversible from the
  Memories page at any time.
- **FORGET** → `setStatus(..., "deleted")` — the same *soft* delete every
  other "forget" action in the app already uses; still recoverable, never
  a permanent purge. Permanent purge remains a separate, explicit,
  user-only action (`DELETE /memories/:id?permanent=true`) that no
  automated flow ever calls.
- **CORRECT** → calls the existing `updateMemory(..., "corrected", reason)`
  with new title/content, recording a `corrected` entry in `MemoryHistory`
  exactly like a manual correction would.

Every action also writes an `AuditLogEntry` (`memory_review.<action>`,
reusing the Phase 9 audit pattern) so what was auto-detected and what a
user chose to do about it stays traceable.

### Explainability: never a fabricated quote

`services/memory/explain.ts` (`GET /api/memories/:id/why`) and
`services/reminders/explain.ts` (`GET /api/reminders/:id/why`) answer "why
did you remember this?", "where did this come from?", "why did you create
this reminder?", and "why did you think this was important?" — entirely
from data already stored, with no AI call involved (deterministic, so it
can't hallucinate a source):

- **Source** — `sourceType`/`sourceId` resolve to the actual originating
  record (a chat message, a voice/image `Capture`, an `EmailRecord`, a
  `DriveFile`/`Document`) with a human label and, where one exists, a
  link back to it.
- **Original excerpt** — `sourceExcerpt`, captured verbatim at creation
  time (the actual chat text, transcript, or OCR text — already wired at
  every creation site since Phase 2/4; Phase 10 adds the same field to
  `Reminder` and populates it at all three reminder-creation call sites:
  chat commands, voice capture, image capture). **If no excerpt was
  captured, the narrative says so explicitly** ("No original wording was
  saved... so I can't quote it") instead of paraphrasing the stored
  summary as if it were a quote.
- **Reasoning** — a new `Memory.reasoning` field. The chat extraction
  prompt (`services/extraction`) now asks the model for one short sentence
  explaining *why* a fact was worth remembering and why it chose that
  importance level, captured verbatim as the model's own stated
  rationale — not invented after the fact. Manual memories have no
  reasoning, and the narrative says exactly that.

A deterministic chat command layer (`services/commands`) answers the same
questions conversationally — `"Why did you remember [fragment]?"`, `"Where
did [fragment] come from?"`, `"Why did you think [fragment] was
important?"`, `"Why did you create this reminder?"` (with or without a
fragment — falls back to the most recently created reminder) — by running
the fragment through the existing hybrid search to find the best match,
then calling the same explain functions. If nothing matches above a
minimum confidence, it says so plainly rather than guessing.

## Phase 11: PWA & mobile

### Installable, without touching the backend

`frontend/vite.config.ts` adds `vite-plugin-pwa` in `injectManifest` mode
— Workbox's precache manifest is injected into a hand-written
`frontend/src/sw.ts` rather than a fully generated service worker, so the
same file can also own push-notification handling (see below). The web
app manifest (`frontend/public/icons/`, generated via a one-off
`frontend/scripts/generate-icons.mjs` from an original SVG mark — not a
copy of Memorae's own branding, consistent with the Phase 8 rule)
declares `display: "standalone"`, `start_url: "/today"`, and both `any`
and `maskable` 512px icons. `index.html` adds the iOS-specific tags
Safari needs that the manifest alone doesn't cover:
`apple-mobile-web-app-capable`, `apple-touch-icon`, and
`viewport-fit=cover` (required for `env(safe-area-inset-*)` to work at
all around the iPhone 13 Pro Max's Dynamic Island/notch).

Install UX is split by platform (`context/PWAContext.tsx`,
`components/pwa/InstallBanner.tsx`) because **iOS Safari never fires
`beforeinstallprompt`**: on Android/Chrome, a captured install prompt
shows an "Install" banner; on iOS (detected via UA + touch points, since
iPadOS reports as Mac), a dismissible banner instead says "tap Share, then
Add to Home Screen" — the only way to install a PWA on iOS. Neither shows
once the app is already running standalone.

### Redesigned mobile nav: capture in one tap, from anywhere

The mobile bottom nav (`components/layout/MobileNav.tsx`) is five items —
Today, Tasks, a raised center **Capture** button, Search, Chat — replacing
the previous four-item bar. Capture opens a bottom sheet
(`components/capture/QuickCaptureSheet.tsx`, mounted once via
`context/CaptureContext.tsx` so it's reachable from the nav, the desktop
topbar, or anywhere else that calls `useCapture().openCapture()`) with
three one-tap options:

- **Voice** — `getUserMedia` + `MediaRecorder` (feature-detecting
  `audio/webm` vs. `audio/mp4` for Safari), uploaded through the existing
  `POST /api/capture/voice` once stopped.
- **Camera** — a native `<input type="file" accept="image/*"
  capture="environment">`, which opens iOS's real camera UI directly
  (deliberately not a custom `getUserMedia` viewfinder — more reliable
  across iOS Safari/PWA standalone quirks for one thing, one tap), through
  the existing `POST /api/capture/image`.
- **Note** — free text sent through the exact same `POST /api/chat` path
  as the Chat page, so it gets the same command parsing and extraction as
  typing in the full chat UI, just faster to reach.

All three were already fully built, working backend endpoints since Phase
3/4 — this phase only adds the one-tap mobile entry point; no capture
logic was duplicated or reimplemented.

### Offline caching and Web Push

`sw.ts` precaches the built app shell (JS/CSS/HTML/icons) and serves a
cached `index.html` for any uncached navigation (so opening the installed
app with no signal shows the SPA shell instead of a browser error), plus a
short-lived (5 minute) `NetworkFirst` cache for read-only `GET` requests
to memories/tasks/reminders/lists/projects/dashboard/search — enough that
reopening the app with a flaky connection shows last-known data. This is
explicitly *not* full offline read/write support: any mutation (POST/PUT/
DELETE — capture uploads, chat messages, task/reminder edits) requires a
live connection, same as before; Workbox only intercepts `GET` by default,
so nothing risks silently queuing/losing a write. Logging out
(`context/AuthContext.tsx`) clears all runtime caches, so a second account
on a shared device never sees the previous user's cached data.

Web Push is real, not a stub: `backend/services/push/` wraps the
`web-push` library with VAPID keys (`VAPID_PUBLIC_KEY`/`VAPID_PRIVATE_KEY`
env vars — the whole feature degrades to "no-op" if unset, same pattern as
every other optional integration in this app), a `PushSubscription` table
(one row per device/browser), and `POST /api/push/subscribe|unsubscribe|
test`. `backend/services/notifications/scheduler.ts` polls every 30
seconds for pending/snoozed reminders whose `remindAt` has arrived and
sends a real push notification, marking `Reminder.notifiedAt` so it never
double-fires; `notifiedAt` is cleared on snooze/reschedule/recurring
advance so the next occurrence notifies again on its own. This closes the
"no scheduler fires reminders" gap Phase 7 explicitly left open. On the
frontend, `hooks/usePushNotifications.ts` handles subscribing
(`PushManager.subscribe` with the VAPID key) and a Settings toggle lets a
user enable/disable it per device — including a note that iOS only
supports Web Push for a PWA actually launched from the Home Screen, not a
regular Safari tab, and detects that case to explain it rather than
silently failing.

### One account, one backend, everywhere

Nothing here introduces a second data path: the mobile capture sheet calls
the identical `/api/capture/*` and `/api/chat` endpoints the desktop app
uses; push subscriptions are rows scoped to the same `userId` as every
other table; offline caching is a read-through cache of the same API
responses, never a separate local store. Signing in on an iPhone and a
laptop is the same account, the same memories, the same tasks/documents/
Google connection — installing the PWA changes how you reach Memorae, not
what Memorae is.

## Cost audit: running Memorae for $0

Every external service the app can talk to, audited for whether it's
actually free, what a local/open-source alternative looks like, and
whether it runs on the two target machines this project is tuned for (a
Mac M2, 16GB RAM; a Windows 11 PC with an i5 9th-gen, GTX 1660 SUPER 6GB,
16GB RAM — both CPU-inference machines for this app's purposes, no CUDA
setup required by anything here).

| Feature | Current technology | Cost | Free/local alternative | Recommended choice |
|---|---|---|---|---|
| Chat replies, memory extraction, voice/image entity extraction, Gmail classify/summarize/extract/draft | OpenAI `gpt-4o-mini` via `services/ai/openaiProvider.ts` | Pay-per-token (no free tier; gpt-4o-mini is already OpenAI's cheapest capable model) | **Ollama** (`services/ai/ollamaProvider.ts`, new) — same `AIProvider` interface, runs `llama3.2:3b` (~2GB) fully offline | **Ollama for $0**, OpenAI opt-in when you want higher quality — both fully supported, pick via `AI_PROVIDER` |
| Daily briefing / weekly review narrative | Same `AIProvider`, wrapped in `narrate()` | Same as above, but **already has a built-in $0 fallback** | Deterministic plain-text narrative (already implemented, always used if AI fails/unset) | **Keep the deterministic fallback as the $0 default** — it's already complete and reliable, AI prose is a nice-to-have layer on top |
| Semantic search embeddings | `@xenova/transformers`, `Xenova/all-MiniLM-L6-v2`, local ONNX/CPU | $0 — one-time ~90MB model download, cached in `node_modules/@xenova/transformers/.cache` (not cwd-dependent, confirmed) | Already local | **Keep** — already optimal, no better $0 option exists |
| Speech-to-text | `@xenova/transformers`, `Xenova/whisper-base.en`, local ONNX/CPU | $0 — one-time ~145MB download | Already local (smaller `whisper-tiny.en` / larger `whisper-small.en` available via `STT_MODEL`) | **Keep** — `whisper-base.en` is the documented sweet spot for both target machines |
| OCR | `tesseract.js`, local | $0 — one-time ~5MB/language download | Already local; hardened this pass — `cachePath`/`langPath` now point at a checked-in `backend/tessdata/` directory instead of relying on process cwd, which previously risked a silent jsDelivr CDN fetch if launched from a different working directory | **Keep** — now deterministic and fully offline regardless of launch directory |
| Document parsing (PDF/DOCX) | `pdf-parse`, `mammoth`, local npm libraries | $0 | Already local | **Keep** |
| Database | SQLite via Prisma | $0 | Already local; Postgres is a config-only swap if you ever need multi-instance scale | **Keep SQLite** — no reason to pay for hosted Postgres at personal-use scale |
| Google Calendar / Gmail / Drive | Google Cloud OAuth APIs | $0 — generous free quota for a single-user personal app, no billing account required for these scopes | None (needed for these specific integrations) | **Keep, optional** — already gated behind `GOOGLE_CLIENT_ID`/`SECRET`; entire app works with it unset |
| Telegram bot | Telegram Bot API | $0 — no paid tier exists for bots at any volume | N/A | **Keep, optional** |
| Push notifications | Self-hosted VAPID keys (`web-push` npm package) + browser-vendor push services (FCM/APNs/Mozilla push, standard Web Push protocol) | $0 — no third-party push-as-a-service (no OneSignal/Pusher/etc. in dependencies) | N/A — already the free/local option | **Keep** |

### What changed this pass (safe, behavior-preserving)

- **Ollama provider added** (`services/ai/ollamaProvider.ts`) — a fully
  local, $0 `AIProvider` implementation, selected via `AI_PROVIDER=ollama`.
  Existing behavior is unchanged unless you opt in: `AI_PROVIDER` still
  defaults to `"openai"`, and every AI-assisted feature's existing
  graceful-degradation logic (try/catch → deterministic fallback, or
  "AI unavailable" for chat) applies identically to an unreachable Ollama
  server as it already did to a missing OpenAI key — no caller needed to
  change.
- **Response caching added** (`services/ai/cachingProvider.ts`) — every
  provider returned by `getAIProvider()` is now wrapped in an exact-match,
  30-minute, in-memory cache. The cache key is the full message array
  (system prompt + history + facts), so a hit only ever happens when the
  request is genuinely byte-identical to a recent one — if anything in the
  underlying data changed, the key changes too and the cache is bypassed
  automatically. This is always *correctness-preserving*, not just fast:
  it mainly eliminates redundant paid/compute calls from things like
  revisiting the Today page without anything having changed.
- **Gmail processing batched 3→1 AI call** (`emailIntelligence.ts`,
  `processEmail`): classify + summarize + extract now run as a single
  structured-JSON request instead of three separate ones, with an
  automatic fallback to the original three-call path if the combined
  response doesn't parse — same reliability guarantee, one-third the
  metered requests in the common case. Verified with a test asserting
  exactly one OpenAI HTTP call for `/api/gmail/messages/:id/process`.
- **OCR CDN dependency closed** — `runOcr()` now passes an explicit
  `cachePath`/`langPath` pointing at `backend/tessdata/` (the trained-data
  file that was previously an implicit, cwd-dependent cache is now a
  checked-in, explicitly-referenced asset), so OCR is guaranteed offline
  regardless of how/where the process is launched.
- 9 new backend tests (Ollama provider, caching correctness including TTL
  expiry via fake timers, provider selection) plus one updated test
  proving the Gmail batching — 240 automated backend tests total, all
  passing.

### Deliberately not changed

- **No downgrade to a smaller/cheaper OpenAI model for "simple" tasks** —
  `gpt-4o-mini` is already OpenAI's cheapest capable model; further
  downsizing (or routing different features to different quality tiers)
  would trade reliability for a marginal saving on top of a service that
  now has a real $0 alternative anyway. Per the "don't sacrifice
  reliability to avoid a small cost" brief, the answer here is "switch to
  Ollama," not "use a worse cloud model."
- **Google, Telegram, and Web Push integrations left exactly as they
  were** — all three were already free (generous free API quotas, no
  paid tier exists, and self-hosted VAPID keys, respectively); the audit
  found nothing to replace, only things to correctly leave alone.
- **No hosted/cloud deployment introduced** — everything above assumes
  running locally, which is what makes it $0 in the first place; nothing
  in this pass adds a VPS, managed database, or cloud function that would
  turn "free local model" into "paid hosting for the thing running the
  free local model."

## What's implemented (Phase 2, on top of Phase 1)

- Memory types, categories, tags, importance, confidence, status,
  source tracking, temporal validity fields, and history/audit trail
- Duplicate detection (blocks creation, returns candidates) and merge
- Conflict detection (flags contradictory same-topic memories via relations)
- Correction (distinct from update — records a reason)
- Archive / restore / soft-delete ("forget") / permanent purge
- Related memories (explicit relations + semantic similarity) and a
  chronological timeline view
- Automatic memory extraction from chat, with type classification and a
  confidence gate — not everything gets remembered
- Local embedding provider (MiniLM via transformers.js) — real semantic
  vectors, no external API, no fake similarity heuristics
- Document upload/indexing for TXT/MD/CSV/PDF/DOCX/images (OCR), with
  content-hash duplicate detection and chunk-level embeddings
- Hybrid (keyword + semantic) search across memories and documents, with
  metadata/date/source filtering and source-attributed results
- RAG-based chat retrieval (top-K only, never the whole store), with
  source citations in the response

## What's implemented (Phase 3, on top of Phase 1 + 2)

- Tasks: subtasks, projects, priorities, deadlines, statuses, recurrence,
  dependencies (cycle-checked), attachments, notes
- Reminders: one-time, recurring, snooze, reschedule, cancel, complete,
  priority
- Lists: arbitrary user-created lists and items, completion state,
  natural-language add/query
- Natural-language command parsing for the exact command shapes in the spec,
  deterministic and offline (no AI key needed for these)
- Reliable IANA timezone handling (DST-correct, tested against real zones)
- Confirmation-gated destructive actions via chat (cancel/delete), with a
  5-minute expiry and explicit yes/no handling
- Tasks/reminders/lists integrated into hybrid search and the dashboard
  summary, alongside memories and documents
- 96 automated backend tests total (up from 47), all passing — see below

## What's implemented (Phase 4, on top of Phase 1 + 2 + 3)

- Local speech-to-text (Whisper via transformers.js, CPU-only) accepting any
  ffmpeg-readable audio format
- Local OCR (tesseract.js, CPU-only) for images/screenshots
- Voice entity extraction: person, memory, amount, task, reminder — matching
  the exact spec example verbatim
- Image entity extraction: dates, deadlines, appointments, tasks, addresses,
  contacts, useful info
- Automatic creation of memories/tasks/reminders from both, with full source
  tracking back to the originating capture, and duplicate-detection reuse
  from Phase 2 so re-processing the same content doesn't double up
- A single `modelConfig` (`backend/src/config/models.ts`) making every local
  model swappable via `.env` with no code changes
- 118 automated backend tests total (up from 96), all passing, including
  real transcription and real OCR against committed audio/image fixtures —
  see below

## What's implemented (Phase 5, on top of Phase 1 + 2 + 3 + 4)

- Telegram text/voice/image/document handling, all routed through the exact
  same services as the REST API (`chatEngine`, `capture`, `documents`) — no
  Telegram-specific memory store
- Secure, single-use, time-limited account linking (`/link CODE` or a
  `/start` deep link)
- Duplicate-delivery protection (`update_id` claimed exactly once)
- Incoming rate limiting (per-chat) and outgoing send throttling
  (per-chat, matching Telegram's own limits)
- Errors degrade to a friendly reply, never a crash or silence
- Long-polling mode for local use (no public URL needed) and a webhook
  endpoint (secret-token verified) for production
- 133 automated backend tests total (up from 118), all passing — see below,
  including the full Telegram → backend → AI → memory workflow tested with
  a mocked Telegram + OpenAI HTTP boundary but a **real** database, real
  command parser, real Whisper transcription, and real OCR underneath

## What's implemented (Phase 6, on top of Phase 1 + 2 + 3 + 4 + 5)

- A provider-agnostic OAuth connector framework (CSRF state, encrypted
  token storage, transparent refresh, revoke/disconnect) with Google as the
  first implementation
- Google Calendar: read/create/update/delete events, conflict detection,
  timezone-aware scheduling
- Gmail: search, AI classify/summarize/extract (tasks/deadlines/events),
  connect emails to memories, **draft-only** replies (structurally
  incapable of sending — no send code path, and the OAuth scope granted
  doesn't include send permission)
- Google Drive: discover, index (reusing the exact Phase 2 document
  extraction/chunking/embedding pipeline), connect to memories, search
  (new `drive_file` source in the same central hybrid search)
- OAuth tokens encrypted at rest (AES-256-GCM); verified in tests that
  stored ciphertext never contains the plaintext token
- 156 automated backend tests total (up from 133), all passing — see below

## What's implemented (Phase 7, on top of Phase 1-6)

- Daily briefing and weekly review combining calendar, tasks, overdue
  tasks, reminders, important emails, projects, and relevant memories —
  structured data always complete, AI narrative optional
- Deterministic "what am I forgetting?" / "what's important today?" /
  "what should I prioritize?" — no AI call required, can't hallucinate
- Five proactive suggestion rules (missing reminder, deadline cluster,
  calendar conflict, unanswered important email, memory cluster) matching
  the spec's exact examples
- Real anti-spam: dedup via stable per-fact keys, 7-day dismissal cooldown,
  and a hard daily cap on new suggestions — all independently enforced
- Full configurability: master enable/disable, per-rule-type toggles, daily
  cap, and quiet hours (generation continues quietly; surfacing stops)
- Fixed a latent Phase-3 bug along the way: `Task`'s `dueToday` filter used
  the server process's local timezone instead of the user's
- 174 automated backend tests total (up from 156), all passing — see below

## What's implemented (Phase 8, on top of Phase 1-7)

- All 14 dashboard sections as real, functional, API-wired pages — Today,
  Calendar, Tasks, Reminders, Memories, Inbox, Documents, Lists, Projects,
  Email, Search, Shared Memory, Activity, Settings
- Global search (command palette, ⌘K) and a full search page, both backed
  by the real hybrid search endpoint
- The command/chat interface restyled and integrated into the shell
- An original design system with independently-tuned dark and light themes,
  persisted across sessions
- Fully responsive: desktop sidebar, mobile bottom-nav + slide-in drawer,
  verified at a 375×812 mobile viewport
- Two new backend features built specifically for this UI: revocable
  read-only memory sharing (`services/sharing/`) and a merged activity feed
  derived from existing tables (`services/activity/`) — no fake/mock data
  anywhere in the dashboard
- 5 new backend tests (sharing + activity) — 179 automated backend tests
  total, all passing — see below

## What's implemented (Phase 9, on top of Phase 1-8)

- A single `requireAccess()` authorization gate retrofitted onto every
  `:id`-addressed memory/list/task/project route — no resource is
  reachable by a non-owner without an accepted `ShareGrant`
- Three ranked permission levels (view/edit/manage) enforced per-endpoint,
  with deletion specifically gated at `manage`
- Invitations scoped to already-registered accounts (no dangling invites),
  pending-until-accepted semantics, decline, and self-revoke ("leave")
  distinct from owner/manager-initiated revoke
- Anti-enumeration access control: no relationship → `404`
  (indistinguishable from "doesn't exist"), insufficient permission → `403`
  (only once existence is already known to the caller)
- An append-only audit log (`AuditLogEntry`) covering invite/accept/
  decline/permission-change/revoke/leave/access-denied, visible only to a
  resource's owner, the acting user, and the target user — never to
  unrelated users
- Note: sharing is per-resource and does not cascade — sharing a project
  does not automatically share its tasks, and vice versa; each resource's
  access is independent by design
- 22 new backend tests (collaboration/sharing) — 201 automated backend
  tests total, all passing — see below

## What's implemented (Phase 10, on top of Phase 1-9)

- Five memory-reliability detectors (duplicate, conflicting, outdated,
  low-confidence, stale-temporary), registered as new rule types in the
  existing Phase 7 suggestion engine — same dedup/cooldown/daily-cap/
  quiet-hours machinery, no parallel notification system
- A Memory Review screen (`/memory-review`) with five actions per
  suggestion (KEEP/MERGE/ARCHIVE/FORGET/CORRECT), each mapped to a real
  existing memory operation — FORGET is always the same recoverable soft
  delete every other "forget" in the app uses, never an automatic
  permanent purge
- Full explainability: `GET /memories/:id/why` and `GET /reminders/:id/why`
  answer "why did you remember this / where did this come from / why did
  you think this was important / why did you create this reminder"
  entirely from stored data (source type, original excerpt, and — for
  AI-extracted memories — the model's own captured reasoning), plus the
  same questions answerable directly in chat via deterministic command
  parsing + hybrid search
- `Reminder.sourceExcerpt` added and populated at all three
  reminder-creation sites, so reminders have the same original-wording
  provenance memories already had
- The AI never claims the user said something without a source: when no
  excerpt was captured, the explanation says so explicitly rather than
  paraphrasing the stored summary as a quote
- 22 new backend tests (detection, review actions, explainability, chat
  explain commands) — 223 automated backend tests total, all passing —
  see below

## What's implemented (Phase 11, on top of Phase 1-10)

- Installable PWA: manifest + `injectManifest`-mode service worker
  (`vite-plugin-pwa`), original app icons (192/512/maskable/apple-touch),
  and the iOS-specific meta tags Safari needs that the manifest alone
  doesn't cover
- Platform-aware install UX: a real install prompt on Android/Chrome, an
  "Add to Home Screen" instructional banner on iOS (which never fires
  `beforeinstallprompt`), neither shown once already running standalone
- A redesigned 5-item mobile bottom nav with a raised center **Capture**
  button, open from anywhere via a shared `CaptureContext`
- A one-tap quick-capture bottom sheet — Voice (`MediaRecorder`), Camera
  (native camera picker), Note (free text) — wired to the exact same
  `POST /api/capture/voice`, `POST /api/capture/image`, and `POST
  /api/chat` endpoints the desktop app already uses; nothing duplicated
- Offline app-shell precaching + a 5-minute `NetworkFirst` cache for
  read-only API GETs, so a flaky/offline reopen shows last-known data;
  mutations are never cached or queued offline (a deliberate scope limit,
  not an oversight); logout clears all runtime caches
- Real Web Push: VAPID-based subscriptions (`PushSubscription` table),
  and a 30-second background scheduler that notifies exactly once per due
  reminder occurrence (`Reminder.notifiedAt`, cleared on
  snooze/reschedule/recurring-advance) — closes the "no scheduler fires
  reminders" gap left open since Phase 7
- Camera and microphone access via standard browser permission prompts
  (`getUserMedia`, `<input capture>`) — no new native permissions
  infrastructure needed
- One account, one backend: capture, chat, memories, tasks, documents, and
  the Google connection are identical between the installed iPhone app and
  desktop — nothing about this phase touches or duplicates backend state
- 8 new backend tests (push subscribe/unsubscribe/send, due-reminder
  scheduler) — 231 automated backend tests total, all passing — see below

## Running tests

```bash
cd backend
npm test
```

Covers (Phase 4, new): real speech-to-text against two distinct committed
WAV fixtures (correct model reported, correct words recognized); real OCR
against a committed PNG fixture with exact-match assertions on every
extracted line; voice entity extraction (person/memory/amount/task/reminder
parsing, empty-result and malformed-output handling, provider-failure
handling) and its date resolution against a real timezone; image entity
extraction (all seven categories, blank-input short-circuit that skips the
AI call entirely, malformed-output handling); the full capture REST API
(upload → transcribe/OCR → store, list/get, auth, missing-file rejection);
and the full capture *service* pipeline end-to-end with an injected fake AI
provider — verifying that a voice note produces a linked memory + reminder
with a resolved future date, that a task-only note produces only a task,
that an image produces tasks/reminders/memories with `sourceType: "image"`
and correct `sourceId` linkage, and that processing the same image twice
does not create a duplicate memory.

Covers (Phase 5, new): account linking (successful link, invalid code,
already-used code, an unlinked user getting instructions instead of any
data access); the exact spec examples end-to-end — "Add eggs to groceries"
creating a real list item, "Remind me tomorrow at 6" creating a real
reminder, "Find everything I said about Project X" retrieving a real
memory via search and answering through a mocked-but-realistic AI round
trip, "What's on my schedule?" falling through to the AI path correctly;
real voice transcription and real OCR triggered from a Telegram message
(through a mocked Telegram file-download boundary); a non-image document
upload indexed through the same document service the web app uses;
graceful error replies when a file download fails; the destructive-action
confirmation flow reachable from Telegram exactly like the web chat;
duplicate `update_id` delivery being processed only once; and per-chat
incoming rate limiting kicking in under a burst of messages.

Covers (Phase 6, new): the full OAuth round trip (authorize URL generation,
callback with a valid state, rejecting an unknown/reused state, rejecting a
failed code exchange) with a direct assertion that stored tokens are
encrypted (ciphertext doesn't contain the plaintext, decrypting it recovers
the original); transparent token refresh before an expired-token API call,
and marking the connection `error` when refresh itself fails; a 428 (not a
crash) when calling Calendar/Gmail/Drive with no connection; Calendar event
creation with correct timezone-aware `dateTime`, conflict detection
blocking creation and `force=true` bypassing it, update, and delete; Gmail
search, classify/summarize/extract, memory linking, and draft-reply with an
explicit assertion that **no request to any `/send` endpoint is ever
made** and that the OAuth scope list contains `gmail.compose` but not
`gmail.send`; Drive discovery, indexing a plain text file and a Google-native
document (via export) with real local chunking/embedding, marking an
unsupported file type as `unsupported` rather than failing, connecting a
Drive file to a memory, and finding it via central hybrid search; and
disconnect actually calling Google's revoke endpoint, removing local
access, and disconnecting twice returning 404 rather than crashing.

Covers (Phase 7, new): settings defaults and updates; disabling proactive
suppressing both generation and surfacing; a daily briefing whose sections
correctly contain seeded overdue tasks, due-today tasks, due-today
reminders, active projects, and high-importance memories (this is also
what caught and proved the fix for the `dueToday` timezone bug); a weekly
review's completed/unfinished/upcoming-deadline sections; all three ad-hoc
queries against real seeded data, including the "nothing pending" graceful
case; all five suggestion rules firing on realistic data (including
verifying that acting on a `missing_reminder` suggestion creates the real
reminder linked back to the source memory); the daily cap holding at
exactly `maxSuggestionsPerDay` even when multiple rules qualify
simultaneously; re-running generation not duplicating an already-pending
suggestion; the dismissal cooldown blocking regeneration and then allowing
it once the cooldown window has passed; quiet hours suppressing
`GET /suggestions` while leaving the underlying rows intact, plus a direct
unit test of the midnight-wrapping quiet-hours logic; and that dismiss/act
correctly 404 when called by a user who doesn't own the suggestion.

Covers (Phase 8, new): creating a memory share link and resolving it
publicly with no auth token at all; a missing token returning 404 and a
revoked one returning 410; listing a user's shares and rejecting an attempt
to share another user's memory; the activity feed correctly aggregating a
memory edit, a completed task, and a completed reminder into one
chronologically-sorted list; and that the activity endpoint requires
authentication.

Covers (Phase 9, new): strict isolation before any sharing exists — a
stranger gets `404` (never `403`) on every read/write endpoint across all
four resource types, and never sees another user's resources in their own
list endpoints; the full invitation lifecycle (inviting a non-existent
email fails, a pending invitation grants no access until accepted, only
the actual invitee can accept/decline their own invitation, a declined
invitation cannot later be accepted); permission-level enforcement at each
boundary (`view` can read but not edit/delete/manage-sharing; `edit` can
modify content but not delete or manage sharing; `manage` can do
everything including inviting others and deleting); an outsider with zero
relationship cannot change a grant's permission or revoke it (`404`); the
owner can revoke a collaborator's access outright, and a grantee can revoke
their own access ("leave") even without `manage`; representative
view/edit/manage flows repeated across shared lists, tasks, and projects
(not just memories); `shared-with-me`/`shared-by-me` listings; and the
audit log recording `share.invite`/`share.accept`/`access.denied` and
correctly hiding another user's private-resource events from a stranger's
own audit log view.

Covers (Phase 10, new): all five detectors firing on real seeded data
(near-identical forced duplicates, contradictory same-category/tag
memories, a `validUntil`-expired memory, a low-confidence memory, a
stale-still-active temporary memory); a disabled rule type producing zero
suggestions even when its underlying condition is met; all five review
actions (KEEP dismisses without touching the memory; MERGE archives the
duplicate and folds its content into the primary; ARCHIVE and FORGET
change status correctly, with FORGET verified recoverable via restore;
CORRECT updates content and records a `corrected` history entry) plus a
403/404 check that another user can't act on someone else's suggestion;
explainability returning the real captured excerpt for an
extraction-sourced memory and explicitly refusing to fabricate one when
none was saved, for both manual and sourced memories and reminders;
cross-user 404s on `/memories/:id/why` and `/reminders/:id/why`; and the
chat-level "why did you remember X" / "why did you create this reminder"
commands correctly finding and explaining the right record, defaulting to
the most recent reminder when no fragment is given, and responding
honestly ("couldn't find") rather than guessing when nothing matches.

Covers (Phase 11, new): VAPID configuration reporting correctly; a
subscription being stored and later removed by its own owner but not by
another user; re-subscribing the same `endpoint` updating keys in place
instead of duplicating; `sendNotification` calling `web-push` for every
subscription and pruning ones that come back expired (410); the due-
reminder scheduler notifying exactly once per due occurrence, never
double-firing on a second run, and correctly re-arming after a snooze
changes `remindAt`.

Covers (Phase 1+2, still passing): memory
create/retrieve/update/correct/archive/restore/delete/purge, duplicate
detection, conflict detection, merge, history, timeline, source tracking;
extraction classification/confidence-gating/malformed-output handling;
full-text/semantic/hybrid search with metadata/date/source filtering and
ranking order, including cross-user isolation; document upload for
txt/md/csv/pdf, real PDF text extraction, duplicate document detection,
large-document chunking, search-after-indexing with source attribution,
deletion, and re-indexing.

Covers (Phase 3, new): timezone conversion across DST/non-DST zones with a
round-trip check; recurrence phrase parsing and next-occurrence math
(including wall-clock time preservation across occurrences); tasks
(creation with priority/deadline/project, subtasks, cyclic-dependency
rejection + blocked-status computation, notes, attachments, status
transitions, recurring-task-spawns-next-instance, overdue filtering,
deletion); reminders (one-time + recurring creation, reschedule, snooze,
cancel, complete-closes-one-time-but-advances-recurring, overdue filtering,
deletion); lists (creation, add/complete/uncomplete/delete items, delete
list); the six example commands from the spec verbatim, end-to-end through
`POST /api/chat`, including timezone-correctness assertions on the produced
dates; the destructive-action confirmation flow (ask → decline → nothing
changes; ask → confirm → action executes) and that non-destructive actions
skip confirmation; dashboard and search integration for tasks/reminders/lists.

## Manually verified (this session, live server)

- The full Phase 11 mobile/PWA experience, live, in a real browser resized
  to an iPhone 13 Pro Max viewport (428×926): registered a new account and
  confirmed the redesigned 5-item bottom nav (Today/Tasks/raised
  Capture/Search/Chat) rendered correctly with proper safe-area spacing;
  tapped the center Capture button and confirmed the quick-capture bottom
  sheet opened with Voice/Camera/Note options; used the Note path, typed
  "Remind me tomorrow at 5pm to water the plants," and confirmed the exact
  same deterministic command parser used by the full Chat page created a
  real reminder — toast read `Reminder set: "water the plants" on Mon, Aug
  24, 5:00 PM.`, and the reminder was then visible on the Reminders page
  with the correct date; opened the hamburger drawer and confirmed all
  sections including the new Memory Review entry render correctly; checked
  Settings and confirmed the new Notifications card renders with an
  "Enable notifications" control (browser push permission prompts and
  service-worker registration itself require a real, non-sandboxed browser
  to complete — not exercisable inside this session's embedded preview
  tool, so that last step is verified by code/test coverage rather than a
  live click here).
- The production build's PWA output verified directly: `npm run build`
  produced a valid `manifest.webmanifest` (correct name, icons, `display:
  "standalone"`, `start_url: "/today"`) with exactly one `<link
  rel="manifest">` injected into `dist/index.html` alongside the
  hand-written iOS meta tags (no duplication), and a real precache
  manifest (14 entries) baked into `dist/sw.js`, confirming the
  `injectManifest` build pipeline works end-to-end.
- The full Phase 9 collaboration flow, live, via curl against the running
  backend: created a memory as one user, confirmed a second user got `404`
  both before any invitation existed and again after being invited but
  before accepting (proving pending grants confer zero access); accepted
  the invitation and confirmed `GET` returned `200` with `"access":"view"`;
  confirmed an edit attempt at view-only permission returned `403`, and
  that this exact request was recorded in the audit log as `access.denied`
  with `{"required":"edit","actual":"view"}`; upgraded the grant to
  `manage` and confirmed the collaborator could then delete the memory
  (`200`); and confirmed the audit log showed the full, correctly-ordered
  sequence — `share.invite` → `share.accept` →
  `access.denied` → `share.permission_change` (with `from`/`to` metadata).
- The full dashboard, live, in a real browser: registered/logged in, then
  navigated Today, Tasks, Memories, Search, Settings, Chat, Lists, and the
  public share view. Created a task via the Tasks page UI, completed it via
  the checkbox, and confirmed it moved between the Open/Done filters
  correctly. Opened a memory's detail modal, created a share link, and
  confirmed the resulting `/shared/:token` page rendered the memory
  read-only with no auth. Used ⌘K to run a real search-as-you-type query
  against the hybrid search API. Typed `"Add bread to my groceries list."`
  into the Chat page and confirmed the resulting list and item actually
  existed on the Lists page afterward — proving the chat command path,
  not just the display, is real. Toggled dark → light mode and confirmed
  every surface (cards, badges, inputs, nav) re-themed correctly. Resized
  to a 375×812 mobile viewport and confirmed the bottom nav, mobile top
  bar, and slide-in drawer (all 14 sections, user footer, theme toggle)
  render and function correctly.
- Proactive intelligence live, end to end with no AI key configured:
  seeded an overdue task, a due-today task, a today reminder, an active
  project, and a high-importance memory, then hit
  `GET /api/proactive/daily-briefing` and confirmed every section was
  correctly populated and the narrative correctly fell back to plain
  deterministic text; then asked `"What's important today?"` and
  `"What should I prioritize?"` via `POST /api/chat` and got correct,
  data-grounded answers with the overdue/urgent item ranked first
- Google connector REST endpoints live: `/connect` correctly returns 503
  when `GOOGLE_CLIENT_ID`/`SECRET` aren't configured (no client credentials
  exist in this environment — same documented limitation as Telegram/OpenAI
  in earlier phases), `/status` correctly reports not-connected, calling
  `/api/calendar/events` with no connection returns `428` rather than
  crashing, and `/disconnect` with no connection returns `404` — the full
  OAuth round trip, token refresh, and all three services are proven by the
  automated test suite's mocked-Google-API integration tests (23 tests,
  real database/encryption/embedding underneath, only the Google/OpenAI
  *HTTP calls* faked, same approach used for Telegram in Phase 5)
- Telegram REST endpoints live: generated a real link code via
  `POST /api/telegram/link-code`, confirmed `GET /api/telegram/status`
  correctly reports unlinked beforehand, and confirmed the webhook endpoint
  correctly rejects requests when no webhook secret is configured (503) —
  full update-processing itself is proven by the automated test suite's
  mocked-boundary integration tests (no real Telegram bot token exists in
  this environment to poll live Telegram servers with, same limitation as
  "no OpenAI key" in earlier phases — documented, not hidden)
- Voice capture live: a real synthesized recording of the spec's own
  example sentence, uploaded through `POST /api/capture/voice`, came back
  with `status: "completed"` and an accurate local transcript
  ("...15,000 and remind me next Friday...") and the correct
  `sttModel: "Xenova/whisper-base.en"`
- Image capture live: a real screenshot-style PNG (invoice date, meeting,
  email, address) uploaded through `POST /api/capture/image` came back with
  every line correctly OCR'd
- Confirmed both capture endpoints degrade honestly with no AI key
  configured — real transcript/OCR text every time, `created` arrays empty
  rather than fabricated, exactly as designed
- All six spec example commands, live, against a user in `Asia/Karachi`:
  correct next-day/next-Monday dates, correct default times, correct list
  creation-on-first-use, correct overdue-task and list-contents answers
- The confirmation flow live: "cancel task X" → asked to confirm → "no" →
  task still active → "cancel task X" again → "yes" → task actually
  cancelled — confirming destructive actions are never silent
- Dashboard summary reflecting chat-created tasks/reminders/lists in the
  same counts/lists a REST client would see
- Duplicate memory detection blocking creation, then `force=true` bypass
- Semantic search matching "plant-based eating habits" to a memory that only
  says "vegetarian... avoid meat" — zero shared keywords, found purely by
  embedding similarity
- Hybrid search returning a real uploaded PDF's extracted text, ranked by
  combined keyword+semantic score, with the correct filename as source
- OCR pipeline end-to-end on a real PNG upload (uploads → extracts →
  chunks → indexes without error)
- `npm run typecheck`, `npm run lint`, `npm run build`, and `npm test` all
  clean in `backend/`; `npm run build` clean in `frontend/`

## What's not built yet

- **No frontend UI for Phase 9 collaboration** — the sharing/invitation/
  permission/audit-log API is fully implemented and tested, but the
  dashboard's "Shared Memory" section still only surfaces the Phase 8
  read-only public share-link feature; inviting a collaborator, accepting/
  declining, changing permissions, and viewing the audit log all currently
  require calling `/api/collaboration/*` directly (e.g. via curl) rather
  than through the UI.
- **No cascading share of related resources** — sharing a project does not
  automatically share its tasks (or vice versa); each resource type's
  sharing is deliberately independent, not a design gap.
- **No frontend UI for Phase 10 Memory Review beyond the Memory Review
  page itself** — the review screen exists and is fully functional, but
  correcting a memory there uses a simple title/content form rather than
  the richer type/category/tag editor available on the Memories page.
- **Task/reminder editing beyond status changes** — the dashboard can
  create, complete/cancel, and delete tasks and reminders, but not yet edit
  an existing one's title/due date/priority in place, or manage subtasks/
  dependencies/notes/attachments visually (all of that exists in the API
  from Phase 3, just not surfaced in this UI pass).
- **Calendar editing beyond create/delete** — no in-UI event update (the
  API supports it); no month-grid view, agenda/list only.
- **No background scheduler** — `POST /api/proactive/suggestions/generate`
  must be called (e.g. on app open, or by a cron job you set up) for new
  suggestions to appear; nothing runs this automatically in the background.
  Briefings are generated on-demand when requested, not pre-computed on a
  schedule either.
- Real Telegram and Google connectivity have not been exercised against
  their live servers in this session — no Telegram bot token or Google
  OAuth client credentials exist in this environment. Both integrations'
  entire pipelines are tested against the real database and real local
  models with only the third-party *HTTP calls* mocked (the correct,
  standard way to test this without live credentials — see "Manually
  verified" above). Once you add real `TELEGRAM_BOT_TOKEN` /
  `GOOGLE_CLIENT_ID`+`GOOGLE_CLIENT_SECRET`, no code changes are needed.
- Real semantic search over *conversation history* (only memories, documents,
  tasks, reminders, lists, and Drive files are indexed/searchable — not raw
  chat logs)
- An AI-fallback layer for productivity commands that don't match the
  deterministic patterns (currently unmatched phrasing falls through to
  normal chat rather than being interpreted as a command)
- True image *understanding* beyond OCR (no vision-language model — see the
  Phase 4 model-choice rationale for why that's a deliberate hardware-driven
  scoping decision, not an oversight)
- Any connector beyond Google (the framework in `services/connectors/` is
  built to make this straightforward, but Microsoft/Outlook, Slack, etc.
  aren't implemented)
- Sending email, or any calendar-invite notification to attendees — this
  app never sends anything on your behalf via Gmail; you send drafts
  yourself from Gmail after reviewing them
- Multi-instance-safe rate limiting for Telegram (current limiters are
  in-process/in-memory — correct for a single backend process, would need a
  shared store like Redis if you ever ran multiple backend instances behind
  a load balancer)
- **Task due-date push notifications** — the Phase 11 scheduler notifies
  for due *reminders* only; tasks with a `dueAt` don't yet trigger a push
  (the same `PushSubscription`/`sendNotification` infrastructure would
  extend to tasks with a small addition to the scheduler, not a new
  system).
- **No offline write queue** — Phase 11's offline caching is deliberately
  read-only (a `NetworkFirst` cache of GETs plus the precached app shell);
  a capture, chat message, or edit made while genuinely offline fails
  rather than queuing to sync later, a scope limit called out explicitly
  above, not an oversight.
- **iOS Web Push requires the PWA to be installed and launched from the
  Home Screen** — this is an iOS/Safari platform restriction (Web Push
  doesn't work in a regular Safari tab, even on iOS 16.4+), not a
  Memorae limitation; Settings detects this case and explains it instead
  of silently failing.
- Background/async indexing (uploads, including voice/image captures, are
  processed synchronously in the request; fine at personal-use scale, would
  need a queue for large files, long recordings, or high concurrency)
- Cloud document/media storage or explicit opt-in cloud sync (not needed yet
  since there's no cloud path at all — everything is local by construction)
- Postgres deployment config, auth hardening (refresh tokens, rate limiting,
  password reset), pagination, frontend tests, CI/deployment
