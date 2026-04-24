# PRD Generator

> Convert a vague idea into a structured Product Requirements Document.
> Fill every section. Leave nothing blank. "TBD" is not allowed.

---

## RULE

```
IF any section is incomplete:
  → Ask user for the missing information.
  → Do NOT proceed to architecture with an incomplete PRD.
```

---

## PRD TEMPLATE

---

### 1. Problem Statement

**Format:** 1–2 sentences. State the pain, not the solution.

```
[WHO] currently [does what workaround] because [root problem].
This causes [measurable consequence].
```

**Example:**
```
Small teams currently track client approvals over email threads because no tool 
fits their workflow without enterprise pricing. This causes missed deadlines and 
lost approval context.
```

---

### 2. User Personas

**Format:** 2–3 named personas. Each must have a goal and a frustration.

```
Persona 1:
  Name: [Name]
  Role: [Role]
  Primary goal: [What they want to accomplish]
  Primary frustration: [What currently blocks them]
  Technical proficiency: [Low / Medium / High]

Persona 2: ...
```

---

### 3. MVP Feature Set

**Format:** Numbered list. Features only — no implementation details.

**Rule:** Max 5 features for MVP. Force prioritization.

```
IF user lists > 5 features:
  → Say: "MVP must be ≤ 5 features. Which 5 deliver the core value?"
  → Wait for reduced list.
```

```
1. [Feature name] — [one sentence description]
2. ...
```

---

### 4. Explicit Out of Scope (MVP)

**Format:** Bulleted list. State what will NOT be built in v1.

**Purpose:** Prevents scope creep. AI must not build these unless PRD is updated.

```
- [Feature] — reason: [why deferred]
- ...
```

---

### 5. Edge Cases

**Minimum:** 3 edge cases. Aim for 5–8.

**Categories to consider:**

```
- Empty states (no data, first-time user)
- Concurrent operations (two users editing same record)
- Partial failures (payment succeeds, email fails)
- Invalid input (malformed data, injection attempts)
- Scale events (10x traffic spike)
- Dependency failures (third-party API is down)
- Permission edge cases (user role changes mid-session)
```

**Format:**

```
EC-01: [Scenario]
  → Expected behavior: [what should happen]
  → Handling: [how the system handles it]

EC-02: ...
```

---

### 6. Scale Targets

**Format:** Fill all three columns. These drive architecture decisions.

```
| Metric              | Launch | 6 Months | 18 Months |
|---------------------|--------|----------|-----------|
| Registered users    |        |          |           |
| Daily active users  |        |          |           |
| Requests / second   |        |          |           |
| Data storage (GB)   |        |          |           |
| Concurrent sessions |        |          |           |
```

**Decision triggers derived from scale:**

```
IF DAU (18mo) > 10,000  → Consider Redis caching
IF RPS (18mo) > 500     → Consider horizontal scaling
IF RPS (18mo) > 2,000   → Consider Kafka for async processing
IF data > 1TB           → Plan sharding or tiered storage strategy
IF concurrent > 5,000   → WebSocket or SSE infrastructure required
```

---

### 7. Data Model Sketch

**Format:** Entity list with relationships. Not a full schema — just key entities.

```
Entity: [Name]
  Fields: [field: type, field: type, ...]
  Relationships: [belongs to / has many / many-to-many with Entity]

Entity: ...
```

**Example:**
```
Entity: User
  Fields: id: uuid, email: string, role: enum(admin|member), created_at: timestamp
  Relationships: has many Projects, has many Invitations

Entity: Project
  Fields: id: uuid, name: string, owner_id: uuid, status: enum, created_at: timestamp
  Relationships: belongs to User (owner), has many Approvals
```

---

### 8. Success Metrics

**Format:** Measurable. Not "users like it."

```
| Metric                     | Target       | Measurement method     |
|----------------------------|--------------|------------------------|
| [Core action completion %] | [> X%]       | [Event tracking]       |
| [Response time]            | [< Xms p99]  | [APM tool]             |
| [Error rate]               | [< X%]       | [Error tracking tool]  |
```

---

### 9. Assumptions Log

**Format:** Everything assumed during clarification. One line each.

```
A-01: [Assumption stated explicitly]
A-02: ...
```

**Rule:** Every "I don't know" answer from Step 1 becomes an entry here.

---

### 10. Open Questions

**Format:** Unresolved decisions that may affect implementation.

```
Q-01: [Question] — Owner: [user/AI/stakeholder] — Resolve by: [phase/date]
Q-02: ...
```

---

## PRD COMPLETION CHECK

Before marking PRD as complete, verify:

```
[ ] Problem statement is one problem, not a list of features
[ ] All personas have a named frustration
[ ] MVP has ≤ 5 features
[ ] Out of scope list exists and is non-empty
[ ] Minimum 3 edge cases with handling defined
[ ] All 5 scale metrics filled for all 3 time horizons
[ ] Data model has all entities referenced in features
[ ] Success metrics are measurable (no vanity metrics)
[ ] Assumptions log is up to date
```

```
IF any box is unchecked:
  → Do not approve PRD.
  → Return to the incomplete section.
```
