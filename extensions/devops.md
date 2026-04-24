# Extension: DevOps

> Load this extension when the project requires production deployment, CI/CD pipelines, or multi-environment infrastructure.
> Do NOT load for local-only or prototype projects.

---

## WHEN TO USE THIS

Use this extension ONLY if:
- Project needs a CI/CD pipeline (automated build, test, deploy)
- Project uses multiple environments (dev / staging / production)
- Project requires Dockerfile, container registry, or cloud deployment
- Project will be maintained by more than one person

Skip if:
- Project is a local prototype with no deployment plan
- Project is a personal script or internal one-off tool

---

## ACTIVATION CHECK

```
IF project needs: production deployment / CI/CD / multi-environment setup / infrastructure config:
  → This extension is active. Apply all rules below.

IF project is: local prototype / dev-only / no stated deployment target:
  → Do NOT load this extension.
  → Revisit when deployment becomes a requirement.
```

---

## SECTION 1 — Deployment Platform Decision

```
IF team size ≤ 3 AND DAU (18mo) < 50,000:
  → Railway or Render
  → Reason: managed infra, zero ops overhead, deploy from git in minutes
  → Cost: $5–$25/month for most apps at this scale

IF team size > 3 OR DAU (18mo) > 50,000:
  → AWS / GCP / Azure
  → Choose based on team familiarity — not theoretical superiority
  → Use managed services (RDS, ElastiCache) before running your own

IF workload is event-driven, short-lived, and stateless:
  → Serverless (Vercel Functions / Cloudflare Workers / AWS Lambda)
  → Caveat: no long-running processes, cold starts, 10s–30s timeout limits

IF compliance or data residency requirements exist:
  → Self-hosted OR cloud in specific region
  → Verify provider's compliance certifications match your requirement

IF self-hosted is required:
  → Docker Compose for single-server deployments (< 3 services)
  → Kubernetes ONLY if team has dedicated platform engineers
```

**Output:** State the deployment choice + reason in the architecture decision record.

---

## SECTION 2 — CI/CD Pipeline

Every production project must have automated build, test, and deploy.

### Minimum Pipeline (required)

```yaml
# .github/workflows/deploy.yml — example structure

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - Checkout code
      - Install dependencies
      - Run linter (fail on error)
      - Run type check (fail on error)
      - Run unit tests (fail on any failure)
      - Run integration tests (fail on any failure)

  build:
    needs: test
    steps:
      - Build Docker image
      - Tag with git SHA (not "latest")
      - Push to container registry

  deploy:
    needs: build
    environment: production
    steps:
      - Pull new image
      - Run database migrations
      - Deploy with zero-downtime strategy
      - Run smoke test against /health endpoint
      - Rollback automatically if smoke test fails
```

### CI/CD Rules

```
[ ] CI runs on every pull request — not just on merge
[ ] Merge to main is blocked if CI fails
[ ] Deployments are triggered by code merge — never manual
[ ] Image tagged with git SHA (not "latest" — "latest" makes rollbacks impossible)
[ ] Secrets stored in CI secret manager (not in workflow files)
[ ] Deployment to production requires passing staging deployment first
[ ] Rollback command documented and tested (not hypothetical)
```

---

## SECTION 3 — Environment Separation

Every production project needs at least two environments. Three is standard.

```
| Environment | Purpose                        | Data                          |
|---|---|---|
| Development | Local dev, fast iteration      | Fake/seeded data only         |
| Staging     | Pre-production validation      | Anonymized copy of prod data  |
| Production  | Real users, real data          | Real data                     |
```

### Environment Rules

```
[ ] Each environment has its own credentials (no shared secrets)
[ ] Each environment has its own database (no shared DB between staging and prod)
[ ] Staging config matches production config (same env vars, same services)
[ ] No feature that "works on staging" can be trusted unless staging mirrors prod
[ ] Migrations tested on staging before production deployment
[ ] Production database never touched directly — all changes via migrations
[ ] Environment name always available as NODE_ENV / APP_ENV in running code
```

```
IF staging database is a copy of production:
  → Anonymize PII fields before copying (email → hashed, names → "Test User")
  → Never copy raw production PII to staging
```

---

## SECTION 4 — Containerization

```
[ ] Dockerfile exists for every deployable service
[ ] Multi-stage build used (builder stage + minimal runtime stage)
[ ] Final image runs as non-root user
[ ] No secrets baked into Docker image (use env vars at runtime)
[ ] .dockerignore excludes: node_modules, .env, .git, test files
[ ] Health check defined in Dockerfile: HEALTHCHECK CMD curl -f /health || exit 1
[ ] Image size validated (flag if > 500MB for a typical Node/Python app)
```

**Multi-stage Dockerfile pattern (required for production):**

```dockerfile
# Stage 1: build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 2: runtime (smaller, no build tools)
FROM node:20-alpine AS runtime
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER appuser
EXPOSE 3000
HEALTHCHECK CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/index.js"]
```

---

## SECTION 5 — Monitoring Basics

```
Minimum required before production launch:

[ ] Uptime check: external ping to /health every 60 seconds
[ ] Alert channel: Slack / PagerDuty / email — somewhere actively watched
[ ] Error rate alert: trigger if 5xx rate > 1% for 5 consecutive minutes
[ ] Latency alert: trigger if p99 > 3× baseline for 5 consecutive minutes
[ ] Disk alert: trigger if usage > 80%
[ ] Memory alert: trigger if sustained > 85%

Optional (add at scale):
[ ] Distributed tracing (Datadog APM / OpenTelemetry)
[ ] Custom business metrics dashboard (signups/day, revenue, key funnel steps)
[ ] Log aggregation with search (Datadog Logs / Loki / Papertrail)
```

```
IF monitoring is not set up before launch:
  → First production incident will be discovered by a user, not by the team.
  → That is not acceptable.
```

---

## SECTION 6 — Database Operations in Production

```
[ ] Database migrations run automatically in CI/CD (before server starts)
[ ] Migration rollback procedure exists and is tested
[ ] No breaking schema changes deployed without backward compatibility period:
      Step 1: Add new column (nullable, no default required from old code)
      Step 2: Deploy code that writes to both old and new column
      Step 3: Backfill old rows
      Step 4: Deploy code that reads from new column only
      Step 5: Drop old column

[ ] Connection pooling configured (PgBouncer or built-in pool)
[ ] Read replica exists if read traffic is dominant (> 70% reads)
[ ] Automated backup verified (not just enabled — actually test restore)
[ ] Point-in-time recovery enabled for production database
```

---

## SECTION 7 — Secrets in CI/CD

```
[ ] All secrets stored in CI secret manager (GitHub Secrets / GitLab Variables)
[ ] No secrets in workflow YAML files
[ ] No secrets in Dockerfile
[ ] Secrets scoped per environment (prod secrets not accessible by staging job)
[ ] Secret rotation procedure documented
[ ] Secret access logged (who/what accessed production secrets)
```

---

## EXTENSION COMPLETE

Apply these rules during:
- Architecture step (deployment platform decision)
- Environment & credentials step (per-environment config)
- Code generation step (Dockerfile, CI config)
- Pre-launch checklist (monitoring, pipeline verification)
