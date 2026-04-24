# Extensions

> Extensions are optional modules that activate only when a project specifically needs them.
> The core flow runs completely without them.

---

## How Extensions Work

Extensions are **OPTIONAL**. They are activated **ONLY** when the project needs them.

```
Frontend project (web app, dashboard, widget)?  → load frontend.md
Deployment needed (CI/CD, multi-env, infra)?    → load devops.md
High scale or performance concern?              → load performance.md
None of the above?                              → skip all extensions
```

Rules:
- Extensions are detected from the PRD automatically — you do not ask for them
- Each extension applies its rules only during the steps it targets
- Simple projects load zero extensions — the core flow is sufficient
- Loading an irrelevant extension adds complexity with no benefit — skip it

---

## What Extensions Are

Extensions are supplementary rule sets for capabilities that:
- are not needed by every project
- would add unnecessary weight to simple projects if always loaded
- require domain-specific decisions beyond the core flow

They are **not plugins**, **not replacements**, and **not shortcuts**.
They are addons that sit on top of the core flow when triggered.

---

## What Extensions Are NOT

```
× Not a replacement for master-flow.md
× Not loaded automatically for every project
× Not a way to skip core steps
× Not required for the repo to work
```

---

## When to Load Each Extension

```
IF project has a user-facing UI (web app, dashboard, widget):
  → Load extensions/frontend.md

IF project requires production deployment, CI/CD, or infrastructure config:
  → Load extensions/devops.md

IF project has explicit scale concerns or performance requirements:
  → Load extensions/performance.md
```

```
IF none of the above apply:
  → Do NOT load any extensions.
  → Core flow is sufficient.
```

---

## Available Extensions

| Extension               | Load when...                                              |
|---|---|
| `frontend.md`           | Project includes a user-facing UI                        |
| `devops.md`             | Project needs CI/CD, infra config, or multi-env setup    |
| `performance.md`        | Scale targets are high or performance is a stated concern |

---

## How Extensions Integrate with the Core Flow

Extensions are evaluated at a single point in the master flow: **after technology decisions, before credentials**.

At that point:
1. Scan the PRD and architecture for extension triggers
2. Load only the extensions that match
3. Apply extension rules in the relevant sections of code generation
4. Skip all extensions that don't match — without mention

Extensions do not add extra steps to the flow. They add targeted rules that apply during the steps that already exist.

---

## Example Triggers

```
"Build a React dashboard for monitoring API usage"
  → frontend.md triggered (UI present)
  → devops.md triggered (production deployment implied)
  → performance.md: check scale targets — if < 5,000 DAU, skip

"Build a REST API for internal use"
  → frontend.md: NOT triggered (no UI)
  → devops.md: triggered if multi-env or CI/CD mentioned
  → performance.md: NOT triggered unless scale concern stated

"Build a data pipeline processing 1M events/day"
  → frontend.md: NOT triggered
  → devops.md: triggered (infra complexity)
  → performance.md: triggered (high volume explicit)
```
