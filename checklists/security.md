# Security Checklist

> Every item must be marked before code ships to production.
> CRITICAL items are non-negotiable. DEFERRED items require written justification.

---

## MOST COMMON FAILURES

These 5 issues cause the majority of production security incidents in AI-generated code:

1. **No input validation** — user data trusted, passed directly to DB or LLM
2. **Hardcoded secrets** — API keys and passwords committed to git
3. **Missing authentication checks** — routes accessible without auth
4. **No rate limiting** — auth endpoints open to brute force
5. **XSS vulnerabilities** — user content rendered as raw HTML

Check these five before anything else. If any are unresolved, stop.

---

## HOW TO USE

```
For each item, mark:
  [x] HANDLED    — implemented and verified
  [~] DEFERRED   — not in MVP, tracked in backlog, date set
  [N] N/A        — genuinely does not apply (explain why)

IF any CRITICAL item is DEFERRED:
  → Production deployment is blocked until resolved.
  → User must explicitly acknowledge risk in writing.
```

---

## SECTION 1 — INPUT VALIDATION

### API Inputs

- [ ] CRITICAL — All API inputs validated with schema (Zod / Joi / Yup / class-validator)
- [ ] CRITICAL — Validation occurs server-side (client-side is UX only, not security)
- [ ] CRITICAL — Reject requests with unknown/extra fields (strict mode on validators)
- [ ] CRITICAL — String length limits enforced on all text fields
- [ ] CRITICAL — Numeric bounds enforced (no negative IDs, no absurd quantities)
- [ ] CRITICAL — File upload types validated by MIME type (not just extension)
- [ ] CRITICAL — File upload size limits enforced at API layer
- [ ] HIGH     — Enum values validated (reject anything outside allowed set)
- [ ] HIGH     — Date/time fields validated (valid formats, reasonable ranges)
- [ ] MEDIUM   — Array inputs have max length limits
- [ ] MEDIUM   — Nested object depth limits enforced (prevents DoS via deep nesting)

### Database Inputs

- [ ] CRITICAL — All database queries use parameterized statements / ORM (no string concatenation)
- [ ] CRITICAL — User IDs in queries always joined with auth context (prevent IDOR)
- [ ] HIGH     — Search inputs sanitized before being passed to LIKE queries
- [ ] HIGH     — Sort/order fields validated against allowlist (not passed directly to SQL)

---

## SECTION 2 — AUTHENTICATION

- [ ] CRITICAL — Passwords hashed with bcrypt or argon2 (cost factor ≥ 12)
- [ ] CRITICAL — Plaintext passwords never logged, never stored, never transmitted
- [ ] CRITICAL — Auth tokens stored in httpOnly, Secure, SameSite=Strict cookies
- [ ] CRITICAL — JWT secrets are ≥ 256-bit random values (not "mysecret" or app name)
- [ ] CRITICAL — JWT expiry set (access token ≤ 15 minutes)
- [ ] CRITICAL — Refresh tokens rotated on every use
- [ ] CRITICAL — All sessions invalidated on password change
- [ ] CRITICAL — Account lockout after N failed auth attempts (N ≤ 10)
- [ ] HIGH     — MFA available for high-privilege accounts
- [ ] HIGH     — Password reset tokens: single-use, expire in ≤ 1 hour
- [ ] HIGH     — OAuth state parameter validated to prevent CSRF in OAuth flow
- [ ] HIGH     — Auth endpoints rate-limited (see redis.md rate limiting section)
- [ ] MEDIUM   — Login events logged (user_id, IP, timestamp, success/failure)
- [ ] MEDIUM   — Suspicious login patterns detected (new device, new geography)

---

## SECTION 3 — AUTHORIZATION

- [ ] CRITICAL — Every authenticated route verifies the requesting user owns the resource
- [ ] CRITICAL — Role checks enforced server-side (never trust client-sent role)
- [ ] CRITICAL — Admin routes require explicit admin role check (not just authentication)
- [ ] CRITICAL — Resource IDs validated against user's allowed scope on every request
- [ ] HIGH     — Horizontal privilege escalation tested (user A cannot access user B's data)
- [ ] HIGH     — Vertical privilege escalation tested (member cannot perform admin actions)
- [ ] HIGH     — Deleted resources return 404, not 403 (prevent enumeration)
- [ ] MEDIUM   — API keys scoped to minimum required permissions
- [ ] MEDIUM   — Audit log for privileged actions (admin actions, role changes, deletions)

---

## SECTION 4 — SECRETS MANAGEMENT

- [ ] CRITICAL — No secrets in source code (no API keys, no DB passwords in code)
- [ ] CRITICAL — No secrets in git history (run: `git log -p | grep -i "password\|secret\|key"`)
- [ ] CRITICAL — `.env` in `.gitignore` — verified before first commit
- [ ] CRITICAL — `env.example` committed with placeholder values only
- [ ] CRITICAL — Production secrets stored in secret manager (AWS Secrets Manager / Vault / Doppler)
- [ ] CRITICAL — Database credentials rotated and not shared across environments
- [ ] HIGH     — Third-party API keys have minimum required scopes
- [ ] HIGH     — API keys for external services stored per-environment (not one key for all)
- [ ] MEDIUM   — Secret rotation process documented
- [ ] MEDIUM   — Secrets access logged and monitored

---

## SECTION 5 — TRANSPORT SECURITY

- [ ] CRITICAL — All traffic over HTTPS (HTTP redirects to HTTPS in production)
- [ ] CRITICAL — TLS 1.2 minimum (TLS 1.3 preferred)
- [ ] CRITICAL — HSTS header set: `Strict-Transport-Security: max-age=31536000`
- [ ] HIGH     — CORS origin whitelist configured (not `*` in production)
- [ ] HIGH     — CORS credentials flag only set when needed
- [ ] HIGH     — Content-Security-Policy header configured
- [ ] MEDIUM   — X-Frame-Options: DENY (or SAMEORIGIN if framing needed)
- [ ] MEDIUM   — X-Content-Type-Options: nosniff
- [ ] MEDIUM   — Referrer-Policy configured
- [ ] MEDIUM   — Permissions-Policy configured (disable unused browser features)

---

## SECTION 6 — INJECTION ATTACKS

- [ ] CRITICAL — SQL injection: parameterized queries everywhere (verified)
- [ ] CRITICAL — XSS: all user content HTML-escaped before rendering
- [ ] CRITICAL — XSS: Content-Security-Policy disables inline scripts
- [ ] CRITICAL — Command injection: no user input passed to shell commands
- [ ] CRITICAL — Path traversal: file paths sanitized (no `../` in user-supplied paths)
- [ ] HIGH     — NoSQL injection: if using MongoDB, operators in input rejected
- [ ] HIGH     — SSRF: outbound HTTP calls validated against allowlist of domains
- [ ] HIGH     — XML/YAML input: external entity processing disabled (XXE prevention)
- [ ] MEDIUM   — Log injection: user input sanitized before being logged

---

## SECTION 7 — PROMPT INJECTION (AI-powered apps only)

```
IF app uses LLM / AI models:
  → All items in this section are CRITICAL
ELSE:
  → Mark entire section as N/A
```

- [ ] CRITICAL — System prompt not exposed to users (not in client-side code)
- [ ] CRITICAL — User input passed to LLM is clearly delimited from system instructions
- [ ] CRITICAL — LLM output not rendered as raw HTML (escape before display)
- [ ] CRITICAL — LLM output not executed as code without explicit user action
- [ ] CRITICAL — LLM cannot call APIs that modify user data without secondary confirmation
- [ ] HIGH     — Indirect prompt injection mitigated: external content (web pages, docs) fetched for context is sandboxed
- [ ] HIGH     — LLM jailbreak attempts logged and rate-limited
- [ ] HIGH     — Maximum token limits set on user inputs
- [ ] HIGH     — Output filtering for: PII, secrets, system internals
- [ ] MEDIUM   — LLM responses validated against expected schema before use in code paths
- [ ] MEDIUM   — AI feature failures degrade gracefully (LLM down ≠ app down)

---

## SECTION 8 — DATA PROTECTION

- [ ] CRITICAL — PII identified and inventoried (what data, where stored, who can access)
- [ ] CRITICAL — PII not logged in plain text (mask: email → `g***@gmail.com`)
- [ ] CRITICAL — Sensitive fields encrypted at rest (payment tokens, SSNs, health data)
- [ ] HIGH     — Data retention policy defined (how long data is kept, how deleted)
- [ ] HIGH     — User data deletion implemented (GDPR right to erasure)
- [ ] HIGH     — Data export implemented if required (GDPR right to portability)
- [ ] MEDIUM   — Backup encryption enabled
- [ ] MEDIUM   — Database access restricted to app servers only (no public DB endpoint)

---

## SECTION 9 — DEPENDENCY SECURITY

- [ ] HIGH     — `npm audit` (or equivalent) run and critical vulnerabilities resolved
- [ ] HIGH     — No packages with known critical CVEs in dependency tree
- [ ] MEDIUM   — Lock file committed (`package-lock.json` / `yarn.lock` / `uv.lock`)
- [ ] MEDIUM   — Dependabot or Renovate configured for automated dependency updates
- [ ] MEDIUM   — No packages with 0 downloads or untrusted publisher in production deps

---

## SECTION 10 — ERROR HANDLING

- [ ] CRITICAL — Stack traces never exposed to API clients in production
- [ ] CRITICAL — Error messages don't reveal system internals (DB names, file paths, versions)
- [ ] HIGH     — Generic error messages returned to clients; detailed errors logged server-side
- [ ] HIGH     — 404 and 403 responses are indistinguishable for unauthorized resource access
- [ ] MEDIUM   — Unhandled promise rejections and uncaught exceptions caught globally and logged

---

## SIGN-OFF

```
Date reviewed: ___________
Reviewed by:   ___________
Environment:   [ ] Development  [ ] Staging  [ ] Production

CRITICAL items unresolved: _____ (must be 0 for production deployment)
DEFERRED items:            _____ (each must have backlog ticket and date)
```
