# API Reference

All endpoints are mounted under `/api`. Every route except `GET
/api/health`, `POST /api/auth/register`, `POST /api/auth/login`, `GET
/api/shared/:token` (the public share view), and `POST
/api/telegram/webhook` (authenticated by its own secret header instead)
requires `Authorization: Bearer <token>` from `/api/auth/login`.

Request bodies are validated with Zod; a failed validation returns `400`
with `{ error: "Validation failed", details: {...} }`. Unauthorized/
missing-relationship access returns `404` (not `403`) for shareable
resources — see SECURITY.md for why. Rate limiting: 20 requests/15min on
`/api/auth/login` and `/api/auth/register`, 300 requests/minute on
everything else under `/api`, both per IP.

## Auth (`/api/auth`)

| Method & path | Description |
|---|---|
| `POST /register` | Create an account, returns `{ token, user }`. Rate-limited. |
| `POST /login` | `{ token, user }`. Rate-limited. |
| `GET /me` | Current user. |
| `PATCH /me` | Update name/timezone. |

## Memories (`/api/memories`)

| Method & path | Description |
|---|---|
| `GET /` | List (filters: `q`, `status`, `type`, `category`, `tag`). |
| `GET /timeline` | Chronological view. |
| `POST /merge` | Merge duplicates into a primary. |
| `POST /` | Create (`?force=true` bypasses duplicate-detection block). |
| `GET /:id` | Read. Requires `view` access. |
| `GET /:id/history` | Full mutation history. |
| `GET /:id/related` | Explicit relations + semantic-similarity matches. |
| `GET /:id/duplicates` | Duplicate candidates for this memory. |
| `GET /:id/why` | Deterministic provenance/explainability. |
| `PUT /:id` | Update. Requires `edit`. |
| `POST /:id/correct` | Update with a recorded correction reason. |
| `POST /:id/archive` / `/:id/restore` | Soft status change. |
| `DELETE /:id` | Soft delete (`?permanent=true` for a real purge — requires `manage`). |

## Memory Review (`/api/memory-review`)

| Method & path | Description |
|---|---|
| `GET /suggestions` | Pending duplicate/conflict/outdated/low-confidence/stale-temporary suggestions, with memories resolved inline. |
| `POST /scan` | Run detection now (respects per-rule settings). |
| `POST /suggestions/:id/act` | Body `{ action: "keep"\|"merge"\|"archive"\|"forget"\|"correct", ... }`. |

## Chat (`/api/chat`)

| Method & path | Description |
|---|---|
| `GET /history` | Full conversation log. |
| `POST /` | `{ message }` → deterministic command, or RAG + AI reply; best-effort memory extraction. |

## Tasks (`/api/tasks`)

| Method & path | Description |
|---|---|
| `GET /` | List (filters: `status`, `projectId`, `overdue`, `dueToday`, `q`, `topLevelOnly`). |
| `POST /` | Create. |
| `GET /:id` | Read (`view`). |
| `PUT /:id` | Update (`edit`). |
| `POST /:id/status` | Change status. |
| `DELETE /:id` | Requires `manage`. |
| `POST /:id/dependencies` / `DELETE /:id/dependencies/:dependsOnId` | Cycle-checked prerequisites. |
| `POST /:id/notes` | Add a note. |
| `POST /:id/attachments` | Upload a file (multipart, 20MB limit). |

## Reminders (`/api/reminders`)

Owner-only — reminders aren't shareable.

| Method & path | Description |
|---|---|
| `GET /` | List (filters: `status`, `upcoming`, `overdue`, `q`). |
| `POST /` | Create (one-time or recurring via `recurrenceText`). |
| `GET /:id` | Read. |
| `GET /:id/why` | Deterministic provenance. |
| `PUT /:id` | Update. |
| `POST /:id/reschedule` / `/:id/snooze` / `/:id/complete` / `/:id/cancel` | Lifecycle actions. |
| `DELETE /:id` | Delete. |

## Lists (`/api/lists`)

| Method & path | Description |
|---|---|
| `GET /` / `POST /` | List / create. |
| `GET /:id` / `PUT /:id` / `DELETE /:id` | Read (`view`) / rename (`edit`) / delete (`manage`). |
| `POST /:id/items` | Add item. |
| `PUT /:id/items/:itemId` | Edit item text. |
| `POST /:id/items/:itemId/complete` | Toggle completion. |
| `DELETE /:id/items/:itemId` | Remove item. |

## Projects (`/api/projects`)

| Method & path | Description |
|---|---|
| `GET /` / `POST /` | List (filter `status`) / create. |
| `GET /:id` / `PUT /:id` | Read (`view`) / update (`edit`). |
| `POST /:id/archive` | Archive. |
| `DELETE /:id` | Requires `manage`. Does not cascade to the project's tasks. |

## Documents (`/api/documents`)

| Method & path | Description |
|---|---|
| `GET /` | List. |
| `POST /` | Upload (multipart, 20MB limit, extension allowlist: txt/md/csv/pdf/docx/png/jpg/webp/bmp). |
| `GET /:id` | Read (includes extracted text). |
| `POST /:id/reindex` | Re-run extraction/chunking. |
| `DELETE /:id` | Delete. |

## Capture (`/api/capture`)

| Method & path | Description |
|---|---|
| `GET /` | List captures (filter `kind`). |
| `GET /:id` | Read one. |
| `POST /voice` | Upload audio (multipart, 25MB) → local transcription → optional entity extraction → memory/task/reminder. |
| `POST /image` | Upload image (multipart, 25MB) → local OCR → optional entity extraction → memory/task/reminder. |

## Search (`/api/search`)

| Method & path | Description |
|---|---|
| `GET /` | Hybrid keyword+semantic search. Query params: `q`, `sources`, `dateFrom`/`dateTo`, `category`, `tag`, `memoryType`, `limit`. |

## Proactive intelligence (`/api/proactive`)

| Method & path | Description |
|---|---|
| `GET /settings` / `PUT /settings` | Per-user proactive-suggestion configuration. |
| `GET /daily-briefing` / `GET /weekly-review` | Structured data + narrative (AI or deterministic fallback). |
| `GET /suggestions` | Active (non-dismissed, non-quiet-hours) suggestions. |
| `POST /suggestions/generate` | Run all enabled rules now. |
| `POST /suggestions/:id/dismiss` / `/:id/act` | Resolve a suggestion; `act` on `missing_reminder` also creates the real reminder. |

## Collaboration / sharing (`/api/collaboration`, `/api/sharing`, `/api/shared`)

| Method & path | Description |
|---|---|
| `POST /collaboration/invite` | Invite a registered user to a memory/list/task/project (`view`/`edit`/`manage`). |
| `GET /collaboration/invitations` | Your pending invitations. |
| `POST /collaboration/invitations/:id/accept` / `/decline` | Respond. |
| `GET /collaboration/grants` | Grants on a resource (`manage` required). |
| `PUT /collaboration/grants/:id` | Change permission. |
| `DELETE /collaboration/grants/:id` | Revoke (owner/manager) or leave (grantee, self only). |
| `GET /collaboration/shared-with-me` / `/shared-by-me` | Both directions. |
| `GET /collaboration/audit-log` | Sharing-related audit trail visible to you. |
| `GET /sharing` / `POST /sharing/memories/:id` / `DELETE /sharing/:token` | Anonymous, unauthenticated public share links. |
| `GET /shared/:token` | **No auth** — public read-only view of a shared memory. |

## Activity (`/api/activity`)

| Method & path | Description |
|---|---|
| `GET /` | Merged feed of memory edits, completed tasks/reminders, captures, indexed documents, resolved suggestions. Query param `limit` (1-100). |

## Dashboard (`/api/dashboard`)

| Method & path | Description |
|---|---|
| `GET /summary` | Aggregate counts + upcoming items for the Today page. |

## Telegram (`/api/telegram`)

| Method & path | Description |
|---|---|
| `POST /link-code` | Generate a single-use, 10-minute linking code. |
| `GET /status` | Linked account status. |
| `POST /unlink` | Remove the link. |
| `POST /webhook` | **No bearer auth** — verified via `X-Telegram-Bot-Api-Secret-Token` instead. |

## Google connector (`/api/connectors/google`)

| Method & path | Description |
|---|---|
| `GET /connect` | Returns the Google OAuth authorize URL (`503` if not configured). |
| `GET /callback` | OAuth callback — validates CSRF state before touching the database. |
| `GET /status` | Connection status. |
| `POST /disconnect` | Revoke and remove. |

## Calendar (`/api/calendar`)

| Method & path | Description |
|---|---|
| `GET /events` | List (`timeMin`/`timeMax`). `428` if not connected. |
| `POST /events` | Create (`?force=true` bypasses conflict detection). |
| `GET /events/:id` / `PATCH /events/:id` / `DELETE /events/:id` | Read/update/delete. |

## Gmail (`/api/gmail`)

| Method & path | Description |
|---|---|
| `GET /search` | Search messages. |
| `GET /messages/:id` | Read one. |
| `POST /messages/:id/process` | Classify + summarize + extract tasks/deadlines/events (single batched AI call). |
| `POST /messages/:id/link-memory` | Connect to a memory. |
| `POST /messages/:id/draft-reply` | Composes reply text and creates a Gmail **draft** — structurally incapable of sending (scope is `gmail.compose`, not `gmail.send`). |
| `GET /records` | Locally cached/processed messages. |

## Drive (`/api/drive`)

| Method & path | Description |
|---|---|
| `GET /files` | Discover files (search `q`). |
| `GET /indexed` | Already-indexed files. |
| `POST /files/:id/index` | Extract, chunk, embed (reuses the document pipeline). |
| `POST /files/:id/link-memory` | Connect to a memory. |

## Push (`/api/push`)

| Method & path | Description |
|---|---|
| `GET /vapid-public-key` | `{ configured, publicKey }`. |
| `POST /subscribe` | Register a browser push subscription. |
| `POST /unsubscribe` | Remove one. |
| `POST /test` | Send yourself a test notification. |
