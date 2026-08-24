# Setup

Local development setup for both target machines this project is tuned
for — a Mac (M2, 16GB) and a Windows 11 PC (i5 9th gen, GTX 1660 SUPER
6GB, 16GB RAM). Both are CPU-inference machines for everything this app
runs locally; no CUDA/GPU setup is required anywhere.

## Prerequisites

- **Node.js 20+** (tested on 22). Mac: `brew install node`. Windows:
  install from [nodejs.org](https://nodejs.org) or `winget install
  OpenJS.NodeJS.LTS`.
- **npm** (ships with Node).
- **ffmpeg** — not required as a system dependency; the backend bundles
  `ffmpeg-static`, a prebuilt binary, so there's nothing to install here.
- Optional, only if you want the features they enable:
  - **Ollama** ([ollama.com](https://ollama.com)) for local, $0 AI chat/
    extraction instead of OpenAI. After installing: `ollama pull
    llama3.2:3b`.
  - **sqlite3 CLI** for the backup script (`brew install sqlite3` on Mac;
    Windows: the [sqlite-tools zip](https://sqlite.org/download.html) or
    `winget install SQLite.SQLite`). Not needed to run the app, only to
    back it up — see BACKUPS.md.

## First-time setup

```bash
git clone <this repo>
cd "Memorae Copy"

cd backend
npm install
cp .env.example .env
npx prisma migrate dev
```

`npx prisma migrate dev` creates `backend/prisma/dev.db` (SQLite, zero
config) and applies every migration. Prisma resolves the `DATABASE_URL`
in `.env` relative to `backend/prisma/`, not `backend/` — so
`DATABASE_URL="file:./dev.db"` really means `backend/prisma/dev.db`. This
matters if you ever write a script that touches the database file
directly (see `backend/scripts/backup.sh` for the correct resolution).

Edit `backend/.env` for anything you want beyond the local-only defaults
— see ENVIRONMENT.md for every variable and what it unlocks. The app
works with zero edits: no AI provider, no Google, no Telegram, no push
notifications, but every local feature (memory storage, search, tasks,
reminders, lists, document upload, OCR, speech-to-text, deterministic
chat commands) works immediately.

```bash
npm run dev
```

Backend now listening on `http://localhost:4000`.

In a second terminal:

```bash
cd frontend
npm install
echo 'VITE_API_URL=http://localhost:4000/api' > .env
npm run dev
```

Frontend now at `http://localhost:5173`. Open it, register an account,
and you're running.

## Running tests

```bash
cd backend
npm test          # full suite — real DB, real local embeddings/STT/OCR, ~40s
npm run typecheck
npm run lint
```

The frontend has `npm run lint` and `npm run build` (which includes a
`tsc -b` typecheck) but no dedicated test suite yet — see DEVELOPMENT.md
for the testing philosophy and what that means in practice.

## Production build

```bash
cd backend
npm run build      # tsc -> dist/
npm start          # node dist/index.js

cd frontend
npm run build      # tsc -b && vite build -> dist/
npm run preview    # serve the built PWA locally to sanity-check it
```

For a real deployment, serve `frontend/dist/` from any static host (or
the backend's own `express.static`, not currently wired up) and point
`VITE_API_URL` at wherever `backend/dist` (via `node dist/index.js`) is
running. Set `CORS_ORIGIN` on the backend to the frontend's real origin.

## Enabling optional features

Each of these is independently optional — the app runs fully without any
of them, only the specific feature they unlock is unavailable until
configured. Full instructions for each are in CONNECTORS.md and
ENVIRONMENT.md:

- **AI (chat Q&A, extraction, briefings, Gmail intelligence)** — set
  `OPENAI_API_KEY` (paid), or install Ollama and set
  `AI_PROVIDER=ollama` (free, local, recommended for $0 operation).
- **Google (Calendar, Gmail, Drive)** — create an OAuth client at
  [console.cloud.google.com](https://console.cloud.google.com), set
  `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET`.
- **Telegram bot** — create a bot via [@BotFather](https://t.me/BotFather),
  set `TELEGRAM_BOT_TOKEN` and `TELEGRAM_POLLING_ENABLED=true` for local
  dev (no public URL needed).
- **Push notifications** — generate VAPID keys with `npx web-push
  generate-vapid-keys` and set `VAPID_PUBLIC_KEY`/`VAPID_PRIVATE_KEY`.

## Platform-specific notes

**Mac (M2, 16GB):** everything above works out of the box. The first
chat message, voice capture, or document upload will trigger a one-time
download of the local embedding/Whisper models (~235MB total, cached in
`backend/node_modules/@xenova/transformers/.cache/`) — expect a short
pause the very first time each is used, not on every request after that.

**Windows 11 PC (i5 9th gen, GTX 1660 SUPER 6GB):** identical setup.
Nothing in this app requires CUDA or GPU drivers — local AI models run on
CPU via ONNX Runtime, and OCR runs via a small native binary — so the
GPU is not a dependency for anything here (Ollama, if you use it, can
optionally use the GPU for faster local chat inference, but CPU-only
Ollama also works fine for the recommended `llama3.2:3b` model size at
this RAM budget).
