# Environment variables

Every configuration point in the app, what it controls, whether it's
required, and what happens if it's left unset. The canonical source is
`backend/.env.example` — this document explains the *why* behind each one.
Frontend config is a single variable, listed at the bottom.

## Backend (`backend/.env`)

### Core (required)

| Variable | Default | Notes |
|---|---|---|
| `DATABASE_URL` | `file:./dev.db` | SQLite by default. Resolved **relative to `backend/prisma/`**, not `backend/` — see SETUP.md. Swap to a `postgresql://...` URL for staging/production; only this value and the Prisma `provider` in `schema.prisma` need to change (see DATABASE.md). |
| `PORT` | `4000` | Backend HTTP port. |
| `JWT_SECRET` | *(must be set)* | Signs auth tokens. Also the fallback source for connector-token encryption if `CONNECTOR_ENCRYPTION_KEY` is unset — use a long, random value. The app refuses to start without this. |
| `JWT_EXPIRES_IN` | `7d` | Auth token lifetime. There is no server-side revocation — see SECURITY.md. |
| `CORS_ORIGIN` | `http://localhost:5173` | Single allowed origin for the frontend. Not a wildcard. |

### AI provider (optional — see README's cost audit)

| Variable | Default | Notes |
|---|---|---|
| `AI_PROVIDER` | `openai` | `"openai"` or `"ollama"`. Powers chat Q&A, memory extraction, briefing narratives, and Gmail classify/summarize/draft. Everything else (memory CRUD, search, tasks/reminders/lists, local STT/OCR, capture) works with **no** AI provider configured at all. |
| `OPENAI_API_KEY` | *(empty)* | Required only if `AI_PROVIDER=openai`. Leaving it empty makes AI-dependent features degrade (deterministic fallback where one exists, e.g. briefings; otherwise the feature returns a clear "AI unavailable" error, e.g. open-ended chat Q&A). |
| `OPENAI_MODEL` | `gpt-4o-mini` | Already OpenAI's cheapest capable model. |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Only used when `AI_PROVIDER=ollama`. |
| `OLLAMA_MODEL` | `llama3.2:3b` | Must be pulled first (`ollama pull llama3.2:3b`). ~2GB, fits both target machines comfortably. |

### Local models (optional — sane defaults, always on)

| Variable | Default | Notes |
|---|---|---|
| `EMBEDDING_MODEL` | `Xenova/all-MiniLM-L6-v2` | 384-dim, ~90MB, local via `@xenova/transformers`. Powers semantic search. |
| `STT_MODEL` | `Xenova/whisper-base.en` | ~145MB, local, CPU-only. Options: `whisper-tiny.en` (~75MB, faster/less accurate) through `whisper-small.en` (~485MB, more accurate/slower) — see `backend/src/config/models.ts`. |
| `OCR_LANGUAGE` | `eng` | tesseract.js language code. The `eng` trained-data file ships in `backend/tessdata/`; other languages will fetch from jsDelivr on first use unless you add the file yourself. |

None of these three require an API key or ever leave the machine — they're
always on, regardless of whether AI/Google/Telegram/Push are configured.

### Telegram (optional)

| Variable | Default | Notes |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | *(empty)* | From [@BotFather](https://t.me/BotFather). Leave blank to disable — nothing else requires it. |
| `TELEGRAM_POLLING_ENABLED` | `false` | Set `true` for local dev — long-polls Telegram, no public URL needed. |
| `TELEGRAM_WEBHOOK_SECRET` | *(empty)* | Only needed for webhook mode (a public HTTPS URL) instead of polling. Any random string; verified against the `X-Telegram-Bot-Api-Secret-Token` header on every webhook request. |

### Google connector (optional)

| Variable | Default | Notes |
|---|---|---|
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | *(empty)* | From an OAuth 2.0 client at console.cloud.google.com. Leave both blank to disable Calendar/Gmail/Drive entirely. |
| `GOOGLE_REDIRECT_URI` | `http://localhost:4000/api/connectors/google/callback` | Must exactly match an "Authorized redirect URI" on the OAuth client. |
| `FRONTEND_URL` | `http://localhost:5173` | Where the OAuth callback redirects back to after linking. |
| `CONNECTOR_ENCRYPTION_KEY` | *(derived from `JWT_SECRET`)* | AES-256-GCM key for OAuth tokens at rest. The derivation (scrypt, not a raw hash) is cryptographically sound for local use, but a single leaked `JWT_SECRET` then compromises both auth tokens and connector credentials — **set this explicitly for any non-local deployment.** |

### Web Push (optional)

| Variable | Default | Notes |
|---|---|---|
| `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY` | *(empty)* | Generate with `npx web-push generate-vapid-keys`. Leave blank to disable — no subscriptions can be created, the reminder-notification scheduler no-ops, everything else works normally. |
| `VAPID_SUBJECT` | `mailto:admin@example.com` | Contact URI/email required by the Web Push protocol; shown to push services, not to end users. |

## Frontend (`frontend/.env`)

| Variable | Default | Notes |
|---|---|---|
| `VITE_API_URL` | `http://localhost:4000/api` | Base URL the frontend calls for every API request. Set to your deployed backend's URL in production. |

## What happens with zero configuration

Register an account and, with every optional variable left blank, you
still get: memory storage/search/sharing, document upload with local
extraction/OCR, task/reminder/list management, deterministic
natural-language commands ("remind me tomorrow at 7pm to call Ali"
works with no AI key), local voice transcription and image OCR via
capture, the full dashboard and mobile PWA, and installability. What you
don't get: open-ended AI chat Q&A, automatic memory extraction from
conversation, AI-narrated briefings (you get the same facts as plain
text instead), Google Calendar/Gmail/Drive, Telegram, and push
notifications. Nothing crashes or 500s because a key is missing —
each feature checks its own prerequisite and responds accordingly.
