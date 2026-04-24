# Environment & Credentials Requirements

> No code is generated until all required credentials are identified and a `.env` template exists.
> "I'll add the API keys later" is not a plan — it is a broken deployment.

---

## ENFORCEMENT RULE

```
IF technology stack is confirmed AND credentials have not been listed:
  → DO NOT proceed to checklists or code generation.
  → Respond: "Before code, I need to define all credentials this system requires."
  → Run Steps 1–5 below.

IF user says "I'll add credentials later":
  → Respond: "Credentials affect how the code is structured — connection strings,
    SDK initialization, and config loading all depend on knowing what's needed now.
    This takes 5 minutes and prevents a broken first run."
  → Continue credential definition.
```

---

## STEP 1 — Identify All External Services

Scan the confirmed technology stack. List every service that requires a credential.

**Categories to check:**

```
Database:
  - Primary DB (PostgreSQL / MySQL / MongoDB / etc.)
  - Read replicas (if any)
  - Connection pooler (PgBouncer — needs its own connection string)

Cache / Queue:
  - Redis (URL + optional password)
  - Kafka (broker URL + SASL credentials if managed)

AI / LLM:
  - OpenAI / Anthropic / Cohere / etc.
  - Embedding model provider (may differ from LLM provider)
  - Vector DB (Pinecone / Weaviate / Qdrant)

Auth:
  - JWT secret (generated — not a third-party key)
  - OAuth provider (Google / GitHub / etc.)
  - Auth platform (Clerk / Auth0 / Supabase — public key + secret key)

Storage:
  - S3 / R2 / GCS (access key ID + secret + bucket name + region)

Email / SMS:
  - Resend / SendGrid / SES (API key + from-address)
  - Twilio (account SID + auth token + phone number)

Payments:
  - Stripe (publishable key + secret key + webhook secret)
  - Paddle / Braintree (equivalent keys)

Monitoring / Logging:
  - Sentry (DSN)
  - Datadog (API key)
  - Logtail / Papertrail (token)

Third-party integrations:
  - Any API the system calls (Zendesk, Slack, GitHub, etc.)

App config (not secrets, but required):
  - PORT
  - NODE_ENV / APP_ENV
  - API base URL
  - CORS allowed origins
  - Log level
```

**Output:** A flat list of every external service this project touches.

---

## STEP 2 — List Required Credentials Per Service

For each service identified in Step 1, list the exact environment variable names.

**Format:**

```
Service: [Name]
  Required:
    - VAR_NAME         — description of what this is
    - VAR_NAME_2       — description
  Optional:
    - VAR_NAME_3       — description, default: [value]
  How to obtain:
    - [One-line instruction: where to find/generate this credential]
```

**Rules:**

```
- Use SCREAMING_SNAKE_CASE for all variable names
- Prefix by service where helpful: DATABASE_URL, REDIS_URL, OPENAI_API_KEY
- Never name a variable just "KEY" or "SECRET" — always include the service name
- Document the default value for optional vars
- Never set a default for secrets (force explicit configuration)
```

---

## COMMON SETUP COMMANDS

```bash
# Generate a secure secret (for JWT_SECRET, encryption keys, etc.)
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"

# First-time local setup
cp .env.example .env
npm install
npm run dev

# Verify env vars load correctly
node -e "require('dotenv').config(); console.log('env ok')"

# Check which required vars are missing
node -e "
  require('dotenv').config();
  const required = ['DATABASE_URL','REDIS_URL','JWT_SECRET']; // adjust per project
  required.forEach(k => { if (!process.env[k]) console.error('MISSING:', k); });
"
```

---

## STEP 3 — Generate `.env` Template

Produce a `.env.example` file. This file:
- Is committed to git
- Contains placeholder values only (never real credentials)
- Documents every variable the app needs to run
- Groups variables by service

**Template format:**

```bash
# ============================================================
# [PROJECT NAME] — Environment Configuration
# Copy this file to .env and fill in all required values.
# NEVER commit .env to git.
# ============================================================

# --- App ---
NODE_ENV=development
PORT=3000
LOG_LEVEL=info
CORS_ALLOWED_ORIGINS=http://localhost:3000

# --- Database ---
# PostgreSQL connection string
# Format: postgresql://USER:PASSWORD@HOST:PORT/DATABASE
DATABASE_URL=postgresql://postgres:password@localhost:5432/myapp

# --- Redis ---
# Redis connection URL
# Format: redis://[:PASSWORD@]HOST:PORT[/DB_NUMBER]
REDIS_URL=redis://localhost:6379

# --- Auth ---
# Generate with: node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
JWT_SECRET=your-256-bit-secret-here
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# --- [Service Name] ---
# Obtain from: [dashboard URL or instruction]
SERVICE_API_KEY=
SERVICE_SECRET=
```

**Rules for the template:**

```
- Every variable must have a comment explaining what it is
- Every required variable must have a placeholder (not just empty)
- Optional variables must have their default value pre-filled
- Group variables by service with a blank line between groups
- Include "How to generate / obtain" for non-obvious credentials
- For generated secrets (JWT, encryption keys): include the exact command to generate
```

---

## STEP 4 — Credentials Rules (enforced in code)

These rules must be applied in every generated codebase:

```
[ ] All secrets loaded from environment variables — no exceptions
[ ] App fails to start if any REQUIRED env var is missing
[ ] Config validation runs at startup (before server begins accepting requests)
[ ] No .env file in production — use secret manager or platform env vars
[ ] .env added to .gitignore — verify before first commit
[ ] .env.example committed and kept up to date (every new var added here first)
[ ] No logging of secret values — mask in startup output if config is printed
[ ] Separate credentials per environment (dev / staging / production never share secrets)
```

**Config validation pattern (required):**

```typescript
// config/env.ts — run this before anything else

import { z } from 'zod';

const EnvSchema = z.object({
  NODE_ENV:     z.enum(['development', 'staging', 'production']),
  PORT:         z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL:    z.string().url(),
  JWT_SECRET:   z.string().min(32),
  // ... add all required vars
});

const parsed = EnvSchema.safeParse(process.env);

if (!parsed.success) {
  console.error('Invalid environment configuration:');
  console.error(parsed.error.flatten().fieldErrors);
  process.exit(1);  // Hard stop — never start with broken config
}

export const env = parsed.data;
```

```python
# config/env.py — Pydantic equivalent

from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    node_env: str = "development"
    port: int = 3000
    database_url: str
    redis_url: str
    jwt_secret: str

    class Config:
        env_file = ".env"

settings = Settings()  # Raises ValidationError on startup if vars are missing
```

---

## STEP 5 — Required Output Before Code

Before code generation begins, produce this block and wait for user confirmation:

```markdown
## Credentials & Environment Setup

### Services Required
| Service      | Purpose                    | Credential vars              |
|---|---|---|
| PostgreSQL   | Primary database           | DATABASE_URL                 |
| Redis        | Rate limiting              | REDIS_URL                    |
| OpenAI       | LLM + embeddings           | OPENAI_API_KEY               |
| [Service]    | [Purpose]                  | [VAR_NAMES]                  |

### Credentials You Need Before Running This App
- [ ] DATABASE_URL — your PostgreSQL connection string
- [ ] REDIS_URL — your Redis connection string
- [ ] OPENAI_API_KEY — from platform.openai.com → API Keys
- [ ] [VAR] — [where to get it]

### Setup Instructions
1. Copy `.env.example` to `.env`
2. Fill in all required values (marked above)
3. Generate secrets where instructed (JWT_SECRET, etc.)
4. Verify: `node -e "require('dotenv').config(); console.log('env ok')"` (or equivalent)

### .env.example
[Full generated template — see Step 3 format]
```

**Gate:**

```
IF user has not confirmed credentials:
  → DO NOT generate code.
  → Wait for: "credentials ready" or "confirmed" or equivalent.

IF user says "I don't have [credential] yet":
  → Respond: "That's fine. Add a placeholder in .env for now.
    The app will fail to start until it's filled in — which is correct behavior.
    The config validation will tell you exactly what's missing."
  → Proceed after user acknowledges.
```
