# Forge Bench — Build Workflow

## Why Supabase Auth over NextAuth

**Supabase Auth wins here because:**

1. **You're already using Supabase Postgres.** Supabase Auth is built into the same project — users, sessions, and RLS (Row Level Security) all connect natively. `auth.uid()` is directly usable inside your SQL policies. With NextAuth you'd need to manually sync a separate auth system with your database's user references.

2. **RLS enforcement is the whole point of your multi-tenancy design.** Your architecture doc relies on RLS as the real security layer ("defense in depth"). Supabase Auth's JWT is understood natively by Postgres RLS policies. NextAuth's JWT isn't — you'd have to build that bridge yourself.

3. **Less code to write.** Supabase gives you signup/login/OAuth (Google) as SDK calls, both frontend and backend verification. NextAuth is more flexible/provider-agnostic but that flexibility is unnecessary complexity for your case — you know you're on Supabase for the DB already.

4. **NextAuth would make sense if:** you were using a different database/ORM setup unrelated to Supabase, or needed many exotic OAuth providers. Neither applies here.

**Verdict:** Supabase Auth is the correct, simpler choice for this stack. Don't second-guess it.

---

## Folder Structure — Two Separate Folders (Not a Monorepo... Yet)

Keep backend and frontend **completely separate** for now. Simpler to reason about, deploy independently (Railway for backend, Vercel for frontend), and matches your architecture doc's deployment plan exactly.

```
forge-bench/
├── forge-bench-backend/          ← Node.js + Express, deploys to Railway
│   ├── src/
│   │   ├── routes/
│   │   │   ├── projects.js
│   │   │   ├── entries.js
│   │   │   └── chat.js
│   │   ├── middleware/
│   │   │   └── auth.js           ← JWT verification middleware
│   │   ├── services/
│   │   │   ├── supabase.js       ← Supabase client init
│   │   │   ├── vision.js         ← image → LLM extraction
│   │   │   ├── pdf.js            ← pdf-parse wrapper
│   │   │   ├── mainDoc.js        ← main_doc rewrite logic
│   │   │   └── chat.js           ← chat-with-project logic
│   │   ├── config/
│   │   │   └── env.js
│   │   └── server.js             ← Express app entry point
│   ├── .env
│   ├── .gitignore
│   ├── package.json
│   └── README.md
│
└── forge-bench-frontend/         ← Next.js, deploys to Vercel
    ├── app/
    │   ├── (auth)/
    │   │   ├── login/page.tsx
    │   │   └── signup/page.tsx
    │   ├── dashboard/
    │   │   ├── page.tsx           ← projects grid/overview
    │   │   └── [projectId]/
    │   │       ├── page.tsx       ← project view (tabs: doc/timeline/chat)
    │   │       └── layout.tsx
    │   ├── layout.tsx
    │   └── page.tsx               ← pre-login homescreen (build later)
    ├── components/
    │   ├── ui/                    ← shadcn components
    │   ├── Sidebar.tsx
    │   ├── ProjectCard.tsx
    │   ├── MainDocView.tsx
    │   ├── Timeline.tsx
    │   ├── ChatPanel.tsx
    │   └── UploadModal.tsx
    ├── lib/
    │   ├── supabase.ts             ← Supabase client (frontend)
    │   └── api.ts                  ← calls to your Express backend
    ├── .env.local
    ├── package.json
    └── README.md
```

**Why separate, not combined:** Your backend needs to run as a **persistent server** (background jobs, WebSocket) — that requires Railway or similar, not serverless. Next.js API routes are serverless by default on Vercel, which conflicts with your background processing needs. Keeping them separate avoids fighting that mismatch. If you ever need a monorepo later (shared types, etc.), you can add a tool like Turborepo — not needed at your current scale.

---

## Build Workflow — Step by Step

### Phase 0 — Setup (Day 1, ~1 hr)
- [ ] Create Supabase project, run schema SQL from architecture doc
- [ ] Enable RLS on `projects` and `entries` tables
- [ ] Grab: project URL, anon key, service role key
- [ ] `mkdir forge-bench-backend forge-bench-frontend`
- [ ] Backend: `npm init -y && npm install express cors dotenv @supabase/supabase-js`
- [ ] Confirm Express runs with one test route

### Phase 1 — Auth Wiring (Day 1-2, ~2 hrs)
- [ ] Frontend: Supabase client setup, login/signup pages (email + Google OAuth)
- [ ] Backend: JWT verification middleware — validates Supabase token, attaches `req.user`
- [ ] Test: log in on frontend, confirm token reaches backend and verifies correctly

### Phase 2 — Core CRUD (Day 2-3, ~3 hrs)
Build and test each with Postman/Thunder Client before moving to the next:
- [ ] `POST /api/projects`
- [ ] `GET /api/projects`
- [ ] `GET /api/projects/:id`
- [ ] `PATCH /api/projects/:id`
- [ ] Confirm RLS blocks cross-user access (test with two accounts)

### Phase 3 — Entry Upload Skeleton (Day 3, ~2 hrs)
- [ ] `POST /api/projects/:id/entries` — accept file, save row with `status: 'processing'`, return immediately
- [ ] No AI yet — just prove the shape works

### Phase 4 — Image Pipeline (Day 3-4, ~3 hrs)
- [ ] Wire vision LLM call (GPT-4o or Claude) for images
- [ ] Full flow: upload → extract → save `extracted_text`, `title`, `tags`, `key_insight` → `status: 'done'`
- [ ] This is your first fully working end-to-end slice — celebrate it

### Phase 5 — Main Doc Rewrite (Day 4, ~2 hrs)
- [ ] LLM call: current `main_doc` + new entry → rewritten `main_doc`
- [ ] This is the most "impressive" piece of the whole project — get it clean

### Phase 6 — PDF Support (Day 5, ~1 hr)
- [ ] Add `pdf-parse`, reuse the same pipeline shape as images

### Phase 7 — Chat Endpoint (Day 5, ~2 hrs)
- [ ] `POST /api/projects/:id/chat` — send `main_doc` + question to LLM, return answer

### Phase 8 — Frontend Dashboard (Day 6-9, ~8 hrs spread out)
- [ ] Sidebar with project list + "New Project"
- [ ] Project view: Main Doc tab (rendered markdown), Timeline tab, Chat tab
- [ ] Upload modal (drag-drop)
- [ ] Framer Motion polish pass last, after functionality works

### Phase 9 — Status Updates (Day 9-10, ~2 hrs)
- [ ] Start with simple polling (`GET /api/entries/:id/status`)
- [ ] Upgrade to WebSocket only after polling confirms the flow works

### Deferred to v2 (don't touch yet)
- Video/link processing (yt-dlp, Whisper)
- BullMQ + Redis job queue
- pgvector cross-project search
- Pre-login marketing homepage

---

## Golden Rule While Building

Don't move to the next phase until the current one is tested and actually works. This project's value on a resume comes from it being real and functional, not from how much code exists. A working image-upload-to-AI-doc pipeline (Phases 0-5) is already a legitimate, demoable portfolio piece even before chat or the frontend dashboard exist.
