# ForgeBench Backend Server

Persistent Node.js + Express backend designed for background media processing (Vision LLM, PDF text extraction), living document rewriting, and chat-with-project query synthesis.

## Architecture
- **Framework**: Express.js
- **Database / Auth**: Supabase (PostgreSQL with Row Level Security)
- **Document Synthesis**: Non-blocking async living document synthesis engine
- **Media Pipeline**: Image vision extraction and PDF parsing

## Planned Structure
```
forge-bench-backend/
├── src/
│   ├── config/
│   │   └── env.js
│   ├── middleware/
│   │   └── auth.js
│   ├── routes/
│   │   ├── projects.js
│   │   ├── entries.js
│   │   └── chat.js
│   ├── services/
│   │   ├── supabase.js
│   │   ├── vision.js
│   │   ├── pdf.js
│   │   ├── mainDoc.js
│   │   └── chat.js
│   └── server.js
└── package.json
```
