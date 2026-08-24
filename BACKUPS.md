# Backups

There was no backup mechanism of any kind before this audit. This
document explains what needs backing up, why a naive file copy is unsafe,
and how to actually do it, using the scripts added this pass.

## What to back up

Three things, none optional if you want a real restore:

1. **The database** — `backend/prisma/dev.db` (SQLite). Everything
   structured: every memory, task, reminder, list, document metadata,
   share grant, audit log entry, and setting.
2. **`backend/uploads/`** — every uploaded document, voice/image capture,
   and task attachment. `Document`, `Capture`, and `TaskAttachment` rows
   in the database store a `storagePath` pointing into this directory —
   the database without these files has metadata but no actual content
   for anything you uploaded.
3. **Nothing else is required.** `backend/tessdata/` is a static OCR
   language-data asset, not user data — it's redownloadable, not
   backup-worthy. `node_modules/@xenova/transformers/.cache/` (the local
   embedding/Whisper model weights) is similarly a redownloadable
   dependency, not data.

If you're running Postgres instead of SQLite (see DATABASE.md), back it
up with `pg_dump`/your usual Postgres tooling instead of the script
below, which is SQLite-specific.

## Why `cp dev.db dev.db.bak` is not safe

A plain file copy of a SQLite database while the server is running can
capture a torn, inconsistent snapshot: a copy that lands mid-write can
miss part of a transaction, and if the database is in WAL mode, committed
data can sit in a separate `-wal` sidecar file that a naive copy of just
the main `.db` file would silently omit entirely. SQLite's own online
backup API (the `.backup` command, or `VACUUM INTO`) is designed
specifically to be safe against a live, in-use database — it's the only
approach used here.

## How to back up

**Mac / Linux:**

```bash
cd backend
./scripts/backup.sh                    # writes to backend/backups/<timestamp>/
./scripts/backup.sh /path/to/elsewhere # or a specific destination
```

**Windows (PowerShell):**

```powershell
cd backend
powershell -ExecutionPolicy Bypass -File scripts\backup.ps1
powershell -ExecutionPolicy Bypass -File scripts\backup.ps1 -OutDir D:\Backups\memorae
```

Both scripts require the `sqlite3` CLI on `PATH` (see SETUP.md for
installing it) and read `DATABASE_URL` from `backend/.env` to find the
database, correctly resolving it relative to `backend/prisma/` (see
DATABASE.md for why that's not simply `backend/`). Each run produces a
timestamped directory containing `memorae.db` and a copy of `uploads/`.
The database backup uses `sqlite3 <db> ".backup '<dest>'"` — safe to run
while the server is running, no downtime needed.

## How to restore

1. Stop the backend (`Ctrl+C` / stop the process).
2. Replace `backend/prisma/dev.db` with the backed-up `memorae.db`.
3. Replace `backend/uploads/` with the backed-up `uploads/` directory.
4. Restart the backend (`npm run dev` or `npm start`).

There's no in-app "restore" flow — this is a manual, deliberate operation
by design, so a bad restore can't happen accidentally from within the UI.

## What's not automated (by design, not oversight)

- **No scheduled/cron backup job ships with the app.** Run the script
  yourself on whatever cadence matters to you — a daily cron job (Mac/
  Linux) or Task Scheduler entry (Windows) calling the script above is
  the recommended way to automate this; it's intentionally left as an
  operator choice rather than a background job baked into the Node
  process (a backup job inside the same process it's backing up adds
  failure modes — e.g. a stuck backup blocking the event loop — for a
  personal-use app where "run it before you do something risky, and
  weekly otherwise" is entirely sufficient).
- **No off-site/cloud backup.** Everything about this app's cost and
  privacy model (see README's cost audit) assumes local-only operation;
  copying the backup output to external storage (an external drive,
  another machine, a cloud drive you already use) is left to you and
  your own risk tolerance, not built into the app.
- **No automatic backup-before-migration.** Running `npx prisma migrate
  dev`/`deploy` does not take a backup first. Take one yourself before
  any schema migration you're not confident about, especially before
  switching to Postgres or making a destructive schema change.
