# ARCHITECTURE.md — MindGrove

This document answers the four required architecture questions, plus the extra advanced-topic design (Redis, Kafka, CI/CD, multi-user) the project roadmap adds on top of the MVP.

---

## 1. How would you scale this to 100k users?

**Stateless API layer.** Express servers hold no session state (JWT auth is stateless), so you can run N identical instances behind a load balancer (Nginx / a managed LB on Render/Railway) and scale horizontally just by adding instances.

**Database.**
- Add read replicas for PostgreSQL once read traffic (viewing entries, insights) outpaces writes — journal apps are read-heavy (users check insights more than they write).
- Index `user_id` on `journal_entries` and `entry_id` on `analysis_results` (already in the schema) so per-user queries stay fast as row count grows.
- Partition `journal_entries` by time (monthly) once a single table crosses tens of millions of rows — most queries only touch recent data anyway.

**Async by default.** Move the LLM call off the request/response path using Kafka (see Q3). This is the single biggest scaling win: LLM latency (1–5s) is the slowest part of the system, and at 100k users you cannot afford to hold an HTTP connection open for every journal write while waiting on OpenRouter.

**CDN + static frontend.** React frontend is static after build — serve it from Vercel/CDN edge, so frontend traffic never touches your backend fleet.

**Connection pooling.** Use PgBouncer or a connection pool (e.g. `pg-pool`) so 100k users' worth of concurrent requests don't exhaust Postgres's connection limit.

**Observability.** At this scale you need metrics (Prometheus/Grafana) and structured logs before you need more servers — most scaling problems show up as "one slow query" or "one hot Redis key," not "not enough boxes."

---

## 2. How would you reduce LLM cost?

- **Cache aggressively (see Q3)** — many journal entries express similar sentiments; a cache hit costs nothing.
- **Use free/cheap OpenRouter models** for the bulk of traffic (e.g. `mistralai/mistral-7b-instruct:free` or similar free-tier models) and reserve a stronger paid model only for cases where the cheap model's output fails validation.
- **Batch where possible.** If insights are recomputed periodically rather than on every request, you can batch multiple entries into fewer, larger LLM calls instead of one call per entry.
- **Short, constrained prompts.** Ask for JSON-only output with a strict schema and low `max_tokens` — most of the cost is output tokens, so capping response length directly caps cost.
- **Deduplicate near-identical input.** Hash normalized text (lowercased, whitespace-collapsed) before calling the LLM — many entries across users ("felt calm", "felt relaxed") can share a cache entry if you fuzzy-match on emotion category rather than exact text.
- **Rate limit per user** so a single abusive user can't run up unbounded LLM cost (also protects against basic DoS).

---

## 3. How would you cache repeated analysis?

- **Redis, key = hash of normalized journal text** (e.g. SHA-256 of lowercased/trimmed text), **value = the JSON analysis result**, TTL ~24h–7d.
- Flow: `/api/journal/analyze` → compute hash → `GET` from Redis → **hit**: return cached JSON immediately (no LLM call, no cost) → **miss**: call OpenRouter → store result in Redis with `SET key value EX <ttl>` → also persist to `analysis_results` in Postgres as permanent history.
- Redis is the *fast, ephemeral* cache; Postgres is the *durable* record — Redis can be flushed/rebuilt without losing data, since Postgres remains the source of truth.
- For the insights endpoint, cache the aggregated result per user (`insights:<userId>`) with a short TTL (e.g. 5 minutes) or invalidate it on new entry/analysis writes, since insights are a rollup that doesn't need to be perfectly real-time.

---

## 4. How would you protect sensitive journal data?

Journal entries are inherently sensitive personal/mental-health-adjacent data, so this gets more care than a typical CRUD field:

- **Encryption in transit:** HTTPS/TLS everywhere (enforced by the hosting platform — Render/Vercel do this by default).
- **Encryption at rest:** enable Postgres disk encryption (managed providers like Supabase/Neon do this by default); consider application-level encryption of the `text` column (e.g. AES-256, key from a secrets manager) for an extra layer beyond disk encryption.
- **Auth & isolation:** JWT-authenticated requests; every query for entries/insights is scoped by the authenticated `user_id` from the token — never trust a `userId` passed in the request body/path directly (the MVP spec's plain `userId` field is fine for grading, but in a real multi-user system it must come from a verified token, not client input).
- **Least-privilege DB access:** the API's DB user should only have the grants it needs (no `DROP`/`ALTER` in production).
- **Secrets management:** API keys (OpenRouter, JWT secret, DB URL) live in environment variables / a secrets manager (Render/Railway secrets, GitHub Actions secrets for CI) — never committed to the repo.
- **Data minimization with the LLM provider:** send only the journal text needed for analysis, not user identifiers, to OpenRouter — the LLM call should be anonymous with respect to who the user is.
- **Audit/delete:** support a "delete my data" endpoint (GDPR-style) that cascades through `journal_entries` → `analysis_results` (already set up via `ON DELETE CASCADE` in the schema).
- **Rate limiting & abuse prevention:** prevents bulk scraping of entries via credential stuffing or brute-forced IDs.

---

## 5. Advanced-topic design (learning goals beyond the MVP)

### Redis
Two independent uses, don't conflate them:
1. **Cache** for `/analyze` results and `/insights` rollups (see Q3).
2. **Rate limiting** — a sliding-window or token-bucket counter per user/IP (`INCR` + `EXPIRE`), rejecting requests once a threshold is hit (e.g. 10 analyze calls/minute).

### Kafka
Introduce one topic, `journal.analyze.requested`:
- `POST /api/journal` writes the entry to Postgres **and** publishes an event `{ entryId, text }` to the topic — the HTTP response returns immediately after the DB write, without waiting on the LLM.
- A separate **worker process** (Node consumer or a FastAPI microservice) subscribes to the topic, calls OpenRouter, writes the result to `analysis_results`, and (optionally) pushes a websocket/SSE update to the frontend so the UI updates when analysis completes.
- This is the architecture that lets the system stay responsive even if OpenRouter is slow or briefly down — the queue absorbs the backlog and retries, rather than the user's request hanging or failing.
- While learning, Redis Streams or BullMQ (Redis-backed job queue) is a lighter-weight substitute for Kafka with the same conceptual shape — worth building first before introducing full Kafka/Zookeeper operational overhead.

### CI/CD (GitHub Actions)
Pipeline: on every push → `install deps` → `lint` → `run tests` → `build` → (on `main` branch only) `deploy backend to Render` + `deploy frontend to Vercel`. Keep secrets (API keys, deploy tokens) in GitHub Actions encrypted secrets, never in the workflow file.

### Multi-user readiness
- Every table keyed by `user_id`, every query scoped by the authenticated user from JWT.
- Horizontal scaling of the API tier is safe *because* there's no server-side session state — any instance can serve any user's request.
- Insights/aggregation queries use `GROUP BY` in SQL rather than in-app loops, so per-user cost stays roughly constant regardless of total user count.

---

## 6. Summary Diagram (request lifecycle with all advanced pieces active)

```
User writes entry
   → POST /api/journal (JWT-authenticated)
       → write row to journal_entries (fast, <50ms)
       → publish event to Kafka topic "journal.analyze.requested"
       → return 201 to frontend immediately

Analysis Worker (separate process)
   → consumes event from Kafka
   → checks Redis cache by text-hash
       → HIT: use cached result
       → MISS: call OpenRouter → cache result in Redis → persist to analysis_results
   → (optional) push update to frontend via SSE/WebSocket

User views insights
   → GET /api/journal/insights/:userId (JWT-authenticated)
       → check Redis cache "insights:<userId>"
       → HIT: return cached rollup
       → MISS: run SQL aggregation over analysis_results → cache → return
```
