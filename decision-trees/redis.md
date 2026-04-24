# Redis Decision Tree

> Run this tree before adding Redis to any architecture.
> Redis is a tool, not a default. Justify it or skip it.

---

## ENTRY POINT

```
Does the project have ANY of the following needs?
  A. Caching API responses or database queries
  B. Rate limiting API endpoints
  C. Session storage (auth tokens, user sessions)
  D. Real-time leaderboards or sorted data
  E. Pub/Sub messaging between services
  F. Distributed locks
  G. Job queues (BullMQ, etc.)

IF none of A–G apply:
  → DO NOT add Redis.
  → Revisit at 1,000 DAU or when a specific need arises.

IF any of A–G apply:
  → Continue to the relevant section below.
```

---

## A — CACHING

### When to use Redis for caching

```
IF query execution time > 100ms AND data changes < once per minute:
  → Cache in Redis. Set TTL = expected data freshness window.

IF same query is executed > 100 times/minute:
  → Cache in Redis regardless of query speed.

IF data is: user profile, config, feature flags, product catalog:
  → Cache in Redis. TTL = 5–60 minutes depending on update frequency.

IF data is: user's own real-time data (cart, drafts):
  → Cache carefully. Invalidate on every write. Or skip caching.
```

### When NOT to use Redis for caching

```
IF data changes on every request:
  → Caching adds complexity with zero benefit. Skip.

IF data is user-specific AND high-churn:
  → Cache cost (memory, invalidation complexity) > benefit. Skip.

IF total dataset < 10MB:
  → Postgres in-memory query cache handles this. Skip Redis.

IF you can't define a TTL or invalidation strategy:
  → Do not cache until you can. Stale data is a bug, not a feature.
```

### TTL Strategy

```
| Data type              | Recommended TTL      |
|------------------------|----------------------|
| Static config / flags  | 1 hour               |
| Product catalog        | 15–30 minutes        |
| User profile           | 5–10 minutes         |
| API aggregation result | 30–60 seconds        |
| Rate limit window      | Matches window (60s) |
| Auth session           | Matches token expiry |
| Search results         | 2–5 minutes          |
```

### Cache Key Convention

```
Format: [project]:[entity]:[id]:[variant]
Example: myapp:user:usr_abc123:profile
Example: myapp:product:prod_xyz:list:page2

Rules:
- Always namespace by project (prevents collision in shared Redis)
- Include version suffix when schema changes: myapp:user:usr_abc123:profile:v2
- Never use user input directly in cache keys (sanitize first)
```

### Invalidation Rules

```
Write-through:   Update cache immediately on DB write (strong consistency)
Write-behind:    Update DB first, invalidate cache (eventual consistency)
TTL-only:        Let cache expire naturally (acceptable for low-criticality data)

IF data is financial or transactional:
  → Write-through or TTL = 0 (don't cache at all)

IF cache miss causes visible UX degradation:
  → Implement cache warming on startup
```

---

## B — RATE LIMITING

### When to use Redis for rate limiting

```
IF app runs on > 1 server instance:
  → Must use Redis for rate limiting (in-memory counters don't work across instances)

IF rate limiting needs to be enforced globally:
  → Redis sliding window counter is the correct approach

IF rate limits must survive server restarts:
  → Redis (in-memory per-process counters reset on restart)
```

### When NOT to use Redis for rate limiting

```
IF app runs on a single instance AND rate limits can reset on restart:
  → In-memory rate limiter is acceptable (simpler)

IF rate limiting is IP-based at high scale (> 10K RPS):
  → Push rate limiting to CDN layer (Cloudflare / AWS WAF) — Redis can't handle this
```

### Rate Limit Patterns

```
Fixed Window:
  - Simple counter per time window
  - Problem: burst allowed at window boundary
  - Use for: internal tools, low-sensitivity limits

Sliding Window (recommended):
  - Accurate, no boundary burst
  - Slightly more memory-intensive
  - Use for: public APIs, auth endpoints

Token Bucket:
  - Allows controlled bursting
  - Use for: APIs with bursty-but-bounded legitimate traffic

Leaky Bucket:
  - Smooths out burst, queues excess
  - Use for: downstream protection
```

### Rate Limit Key Structure

```
Format: rl:[endpoint]:[identifier]:[window]
Example: rl:auth_login:ip_1.2.3.4:60s
Example: rl:api_v1:user_usr_abc123:60s

Limits (sensible defaults — adjust per requirement):
  Auth endpoints:     5 requests / 60 seconds per IP
  Public API:         60 requests / 60 seconds per user
  Internal API:       500 requests / 60 seconds per service
  Password reset:     3 requests / 3600 seconds per email
```

---

## C — SESSION STORAGE

### When to use Redis for sessions

```
IF using stateful sessions (not JWT):
  IF single server:   → OK to use in-memory or DB sessions
  IF multiple servers → MUST use Redis (session must be shared)

IF using JWT + refresh tokens:
  → Store refresh token metadata in Redis (fast lookup, easy revocation)
  → TTL = refresh token lifetime

IF token revocation must be instant (security-critical apps):
  → Store revoked token IDs in Redis with TTL = token expiry
  → Check on every authenticated request
```

---

## D — SORTED SETS / LEADERBOARDS

### When to use Redis Sorted Sets

```
IF need: real-time leaderboard → ZSET (score = points, member = user_id)
IF need: priority queue        → ZSET (score = priority, member = job_id)
IF need: time-windowed ranking → ZSET with TTL-based cleanup

IF leaderboard updates < once/second AND total entries < 10,000:
  → Postgres ORDER BY with index is acceptable. Skip Redis.

IF leaderboard updates > 10/second OR total entries > 100,000:
  → Redis ZSET is the correct choice.
```

---

## E — PUB/SUB

### When to use Redis Pub/Sub

```
IF messages are: fire-and-forget, at-most-once delivery acceptable:
  → Redis Pub/Sub is fine

IF messages are: critical, must not be lost, need delivery guarantee:
  → DO NOT use Redis Pub/Sub → Use Kafka or Redis Streams instead

IF consumers are: short-lived or dynamic:
  → Redis Pub/Sub (no persistent subscription needed)
```

---

## F — DISTRIBUTED LOCKS

```
IF concurrent operations must be mutually exclusive AND multiple servers:
  → Redis SET NX EX (Redlock pattern for multi-node Redis)

IF lock failure must not corrupt data:
  → Use Redlock with retry + jitter
  → Always set lock TTL (never infinite locks)
  → Always release lock in finally block

IF single-server deployment:
  → DB-level advisory locks (Postgres pg_advisory_lock) are simpler
```

---

## G — JOB QUEUES

```
IF using BullMQ or similar:
  → Redis is required as the backing store
  → Use separate Redis instance or database number for queues (don't mix with cache)
  → Enable Redis persistence (AOF) for queues — job loss is a bug

IF job volume < 100/hour:
  → Consider Postgres-backed queue (pg-boss) — no extra infrastructure

IF job volume > 10,000/hour AND complex routing:
  → Consider Kafka instead
```

---

## REDIS OPERATIONAL RULES

```
[ ] Never use Redis as primary/only database for critical data
[ ] Enable Redis persistence (RDB snapshots minimum; AOF for queues)
[ ] Set maxmemory policy: allkeys-lru for caches, noeviction for queues
[ ] Monitor: memory usage, eviction rate, hit/miss ratio, latency
[ ] Use Redis Cluster IF single-node memory > 25GB or RPS > 100K
[ ] Set connection pool limits — unbounded connections will crash Redis
[ ] Separate Redis instances (or DBs) for: cache / sessions / queues
[ ] All Redis keys must have TTL (except persistent data like queues)
```

---

## DECISION SUMMARY

```
Need caching?              → Redis (IF > 100 req/min or > 100ms queries)
Need rate limiting?        → Redis (IF multi-server deployment)
Need session storage?      → Redis (IF multi-server OR instant revocation)
Need leaderboards?         → Redis (IF > 10 updates/sec or > 100K entries)
Need pub/sub?              → Redis (IF at-most-once acceptable)
Need distributed locks?    → Redis (IF multi-server)
Need job queues?           → Redis/BullMQ (IF < 10K/hr); Kafka (IF > 10K/hr)
None of the above?         → DO NOT add Redis
```

---

## FINAL DECISION SUMMARY

**Use Redis IF:**
- caching repeated queries (same query > 100×/min or > 100ms)
- rate limiting is required across multiple server instances
- multiple server instances share session or auth state

**Avoid Redis IF:**
- traffic is low (< 1,000 DAU) — database query cache is sufficient
- no performance bottlenecks have been measured
- data changes on every request — caching adds complexity with zero gain
