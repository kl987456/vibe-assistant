# Quick Mode

> ⚠️ Quick Mode is NOT for production systems. Use for prototypes, internal tools, and personal projects only.

> Use when speed is needed but basic safety must be maintained.
> Quick Mode is NOT a shortcut past security. It is a shortcut past deep architecture planning.

---

## WHEN TO USE QUICK MODE

```
IF: project is a prototype / internal tool / personal project
AND: time is constrained
AND: user explicitly says "use quick mode"
THEN: activate Quick Mode

DO NOT use Quick Mode for:
  × Production systems handling real user data
  × Financial or healthcare applications
  × Apps with PII
  × Any system with > 100 real users
```

---

## QUICK MODE FLOW

### Step 1 — Ask 5 Critical Questions Only

```
1. Who uses this? (internal / public / developers only)
2. What is the single core action?
3. What scale? (users at launch and in 6 months)
4. Is there any sensitive data? (yes/no — if yes, stop Quick Mode)
5. What platform? (web / API / mobile)
```

Wait for all 5 answers. Do not proceed without them.

---

### Step 2 — Generate Minimal PRD

```markdown
## Quick PRD: [Name]

**Problem:** [1 sentence]
**User:** [who]
**Core feature:** [1 feature only]
**Scale:** [launch users] → [6mo users]
**Out of scope:** [anything not the core feature]
**Assumptions:** [list "I don't know" answers as assumptions]
```

Present to user. Get a "looks good" before continuing.

---

### Step 3 — Skip Deep Architecture

```
Make these decisions with no discussion:

IF scale < 5,000 users:
  → REST API + PostgreSQL + no Redis + no Kafka
  → Deploy to Railway or Render
  → No read replicas, no caching, no queues

IF scale 5,000–50,000 users:
  → Add Redis for rate limiting only
  → Discuss caching need after launch

IF scale > 50,000 users:
  → EXIT Quick Mode → use full master-flow.md
```

State the stack in one table. No discussion required.

---

### Step 4 — Run CRITICAL Security Checks Only

These 5 checks are non-negotiable even in Quick Mode:

```
[ ] All inputs validated server-side (Zod / Joi)
[ ] No secrets in code (env vars only, .env in .gitignore)
[ ] Auth on all non-public routes
[ ] Parameterized DB queries (no string concatenation)
[ ] Error responses don't expose stack traces
```

```
IF any of these 5 are not handled:
  → Flag it explicitly before generating code.
  → Do NOT skip. These are the 5 that cause production incidents.
```

---

### Step 5 — Generate Code

Generate the full working implementation immediately after Step 4.

Order:
1. Types + DB schema
2. Core business logic
3. API routes + middleware
4. env.example

No waiting between sections. Deliver everything in one pass.

---

## QUICK MODE RULES

```
- Skip: deep architecture docs
- Skip: detailed technology decision trees (use defaults below)
- Skip: production checklist (defer to pre-launch review)
- DO NOT skip: the 5 security checks
- DO NOT skip: the 5 questions
- DO NOT skip: env vars for secrets

Default stack (Quick Mode):
  Language:    TypeScript
  Framework:   Express (Node.js) or FastAPI (Python)
  Database:    PostgreSQL
  ORM:         Drizzle (TS) or SQLAlchemy (Python)
  Validation:  Zod (TS) or Pydantic (Python)
  Auth:        JWT (httpOnly cookie)
  Deploy:      Railway or Render
  Redis:       Only if rate limiting explicitly required
  Kafka:       Never in Quick Mode
```

---

## EXITING QUICK MODE

```
IF at any point:
  - User reveals sensitive data handling
  - Scale target exceeds 50,000 users
  - Compliance requirement is mentioned (GDPR / HIPAA / SOC2)
  - Multiple services or teams are involved

→ STOP Quick Mode
→ Say: "This project needs full planning. Switching to master-flow.md."
→ Restart from master-flow.md Step 1
```

---

## QUICK MODE SUMMARY

```
Questions:    5 (not 10)
PRD:          1 page (not full template)
Architecture: Default stack, no discussion
Security:     5 critical items only
Code:         Full implementation, one pass

Time saved:   ~30 minutes vs full flow
Risk added:   Deferred production readiness (acceptable for prototypes)
Risk NOT added: Security basics — always enforced
```
