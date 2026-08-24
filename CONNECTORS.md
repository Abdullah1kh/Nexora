# Connectors

Step-by-step setup for each optional external integration. All three are
independently optional — skip any you don't need.

## Google (Calendar, Gmail, Drive)

1. Go to [console.cloud.google.com](https://console.cloud.google.com),
   create a project (or use an existing one).
2. **APIs & Services → Library** — enable the Google Calendar API, Gmail
   API, and Google Drive API.
3. **APIs & Services → OAuth consent screen** — configure it (External is
   fine for personal use; add your own email as a test user if the app
   stays in "Testing" publish status, which avoids Google's app-review
   process entirely for personal use).
4. **APIs & Services → Credentials → Create Credentials → OAuth client
   ID** — Application type: **Web application**. Add an "Authorized
   redirect URI" matching `GOOGLE_REDIRECT_URI` exactly (default:
   `http://localhost:4000/api/connectors/google/callback`).
5. Copy the Client ID and Client Secret into `backend/.env`:
   `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`.
6. Restart the backend, then in the app go to **Settings → Google** and
   click Connect.

**Scopes requested** (least-privilege, deliberately): `openid`, `email`,
`calendar` (full read/write — needed for create/update/delete),
`gmail.readonly` (search/read), `gmail.compose` (draft creation
**only** — the app is structurally incapable of sending mail; this is
enforced by Google at the OAuth-grant level, not just in app code, since
`gmail.send` is never requested), `drive.readonly`.

**What each API powers:** Calendar — read/create/update/delete events,
conflict detection on create, timezone-aware scheduling, and the daily
briefing's calendar section. Gmail — search, AI classify/summarize/
extract-tasks (one batched AI call per message), link an email to a
memory, and draft (never send) replies. Drive — discover files, index
them through the exact same extraction/chunking/embedding pipeline as a
locally uploaded document, connect them to memories, and find them via
the same hybrid search.

**Token storage:** encrypted at rest (AES-256-GCM). See
`CONNECTOR_ENCRYPTION_KEY` in ENVIRONMENT.md and SECURITY.md for the key
management details — set it explicitly for anything beyond local dev.

**Disconnecting:** Settings → Google → Disconnect calls Google's own
token-revocation endpoint, not just a local flag flip.

## Telegram bot

1. Message [@BotFather](https://t.me/BotFather) on Telegram, send
   `/newbot`, follow the prompts. You'll get a bot token.
2. Set `TELEGRAM_BOT_TOKEN` in `backend/.env`.
3. For local development: set `TELEGRAM_POLLING_ENABLED=true` and
   restart. The backend long-polls Telegram for updates — no public URL
   needed.
4. For production behind a public HTTPS URL, use a webhook instead: set
   `TELEGRAM_WEBHOOK_SECRET` to any random string, then register the
   webhook (the app validates every incoming webhook request against
   this secret via the `X-Telegram-Bot-Api-Secret-Token` header, and
   rejects anything that doesn't match — including requests without the
   header at all).
5. In the app, go to **Settings → Telegram**, click "Generate code," and
   send `/link <code>` to your bot within 10 minutes (codes are
   single-use and expire).

Once linked, the bot handles text, voice notes, photos, and documents
through the **exact same** `chatEngine`/`capture`/`documents` services
the web app uses — there is no Telegram-specific memory store. Duplicate
Telegram updates (its "at-least-once" delivery guarantee) are
deduplicated by `update_id`; incoming messages are rate-limited per chat.

## Web Push (browser/PWA notifications)

1. `npx web-push generate-vapid-keys` (from `backend/`, or anywhere with
   the `web-push` package available).
2. Set `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY` in `backend/.env`.
   `VAPID_SUBJECT` should be a `mailto:` address or URL identifying you
   as the sender (shown to push services, not end users).
3. Restart the backend. In the app, **Settings → Notifications → Enable
   notifications** on each device you want to receive them on.

Delivery goes through the browser vendor's own push service (FCM for
Chrome, Apple's for Safari/iOS, Mozilla's for Firefox) via the standard
Web Push protocol — no third-party push-as-a-service is involved, and
it's free regardless of volume.

**iOS specifically:** Web Push only works for a PWA actually installed
and launched from the Home Screen, not a regular Safari tab (an iOS/
Safari platform restriction, not something this app can work around).
The Settings page detects this and explains it rather than silently
failing to subscribe.

A background job polls every 30 seconds for reminders that have come due
and sends a push notification, marking each reminder so it's never
notified twice for the same occurrence (see SECURITY.md and the
reliability notes in ARCHITECTURE.md for how the double-notify race is
prevented under concurrent poll ticks).

## Adding another connector later

`backend/src/services/connectors/` (OAuth state, token encryption,
refresh, revoke) is provider-agnostic by design —
`backend/src/services/connectors/google/` is the only implementation
today, but a second provider (Microsoft/Outlook, Slack, etc.) would reuse
the same framework rather than duplicating the OAuth plumbing. See the
comment in `backend/src/routes/connectors/index.ts`.
