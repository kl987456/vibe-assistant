# Master Execution Flow

> AI MUST follow this sequence. No step may be skipped. No code is written until Step 7.

---

## ENFORCEMENT RULE

```
IF user asks for code BEFORE steps 1–6 are complete:
  → Respond: "I need to complete the planning steps first."
  → Return to the current incomplete step.
  → Do NOT generate code.
```

---

## STEP 1 — Requirement Clarification

**Trigger:** User provides a project idea (any format).

**Action:** Ask ALL of the following questions. Do not assume answers.

### Required Questions

```
1. Who are the primary users? (internal team / consumers / businesses / developers)
2. What is the ONE core action a user must be able to do?
3. What does failure look like? (what breaks if this doesn't work?)
4. Expected users at launch?
5. Expected users at 6 months? At 18 months?
6. Is there any existing system this must integrate with?
7. What is the data sensitivity level? (public / internal / PII / financial / healthcare)
8. Is real-time response required? (< 500ms / < 2s / async is fine)
9. Are there regulatory requirements? (GDPR / HIPAA / SOC2 / none)
10. What platforms? (web / mobile / API-only / desktop)
```

**Gate:** All 10 questions must have answers before proceeding.

```
IF user does not answer a required question:
  → DO NOT assume an answer.
  → Provide 2–3 options with clear tradeoffs.
  → Force user to choose before proceeding.
  → Record chosen option as an assumption in the PRD.

IF any answer is "I don't know":
  → Provide 2–3 reasonable options with tradeoffs.
  → Ask user to choose.
  → Record chosen option as assumption.
```

---

## STEP 2 — PRD Generation

**Trigger:** Step 1 is complete (all questions answered).

**Action:** Use `rules/prd-generator.md` to produce a structured PRD.

**Required output sections:**
- Problem statement (1–2 sentences)
- User personas (named, with goals)
- Core features (MVP scope only)
- Out of scope (explicit list)
- Edge cases (minimum 3)
- Scale targets (launch / 6mo / 18mo)
- Data model sketch (entities and relationships)

**Gate:** Present PRD to user. Wait for explicit approval.

```
IF user modifies PRD:
  → Update and re-present.
  → Do NOT proceed until user says "approved" or equivalent.
```

---

## STEP 3 — Architecture Design

**Trigger:** PRD is approved.

**Action:** Use `rules/architecture.md` to design the system.

**Required decisions:**
- Client layer (web / mobile / API)
- API layer (REST / GraphQL / gRPC — with reasoning)
- Auth strategy (JWT / session / OAuth — with reasoning)
- Primary database (with reasoning)
- Caching layer (required? → use `decision-trees/redis.md`)
- Async processing (required? → use `decision-trees/kafka.md`)
- File storage (required? S3 / local / none)
- Deployment target (cloud / self-hosted / serverless)

**Output format:** Architecture diagram (text-based) + decision log.

**Gate:** Present architecture to user. Wait for explicit approval.

---

## STEP 4 — Technology Decisions

**Trigger:** Architecture is approved.

**Action:** For each technology choice in the architecture, run the appropriate decision tree.

### Decision Trees to Run

```
Caching needed?       → Run decision-trees/redis.md
Async processing?     → Run decision-trees/kafka.md
Search needed?        → Evaluate: Postgres full-text / Elasticsearch / Typesense
File uploads?         → Evaluate: S3 / Cloudinary / local disk
Background jobs?      → Evaluate: Redis queues / Kafka / cron / serverless functions
Email/SMS?            → Evaluate: Resend / SendGrid / SES / Twilio
Payments?             → Evaluate: Stripe (default) / Paddle / Braintree
```

**Output:** Technology stack table with explicit reasoning for each choice.

```
| Layer        | Technology | Why | Why NOT alternative |
|---|---|---|---|
| ...          | ...        | ... | ...                 |
```

**Gate:** User must confirm technology stack before proceeding.

---

## OPTIONAL STEP — Extensions

**Trigger:** Technology stack is confirmed.

**Action:** Scan the PRD and confirmed architecture for extension triggers. Load only what matches.

```
IF project includes a user-facing UI (web app / dashboard / widget / mobile):
  → Load extensions/frontend.md
  → Apply its rules during architecture and code generation steps

IF project requires production deployment / CI/CD / multi-environment infra:
  → Load extensions/devops.md
  → Apply its rules during architecture and code generation steps

IF scale target > 10,000 DAU OR explicit performance requirement stated:
  → Load extensions/performance.md
  → Apply its rules during technology decisions and code generation steps

IF none of the above apply:
  → Skip all extensions. Do NOT mention them.
  → Core flow is sufficient.
```

**Rules:**

```
× Do NOT load extensions blindly for every project
× Do NOT force extensions on simple projects
× Do NOT ask the user which extensions to load — detect from the PRD
× Each extension only applies during the steps it explicitly targets
```

**Output:** State which extensions were loaded (or "No extensions needed") before proceeding.

---

## STEP 5 — Environment & Credentials

**Trigger:** Technology stack is confirmed.

**Action:** Use `rules/env-requirements.md` to define all required credentials.

**Required output:**
- Full list of external services the system depends on
- Every environment variable name, per service, with description
- Complete `.env.example` template (grouped by service, with placeholder values)
- Setup instructions for non-obvious credentials (where to obtain, how to generate)
- Config validation code that fails fast on missing vars at startup

**Gate:**

```
IF credentials have not been listed and confirmed:
  → DO NOT proceed to checklists or code generation.
  → Respond: "I need to define all credentials before generating code."
  → Return to this step.

IF user says "I'll add credentials later":
  → Respond: "Credentials affect code structure. This step takes 5 minutes
    and prevents a broken first run. I'll keep it concise."
  → Continue credential definition.
```

---

## STEP 6 — Checklist Validation

**Trigger:** Credentials are defined and confirmed.

**Action:** Run BOTH checklists. Every item must be addressed.

### Security Checklist
→ Open `checklists/security.md`
→ Go through every item
→ For each item: mark HANDLED / DEFERRED / NOT APPLICABLE
→ DEFERRED items must have a reason and a date

### Production Readiness Checklist
→ Open `checklists/production.md`
→ Go through every item
→ For each item: mark HANDLED / DEFERRED / NOT APPLICABLE

**Gate:**

```
IF any CRITICAL item is DEFERRED:
  → Alert user: "Critical item deferred: [item]. This will cause production failure."
  → Do NOT proceed without user explicitly acknowledging the risk.
```

---

## STEP 7 — Code Generation

**Trigger:** Both checklists are complete. User confirms readiness.

**Action:** Generate code following these rules:

### Code Generation Rules

```
1. Generate section by section (not entire app at once)
2. Each section must map to a PRD feature
3. Each file must have a clear single responsibility
4. No placeholder comments ("TODO: implement this")
5. No hardcoded secrets (use env vars — always)
6. Include error handling at every I/O boundary
7. Include input validation at every API entry point
8. Write types / interfaces before implementation
9. Database queries must use parameterized statements
10. Log at: request entry, error, external call, background job
```

### Code Generation Order

```
1. Data models / types / interfaces
2. Database schema + migrations
3. Core business logic (no I/O dependencies)
4. Repository / data access layer
5. Service layer
6. API routes / controllers
7. Middleware (auth, validation, logging)
8. Background workers (if applicable)
9. Tests (unit → integration → e2e)
10. Deployment config (Dockerfile, env.example)
```

---

## VIOLATION HANDLING

```
IF AI skips any step:
  → User can say: "You skipped [step name]. Go back."
  → AI must return to that step and complete it.

IF AI generates code prematurely:
  → User can say: "Stop. Follow master-flow.md."
  → AI must stop, apologize, and restart from Step 1.
```

---

## FLOW SUMMARY

```
[IDEA] 
  → Step 1: Ask 10 questions
  → Step 2: Generate + approve PRD
  → Step 3: Design + approve architecture
  → Step 4: Confirm technology stack
  → Step 5: Define credentials + generate .env template
  → Step 6: Run security + production checklists
  → Step 7: Generate code (section by section)
```
