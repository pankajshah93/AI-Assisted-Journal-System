# 🌿 MindGrove — AI-Assisted Journal & Emotion Insight Platform

MindGrove lets users complete immersive nature sessions (forest / ocean / mountain), write a journal entry afterward, and get LLM-powered emotional insight into their own mental state over time. This README covers the full build plan, tech stack, setup, and a phase-by-phase roadmap — written so you can build it, learn every underlying concept, and explain the whole flow confidently in interviews.

---

## 1. Product Vision

- Structured from day one as a real product you can grow into a startup, not just a demo.
- Core loop: **Session → Journal → AI Analysis → Insights over time.**
- Designed to scale from "1 user on your laptop" to "100k users in production" without a rewrite — that's why Redis, Kafka, and CI/CD show up in the roadmap even though they're optional for the MVP.

---

## 2. Feature List

### MVP (core, must-have)
- [ ] Journal entry creation (`POST /api/journal`)
- [ ] Fetch all entries for a user (`GET /api/journal/:userId`)
- [ ] LLM emotion analysis on a journal entry (`POST /api/journal/analyze`) — real LLM call, not hardcoded output
- [ ] Aggregated insights per user (`GET /api/journal/insights/:userId`)
- [ ] Minimal frontend: write entry → view entries → click Analyze → view insights
- [ ] `README.md` + `ARCHITECTURE.md`

### Bonus (score boosters)
- [ ] Streaming LLM response (token-by-token on the frontend)
- [ ] Caching analysis results (Redis)
- [ ] Rate limiting (per-user, per-IP)
- [ ] Docker Compose (one command spins up everything)
- [ ] Deployed demo (Render/Railway backend + Vercel frontend)

### Advanced / "learn for the future" layer
- [ ] Redis — cache LLM results + rate-limit counters
- [ ] Kafka — async event pipeline for analysis jobs (decouples journal-write from LLM-call)
- [ ] CI/CD — GitHub Actions: lint → test → build → deploy on push to `main`
- [ ] JWT auth + multi-user isolation
- [ ] Horizontal scaling readiness (stateless API servers behind a load balancer)

---

## 3. Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Backend | **Node.js + Express.js** (primary) | Fast to build REST APIs, huge ecosystem, easy to add middleware (rate limit, auth) |
| Backend (alt/microservice) | **Python FastAPI** | Optional — use it later if you want to isolate the LLM-calling service from the main CRUD API (good excuse to practice polyglot microservices) |
| Frontend | **React** (Vite) + Tailwind CSS | Simple state management is enough for this scope; Tailwind keeps UI fast since "UI quality is not important" |
| Database | **PostgreSQL** | Relational fits journal entries + users + emotion tags well; free tier available everywhere (Supabase/Neon/Railway) |
| Cache / Rate limit | **Redis** | Cache repeated `/analyze` calls, store rate-limit counters, later used as a Kafka-adjacent job queue (BullMQ) |
| Message queue | **Kafka** (or Redis Streams/BullMQ as a lighter substitute while learning) | Decouple "entry saved" from "LLM analysis" — write path stays fast, analysis happens async |
| LLM Provider | **OpenRouter API** (you already have a key) | Free/cheap access to multiple models (e.g. `mistralai/mistral-7b-instruct:free`, `meta-llama/llama-3.1-8b-instruct:free`) through one unified API |
| Auth | **JWT** | Needed the moment you support multiple real users, not just a `userId` string in the body |
| Containerization | **Docker + Docker Compose** | One command (`docker compose up`) runs API + DB + Redis + frontend together |
| CI/CD | **GitHub Actions** | Auto-run tests + build on every push; auto-deploy on merge to `main` |
| Deployment | Backend → Render/Railway. Frontend → Vercel. DB → Supabase/Neon (managed Postgres). Redis → Upstash (free tier). | All have generous free tiers, so the whole stack can be deployed at $0 to start |

---

## 4. High-Level Architecture

```
┌─────────────┐        ┌──────────────────┐       ┌────────────────────┐
│   React     │  HTTP  │  Express API      │  SQL  │  PostgreSQL        │
│  Frontend   │ ─────► │  (auth, CRUD,     │ ─────►│  users / entries / │
│  (Vercel)   │        │   rate-limit)     │       │  analysis          │
└─────────────┘        └────────┬─────────┘       └────────────────────┘
                                 │
                        ┌────────▼─────────┐        ┌───────────────┐
                        │  Redis            │◄──────►│  Rate limiter │
                        │  (cache + queue)   │        │  counters     │
                        └────────┬─────────┘        └───────────────┘
                                 │ enqueue "analyze job"
                        ┌────────▼─────────┐
                        │  Kafka topic:      │
                        │  journal.analyze   │
                        └────────┬─────────┘
                                 │ consumed by
                        ┌────────▼─────────┐        ┌───────────────┐
                        │  Analysis Worker   │ ─────►│  OpenRouter   │
                        │  (Node or FastAPI) │        │  LLM API      │
                        └────────┬─────────┘        └───────────────┘
                                 │ writes result back
                                 ▼
                        PostgreSQL (analysis table) + Redis (cache)
```

For the MVP you can build the **top half only** (Frontend → API → Postgres, with a direct synchronous call to OpenRouter inside `/analyze`). The Kafka/worker half is the "v2, learn async processing" layer — add it once the MVP works end-to-end.

---

## 5. Folder Structure

```
mindgrove/
├── backend/
│   ├── src/
│   │   ├── routes/
│   │   │   ├── journal.routes.js
│   │   │   └── insights.routes.js
│   │   ├── controllers/
│   │   │   ├── journal.controller.js
│   │   │   └── insights.controller.js
│   │   ├── services/
│   │   │   ├── llm.service.js        # OpenRouter calls live here
│   │   │   └── cache.service.js      # Redis get/set wrapper
│   │   ├── middleware/
│   │   │   ├── auth.middleware.js
│   │   │   └── rateLimiter.middleware.js
│   │   ├── models/
│   │   │   ├── entry.model.js
│   │   │   └── user.model.js
│   │   ├── db/
│   │   │   └── index.js              # pg pool / Prisma client
│   │   └── app.js
│   ├── worker/                       # optional: Kafka consumer for async analysis
│   │   └── analyzeWorker.js
│   ├── Dockerfile
│   ├── package.json
│   └── .env.example
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── JournalForm.jsx
│   │   │   ├── EntryList.jsx
│   │   │   └── InsightsPanel.jsx
│   │   ├── api/
│   │   │   └── client.js
│   │   ├── App.jsx
│   │   └── main.jsx
│   ├── Dockerfile
│   └── package.json
├── docker-compose.yml
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── README.md
└── ARCHITECTURE.md
```

---

## 6. Database Schema (PostgreSQL)

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE journal_entries (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  ambience TEXT NOT NULL CHECK (ambience IN ('forest','ocean','mountain')),
  text TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE analysis_results (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  entry_id UUID REFERENCES journal_entries(id) ON DELETE CASCADE,
  emotion TEXT,
  keywords TEXT[],
  summary TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_entries_user_id ON journal_entries(user_id);
CREATE INDEX idx_analysis_entry_id ON analysis_results(entry_id);
```

`analysis_results` is a separate table (not a column on `journal_entries`) so you can cache/re-run analysis without mutating the original entry, and so `insights` queries stay fast with an index.

---

## 7. API Endpoints

### `POST /api/journal`
```json
// Request
{ "userId": "123", "ambience": "forest", "text": "I felt calm today after listening to the rain." }

// Response 201
{ "id": "uuid", "userId": "123", "ambience": "forest", "text": "...", "createdAt": "..." }
```

### `GET /api/journal/:userId`
Returns array of all entries for that user, newest first, paginated (`?page=1&limit=20`) once you go past MVP.

### `POST /api/journal/analyze`
```json
// Request
{ "text": "I felt calm today after listening to the rain" }

// Response 200
{ "emotion": "calm", "keywords": ["rain","nature","peace"], "summary": "User experienced relaxation during the forest session" }
```
Internally: check Redis cache by hash of `text` → if miss, call OpenRouter → parse JSON → cache result (e.g. 24h TTL) → return.

### `GET /api/journal/insights/:userId`
```json
{ "totalEntries": 8, "topEmotion": "calm", "mostUsedAmbience": "forest", "recentKeywords": ["focus","nature","rain"] }
```
Computed via a single aggregation query (`GROUP BY emotion`, `GROUP BY ambience`, `ORDER BY count DESC LIMIT 1`) rather than pulling all rows into app memory — this matters once a user has thousands of entries.

---

## 8. LLM Integration (OpenRouter)

```js
// backend/src/services/llm.service.js
const OPENROUTER_URL = "https://openrouter.ai/api/v1/chat/completions";

async function analyzeEmotion(text) {
  const prompt = `Analyze the emotional tone of this journal entry. Respond ONLY with valid JSON in this exact shape, no extra text:
{"emotion": "<single word>", "keywords": ["<3 words>"], "summary": "<one sentence>"}

Journal entry: "${text}"`;

  const res = await fetch(OPENROUTER_URL, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.OPENROUTER_API_KEY}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      model: "mistralai/mistral-7b-instruct:free",
      messages: [{ role: "user", content: prompt }]
    })
  });

  const data = await res.json();
  const raw = data.choices[0].message.content;
  return JSON.parse(raw.replace(/```json|```/g, "").trim());
}
```
Wrap this in try/catch, validate the parsed shape, and fall back to a retry-with-stricter-prompt if JSON parsing fails once — never fall back to a hardcoded/dummy response; a real analysis pipeline is the whole point of the product.

---

## 9. Setup & Run Locally

### Prerequisites
- Node.js 18+, PostgreSQL 14+, Redis (optional for MVP), an OpenRouter API key

### Backend first (your plan — build & test this before touching frontend)
```bash
cd backend
cp .env.example .env        # fill in DATABASE_URL, OPENROUTER_API_KEY, REDIS_URL, JWT_SECRET
npm install
npm run migrate             # creates tables from schema above
npm run dev                 # starts on http://localhost:5000
```
Test each endpoint with `curl` or Postman before writing a single line of frontend code:
```bash
curl -X POST http://localhost:5000/api/journal -H "Content-Type: application/json" \
  -d '{"userId":"123","ambience":"forest","text":"I felt calm today."}'

curl -X POST http://localhost:5000/api/journal/analyze -H "Content-Type: application/json" \
  -d '{"text":"I felt calm today after listening to the rain"}'
```

### Then frontend
```bash
cd frontend
npm install
npm run dev                 # starts on http://localhost:5173
```

### Or everything at once with Docker
```bash
docker compose up --build
```

---

## 10. Suggested Build Order (efficient, in-order roadmap)

1. **DB + models** — write the schema, run migrations, seed one test user.
2. **Journal CRUD API** (`POST /api/journal`, `GET /api/journal/:userId`) — no LLM yet, just prove storage works.
3. **LLM analyze endpoint** — wire up OpenRouter, get real JSON back, store in `analysis_results`.
4. **Insights endpoint** — aggregation query over stored analysis.
5. **Frontend** — one page: entry form → list → Analyze button → insights panel. Call the four endpoints in order.
6. **Auth (JWT)** — swap the raw `userId` string for a real logged-in user; this is what makes it genuinely multi-user.
7. **Redis caching** — cache `/analyze` by text-hash; add rate limiting middleware.
8. **Dockerize** — Dockerfile per service + `docker-compose.yml`.
9. **CI/CD** — GitHub Actions workflow: install → lint → test → build → deploy.
10. **Deploy** — backend to Render/Railway, frontend to Vercel, DB to Supabase/Neon, Redis to Upstash.
11. **(Learning stretch) Kafka** — introduce a `journal.analyze` topic; API publishes an event on entry creation, a separate worker consumes it and calls the LLM, decoupling write-latency from LLM-latency. Do this last — it's the piece that's optional for the MVP but valuable for your own understanding of async architectures.

Building in this order means you always have something *runnable* at every step, which keeps momentum up and makes the project easy to demo at any point.

---

## 11. Interview Talking Points (so you can explain 0 → advanced)

- **Why separate `analysis_results` from `journal_entries`?** Keeps the write path fast and lets you re-run/version analysis without touching source data.
- **Why cache in Redis instead of just relying on Postgres?** Sub-millisecond reads for repeated identical text, and it takes load off both Postgres and OpenRouter (cost + latency).
- **Why Kafka instead of just calling the LLM inline?** At scale, LLM calls are the slowest and least reliable part of the request — decoupling means a slow/failed LLM call never blocks the user's journal save, and you can retry/backoff independently.
- **Why JWT over sessions?** Stateless auth scales horizontally without sticky sessions or a shared session store.
- **What breaks first at 100k users, and how do you know?** (See `ARCHITECTURE.md` — this is the exact kind of question interviewers ask.)

See `ARCHITECTURE.md` for the full scaling / cost / caching / security answers.

---

## 12. Environment Variables (`.env.example`)

```
DATABASE_URL=postgresql://user:password@localhost:5432/mindgrove
OPENROUTER_API_KEY=sk-or-xxxxxxxx
JWT_SECRET=change-me
REDIS_URL=redis://localhost:6379
PORT=5000
NODE_ENV=development
```

## 13. License
MIT — build on it freely.
