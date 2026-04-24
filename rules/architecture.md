# Architecture Design Rules

> Define the system structure before any code is written.
> Every layer must have a justification. "It's common" is not a justification.

---

## RULE

```
IF architecture has an unjustified component:
  → Flag it: "Why [component]? What requirement drives this?"
  → Remove unjustified components (YAGNI).
```

---

## LAYER 1 — Client Layer

**Decision:**

```
IF web app needed:
  → Choose: SPA (React/Vue) OR SSR (Next.js/Nuxt) OR static (Astro)

  IF SEO is required           → SSR
  IF auth-gated (no SEO need)  → SPA
  IF mostly static content     → Static site generator

IF mobile app needed:
  → Choose: Native (Swift/Kotlin) OR cross-platform (React Native/Flutter)

  IF budget-constrained          → React Native / Flutter
  IF performance-critical native → Native per platform

IF API-only (no UI):
  → No frontend. Proceed to API layer.
```

---

## LAYER 2 — API Layer

**Decision:**

```
IF clients are: web + mobile + potentially third-party
  → REST (default choice — widest compatibility)

IF data requirements are: complex, nested, client-driven queries
  → GraphQL (accept added complexity cost)

IF internal service-to-service only AND latency-critical
  → gRPC

IF real-time bidirectional communication required
  → WebSocket (add alongside REST, not instead)
```

**API Design Rules:**

```
- Version all APIs from day 1: /api/v1/...
- Paginate all list endpoints (default: cursor-based)
- Return consistent error envelope:
  { error: { code: string, message: string, details?: object } }
- Use HTTP status codes correctly (don't return 200 with error body)
- All endpoints must be authenticated by default; whitelist public ones
```

---

## LAYER 3 — Auth Layer

**Decision:**

```
IF B2C app (users sign up themselves):
  → JWT (stateless, scalable) + refresh token rotation
  → Consider: social OAuth (Google / GitHub) via Auth.js or Clerk

IF B2B app (org-based access):
  → JWT + RBAC (Role-Based Access Control)
  → Consider: Clerk or Auth0 for SSO support

IF internal tooling only:
  → Session-based auth (simpler, fine for low scale)
  → OR SSO via existing company IdP

IF API-only, machine-to-machine:
  → API keys with prefix (sk_live_, sk_test_) + hashed storage
  → Scoped permissions per key
```

**Auth Rules:**

```
- NEVER store plaintext passwords (bcrypt / argon2, cost factor ≥ 12)
- NEVER store tokens in localStorage (use httpOnly cookies)
- Rotate refresh tokens on every use
- Short-lived access tokens (15 min default)
- Invalidate all sessions on password change
```

---

## LAYER 4 — Database Layer

**Decision:**

```
IF data is: relational, structured, consistency-critical
  → PostgreSQL (default for almost everything)

IF data is: document-like, schema-flexible, team familiar with Mongo
  → MongoDB (but define schemas anyway — Mongoose or Zod)

IF data is: time-series (metrics, logs, sensor data)
  → TimescaleDB (Postgres extension) OR InfluxDB

IF data is: graph (social networks, recommendation engines)
  → Neo4j OR PostgreSQL with recursive CTEs (for simpler cases)

IF scale > 1M rows per table AND read-heavy:
  → Add read replicas
  → Add indexes on all foreign keys and filter columns
  → Plan query analysis before launch (EXPLAIN ANALYZE)
```

**Database Rules:**

```
- Always use migrations (never manual schema changes in production)
- Foreign keys must be enforced at DB level, not just app level
- Soft deletes (deleted_at timestamp) for user-facing data
- created_at + updated_at on every table
- UUIDs for primary keys (not sequential integers — prevents enumeration)
- Connection pooling required (PgBouncer or built-in pool)
```

---

## LAYER 5 — Caching Layer

**Run decision tree:** `decision-trees/redis.md`

**Default rule:**

```
IF scale < 1,000 DAU AND no rate limiting needed:
  → Skip Redis. Use database + query optimization.
  → Re-evaluate at 1,000 DAU.

IF scale ≥ 1,000 DAU OR rate limiting needed:
  → Introduce Redis. Use decision tree.
```

---

## LAYER 6 — Async / Queue Layer

**Run decision tree:** `decision-trees/kafka.md`

**Default rule:**

```
IF async jobs are: simple, low-volume, delay-tolerant
  → Redis queues (BullMQ) — simpler, no new infrastructure

IF async jobs are: high-volume, require replay, multi-consumer
  → Kafka — accept operational overhead

IF async jobs are: scheduled (cron-like), simple
  → pg-boss (Postgres-backed) OR Inngest — no new infrastructure
```

---

## LAYER 7 — File Storage

**Decision:**

```
IF file uploads required:
  IF cloud-deployed  → S3 (or compatible: R2, MinIO)
  IF self-hosted     → MinIO

  Rules:
  - Never store files in the database (not even small ones)
  - Generate pre-signed URLs — never expose storage credentials to client
  - Validate file type server-side (not just extension — check MIME)
  - Set max file size at both API layer AND storage layer
  - Virus-scan uploads if sensitive (ClamAV or cloud equivalent)
```

---

## LAYER 8 — Deployment

**Decision:**

```
IF team size ≤ 3 AND scale < 50K MAU:
  → Railway / Render / Fly.io (managed, low ops burden)

IF team size > 3 OR scale > 50K MAU:
  → AWS / GCP / Azure with container orchestration

IF serverless is preferred:
  → Vercel (frontend) + AWS Lambda / Cloudflare Workers (API)
  → Caveat: cold starts, stateless only, no long-running processes

IF self-hosted required:
  → Docker Compose (small) OR Kubernetes (scale)
```

**Deployment Rules:**

```
- Containerize everything (Dockerfile required)
- Never run as root in containers
- Secrets via environment variables (never baked into image)
- Health check endpoint: GET /health → { status: "ok", version: string }
- Separate environments: local / staging / production
- Staging must mirror production config (no "it worked on staging" excuses)
```

---

## DECISION CONFIDENCE

For every architecture decision, state:

```
| Layer       | Choice     | Confidence        | Risk if wrong                              |
|---|---|---|---|
| Client      | [choice]   | High/Medium/Low   | [what breaks if this is the wrong call]    |
| API         | [choice]   | High/Medium/Low   | [what breaks]                              |
| Database    | [choice]   | High/Medium/Low   | [what breaks]                              |
| Cache       | [choice]   | High/Medium/Low   | [what breaks]                              |
| Queue       | [choice]   | High/Medium/Low   | [what breaks]                              |
```

Confidence levels:
- **High** — decision is clear from requirements; alternatives have obvious drawbacks
- **Medium** — decision is reasonable but assumptions may not hold (flag the assumption)
- **Low** — genuine uncertainty; state the tradeoff and document the trigger to revisit

```
IF any decision is Low confidence:
  → State: "This is a Low confidence decision because [reason]."
  → State: "We should revisit if [condition] changes."
  → Record as an open question in the PRD.
```

---

## ARCHITECTURE OUTPUT FORMAT

After completing all decisions, produce this output:

```
## Architecture Decision Record

### System: [Project Name]
### Date: [Date]
### Scale target: [from PRD]

---

Client Layer:    [choice] — [reason] — Confidence: [High/Medium/Low]
API Layer:       [choice] — [reason] — Confidence: [High/Medium/Low]
Auth:            [choice] — [reason] — Confidence: [High/Medium/Low]
Primary DB:      [choice] — [reason] — Confidence: [High/Medium/Low]
Cache:           [choice] — [reason] | SKIPPED — [reason]
Queue:           [choice] — [reason] | SKIPPED — [reason]
File Storage:    [choice] — [reason] | NOT NEEDED
Deployment:      [choice] — [reason] — Confidence: [High/Medium/Low]

---

## Text Architecture Diagram

[Client: React SPA]
      |
      | HTTPS
      v
[API Gateway / Load Balancer]
      |
      v
[Express API — /api/v1]
  |         |         |
Auth MW  Rate Limit  Validation
      |
      +——→ [PostgreSQL — primary]
      |         |
      |    [Read replica]
      |
      +——→ [Redis — cache + sessions]
      |
      +——→ [BullMQ — job queue]
                |
           [Worker processes]
```

---

## ARCHITECTURE ANTI-PATTERNS (forbidden)

```
× Microservices for a new product with < 3 engineers — use monolith first
× Multiple databases for MVP — one source of truth
× Redis as primary database — it is a cache, not a store
× Syncing files through API (base64 in JSON) — use pre-signed URLs
× In-memory session state on multi-instance deployment — use Redis or DB
× HTTP between services without timeout and retry — always add both
```

---

## COMMON MISTAKES

These are the most frequent architecture errors in AI-generated systems:

- **Adding Redis without a measured need** — adds ops overhead for zero benefit at low traffic; run the Redis decision tree first
- **Adding Kafka too early** — Kafka is operationally expensive; BullMQ handles 95% of async needs without a broker
- **Overusing microservices** — a monolith ships faster and is easier to debug; split only when team ownership or scale forces it
- **Skipping database indexing** — queries that work in development destroy performance at scale; index every foreign key and filter column before launch
- **No connection pooling** — every new request opening a DB connection will exhaust limits under load; PgBouncer or built-in pooling is required
- **Shared staging/production config** — "it worked on staging" means nothing if staging uses different secrets, different DB size, or different services
