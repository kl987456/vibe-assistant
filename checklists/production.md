# Production Readiness Checklist

> A system is not ready for production until every CRITICAL item is checked.
> "It works on my machine" is not production readiness.

---

## HOW TO USE

```
For each item, mark:
  [x] DONE       — implemented and verified
  [~] DEFERRED   — not in v1, has a backlog ticket, date set
  [N] N/A        — does not apply (explain why inline)

Gate rule:
  IF any CRITICAL item is unchecked:
    → Do not deploy to production.
    → Do not declare the feature complete.
```

---

## SECTION 1 — OBSERVABILITY / LOGGING

- [ ] CRITICAL — Structured logging enabled (JSON format, not plain strings)
- [ ] CRITICAL — Log levels used correctly: ERROR / WARN / INFO / DEBUG
- [ ] CRITICAL — Errors logged with: message, stack trace, request ID, user ID (masked)
- [ ] CRITICAL — Request logging: method, path, status code, duration, request ID
- [ ] CRITICAL — Request ID generated and propagated through all log lines per request
- [ ] CRITICAL — External call logging: service name, endpoint, duration, success/fail
- [ ] CRITICAL — Background job logging: job type, job ID, start, end, success/fail
- [ ] HIGH     — Log shipping to centralized system (Datadog / Loki / CloudWatch / Papertrail)
- [ ] HIGH     — PII masked in logs before shipping (emails, names, tokens)
- [ ] HIGH     — Log retention policy set (≥ 30 days for production logs)
- [ ] MEDIUM   — Audit log for sensitive actions (separate from app log, longer retention)
- [ ] MEDIUM   — Log sampling configured for high-volume routes (avoid log flood)

---

## SECTION 2 — ERROR HANDLING

- [ ] CRITICAL — Global error handler catches all unhandled exceptions
- [ ] CRITICAL — Unhandled Promise rejections caught globally (not silently swallowed)
- [ ] CRITICAL — All async operations wrapped in try/catch (no floating Promises)
- [ ] CRITICAL — API errors return consistent error envelope (never raw stack traces)
- [ ] CRITICAL — Error handler sends alert/notification for 5xx errors
- [ ] HIGH     — Errors classified: expected (4xx) vs unexpected (5xx)
- [ ] HIGH     — Circuit breaker implemented for external service calls
- [ ] HIGH     — Graceful degradation: if optional external service fails, app continues
- [ ] MEDIUM   — Error rate tracked as a metric (not just logged)
- [ ] MEDIUM   — Error tracking tool integrated (Sentry / Bugsnag / Rollbar)
- [ ] MEDIUM   — Source maps configured so error stack traces map to source code

---

## SECTION 3 — RETRIES AND RESILIENCE

- [ ] CRITICAL — External HTTP calls have timeout set (no indefinite hanging)
- [ ] CRITICAL — External HTTP calls retry on 5xx and network errors (not on 4xx)
- [ ] HIGH     — Retry logic uses exponential backoff with jitter
- [ ] HIGH     — Max retry count set (not infinite)
- [ ] HIGH     — Retry budget defined (don't retry so aggressively you amplify incidents)
- [ ] HIGH     — Idempotency ensured for retried operations (same request = same result)
- [ ] MEDIUM   — Timeout values tuned per external service (don't use global default for all)
- [ ] MEDIUM   — Database queries have statement timeout set
- [ ] MEDIUM   — Background jobs have max execution time set (stuck job detection)

---

## SECTION 4 — MONITORING AND ALERTING

- [ ] CRITICAL — Health check endpoint: `GET /health` returns 200 with `{ status: "ok" }`
- [ ] CRITICAL — Health check verifies: DB connection, Redis connection (if applicable)
- [ ] CRITICAL — Uptime monitoring configured (checks /health every 60s minimum)
- [ ] CRITICAL — Alert on: sustained error rate > 1% for > 5 minutes
- [ ] CRITICAL — Alert on: p99 response time > 3x baseline for > 5 minutes
- [ ] HIGH     — Alert on: DLQ depth > 0 (any failed background job)
- [ ] HIGH     — Alert on: disk usage > 80%
- [ ] HIGH     — Alert on: memory usage > 85% sustained
- [ ] HIGH     — Alert on: database connection pool exhaustion
- [ ] HIGH     — Alerts go to: a channel that is actively monitored (not just email)
- [ ] MEDIUM   — Dashboard created for: RPS, error rate, p50/p95/p99 latency
- [ ] MEDIUM   — Business metrics tracked (signups, conversions, key user actions)
- [ ] MEDIUM   — Alert fatigue avoided: alerts are actionable (not informational noise)
- [ ] LOW      — Runbook linked from each alert (what to do when this fires)

---

## SECTION 5 — PERFORMANCE

- [ ] CRITICAL — Database queries tested with production-representative data volume
- [ ] CRITICAL — N+1 query problem eliminated (use joins or data loaders)
- [ ] CRITICAL — All database columns used in WHERE / JOIN / ORDER BY are indexed
- [ ] CRITICAL — No unbounded queries (all list queries are paginated)
- [ ] HIGH     — Slow query log enabled and threshold set (flag queries > 1s)
- [ ] HIGH     — Connection pool size tuned for expected concurrency
- [ ] HIGH     — Load test run against staging (simulate expected peak traffic)
- [ ] HIGH     — Memory leak check: process memory stable over 30 minutes under load
- [ ] MEDIUM   — Static assets served from CDN (not from app server)
- [ ] MEDIUM   — API response compression enabled (gzip / brotli)
- [ ] MEDIUM   — Database indexes reviewed by EXPLAIN ANALYZE on critical queries
- [ ] LOW      — Caching headers set for cacheable API responses

---

## SECTION 6 — DEPLOYMENTS

- [ ] CRITICAL — CI/CD pipeline exists (no manual deployments to production)
- [ ] CRITICAL — Automated tests run on every pull request (block merge on failure)
- [ ] CRITICAL — Production deployment is one-command or one-click (not manual steps)
- [ ] CRITICAL — Rollback procedure documented and tested (not hypothetical)
- [ ] CRITICAL — Database migrations are backward-compatible (no breaking schema changes in same deploy)
- [ ] HIGH     — Blue/green or rolling deployment (zero-downtime deploys)
- [ ] HIGH     — Environment config via environment variables (not config files in repo)
- [ ] HIGH     — Staging environment mirrors production (same config, similar data)
- [ ] HIGH     — Docker image built with non-root user
- [ ] HIGH     — Container image scanned for known CVEs before production deploy
- [ ] MEDIUM   — Deployment notifications sent to team channel (what, when, who)
- [ ] MEDIUM   — Feature flags available to disable features without redeployment

---

## SECTION 7 — DATA AND BACKUPS

- [ ] CRITICAL — Database backup strategy defined and automated
- [ ] CRITICAL — Backup restoration tested (backup that is never restored is not a backup)
- [ ] CRITICAL — Backup frequency: daily minimum; hourly for financial/critical data
- [ ] CRITICAL — Backups stored in separate region/location from primary
- [ ] HIGH     — Point-in-time recovery (PITR) enabled for production database
- [ ] HIGH     — Data retention policy defined and enforced
- [ ] HIGH     — Recovery time objective (RTO) defined: "we can recover in X minutes"
- [ ] HIGH     — Recovery point objective (RPO) defined: "we can lose at most X minutes of data"
- [ ] MEDIUM   — Backup access restricted (not accessible from app servers)

---

## SECTION 8 — CONFIGURATION

- [ ] CRITICAL — `env.example` maintained and up-to-date (every env var documented)
- [ ] CRITICAL — No default credentials in production (all secrets changed from defaults)
- [ ] CRITICAL — Debug mode / verbose error mode disabled in production
- [ ] CRITICAL — Production config separate from development config
- [ ] HIGH     — Config validation on startup (app fails fast if required env vars missing)
- [ ] HIGH     — All third-party services configured for production accounts (not sandbox/test)
- [ ] MEDIUM   — Config changes audited (who changed what and when)

---

## SECTION 9 — CAPACITY AND SCALING

- [ ] HIGH     — Horizontal scaling tested (app works with > 1 instance, no shared in-process state)
- [ ] HIGH     — Auto-scaling policy configured (scale up on CPU/RPS threshold)
- [ ] HIGH     — Rate limits documented and implemented (prevents single user exhausting resources)
- [ ] MEDIUM   — Cost alerting set (alert if monthly cloud spend > expected threshold)
- [ ] MEDIUM   — Database read replica configured if read traffic is dominant
- [ ] LOW      — Scale-down behavior tested (no stuck connections on scale-down)

---

## SECTION 10 — INCIDENT RESPONSE

- [ ] HIGH     — On-call rotation defined (who is paged for production incidents)
- [ ] HIGH     — Incident communication plan: how users are notified of outages
- [ ] HIGH     — Status page exists or is planned
- [ ] MEDIUM   — Post-mortem process defined (blameless, documented, actioned)
- [ ] MEDIUM   — Runbooks for common failures exist (DB down, cache down, queue backed up)
- [ ] LOW      — Incident severity levels defined (P0/P1/P2/P3 with response SLAs)

---

## SECTION 11 — TESTING

- [ ] CRITICAL — Unit tests for all business logic (not just happy paths)
- [ ] CRITICAL — Integration tests for all API endpoints
- [ ] HIGH     — Auth and authorization paths tested (both success and failure cases)
- [ ] HIGH     — Edge cases from PRD tested
- [ ] HIGH     — Tests run in CI on every commit
- [ ] MEDIUM   — Test coverage threshold enforced in CI (suggest ≥ 70%)
- [ ] MEDIUM   — E2E tests for critical user flows (signup, purchase, core action)
- [ ] MEDIUM   — Load test results documented and reviewed
- [ ] LOW      — Contract tests if multiple teams consume the same API

---

## SIGN-OFF

```
Date reviewed:          ___________
Reviewed by:            ___________
Target deploy date:     ___________

CRITICAL items incomplete:   _____ (must be 0)
HIGH items incomplete:       _____ (risk accepted by: _______)
DEFERRED items total:        _____ (all have ticket links)

Deployment approved:    [ ] Yes   [ ] No — blocked by: ___________
```
