# 🌿 MindGrove — AI-Assisted Journal & Emotion Insight Platform

MindGrove lets users complete immersive nature sessions (forest / ocean / mountain), write a journal entry afterward, and get LLM-powered emotional insight into their own mental state over time. This README covers the full build plan, tech stack, setup, and a phase-by-phase roadmap.

---

## 1. Product Vision

- Structured from day one as a real product you can grow into a startup, not just a demo.
- Core loop: **Session → Journal → AI Analysis → Insights over time.**
- Designed to scale from "1 user on your laptop" to "100k users in production" without a rewrite — that's why Redis, Kafka, and CI/CD show up in the roadmap even though they're optional for the core version.

---

## 2. Feature List

### Core Features
- [ ] Journal entry creation (`POST /api/journal`)
- [ ] Fetch all entries for a user (`GET /api/journal/:userId`)
- [ ] LLM emotion analysis on a journal entry (`POST /api/journal/analyze`) — real LLM call, not hardcoded output
- [ ] Aggregated insights per user (`GET /api/journal/insights/:userId`)
- [ ] Minimal frontend: write entry → view entries → click Analyze → view insights
- [ ] `README.md` + `ARCHITECTURE.md`

### Extended Features
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
- [ ] MongoDB replica set (built into Atlas by default) + a look at sharding for future scale
- [ ] Horizontal scaling readiness (stateless API servers behind a load balancer)

---

## 3. Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Backend | **Node.js + Express.js** | Fast to build REST APIs, huge ecosystem, easy to add middleware (rate limit, auth) |
| Frontend | **React** (Vite) + Tailwind CSS | Simple state management is enough for this scope; Tailwind keeps UI fast since UI polish isn't the priority yet |
| Database | **MongoDB** | Document model fits journal entries naturally (embed the analysis result inside the entry); free tier via MongoDB Atlas; matches what you're actively learning (replica sets, sharding) |
| Cache / Rate limit | **Redis** | Cache repeated `/analyze` calls, store rate-limit counters, later used as a Kafka-adjacent job queue (BullMQ) |
| Message queue | **Kafka** (or Redis Streams/BullMQ as a lighter substitute while learning) | Decouple "entry saved" from "LLM analysis" — write path stays fast, analysis happens async |
| LLM Provider | **OpenRouter API** (you already have a key) | Free/cheap access to multiple models (e.g. `mistralai/mistral-7b-instruct:free`, `meta-llama/llama-3.1-8b-instruct:free`) through one unified API |
| Auth | **JWT** | Needed the moment you support multiple real users, not just a `userId` string in the body |
| Containerization | **Docker + Docker Compose** | One command (`docker compose up`) runs API + DB + Redis + frontend together |
| CI/CD | **GitHub Actions** | Auto-run tests + build on every push; auto-deploy on merge to `main` |
| Deployment | Backend → Render/Railway. Frontend → Vercel. DB → MongoDB Atlas (free tier, replica set included). Redis → Upstash (free tier). | All have generous free tiers, so the whole stack can be deployed at $0 to start |

---

## 4. High-Level Architecture

```
┌─────────────┐        ┌──────────────────┐       ┌────────────────────┐
│   React     │  HTTP  │  Express API      │ Mongo │  MongoDB            │
│  Frontend   │ ─────► │  (auth, CRUD,     │Driver ►│  users /            │
│  (Vercel)   │        │   rate-limit)     │       │  journalEntries      │
└─────────────┘        └────────┬─────────┘       │  (analysis embedded) │
                                 │                  └────────────────────┘
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
                        │  (Node.js)         │        │  LLM API      │
                        └────────┬─────────┘        └───────────────┘
                                 │ writes result back
                                 ▼
                        MongoDB (embedded in entry doc) + Redis (cache)
```

For the first working version you can build the **top half only** (Frontend → API → MongoDB, with a direct synchronous call to OpenRouter inside `/analyze`). The Kafka/worker half is the "v2, learn async processing" layer — add it once the core version works end-to-end.

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
│   │   │   ├── journalEntry.model.js # Mongoose schema
│   │   │   └── user.model.js         # Mongoose schema
│   │   ├── db/
│   │   │   └── index.js              # MongoDB connection (mongoose.connect)
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

## 6. Database Schema (MongoDB / Mongoose)

Two collections. Analysis is **embedded** inside the journal entry document since it's a one-to-one relationship — no need for a separate collection or a join.

```js
// models/user.model.js
const userSchema = new Schema({
  email: { type: String, required: true, unique: true },
  passwordHash: { type: String, required: true },
  createdAt: { type: Date, default: Date.now }
});

// models/journalEntry.model.js
const journalEntrySchema = new Schema({
  userId: { type: Schema.Types.ObjectId, ref: "User", required: true, index: true },
  ambience: { type: String, enum: ["forest", "ocean", "mountain"], required: true },
  text: { type: String, required: true },
  analysis: {
    emotion: String,
    keywords: [String],
    summary: String
  },
  createdAt: { type: Date, default: Date.now }
});

// compound index — speeds up insights aggregation per user
journalEntrySchema.index({ userId: 1, createdAt: -1 });
```

Embedding `analysis` inside the entry document (instead of a separate `analysisResults` collection) means one query returns the entry *and* its analysis together — no join needed, which is exactly what MongoDB's document model is good at for one-to-one/one-to-few relationships.

---

## 7. API Endpoints

### `POST /api/journal`
```json
// Request
{ "userId": "123", "ambience": "forest", "text": "I felt calm today after listening to the rain." }

// Response 201
{ "id": "ObjectId", "userId": "123", "ambience": "forest", "text": "...", "createdAt": "..." }
```

### `GET /api/journal/:userId`
Returns array of all entries for that user, newest first, paginated (`?page=1&limit=20`) once you go past the core version.

### `POST /api/journal/analyze`
```json
// Request
{ "text": "I felt calm today after listening to the rain" }

// Response 200
{ "emotion": "calm", "keywords": ["rain","nature","peace"], "summary": "User experienced relaxation during the forest session" }
```
Internally: check Redis cache by hash of `text` → if miss, call OpenRouter → parse JSON → cache result (e.g. 24h TTL) → also write into the entry's `analysis` field → return.

### `GET /api/journal/insights/:userId`
```json
{ "totalEntries": 8, "topEmotion": "calm", "mostUsedAmbience": "forest", "recentKeywords": ["focus","nature","rain"] }
```
Computed via a MongoDB aggregation pipeline (`$match` on userId → `$group` by `analysis.emotion` and by `ambience` → `$sort` → `$limit`) rather than pulling all documents into app memory — this matters once a user has thousands of entries.

```js
// insights.controller.js — top emotion example
const topEmotion = await JournalEntry.aggregate([
  { $match: { userId } },
  { $group: { _id: "$analysis.emotion", count: { $sum: 1 } } },
  { $sort: { count: -1 } },
  { $limit: 1 }
]);
```

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
- Node.js 18+, MongoDB (local via `mongod`, or a free MongoDB Atlas cluster), Redis (optional for the core version), an OpenRouter API key

### Backend first (your plan — build & test this before touching frontend)
```bash
cd backend
cp .env.example .env        # fill in MONGODB_URI, OPENROUTER_API_KEY, REDIS_URL, JWT_SECRET
npm install
npm run dev                 # starts on http://localhost:5000
```
Mongoose creates collections automatically the first time you insert a document — no separate migration step like SQL needs.

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

1. **DB + models** — write the Mongoose schemas, connect to MongoDB (local or Atlas), seed one test user.
2. **Journal CRUD API** (`POST /api/journal`, `GET /api/journal/:userId`) — no LLM yet, just prove storage works.
3. **LLM analyze endpoint** — wire up OpenRouter, get real JSON back, store it in the entry's embedded `analysis` field.
4. **Insights endpoint** — aggregation pipeline over stored entries.
5. **Frontend** — one page: entry form → list → Analyze button → insights panel. Call the four endpoints in order.
6. **Auth (JWT)** — swap the raw `userId` string for a real logged-in user; this is what makes it genuinely multi-user.
7. **Redis caching** — cache `/analyze` by text-hash; add rate limiting middleware.
8. **Dockerize** — Dockerfile per service + `docker-compose.yml`.
9. **CI/CD** — GitHub Actions workflow: install → lint → test → build → deploy.
10. **Deploy** — backend to Render/Railway, frontend to Vercel, DB to MongoDB Atlas, Redis to Upstash.
11. **(Learning stretch) Kafka** — introduce a `journal.analyze` topic; API publishes an event on entry creation, a separate worker consumes it and calls the LLM, decoupling write-latency from LLM-latency. Do this last — it's optional for the core version but valuable for understanding async architectures.
12. **(Learning stretch) Replica sets & sharding** — MongoDB Atlas gives you a 3-node replica set by default even on the free tier; read up on how primary/secondary failover works. Sharding only matters at a much bigger data size, so treat it as a documented "here's how I'd scale this further" section in `ARCHITECTURE.md` rather than something you need to implement now.

Building in this order means you always have something *runnable* at every step, which keeps momentum up and makes the project easy to demo at any point.

---

## 11. Design Rationale

- **Why embed `analysis` inside `journalEntries` instead of a separate collection?** It's a one-to-one relationship — one entry has exactly one analysis. Embedding means one query returns both, no join needed.
- **Why cache in Redis instead of just relying on MongoDB?** Sub-millisecond reads for repeated identical text, and it takes load off both MongoDB and OpenRouter (cost + latency).
- **Why Kafka instead of just calling the LLM inline?** At scale, LLM calls are the slowest and least reliable part of the request — decoupling means a slow/failed LLM call never blocks the user's journal save, and you can retry/backoff independently.
- **Why JWT over sessions?** Stateless auth scales horizontally without sticky sessions or a shared session store.
- **Why MongoDB over a relational database here?** You're actively learning MongoDB (replica sets, sharding), and the data shape — a journal entry with an embedded analysis result — maps naturally onto a document rather than needing a join across tables.

See `ARCHITECTURE.md` for the full scaling / cost / caching / security details.

---

## 12. Environment Variables (`.env.example`)

```
MONGODB_URI=mongodb+srv://user:password@cluster.mongodb.net/mindgrove
OPENROUTER_API_KEY=sk-or-xxxxxxxx
JWT_SECRET=change-me
REDIS_URL=redis://localhost:6379
PORT=5000
NODE_ENV=development
```

## 13. License
MIT — build on it freely.
