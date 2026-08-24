# Security

This document is the result of a full production security audit
(authentication, authorization, memory isolation, secrets handling,
OAuth, document/email access, Telegram auth, input validation, rate
limiting, and error handling), what was found, what was fixed, and what
remains a known, accepted limitation for a personal-use application.

## Authentication

- Passwords hashed with bcrypt (cost factor 10), minimum 8 characters
  enforced server-side.
- JWTs signed with `JWT_SECRET` (required — the app refuses to start
  without it), 7-day default expiry, minimal payload (`{ userId }` only,
  no PII).
- **No rate limiting or lockout on login/register** was found and has
  been **fixed** — see "Rate limiting" below.
- **No password-reset flow exists.** If a user forgets their password,
  there is currently no recovery path. This is a real, known gap — not
  implemented in this pass because building one (email delivery, reset
  tokens, expiry) is a genuine new feature, not an audit fix. Documented
  in the final report's "known limitations."
- **No server-side token revocation/logout.** A token is valid for its
  full 7-day life regardless of an explicit "log out" or a password
  change. Acceptable for a personal-use app; would need a token-blocklist
  or short-lived-token+refresh-token scheme to close.
- The frontend stores the token in `localStorage` (not an `httpOnly`
  cookie) — trades XSS exposure for CSRF-immunity. A reasonable choice
  for a Bearer-token API with no cookie-based session, but worth knowing
  that any successfully injected script has full account access for up
  to 7 days.

## Authorization & memory isolation

Every `:id`-addressed route for memories, lists, tasks, and projects
calls `requireAccess(userId, resourceType, id, minPermission)` —
`backend/src/services/sharing/access.ts` — **before** touching any data.
This includes easy-to-miss nested endpoints (task dependencies/notes/
attachments, list items). Reminders (not shareable) scope every operation
through an owner-checked lookup instead. Verified directly against the
route source, not assumed from prior documentation: no gap found.

**Anti-enumeration by design:** a user with *no* relationship to a
resource gets `404` (indistinguishable from "doesn't exist"); a user with
a real but insufficient grant gets `403` (their knowledge that the
resource exists is already established, so there's nothing left to hide).
Every `403` is also written to the audit log; `404`s are not, since
logging every failed probe against resources a stranger has zero
relationship to would itself leak information through log visibility.

Permission levels are ranked `view < edit < manage`. Deletion specifically
requires `manage`, not just `edit` — an editor can't unilaterally destroy
something they were only given content-editing rights over.

## Secrets handling

- No API key, token, or secret is ever logged (verified via repo-wide
  grep near log statements).
- The central error handler (`middleware/errorHandler.ts`) is a strict
  allowlist of known error types; anything else collapses to a generic
  `500 Internal server error` with full detail logged server-side only —
  raw Prisma errors, stack traces, and internal file paths never reach a
  client response.
- `.gitignore` (root, `backend/`, `frontend/`) excludes `.env`, `*.db`,
  `uploads/`, and `backups/`. No secret or personal-data file is
  committed. Verified no hardcoded API keys/private keys anywhere in
  source via pattern search (OpenAI `sk-`, Google `AIza`, PEM private-key
  headers, GitHub tokens).

## OAuth (Google connector)

- Tokens encrypted at rest with AES-256-GCM (random IV per encryption,
  authenticated). If `CONNECTOR_ENCRYPTION_KEY` is unset, the key is
  derived from `JWT_SECRET` via `scrypt` (a real KDF, not a shortcut) —
  the operational risk is that a single leaked `JWT_SECRET` then
  compromises both auth tokens and connector credentials, which is why
  the code and ENVIRONMENT.md both instruct setting a dedicated key for
  any non-local deployment.
- CSRF `state` is single-use, deleted on first use, and expires after 10
  minutes. The OAuth callback validates it **before** any database write.
- A failed token refresh marks the connection `status: "error"` with the
  upstream error message stored — no infinite retry loop, no token
  leakage. The stored error text is returned only to the connection's own
  owner (via `GET /connectors/google/status`), never cross-user.

## Document / email access

Every document, capture, and Gmail/Drive route scopes by the requesting
user — no cross-user access possible via ID guessing. File uploads have
size limits (20MB documents/attachments, 25MB captures) and, as of this
audit, an **extension allowlist on document uploads** (txt/md/csv/pdf/
docx/png/jpg/webp/bmp — exactly what the extraction pipeline actually
supports), added because arbitrary files could previously be persisted
to disk even though they'd never be processed. Uploaded files are never
served statically by Express (no `express.static` mount over `uploads/`),
so there's no direct file-execution or stored-content-serving risk from
an unexpected file type, but rejecting them up front is still the
correct default. Path traversal is not currently exploitable: stored
filenames are always `${randomUUID()}-${originalName}` inside a fixed
per-user directory — the original name is never used to determine the
directory.

## Telegram authentication

Link codes are single-use (checked-and-consumed atomically) and expire
after 10 minutes; creating a new code deletes prior unused ones. The
webhook endpoint rejects any request whose
`X-Telegram-Bot-Api-Secret-Token` header doesn't match
`TELEGRAM_WEBHOOK_SECRET` (503 if unconfigured, 401 on mismatch). Every
data-touching action in the message handler requires a resolved,
linked `userId` first — an unlinked Telegram user can only trigger the
linking flow or a "not linked" reply.

## Input validation

Nearly every POST/PUT route validates its body with Zod. One gap found
and **fixed**: `GET /api/activity` accepted an unvalidated, unbounded
`limit` query param (a non-numeric value would propagate as `NaN` into a
Prisma query); it now uses a Zod schema (`z.coerce.number().int().min(1)
.max(100).optional()`).

## Rate limiting — fixed this pass

Previously, **no rate limiting existed anywhere** in the Express stack —
the only throttling in the codebase was Telegram-specific and per-chat.
This meant `/api/auth/login` was fully brute-forceable and every other
endpoint (including AI-cost-incurring ones) was hammerable without limit.
Fixed with `express-rate-limit` (`backend/src/middleware/rateLimit.ts`):

- `authRateLimiter` — 20 requests/15 minutes per IP on
  `POST /api/auth/login` and `POST /api/auth/register`.
- `apiRateLimiter` — 300 requests/minute per IP, applied globally to
  `/api/*`, as a backstop against runaway abuse rather than a constraint
  on normal use.

Both are in-memory and per-process — correct for this app's documented
single-process deployment model (the same tradeoff already made for the
Telegram rate limiter); a multi-instance deployment would need a shared
store (Redis) instead. Both are automatically skipped when
`NODE_ENV=test` (set automatically by Vitest) so the test suite's
legitimate high-volume account creation is never throttled.

## Reminder scheduler reliability — fixed this pass

Two real bugs found in `services/notifications/scheduler.ts` and
`services/reminders/index.ts`, both fixed and covered by new tests:

1. **Double-notify race**: the 30-second poll had no re-entrancy guard,
   so a slow tick overlapping the next one could send the same push
   notification twice. Fixed with an in-process `isRunning` flag plus an
   atomic per-row claim (`updateMany` with a `notifiedAt: null` guard in
   the `WHERE` clause) before sending — verified with a test that fires
   two overlapping `runDueReminderCheck()` calls and asserts exactly one
   send.
2. **Recurring reminder catch-up bug**: completing a recurring reminder
   advanced it by exactly one interval from its own (possibly long-past)
   `remindAt`, which could still land in the past if the server had been
   down for a while — requiring the user to complete the same reminder
   repeatedly just to catch up. Fixed by advancing from `max(remindAt,
   now)` instead, so one "complete" always lands on a genuinely future
   occurrence.

## AI hallucination safeguards

`services/memory/explain.ts` never fabricates a quote — every branch
either shows the actual `sourceExcerpt` captured verbatim at creation
time, or explicitly states that no original wording was saved. One
inconsistency found and **fixed**: the `email` source type fell back to
Gmail's own auto-generated preview snippet when no excerpt was captured,
and presented it as "the relevant original text" — real text, but Gmail's
algorithmic truncation, not provably the exact sentence a memory was
extracted from. The response now carries an `excerptIsVerbatim` flag and
the narrative wording differs accordingly ("An auto-generated preview...
not necessarily the exact original wording" vs. "The relevant original
text"). The chat RAG system prompt explicitly instructs the model to cite
sources and say so plainly rather than guess when context doesn't answer
a question — a real, present instruction, not just a documented
intention, though it remains a soft instruction with no structural
enforcement against the underlying model ignoring it.

## Known vulnerabilities (dependency-level, monitored, not exploitable here)

`npm audit` reports a critical/high advisory chain in `protobufjs` and
`sharp`, both deep transitive dependencies of `@xenova/transformers`
(local embeddings/speech-to-text). A fix is only available via
`npm audit fix --force`, which would downgrade `@xenova/transformers` to
a much older major version — a breaking change to the app's core local-AI
engine, not applied. Risk assessment: `protobufjs` is only used
internally by `onnxruntime-web` to load bundled, developer-controlled
ONNX model files (never attacker-supplied protobuf input), and `sharp`
isn't invoked by any code path this app actually exercises. Monitored,
not currently exploitable, intentionally not force-downgraded per "don't
break existing functionality."

## CORS & error handling

`CORS_ORIGIN` is a single configurable origin (never a wildcard), no
route bypasses it. The error handler always returns a generic message on
any unrecognized error path (not just in a production-only branch),
which is stricter than the common dev/production split.

## Reporting a concern

This is a personal-use, self-hosted application with no bug bounty
program. If you find something here, the fix is: don't expose the
backend directly to the public internet without a reverse proxy/TLS
termination and the rate limits above, keep `JWT_SECRET`/
`CONNECTOR_ENCRYPTION_KEY` out of version control (already enforced by
`.gitignore`), and treat the SQLite file and `uploads/` directory as
containing your full personal data — see BACKUPS.md for how to protect
them.
