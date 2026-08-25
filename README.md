<div align="center">

<img src="https://github.com/user-attachments/assets/708e2230-aa16-4f3a-acc6-6e08c0a2e111" alt="Nexora" width="100%">

<br>

### Your personal AI memory layer.

Capture information once. Find it later. Let it help you act.

<br>

![Status](https://img.shields.io/badge/status-in%20development-8B5CF6?style=for-the-badge)
![Architecture](https://img.shields.io/badge/architecture-local--first-06B6D4?style=for-the-badge)
![Platform](https://img.shields.io/badge/platform-web%20%7C%20PWA-111827?style=for-the-badge)
![AI](https://img.shields.io/badge/AI-pluggable-10B981?style=for-the-badge)

</div>

---

## Overview

Nexora is a personal AI memory and productivity system built around a simple idea:

> **You shouldn't have to remember where you put something.**

It brings memories, documents, tasks, reminders, lists, search, voice capture, images, email, calendar data, and messaging into one connected system.

The emphasis is on **useful memory rather than just chat**. Information can be captured from different places, organized into structured records, linked to its source, and retrieved later through natural language.

Nexora is designed to be **local-first, self-hostable, and inexpensive to run**. Cloud services are optional where local alternatives are practical.

---

## At a glance

<table>
<tr>
<td width="50%">

### 🧠 Memory

Long-term memories with:

- Source tracking
- Importance & confidence
- Temporal validity
- Related memories
- Duplicate detection
- Conflict detection
- History & recovery
- Explainable provenance

</td>
<td width="50%">

### 🔎 Search

One search layer across:

- Memories
- Documents
- Tasks
- Reminders
- Lists
- Google Drive files

Uses hybrid keyword + semantic retrieval.

</td>
</tr>

<tr>
<td>

### ⚡ Productivity

Natural-language control for:

- Tasks
- Subtasks
- Projects
- Reminders
- Recurring items
- Lists
- Dependencies

Destructive actions require confirmation.

</td>
<td>

### 🎙️ Multimodal capture

Capture information through:

- Text
- Voice
- Images
- Screenshots
- Documents

Speech recognition and OCR can run locally.

</td>
</tr>

<tr>
<td>

### 🔗 Integrations

Current connector work includes:

- Telegram
- Google Calendar
- Gmail
- Google Drive

Connectors share the same memory and search layer.

</td>
<td>

### 📱 Mobile

An installable PWA with:

- Quick capture
- Voice capture
- Camera capture
- Mobile search
- Push notifications
- Offline app-shell caching

</td>
</tr>
</table>

---

## Architecture

<img src="https://github.com/user-attachments/assets/d8a5920b-019b-493d-8ec9-a4761b85158e" alt="Nexora architecture" width="100%">

The application is split into a frontend, API/backend, memory and search services, AI providers, and external connectors.

The important architectural decision is that the AI provider is **replaceable**. Nexora can use a cloud model when needed or a local provider such as Ollama without changing the rest of the application.

---

## Memory is the core

<img src="https://github.com/user-attachments/assets/af8b0b80-6727-4fd2-8bcd-75b32286ee5f" alt="Nexora memory system" width="100%">

Nexora does not treat every conversation message as permanent memory.

When information is worth keeping, the system can extract a structured memory with its source, confidence, importance, and temporal context. Memories can then be corrected, merged, archived, restored, or forgotten.

Every sourced memory can be traced back to the information that created it.

This also powers the RAG layer: relevant memories and document chunks are retrieved first, and only that context is passed to the AI.

---

## Local-first AI

<img src="https://github.com/user-attachments/assets/74a9119c-9dc5-42fc-b3d8-c2c691ef1f9a" alt="Nexora local-first architecture" width="100%">

The project aims for a **$0 recurring cost for personal use** where realistically possible.

Local components currently include:

| Capability | Technology |
|---|---|
| Embeddings | `Xenova/all-MiniLM-L6-v2` |
| Speech-to-text | `Xenova/whisper-base.en` |
| OCR | `tesseract.js` |
| Document parsing | `pdf-parse`, `mammoth` |
| Database | SQLite + Prisma |
| Local AI option | Ollama |
| Cloud AI option | Pluggable `AIProvider` |

The system does not require a cloud AI provider for embeddings, OCR, or speech transcription.

Cloud AI remains optional for tasks where a stronger language model is useful.

---

## Search & RAG

Nexora uses a hybrid retrieval pipeline rather than sending an entire personal database to an AI model.

```text
Query
  │
  ▼
┌──────────────────────┐
│ Hybrid Search        │
│                      │
│ Keyword + Semantic   │
└──────────┬───────────┘
           │
           ▼
   Ranked relevant data
           │
      ┌────┴────┐
      ▼         ▼
   Memories   Documents
      │         │
      └────┬────┘
           ▼
      Context window
           │
           ▼
        AI answer
           │
           ▼
      Source references
```

Results retain source information so an answer can show what actually informed it.

---

## Documents

Supported document and media inputs include:

- TXT
- Markdown
- CSV
- PDF
- DOCX
- PNG
- JPEG
- WebP
- BMP
- Audio files supported by the bundled FFmpeg pipeline

Documents are extracted, chunked, embedded, and indexed.

Duplicate files are detected using content hashes, and document chunks participate in the same central search system as memories.

---

## Voice & image capture

Voice notes are transcribed locally using Whisper.

Images and screenshots are processed locally with OCR.

The resulting text can optionally be passed through the configured AI provider to extract useful entities such as:

- Tasks
- Reminders
- Dates
- Deadlines
- Contacts
- Addresses
- Useful information

The important part is that the original capture remains linked to the records created from it.

---

## Productivity

Nexora understands a set of deterministic natural-language commands without requiring an AI call.

Examples:

```text
"Remind me tomorrow at 7 PM to review the project."

"Every Monday remind me to review finances."

"Add milk and eggs to my grocery list."

"Create a task to finish the website by Friday."

"Show my overdue tasks."
```

Recurring tasks and reminders use timezone-aware recurrence rules.

Actions that can cause data loss require explicit confirmation.

---

## Proactive intelligence

Nexora can combine information from different parts of the system to produce:

- Daily briefings
- Weekly reviews
- Priority suggestions
- Missing-reminder suggestions
- Deadline clustering
- Calendar conflict detection
- Unanswered important-email detection
- Memory organization suggestions

The structured briefing works without an AI provider. AI-generated prose is an optional layer on top.

Suggestions also include deduplication, dismissal cooldowns, daily limits, and quiet hours to avoid turning the assistant into a notification machine.

---

## Gmail, Calendar & Drive

Google integrations use a reusable OAuth connector layer.

### Google Calendar

- Read events
- Create events
- Update events
- Delete events
- Timezone-aware scheduling
- Conflict detection

### Gmail

- Search
- Classify
- Summarize
- Extract tasks/deadlines/events
- Link emails to memories
- Generate drafts

Nexora intentionally does **not** send email on the user's behalf.

### Google Drive

- Discover files
- Index supported files
- Search indexed content
- Link files to memories

OAuth tokens are encrypted at rest.

---

## Telegram

Telegram is treated as another interface to the same Nexora backend.

Messages, voice notes, photos, and documents use the same underlying services as the web application.

That means there is no separate Telegram memory database.

Account linking is single-use and time-limited, with duplicate-update protection and rate limiting.

---

## Sharing & collaboration

Nexora includes resource-level sharing for memories, lists, tasks, and projects.

Permission levels:

| Permission | Access |
|---|---|
| `view` | Read |
| `edit` | Read + modify |
| `manage` | Modify + manage access + delete |

Authorization is enforced before resource access, with anti-enumeration behavior and an append-only audit log.

Sharing is intentionally explicit: sharing one resource does not automatically expose related resources.

---

## Mobile

<img src="https://github.com/user-attachments/assets/9fb92e4e-c7ed-465f-b9a9-1b590e1369e5" alt="Nexora mobile preview" width="45%">

Nexora is installable as a PWA and is designed around quick capture rather than simply shrinking the desktop interface.

The mobile experience includes:

- Today
- Tasks
- Capture
- Search
- Chat
- Camera capture
- Voice capture
- Quick notes
- Push notifications
- Short-lived read-only caching

Mutating operations still require a live connection; the current offline layer deliberately does not queue writes.

---

## Dashboard

<img src="https://github.com/user-attachments/assets/888e060c-048c-4044-80d8-e290f57003f0" alt="Nexora dashboard preview" width="100%">

The dashboard brings the system together in one interface.

Current sections include:

**Today · Calendar · Tasks · Reminders · Memories · Inbox · Documents · Lists · Projects · Email · Search · Shared Memory · Activity · Settings**

The interface supports responsive desktop/mobile layouts and dark, light, and system themes.

---

## Technology

### Frontend

- React
- TypeScript
- Vite
- React Router
- Custom CSS design system
- PWA / Workbox

### Backend

- Node.js
- Express
- TypeScript
- Prisma
- SQLite
- JWT authentication

### AI & retrieval

- Pluggable AI provider
- OpenAI provider
- Ollama provider
- Transformers.js
- Local embeddings
- Whisper
- Tesseract OCR
- Hybrid retrieval
- RAG

### Integrations

- Telegram Bot API
- Google OAuth
- Google Calendar API
- Gmail API
- Google Drive API
- Web Push

---

## Project structure

```text
nexora/
├── backend/
│   ├── prisma/
│   └── src/
│       ├── routes/
│       ├── services/
│       └── config/
│
├── frontend/
│   ├── public/
│   └── src/
│       ├── components/
│       ├── pages/
│       ├── context/
│       └── styles/
│
├── docs/
│   └── images/
│
├── .env.example
├── .gitignore
└── README.md
```

---

## Getting started

### Requirements

- Node.js 18+
- npm
- A local browser

Optional:

- Ollama for fully local AI
- Google OAuth credentials
- Telegram bot credentials
- VAPID keys for Web Push

### Backend

```bash
cd backend
cp .env.example .env
npm install
npx prisma migrate dev
npm run dev
```

The API runs on:

```text
http://localhost:4000
```

### Frontend

```bash
cd frontend
cp .env.example .env
npm install
npm run dev
```

The development frontend runs on:

```text
http://localhost:5173
```

The first semantic-search operation downloads the configured embedding model and caches it locally.

---

## Configuration

AI providers are selected through environment variables.

```env
AI_PROVIDER=ollama
```

or:

```env
AI_PROVIDER=openai
OPENAI_API_KEY=...
OPENAI_MODEL=...
```

Other integrations are optional and can be enabled independently.

**Never commit `.env` files or credentials.**

---

## Testing

Backend tests:

```bash
cd backend
npm test
```

Useful checks:

```bash
npm run typecheck
npm run lint
npm run build
```

The test suite covers memory isolation, retrieval, document processing, commands, recurring tasks/reminders, multimodal capture, Telegram flows, Google OAuth/connectors, proactive intelligence, sharing, memory reliability, and push notifications.

---

## Current status

**Early development / active build**

The core system is functional, but the project is not being presented as production-ready yet.

### Working

- Core authentication
- Memory engine
- Memory search
- Hybrid search
- RAG chat
- Documents
- Tasks
- Reminders
- Lists
- Voice capture
- Image/OCR capture
- Telegram integration
- Google Calendar / Gmail / Drive connectors
- Daily briefing
- Weekly review
- Proactive suggestions
- Sharing and permissions
- Memory review
- PWA
- Web Push
- Local embeddings
- Ollama provider

### Still being developed

- Collaboration UI polish
- More complete task editing UI
- More complete calendar editing
- Automatic background generation of proactive suggestions
- Conversation-history indexing
- Richer image understanding beyond OCR
- Additional connectors
- Offline write/sync support
- Production deployment and hardening

---

## Design principles

Nexora is being built around a few rules:

**Local when practical.**  
Personal information should not need to leave the machine unless a feature genuinely requires it.

**AI should be replaceable.**  
The application should not be tied to one model or provider.

**Memory should be explainable.**  
If Nexora remembers something, it should be possible to understand where it came from.

**Destructive actions should be deliberate.**  
The assistant should not silently delete or send things on the user's behalf.

**One backend, multiple interfaces.**  
Web, PWA, Telegram, and future clients should use the same underlying memory and productivity system.

**No fake completeness.**  
A feature should only be marked complete when it actually works.

---

## Roadmap

### Foundation
- [x] Authentication
- [x] Core API
- [x] Database
- [x] Memory engine

### Intelligence
- [x] Semantic search
- [x] Hybrid retrieval
- [x] RAG
- [x] Memory reliability
- [x] Explainability
- [x] Proactive suggestions

### Productivity
- [x] Tasks
- [x] Reminders
- [x] Lists
- [x] Projects
- [x] Recurrence

### Multimodal
- [x] Voice transcription
- [x] OCR
- [x] Document indexing

### Integrations
- [x] Telegram
- [x] Google Calendar
- [x] Gmail
- [x] Google Drive

### Mobile
- [x] PWA
- [x] Quick capture
- [x] Push notifications
- [ ] Offline write/sync

### Future
- [ ] More connectors
- [ ] Rich image understanding
- [ ] Conversation-history search
- [ ] More advanced automation
- [ ] Production deployment

---

## Privacy & security

Nexora is intended to handle personal information, so privacy is part of the architecture.

The project aims to:

- Keep credentials out of the frontend
- Encrypt external OAuth tokens at rest
- Enforce user-level data isolation
- Keep local processing local where practical
- Make external integrations explicit
- Support recoverable deletion
- Maintain audit trails for sensitive actions
- Avoid sending the entire personal database to an AI model

Before production use, the application should still receive a dedicated security review and deployment hardening pass.

---

## Contributing

Nexora is currently being developed as an independent project.

Contribution guidelines will be added once the core architecture and public development workflow are settled.

---

<div align="center">

<br>

<img src="https://github.com/user-attachments/assets/8072e1c0-4ca3-40d8-bb26-fd80064bf019" alt="Nexora logo" width="140">

<br><br>

**NEXORA**

*Capture · Remember · Find · Act*

</div>
