# Development

## Testing philosophy

**Mock only genuinely external third-party HTTP APIs** — OpenAI, Google,
Telegram, web-push delivery — via `vi.stubGlobal("fetch", ...)` or, where
`fetch` isn't the transport (the `web-push` npm package uses Node's
`https` module directly), `vi.spyOn(webpushModule, "sendNotification")`.
Everything else — the database, local embeddings, local speech-to-text,
local OCR — runs for real in every test. This is deliberate: it means the
test suite is also proof that the local-AI stack actually works on real
audio/image fixtures, not just that the code compiles.

Each test file owns its own SQLite database (`<name>-test.db`, created
via `prisma db push --skip-generate` in `beforeAll`, deleted in
`afterAll`) and imports `createApp()` dynamically after setting
`process.env.*`, so test files can run with different configuration
(different `AI_PROVIDER`, different VAPID keys, etc.) without leaking
into each other.

```bash
cd backend
npm test              # full suite (~40s — most of that is loading local ML models once)
npm run typecheck
npm run lint
npm run build          # tsc -> dist/ (excludes __tests__ — see tsconfig.json)
```

The frontend has `npm run lint` (oxlint) and `npm run build` (`tsc -b &&
vite build`, which is also the typecheck) but **no dedicated test suite**
— a known, current limitation (see the final report). If you add one,
match the backend's philosophy: prefer exercising real components over
extensive mocking.

## Project scripts

| Command | Where | What |
|---|---|---|
| `npm run dev` | `backend/` | `tsx watch src/index.ts` — auto-restart on change |
| `npm run dev` | `frontend/` | Vite dev server |
| `npm run build` | both | Production build |
| `npm start` | `backend/` | Run the built `dist/index.js` |
| `npm run preview` | `frontend/` | Serve the built PWA locally (needed to test service-worker/offline behavior — the dev server runs with `devOptions.enabled: false` for stability) |
| `./scripts/backup.sh` / `.ps1` | `backend/` | Safe SQLite backup — see BACKUPS.md |
| `node scripts/generate-icons.mjs` | `frontend/` | Regenerate PWA icons from `scripts/icon-master.svg` (requires `npm install --no-save sharp` first — not a persistent dependency) |

## Conventions

- **One memory system, one command parser, one chat engine.** Every
  channel (web, Telegram, mobile capture) calls the same
  `services/chatEngine/processChatMessage()`. If you're adding a new
  input channel, it should call this function, not reimplement any part
  of it.
- **Everything optional degrades, never crashes.** Any new AI-assisted,
  Google-, Telegram-, or push-dependent feature must check its own
  prerequisite and degrade (skip the feature, or fall back to
  deterministic behavior) rather than throwing an unhandled error when
  unconfigured.
- **Never fabricate a source.** If you're building anything that shows
  the user "why" something happened or "where it came from," it must be
  traceable to actual stored data (see `services/memory/explain.ts` for
  the pattern) — never a paraphrase presented as a quote.
- **`requireAccess()` for anything shareable.** Any new `:id`-addressed
  route on a resource type that could ever be shared must call
  `services/sharing/access.ts`'s `requireAccess()` before touching data,
  following the existing 404-for-no-relationship/403-for-insufficient-
  permission pattern.
- **No premature abstraction.** Three similar lines are better than a
  shared helper built for a hypothetical second use. Match the existing
  file's structure before introducing a new pattern.
- **Small, focused commits/changes.** This codebase has been built in
  well-scoped phases, each with its own tests and documentation updates
  landing together — follow that shape rather than large, mixed changes.

## Adding a new local model / swapping an existing one

Every local model (embedding, speech-to-text, OCR language) is
configured in `backend/src/config/models.ts` and overridable via env var
— see ENVIRONMENT.md. Swapping a model is a one-line `.env` change, not a
code change, as long as the new model exposes the same
`@xenova/transformers` pipeline interface (`feature-extraction` for
embeddings, `automatic-speech-recognition` for STT) or is a tesseract.js-
supported language code.

## Adding a new AI provider

`backend/src/services/ai/` defines a minimal `AIProvider` interface
(`{ name, generateReply(messages) }`). `OllamaProvider` is the reference
example for adding a provider beyond `OpenAIProvider` — implement the
interface, register it in the `providers` map in `services/ai/index.ts`,
and every existing AI-assisted feature (chat, extraction, briefings,
Gmail intelligence) picks it up automatically, already wrapped in the
shared response cache (`CachingAIProvider`).

## CI

There is no CI pipeline configured in this repo yet. Before considering
any change complete, run the full sequence yourself:

```bash
cd backend && npm run typecheck && npm run lint && npm test && npm run build
cd ../frontend && npm run lint && npm run build
```
