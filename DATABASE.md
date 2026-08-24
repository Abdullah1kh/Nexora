# Database

SQLite via Prisma for local development and personal-scale deployment
(zero setup, one file); swappable to Postgres for staging/production by
changing the datasource `provider` and `DATABASE_URL` — the schema uses
only types supported by both, so no model changes are needed either way.

## Where the file actually lives

`DATABASE_URL="file:./dev.db"` in `backend/.env` resolves **relative to
`backend/prisma/`**, not `backend/` — Prisma resolves relative SQLite
paths relative to `schema.prisma`'s own directory for both the CLI and
the generated client. The real path is `backend/prisma/dev.db`. This
matters for any script that touches the file directly (see
`backend/scripts/backup.sh`, and BACKUPS.md generally).

## Schema overview (33 models)

Grouped by the feature area that owns them — full definitions are in
`backend/prisma/schema.prisma`, which is kept comment-annotated with the
valid values for every enum-like string field (SQLite has no native enum
type, so status/type fields are plain `String` with a comment above the
model documenting the closed set of values the application code enforces).

**Identity** — `User` (the root of nearly every relation; email/password
hash, timezone, cascades to everything else on delete).

**Memory engine** — `Memory` (type, category, importance, confidence,
status, source tracking, temporal validity, `reasoning` for
explainability), `MemoryRelation` (duplicate/related/conflicts/supersedes/
part_of links between memories), `MemoryHistory` (append-only snapshot on
every mutation), `ChatMessage` (raw conversation log, also RAG-indexed).

**Documents** — `Document` + `DocumentChunk` (chunked, embedded text for
hybrid search), `Capture` (voice/image uploads with transcript/OCR text
and extracted entities).

**Productivity** — `Project`, `Task` + `TaskDependency` + `TaskNote` +
`TaskAttachment` (subtasks via self-relation), `Reminder` (one-time or
recurring, `notifiedAt` tracks push delivery), `List` + `ListItem`,
`PendingConfirmation` (short-lived, for the "are you sure?" flow before
any destructive chat command executes).

**Telegram** — `TelegramAccount` (one-to-one link to `User`),
`TelegramLinkCode` (single-use, expiring), `TelegramProcessedUpdate`
(dedupes Telegram's at-least-once delivery guarantee).

**Google connector** — `Connection` (one row per user+provider; both
tokens encrypted at rest), `OAuthState` (short-lived CSRF token for the
authorize→callback round trip), `EmailRecord` + `EmailDraft` (Gmail
cache and drafts — never a sent-mail record, because the app never sends),
`DriveFile` + `DriveFileChunk` (mirrors the `Document`/`DocumentChunk`
shape for Drive-sourced content).

**Proactive intelligence** — `ProactiveSettings` (per-user config),
`Suggestion` (the anti-spam-controlled suggestion queue — also reused by
the memory-reliability review screen), `BriefingLog` (a JSON snapshot of
every generated daily/weekly briefing).

**Sharing & collaboration** — `MemoryShare` (anonymous, unauthenticated,
revocable read-only links — the public "Shared Memory" feature),
`ShareGrant` (real user-to-user permission grants with view/edit/manage
levels — distinct from `MemoryShare`), `AuditLogEntry` (append-only
security-relevant log of sharing actions and access denials).

**Push** — `PushSubscription` (one row per browser/device).

## Cascade behavior

Almost every child record cascades on its parent's deletion (a memory's
history/relations/shares, a task's notes/attachments/dependencies, a
document's chunks, and so on) — deleting a `User` cleans up everything
they own. Two intentional exceptions: `Task.projectId` uses `SetNull`
(deleting a project doesn't delete its tasks, just detaches them), and
`ShareGrant`/collaboration data references both an owner and a grantee
independently. One known gap: `AuditLogEntry.actor` cascades on the
actor's user deletion, which would erase their audit trail entries rather
than preserving them anonymized — currently unreachable in practice since
there is no account-deletion feature yet (see SECURITY.md's known
limitations).

## Indexes

Every model that's filtered by `userId` in the service layer has a
matching `@@index`. Two indexes exist purely to back cross-user
background/admin-style queries rather than a specific user's own
requests: `Reminder` has `@@index([status, remindAt, notifiedAt])` for
the notification scheduler's global due-reminder poll (not scoped to one
user), and `AuditLogEntry` has `@@index([targetUserId])` alongside
`actorId`/`resourceType+resourceId`/`createdAt` for the audit-log query
that checks whether the requesting user was the *target* of an action.

## Migrations

```bash
cd backend
npx prisma migrate dev --name <description>   # create + apply a migration locally
npx prisma migrate deploy                      # apply pending migrations, no prompts (CI/production)
npx prisma studio                               # browse the data in a local GUI
```

Migrations are checked into `backend/prisma/migrations/` — never edit an
already-applied migration file; create a new one instead.

## Backups

See BACKUPS.md. Short version: never `cp` the SQLite file while the
server is running — use `backend/scripts/backup.sh` (or `.ps1` on
Windows), which uses SQLite's own online `.backup` API and is safe to run
against a live database with no downtime.

## Scaling notes (personal-use app, not currently a concern)

Two service-layer patterns are O(n) or O(n²) over a single user's own
data with no pagination: `services/search/index.ts`'s hybrid search loads
a user's full memory/document-chunk/task/reminder/list-item/drive-chunk
set into memory per query, and `services/memory/reliability.ts`'s
duplicate scan is an all-pairs comparison over a user's active memories.
Both are fine at hundreds to low thousands of rows (a realistic personal
account) and would need query-level pagination/filtering if this app ever
served many users sharing one deployment — not a concern at the stated
single-user scale, and deliberately not "fixed" pre-emptively per this
project's own guidance against solving problems that don't exist yet.
