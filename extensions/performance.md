# Extension: Performance

> Load this extension when the project has explicit scale targets or performance requirements.
> Do NOT load for early-stage prototypes where scale is not yet a concern.

---

## WHEN TO USE THIS

Use this extension ONLY if:
- PRD scale target exceeds 10,000 DAU at 18 months
- Response time requirement is stated (e.g. < 200ms, real-time)
- Write volume is high (> 1,000 writes/minute)
- Background processing of large datasets is required

Skip if:
- Project is early-stage with < 5,000 expected users
- No performance requirement has been stated
- Scale concerns are hypothetical, not driven by actual targets in the PRD

---

## ACTIVATION CHECK

```
IF any of the following are true:
  - PRD scale target > 10,000 DAU at 18 months
  - Stated requirement: response time < 200ms
  - Stated requirement: real-time or near-real-time (< 500ms)
  - High write volume (> 1,000 writes/minute)
  - Background processing of large datasets
  → This extension is active. Apply all rules below.

IF project is early-stage AND scale target < 5,000 DAU:
  → Do NOT load this extension.
  → Respond: "Performance optimization is premature at this scale.
    Build correctly first. Optimize when you have measured a bottleneck."
```

---

## RULE ZERO — Measure Before Optimizing

```
NEVER optimize without a measurement.
NEVER add complexity "just in case."

IF there is no benchmark showing a problem:
  → Do NOT add caching, queues, read replicas, or partitioning.
  → Build the simple version. Measure. Then optimize what is slow.

"Premature optimization is the root of all evil." — Knuth
```

---

## SECTION 1 — When to Optimize (Triggers)

```
Trigger: Response time > 500ms on any user-facing endpoint
  → Profile first: is it DB query? External call? CPU? Memory?
  → Fix the actual bottleneck. Do not guess.

Trigger: Database CPU > 70% sustained
  → Run EXPLAIN ANALYZE on the slowest queries
  → Add missing indexes
  → Consider read replica before anything else

Trigger: Memory usage growing over time (memory leak pattern)
  → Profile heap (Node: --inspect + Chrome DevTools / clinic.js)
  → Fix leak before adding more RAM

Trigger: Error rate spike under load
  → Check: connection pool exhaustion? Rate limit hit? Queue backed up?
  → Fix the constraint, not the symptom

Trigger: DAU crosses scale threshold from PRD
  → Revisit architecture decisions from Step 3
  → Run performance.md rules again with new scale context
```

---

## SECTION 2 — Database Optimization

### Index Rules

```
Add an index on EVERY column used in:
  - WHERE clauses
  - JOIN conditions
  - ORDER BY clauses
  - Foreign keys (Postgres does NOT auto-index FK columns)

Commands to find missing indexes:

-- Queries doing sequential scans on large tables
SELECT schemaname, tablename, seq_scan, idx_scan
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- Unused indexes (cost without benefit)
SELECT indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND schemaname = 'public';
```

### Query Optimization

```
[ ] EXPLAIN ANALYZE run on every query that touches > 10,000 rows
[ ] N+1 queries eliminated (use JOINs or data loaders, not loops)
[ ] Pagination on all list queries (cursor-based for large datasets)
[ ] SELECT * replaced with explicit column selection on hot paths
[ ] Aggregate queries (COUNT, SUM, GROUP BY) cached if called frequently
[ ] Long-running queries have statement_timeout set
```

### Connection Pool Rules

```
IF using PostgreSQL:
  Pool size = (number of CPU cores × 2) + number of spindle disks
  Typical starting point: 10–20 connections per app instance

IF pool is exhausted under load:
  → Add PgBouncer in transaction pooling mode
  → Do NOT just increase pool size (you will hit DB limits)

IF multiple app instances share one DB:
  → Total connections = instances × pool size
  → Must stay below DB max_connections (check with: SHOW max_connections;)
```

### Scaling Triggers

```
IF query time > 100ms on indexed queries:
  → Table may be too large for current indexing strategy
  → Consider: table partitioning (by date/tenant) OR archiving old data

IF single DB node is the bottleneck:
  Read-heavy (> 70% reads): → Add read replica, route reads to replica
  Write-heavy (> 500 writes/sec): → Consider sharding or write-optimized DB

IF > 1TB of data:
  → Plan data tiering: hot data in primary, cold data in archive storage
  → Implement data retention policy if not already done
```

---

## SECTION 3 — Caching Strategy

Run `decision-trees/redis.md` first. This section assumes caching is justified.

### What to Cache

```
Cache (high value):
  - Results of expensive aggregation queries (> 100ms)
  - Data fetched > 100 times/minute with low change frequency
  - External API responses with a TTL that matches data freshness
  - User session data (if stateful sessions)
  - Feature flags and config (changes rarely, read constantly)

Do NOT cache:
  - Data that changes on every request
  - Data unique to each request with no sharing potential
  - Data where stale = incorrect (financial balances, inventory counts)
  - Data you cannot define a TTL or invalidation strategy for
```

### Cache Invalidation Rules

```
Write-through (strong consistency):
  Update cache immediately on every write
  Use when: stale data is a bug (user profile, permissions)

TTL-only (eventual consistency):
  Let cache expire naturally after TTL
  Use when: slight staleness is acceptable (product catalog, blog posts)

Event-driven invalidation:
  Invalidate cache when a specific event occurs
  Use when: change events are published to a queue you control

IF cache miss rate > 50%:
  → Cache is not helping. Re-evaluate what is being cached.

IF memory pressure from cache:
  → Set maxmemory-policy: allkeys-lru
  → Review TTLs — reduce for low-value items
```

---

## SECTION 4 — Async and Queue Optimization

```
IF background jobs are slow:
  → Profile: is it DB query? External API? CPU? File I/O?
  → Fix the bottleneck, not the symptom

IF queue is backing up:
  → Add more workers (horizontal scaling of workers, not the API)
  → Check: are workers blocked waiting for slow external calls?
  → Add timeout to every job step

IF jobs are failing and retrying:
  → Check DLQ — how many messages, what error?
  → Fix the root cause before adding more retry attempts

IF job volume grows:
  < 10K/hr:  → BullMQ with 2–4 workers
  10K–100K/hr: → BullMQ with autoscaling workers OR switch to Kafka
  > 100K/hr: → Kafka (run decision-trees/kafka.md)
```

---

## SECTION 5 — Horizontal Scaling Rules

```
Before scaling horizontally, verify:
[ ] App is stateless (no in-process state shared between requests)
[ ] Sessions are in Redis or DB (not in-memory)
[ ] File uploads go to S3 (not local disk — local disk is not shared)
[ ] Rate limiting is in Redis (not in-memory per instance)
[ ] Cron jobs run once (not once per instance — use distributed locking)
[ ] WebSocket connections route to the same instance (sticky sessions or pub/sub)

IF any of the above are NOT met:
  → Do NOT add instances yet — you will get inconsistent behavior.
  → Fix the statefulness problem first.
```

### Scaling Thresholds

```
| Signal                          | Action                                   |
|---|---|
| CPU > 70% sustained (5+ min)    | Scale out (add instances)                |
| Memory > 85% sustained          | Scale up (larger instance) OR fix leak   |
| DB connections exhausted        | Add PgBouncer, then read replica         |
| Cache hit rate < 50%            | Re-evaluate cache strategy               |
| Queue lag > 1,000 messages      | Add workers                              |
| p99 latency > 3× baseline       | Profile and find bottleneck              |
```

---

## SECTION 6 — Load Testing (Required Before Launch)

```
IF scale target > 5,000 DAU:
  → Load test required before production launch

Minimum load test:
  - Simulate: expected peak concurrent users
  - Duration: 10 minutes sustained
  - Measure: p50, p95, p99 response times; error rate; DB CPU; memory
  - Pass criteria: p99 < 2s AND error rate < 0.1% at peak load

Tools:
  - k6 (JavaScript, scriptable, free)
  - Artillery (YAML config, Node.js)
  - Locust (Python, good for complex scenarios)

IF load test fails:
  → Profile the bottleneck (use APM or EXPLAIN ANALYZE)
  → Fix one thing at a time
  → Re-run load test after each fix
  → Do NOT launch until pass criteria are met
```

---

## SECTION 7 — Performance Budget

Define these before writing code. Enforce them before launch.

```
| Metric                     | Target            | How to measure              |
|---|---|---|
| API p99 response time       | < 500ms           | APM / load test             |
| API p50 response time       | < 100ms           | APM / load test             |
| DB query time (slow log)    | < 100ms           | pg slow query log           |
| Frontend bundle size (gzip) | < 200KB initial   | webpack-bundle-analyzer     |
| Frontend LCP                | < 2.5s            | Lighthouse / Core Web Vitals|
| Error rate                  | < 0.1%            | Error tracking tool         |
| Cache hit rate              | > 80%             | Redis INFO stats            |
```

```
IF any target is not met before launch:
  → Treat it as a CRITICAL issue (not a backlog item)
  → Fix or explicitly accept the risk in writing
```

---

## EXTENSION COMPLETE

Apply these rules during:
- Architecture step (inform caching and scaling decisions)
- Technology decisions step (validate Redis / Kafka choices against thresholds)
- Code generation step (query optimization, connection pooling, async patterns)
- Pre-launch checklist (load test, performance budget verification)
