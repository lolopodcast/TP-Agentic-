# SMMC AI 企業策略協同工作站 — Project Instructions

> This file is read by Claude Code at the start of every session.
> Last updated: 2026-04-08

---

## Project Owner

**Prof. Shihmin Lo (Jimmy)**
Department of International Business, National Chi Nan University (NCNU)
- Does NOT write code directly; works iteratively via natural language with AI assistants
- Confirms outlines before full generation; identifies bugs with concrete examples; applies incremental fixes
- TAs operate interfaces but do not write code

---

## Project Overview

A bilingual (zh-TW / en-GB) AI-powered enterprise strategy analysis workstation for the SMMC-EMI course. Students input a target company; the system searches for public data, builds a knowledge base, then runs three AI agents sequentially to produce a comprehensive strategy report following the TP0–TP3 analytical framework.

**Live URL (frontend):** `https://lolopodcast.github.io/SMMC-Strategy-Workstation/`
**Worker endpoint:** `https://smmc-strategy-api.{account}.workers.dev`

---

## Architecture

```
Frontend (GitHub Pages)          Cloudflare Worker (/api)         Cloudflare D1
┌──────────────────────┐        ┌──────────────────────────┐     ┌─────────────────┐
│ Single-file HTML SPA │──fetch─▶│ Auth gate (token-based)  │     │ knowledge_base  │
│ React 18 + Tailwind  │        │ Gemini proxy + grounding │─────▶│ reports         │
│ Babel CDN transform  │◀──SSE──│ ReAct engine (3 rounds)  │     │ tokens          │
│ No API keys in FE    │        │ Report CRUD              │     │ usage_log       │
└──────────────────────┘        │ Admin APIs               │     └─────────────────┘
                                └──────────────────────────┘
```

### Key Files
- `index.html` — the entire frontend SPA (single file, ~1200+ lines)
- `worker.js` — Cloudflare Worker backend (single file)
- `wrangler.toml` — Worker configuration (D1 binding name: `DB`, database: `smmc-strategy-db`)

### Tech Stack
| Layer | Technology |
|-------|-----------|
| Frontend | React 18 + Tailwind CSS + Babel (all via CDN, single HTML file) |
| Backend | Cloudflare Worker (single `worker.js`) |
| Database | Cloudflare D1 (SQLite) |
| LLM | Google Gemini API (with grounding/search) |
| Deployment | GitHub Pages (frontend) + Cloudflare Workers (backend) |

---

## Design System — BCG Academic Style

All UI must follow this established design system consistently:

| Token | Value | Usage |
|-------|-------|-------|
| Primary (deep green) | `#004030` | Headers, buttons, accents, brand |
| Secondary (warm gold) | `#D8C3A5` | Highlights, borders, hover states |
| Font — headings | `Merriweather` (serif) | All headings, brand text |
| Font — body | `Inter` (sans-serif) | Body text, UI elements |
| Dark mode | Slate-800/900 backgrounds | Full dark mode support required |
| Border radius | `rounded-xl` (16px) for cards | Consistent rounded corners |

**Bilingual requirements:**
- zh-TW (台灣繁體中文) as primary; en-GB as secondary
- Use `dict` object with `zh` and `en` keys for ALL user-facing strings
- NEVER use Simplified Chinese terms (項目→專案, 數據庫→資料庫, 視頻→影片, 賦能→賦予能力)
- Language toggle in header; default follows browser locale

---

## Authentication & Authorization

Three-role system (NO user registration — token-based):

| Role | Token source | Capabilities |
|------|-------------|-------------|
| **admin** (Professor) | Hardcoded: `proflolothebestever` | Everything + Admin panel + PDF export + no rate limit |
| **ta** | Set via Admin panel, stored in D1 `tokens` table | View all reports + ReAct trace panel |
| **student** | Shared token set via Admin panel; selects group (G1–G12) at login | Run analysis + view own group's history only |

### Rate Limiting
- Per-group daily analysis limit (configurable by admin)
- Admin is exempt from all rate limits
- Tracked in D1 `usage_log` table

---

## D1 Database Schema

```sql
-- Knowledge base for RAG
CREATE TABLE knowledge_base (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  company TEXT NOT NULL,
  source_url TEXT DEFAULT '',
  content TEXT NOT NULL,
  created_at TEXT DEFAULT (datetime('now'))
);

-- Analysis reports
CREATE TABLE reports (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  company TEXT NOT NULL,
  lang TEXT DEFAULT 'zh',
  agent1 TEXT, agent2 TEXT, agent3 TEXT,
  agent1_trace TEXT DEFAULT '',
  agent2_trace TEXT DEFAULT '',
  agent3_trace TEXT DEFAULT '',
  owner_group TEXT DEFAULT '',
  owner_role TEXT DEFAULT 'student',
  rag_context TEXT,
  options TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);

-- Auth tokens
CREATE TABLE tokens (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  token TEXT UNIQUE NOT NULL,
  role TEXT NOT NULL,        -- 'student' or 'ta'
  label TEXT DEFAULT '',
  daily_limit INTEGER DEFAULT 5,
  expires_at TEXT,
  created_at TEXT DEFAULT (datetime('now'))
);

-- Usage tracking
CREATE TABLE usage_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  group_name TEXT NOT NULL,
  action TEXT DEFAULT 'analyze',
  created_at TEXT DEFAULT (datetime('now'))
);
```

---

## ReAct Agent System

Each of the 3 agents runs an autonomous ReAct loop:

```
Thought → Action → Observation → (repeat up to 3 rounds) → Finish
```

### Available Tools
| Tool | Description |
|------|-----------|
| `search(query)` | Trigger Gemini grounding search on a specific subtopic |
| `lookup_kb(keyword)` | Full-text search D1 knowledge_base (SQL LIKE) |
| `calculate(expression)` | Simple financial calculations |
| `finish(report)` | Output final report in Markdown and end the loop |

### SSE Streaming
The `analyze` action returns Server-Sent Events for real-time display:
- Each Thought/Action/Observation step is streamed as it happens
- Frontend shows a live trace panel (visible to admin and TA only)

---

## TP0–TP3 Analytical Framework

This is the core academic framework. The full prompts are embedded in `worker.js` as system prompts.

### Agent 1 → TP0 (Company Background) + TP1 (External Environment & Industry)

**TP0 must cover 6 items:**
1. 1-2 latest major news items
2. History, Vision, Mission, Core Values
3. Organizational structure & senior management team
4. Business & product portfolio (with revenue % breakdown)
5. 3-5 year financial performance (profitability, operational efficiency, financial structure, solvency, stock price)
6. Source citations

**TP1 must follow this sequence:**
1. Preamble (0.5-1 paragraph): summarize company significance from TP0; specify research scope (whole group vs. specific BU); reference annual report on opportunities/threats/risks
2. PESTEL — focus on 2-3 most relevant factors (depth over breadth)
3. Industry value chain — identify upstream/midstream/downstream partners & competitors
4. Industry life cycle — specify current stage (introduction/growth/maturity/decline)
5. Five Forces — focus on 2-3 most critical forces (cite concentration ratios if available)
6. Competitor analysis — market commonality + resource similarity
7. Competitive dynamics — actions vs. responses
8. Summary: 4P (Position, Priority, Posture, Profit potential)

### Agent 2 → TP2 (Resources & Capabilities)

**Must cover 6 layers:**
1. Organizational structure & evolution (BU/FU/GU design logic, tied to VP and TA)
2. Corporate value activity chain/system/network (KA, CR, CH interrelationships → VP delivery to TA)
3. Key resource inventory (focus 3-4): tangible + intangible (human capital, structural/organizational capital, relational/social capital), with CS and RS data
4. VRIN assessment (Valuable, Rare, Inimitable, Non-substitutable) — must cite investment news / industry reports as evidence
5. Core competencies + complementary assets + key partners (KP)
6. Dynamic capabilities (sensing, seizing, transforming) — from recent 3-5 year annual reports

### Agent 3 → TP3 (Corporate Strategy & Future Development)

**Must analyze 3 growth dimensions with theory:**
1. Vertical Integration (VI) — using Transaction Cost Theory:
   - Direction (upstream/downstream)
   - 3 evaluation dimensions: asset specificity, frequency, uncertainty
   - Governance mechanism: greenfield, strategic alliance, or M&A
2. Horizontal Diversification (HD):
   - BCG Matrix positioning
   - Synergy analysis: economies of scale/scope/learning, network effects
3. Geographic Expansion (GE):
   - Internationalization strategy type: international/global/multi-domestic/transnational
   - Organizational design for international operations
   - National competitive advantage (Porter's Diamond)
4. Corporate life cycle perspective on innovation/transformation/growth
5. Agency problem and organizational inertia considerations
6. Future challenges & governance (ESG, stakeholder interests)

---

## PDF Export

- Only visible to admin role
- Uses `window.print()` with CSS `@page` rules (no third-party PDF library)
- BCG-styled layout: cover page (deep green #004030 background, gold accents), table of contents, 3 agent sections, footer with version and architect info
- Opens in new window → triggers browser print dialog → user saves as PDF

---

## Worker API Reference

Single endpoint: `POST /api` with `{ action, token, ... }` body.

| Action | Auth Required | Role | Description |
|--------|:---:|------|-----------|
| `auth` | — | any | Verify token, returns `{ role, group }` |
| `search` | ✅ | any | Gemini grounding search → save to D1 |
| `rag_retrieve` | ✅ | any | Full-text search knowledge_base |
| `analyze` | ✅ | any | Run 3-agent ReAct analysis (SSE response) |
| `save_report` | ✅ | any | Save completed report |
| `list_reports` | ✅ | any | List reports (students: own group only) |
| `get_report` | ✅ | any | Get single report by ID |
| `admin_get_tokens` | ✅ | admin | List all tokens |
| `admin_set_token` | ✅ | admin | Create/update a token |
| `admin_delete_token` | ✅ | admin | Delete a token |
| `admin_kb_stats` | ✅ | admin | Knowledge base statistics |
| `admin_clear_kb` | ✅ | admin | Clear KB (by company or all) |
| `admin_usage_stats` | ✅ | admin | Usage statistics |

---

## Cloudflare Environment Variables

| Variable | Type | Description |
|----------|------|-----------|
| `GEMINI_KEY` | Secret | Google Gemini API key |
| `CORS_ORIGIN` | Text | Allowed frontend origin (e.g., `https://lolopodcast.github.io`) |

Note: The old `AUTH_TOKEN` secret is no longer used (replaced by D1 `tokens` table + hardcoded admin token).

D1 binding variable name: `DB`

---

## Coding Conventions

1. **Frontend is always a single HTML file** — React + Tailwind + Babel via CDN. No build tools, no npm.
2. **Worker is a single JS file** — no bundler, plain ES modules compatible with Cloudflare Workers runtime.
3. **All user-facing strings** go in the bilingual `dict` object (`dict.zh` / `dict.en`). Never hardcode Chinese or English directly in JSX.
4. **Version string** format: `Version X.Y.Z (YYYY.MM) — Edition Name` in both zh and en dict entries (`footer_version`).
5. **Architect credit** always present in footer: `Prof. Shihmin Lo` with link to NCNU faculty page.
6. **Error messages** must be bilingual and user-friendly (students are non-technical).
7. **No Simplified Chinese** — always use Taiwan standard terminology.
8. **CSS variables** for theming where possible; dark mode via Tailwind `dark:` prefix.

---

## Deployment Checklist

### Frontend (GitHub Pages)
1. Update `API_BASE` constant in `index.html` to match Worker URL
2. Upload to GitHub repo → Settings → Pages → Deploy from `main` branch

### Backend (Cloudflare Worker)
1. Paste `worker.js` into Cloudflare Dashboard editor → Save and deploy
2. Bind D1 database (`DB` → `smmc-strategy-db`)
3. Set `GEMINI_KEY` secret
4. Set `CORS_ORIGIN` text variable
5. D1 tables are auto-created on first request; for upgrades, run ALTER TABLE manually in D1 Console

---

## Known Issues & Gotchas

- **Gemini grounding search** sometimes returns thin results for smaller/non-public companies — the system gracefully falls back to Gemini's general knowledge
- **Agent output formatting** can occasionally break if Gemini injects malformed JSON escape characters in the ReAct `finish` action_input — the Worker has a sanitization step but edge cases may slip through
- **D1 Console** only accepts pure SQL — do NOT paste markdown code fences (```sql) into it
- **SSE streaming** requires the frontend to use `EventSource` or manual `fetch` with `ReadableStream` — standard `fetch().json()` will NOT work for the `analyze` action
- **Admin token `proflolothebestever`** is hardcoded in `worker.js` — change it there if needed (not in environment variables)

---

## Communication Preferences

- Jimmy prefers warm, plain language — avoid bureaucratic or ceremonial phrasing
- When proposing changes, always outline the plan first and get confirmation before writing code
- Use structured inputs (multiple-choice, confirm/deny) to clarify requirements efficiently
- When reporting bugs, provide the exact error message and steps to reproduce
- Keep explanations bilingual when discussing with Jimmy (zh-TW primary, English terms for technical concepts)
