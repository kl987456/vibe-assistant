# vibe-coder-guardrails

> Turn AI-generated code into production-ready systems — automatically.

---

## Try in 30 seconds

1. Copy `prompts/master-prompt.md`
2. Paste into Claude
3. Type: `"Build a chatbot"`

That's it. The guardrails activate automatically.

---

## The Problem

AI can build your app in minutes.

It can also silently introduce:
- security vulnerabilities
- bad architecture decisions
- scalability bottlenecks

This repo exists to prevent that.

**Input:** a vague idea
**Output:** production-ready system design + code

---

## What This Repo Does

This repo is a **rulebook for AI**. It forces any AI assistant (Claude, GPT, Gemini) to behave like a senior engineer — structured, deliberate, and production-aware — before a single line of code is written.

**It is NOT documentation. It is NOT a tutorial.**

It is a system of enforceable rules, decision trees, and checklists that AI must follow when helping you build software.

---

## The Value

```
Normal AI:    Idea → Code          (fast, broken in production)
This repo:    Idea → System → Code (structured, production-ready)
```

The difference: 10 minutes of forced planning before a single line is written.

---

## What You Get

For every project this repo produces:

- **Structured PRD** — problem, users, features, edge cases, scale targets
- **Architecture decisions** — every layer chosen with explicit reasoning
- **Technology decisions** — Redis, Kafka, database, auth — IF/THEN, not guesswork
- **Credentials & .env setup** — every required env var listed before code is written
- **Security validation** — 70+ item checklist covering input, auth, secrets, XSS, injection
- **Production readiness** — logging, retries, monitoring, error handling, health checks
- **Code** — typed, validated, with error handling — generated only after all of the above

---

## Where This Works

Designed for most modern application development:

- Backend APIs and SaaS applications
- AI-powered apps (chatbots, RAG systems, assistants)
- Web applications with moderate scale
- Internal tools and dashboards
- Multi-service systems requiring structured architecture decisions

## Where This Does NOT Apply

Not designed for:

- Highly regulated domains (fintech, healthcare, legal compliance-heavy systems)
- Low-level systems (embedded, firmware, hardware programming)
- Game engines or real-time physics systems
- Advanced infrastructure engineering (Kubernetes-heavy, multi-region, edge networks)
- Hyperscale distributed systems (millions of requests/second)

## Why This Distinction Matters

This repo focuses on structured thinking, production-ready practices, and real-world application building.

It does NOT replace:
- Domain expertise for specialized industries
- Deep infrastructure or platform engineering knowledge
- Compliance frameworks for regulated systems

---

## How It's Different

| Normal AI Coding | With This Repo |
|---|---|
| Gets idea → writes code | Gets idea → asks questions first |
| Makes technology assumptions | Uses decision trees to choose tech |
| Skips security | Runs security checklist before code |
| Skips architecture | Forces architecture design |
| No PRD | Generates structured PRD |
| Hardcodes credentials | Generates .env template first |
| Vibe-coded output | Production-ready output |

---

## Repository Structure

```
vibe-coder-guardrails/
├── prompts/
│   └── master-prompt.md     ← START HERE — paste into Claude or ChatGPT
├── rules/
│   ├── master-flow.md       ← 7-step execution order AI must follow
│   ├── prd-generator.md     ← Converts vague ideas into structured PRDs
│   ├── architecture.md      ← System design blueprint
│   ├── env-requirements.md  ← Credential definition + .env template generation
│   └── quick-mode.md        ← Accelerated flow for prototypes
├── extensions/              ← Optional — load only what your project needs
│   ├── README.md            ← When and how to use extensions
│   ├── frontend.md          ← UI architecture, state handling, API patterns
│   ├── devops.md            ← Deployment, CI/CD, environments, containers
│   └── performance.md       ← DB optimization, caching, scaling triggers
├── decision-trees/
│   ├── redis.md             ← When to use / not use Redis
│   └── kafka.md             ← When to use / not use Kafka
├── checklists/
│   ├── security.md          ← Security gates before shipping
│   └── production.md        ← Production readiness gates
├── examples/
│   └── chatbot-example.md   ← Full walkthrough: idea → production code
└── WHY.md                   ← Why this repo exists
```

---

## Quick Start

**1.** Copy `prompts/master-prompt.md` — paste as your first message in Claude or ChatGPT

**2.** Describe your project — even a vague one-liner works

**3.** The AI will run through the structured flow automatically:
- Clarifying questions → PRD → Architecture → Tech decisions → Credentials → Checklists → Code

**4.** Review the decisions before accepting any code — the PRD, architecture, and tech choices are explicit and adjustable

---

## Quick Setup (for generated projects)

Every project this repo produces includes a `.env.example`. To run it locally:

```bash
# 1. Copy the environment template
cp .env.example .env

# 2. Fill in your credentials (the file tells you where to get each one)
# Edit .env and replace placeholder values

# 3. Generate any secrets the project needs
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"

# 4. Install and run
npm install
npm run dev
```

The generated project will exit with a clear error listing any missing env vars — it will not start with broken config.

---

## How to Use with Claude

**Option A — Single session**
Paste `master-prompt.md` at the start of every Claude conversation where you're building something.

**Option B — CLAUDE.md integration**
Add the contents of `master-prompt.md` to your project's `CLAUDE.md` file so Claude automatically follows these rules in that project.

**Option C — Reference specific files**
Tell Claude: *"Follow the Redis decision tree at decision-trees/redis.md before choosing caching tech."*

---

## Extensions (Optional)

Advanced modules that activate only when a project needs them. The core flow runs without them.

| Extension | Loads when... |
|---|---|
| `extensions/frontend.md` | Project includes a user-facing UI |
| `extensions/devops.md` | Project needs CI/CD, multi-env setup, or infra config |
| `extensions/performance.md` | Scale targets are high or performance is a stated concern |

Extensions are **detected automatically** from the PRD — you don't have to ask for them.
Simple projects load zero extensions. Complex projects load what they need.

See `extensions/README.md` for the full rules.

---

## Who This Is For

- Developers using AI to build real products
- Teams that want AI-generated code to be reviewable and maintainable
- Anyone who has been burned by AI code that "worked in demo but broke in production"

---

## Principles

1. **Questions before code** — always
2. **Explicit decisions** — IF/THEN, not "consider"
3. **Checklists are gates** — not suggestions
4. **Architecture before implementation** — always
5. **Scale assumptions must be stated** — never implicit

---

> **Stop vibe coding. Start shipping real systems.**
