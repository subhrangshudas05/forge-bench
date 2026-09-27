# Forge Bench — Full Architecture Document

## 1. Product Summary

A personal knowledge-forge app. Users create **Projects** (topics they want to develop — e.g. "Business Pricing Ideas", "IG Content Hooks"). Inside a project, they dump raw inputs — images, PDFs/docs, and later links/videos. The system extracts insight from each input and continuously rewrites a single, readable **Main Doc** per project — a living, organized note, not a growing log. Users can chat with a project (Q&A, summaries, to-do extraction) using the Main Doc as context.

Multi-user from day one (real auth), since friends/family will use it. Resume-facing — built with production patterns, not a toy demo.

---

## 2. Tech Stack

**Backend:** Node.js + Express (persistent server, not serverless — needed for background processing, WebSocket, job handling)

**Database:** Supabase (Postgres) — single database, no separate vector DB in v1

**Auth:** Supabase Auth (email/password + Google OAuth), Row Level Security (RLS) enforcing per-user data isolation

**File/Media processing:**
- Vision LLM (GPT-4o / Claude) — image understanding, no separate OCR tool
- `pdf-parse` — PDF text extraction
- `yt-dlp` — universal video/audio extraction (YouTube, IG, TikTok, etc.)
- Whisper API — audio-to-transcript

**AI orchestration:** Direct LLM API calls (OpenAI/Anthropic). Langchain optional, not required for v1 — single-step prompts don't need chain abstraction.

**Background jobs:** Async processing in Node initially. BullMQ + Redis as a documented upgrade path (v1.5) once reliability/retry needs justify it.

**Real-time:** WebSocket — scoped only to processing-status notifications (upload → processing → done/failed). Polling as fallback/simpler alternative if WebSocket adds too much overhead early.

**Deployment:** Backend on Railway (persistent server needed). Frontend on Vercel.

**Frontend:** Next.js (App Router) — same stack you already know. Framer Motion for animation/polish, Tailwind for styling, shadcn/ui for component primitives. This is a dashboard-style SaaS UI, not marketing-heavy, so keep it functional-first, polish later.

---

## 3. Database Schema (Postgres via Supabase)

```sql
-- users table handled automatically by Supabase Auth (auth.users)

-- projects
create table projects (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) not null,
  name text not null,
  overview text default '',        -- short, ~100-150 words, used as context for new-entry extraction
  main_doc text default '',        -- full rewritten doc, markdown formatted
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- entries (each raw input the user adds)
create table entries (
  id uuid primary key default gen_random_uuid(),
  project_id uuid references projects(id) not null,
  user_id uuid references auth.users(id) not null,
  type text not null,              -- 'image' | 'doc' | 'link' | 'video'
  source_url text,                 -- null for direct uploads
  extracted_text text,             -- raw extraction (OCR/vision output, pdf text, transcript)
  title text,
  tags text[],
  key_insight text,
  status text default 'processing', -- 'processing' | 'done' | 'failed'
  error_message text,
  created_at timestamptz default now()
);

-- Row Level Security: enforce user_id = auth.uid() on all reads/writes
```

**Note on SQL practice:** Prisma/Supabase client can handle most of this, but deliberately write a few raw SQL queries yourself (timeline query, any filtering/joins) to actually exercise SQL — this schema alone won't force that if you only use the ORM.

---

## 4. Processing Pipeline (per entry type)

### Image
1. User uploads image → saved temporarily (not stored long-term, per product decision — only extracted text persists)
2. Send directly to vision LLM (GPT-4o/Claude) with prompt: "Extract all text and describe the key insight/idea in this image."
3. Output → `extracted_text`

### Doc (PDF)
1. `pdf-parse` extracts raw text from uploaded PDF
2. Raw text → LLM for insight extraction if needed (or used directly if already clean)
3. Output → `extracted_text`

### Video (any platform — YouTube, IG, TikTok, etc.) — v2 feature, architecture ready
1. `yt-dlp -x --audio-format mp3 <url>` → downloads audio only
2. Audio → Whisper API → transcript
3. Transcript → `extracted_text`

### Link (non-video) — v2 feature
1. Fetch URL server-side, check Open Graph metadata / oEmbed first (fast, cheap)
2. Fallback to Puppeteer only if metadata insufficient
3. Extracted content → `extracted_text`

### Common final step (all types)
Once `extracted_text` exists:
1. **Entry-level extraction:** LLM call using `extracted_text` + project's `overview` (~100-150 words, short context) → produces `title`, `tags`, `key_insight` for this entry
2. **Main doc rewrite (async, non-blocking):** LLM call using current `main_doc` (full) + new entry's `extracted_text`/`key_insight` → produces fully rewritten `main_doc`
3. Both writes committed to DB, entry `status` set to `done`
4. WebSocket event (or poll response) notifies frontend

---

## 5. Main Doc Structure (what gets rewritten each time)

```markdown
# [Project Name]

## Overview
Short evolving description (this is what's reused as extraction context — keep ~100-150 words)

## Key Insights
### [Theme]
- Insight (source: entry title, date)
### [Theme]
- Insight (source: entry title, date)

## Recent Additions
- [date] entry title — one line
```

Rewritten (not appended) on every new insert, to stay readable long-term.

---

## 6. Chat-with-Project

- User asks a question inside a project
- Full current `main_doc` sent as context to LLM
- LLM answers (Q&A, "summarize this", "give me a to-do list from this") — summary/to-do outputs are generated on-demand, not stored
- No vector search / no per-entry retrieval in v1 — the Main Doc already serves as the single organized source of truth

**v2 addition (documented, not built yet):** pgvector-based cross-project search ("show me anything about pricing across all my projects") — legitimate use case for embeddings later, since main-doc-as-context doesn't scale across projects.

---

## 7. API Routes (Express)

```
Auth handled by Supabase client-side + JWT verification middleware server-side

POST   /api/projects                 - create project
GET    /api/projects                 - list user's projects
GET    /api/projects/:id             - get project + main_doc
PATCH  /api/projects/:id             - update project (rename, etc.)

POST   /api/projects/:id/entries     - upload new entry (image/doc/link/video)
GET    /api/projects/:id/entries     - list entries (timeline)
GET    /api/entries/:id/status       - poll processing status (or WebSocket equivalent)

POST   /api/projects/:id/chat        - ask a question, get LLM response using main_doc context
```

Middleware: JWT verification (via Supabase's issued token) on every route, RLS as the actual enforcement layer at DB level (defense in depth — even if middleware has a bug, DB still blocks cross-user access).

---

## 8. WebSocket Scope

Single purpose: notify frontend when an entry's processing status changes (`processing` → `done`/`failed`). Not used for chat streaming or anything else in v1 — kept deliberately narrow so it's a clean, demonstrable skill rather than added complexity everywhere.

---

## 9. Frontend Structure (after-login dashboard — primary focus)

**Stack:** Next.js (App Router), Tailwind, shadcn/ui, Framer Motion for transitions/polish

**Layout:**
```
Sidebar (persistent) — list of projects, "+ New Project"
Main area — changes based on what's selected:

  Dashboard/Overview (default view)
    → Grid/list of all projects, last updated, quick stats

  Project view (tabs within)
    → Main Doc (rendered markdown, read-only, styled like a clean notepad)
    → Timeline (chronological list of entries, type icon, title, date, status)
    → Chat (ask questions, get summaries/to-dos)
    → (future: Settings/context tuning)

Global "+" button (always accessible)
    → Dropdown: pick existing project OR "New Project"
    → Then upload flow (drag-drop image/PDF, or paste link later)
```

**Processing UX:** entry shows "processing" spinner state in timeline immediately after upload, updates live via WebSocket (or poll) to "done" with extracted title/tags, or "failed" with retry option.

**Pre-login (design later, not priority now):** simple homescreen — login + short SaaS overview. Explicitly deferred per your instruction — after-login dashboard is the v1 focus.

---

## 10. What This Project Covers (honest mapping)

**Phase 1 (Core Backend) — covered:**
- Node.js async patterns (real async processing pipeline)
- HTTP/REST API design (routes above)
- Auth done properly (Supabase Auth + JWT verification + RLS)
- Database design (schema above, relationships, indexing on `project_id`/`user_id`)

**Not covered by this project (needs separate practice):**
- Raw SQL fluency — deliberately write some raw queries in this project, plus separate focused SQL study
- Manual JWT-from-scratch — do as an isolated exercise elsewhere, not needed here since Supabase handles it

**Phase 4 (AI Integration) — covered:**
- LLM API usage (vision, text extraction, summarization)
- Context management (dual-context strategy: short overview vs full doc)
- Whisper (audio transcription)
- Real production pattern: async job status, structured extraction, doc-rewrite-not-append strategy
- Documented, justified path to vector search (v2) — shows engineering judgment, not just tool-stacking

**Resume framing:** "Built a multi-user AI knowledge-organization tool — Node/Express backend, Supabase Postgres with RLS-enforced multi-tenancy, async media processing pipeline (vision LLM, Whisper transcription via yt-dlp), and a context-aware document synthesis system that maintains a single coherent knowledge doc per topic rather than raw chat logs." — strong, specific, defensible in an interview.
