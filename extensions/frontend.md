# Extension: Frontend

> Load this extension when the project includes a user-facing UI.
> Do NOT load for API-only projects.

---

## WHEN TO USE THIS

Use this extension ONLY if:
- Project has a web app, dashboard, admin panel, or embeddable widget
- Users interact with a visual interface (not just an API)

Skip if:
- Project is API-only, a CLI tool, a background service, or a data pipeline

---

## ACTIVATION CHECK

```
IF project has: web app / dashboard / widget / admin panel / mobile UI:
  → This extension is active. Apply all rules below during relevant steps.

IF project is: API-only / backend service / CLI tool / data pipeline:
  → Do NOT load this extension. Skip entirely.
```

---

## SECTION 1 — UI Architecture Decision

Choose rendering strategy before writing any frontend code.

```
IF content is: public-facing, SEO-required, or marketing pages:
  → SSR (Next.js / Nuxt / Remix)
  → Reason: search engine indexability, fast first paint

IF content is: auth-gated (user must log in to see it):
  → SPA (React / Vue / Svelte)
  → Reason: no SEO needed; simpler deployment; better interactivity

IF content is: mostly static with minor interactivity:
  → Static site generator (Astro / Eleventy)
  → Reason: lowest infra cost, fastest load times, CDN-friendly

IF content is: mixed (public landing + auth-gated app):
  → SSR for public pages + SPA for app shell (Next.js handles both)

IF building an embeddable widget:
  → Vanilla TypeScript (no framework)
  → Reason: no framework dependency for host app; smallest bundle
```

---

## SECTION 2 — State Management

```
IF state is: server data (fetched from API):
  → Use: React Query / SWR / TanStack Query
  → Do NOT use Redux/Zustand for server data — it duplicates the cache

IF state is: UI-only (modals, tabs, form steps):
  → Use: local component state (useState)
  → Do NOT reach for a state library for simple UI state

IF state is: shared across many unrelated components (theme, auth user, cart):
  → Use: Zustand (simple) OR React Context (simpler, but re-render risk)

IF state is: complex with many transitions (multi-step workflow, undo/redo):
  → Use: XState (state machines — predictable, testable)
```

---

## SECTION 3 — Required UI States

Every user-facing feature must handle all three states before it is considered complete.

### Loading State

```
IF data is being fetched:
  → Show: skeleton loader OR spinner (not a blank screen)
  → Rule: loading state must be visually distinct from empty state
  → Rule: disable interactive elements during loading (prevent double-submit)
```

### Error State

```
IF an API call fails:
  → Show: human-readable error message (not "Error 500" or raw stack trace)
  → Show: a recovery action where possible ("Try again" / "Go back")
  → Log: error to monitoring tool (Sentry / console in dev)
  → Rule: errors must not crash the entire page (use error boundaries)

Error message format:
  × Bad:  "Request failed with status code 400"
  ✓ Good: "We couldn't save your changes. Please check your connection and try again."
```

### Empty State

```
IF a list or data view has no items:
  → Show: a meaningful empty state (not a blank space)
  → Empty state must include:
    - What is empty ("No projects yet")
    - Why it might be empty (first-time user, filter applied)
    - A clear action to fix it ("Create your first project →")

IF filters are applied AND result is empty:
  → Show: "No results match your filters" + "Clear filters" action
  → Do NOT show the generic empty state (they are different situations)
```

---

## SECTION 4 — API Integration Patterns

```
[ ] All API calls go through a typed client (not raw fetch scattered in components)
[ ] Base URL comes from environment variable (never hardcoded)
[ ] Auth token attached in one place (interceptor / client wrapper)
[ ] All API responses typed (no `any` return types)
[ ] Network errors handled at the client level (not in every component)
[ ] Loading and error states derived from query state (not manual booleans)
[ ] Optimistic updates used for fast UX on mutations where appropriate
[ ] Stale data strategy defined (refetch on focus / on interval / manual)
```

**API client pattern (required):**

```typescript
// lib/api-client.ts — one place, all calls go through here

const client = {
  get: <T>(path: string): Promise<T> =>
    fetch(`${env.API_BASE_URL}${path}`, {
      headers: { Authorization: `Bearer ${getToken()}` },
    }).then(handleResponse<T>),

  post: <T>(path: string, body: unknown): Promise<T> =>
    fetch(`${env.API_BASE_URL}${path}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${getToken()}`,
      },
      body: JSON.stringify(body),
    }).then(handleResponse<T>),
};

async function handleResponse<T>(res: Response): Promise<T> {
  if (!res.ok) {
    const error = await res.json().catch(() => ({ message: 'Request failed' }));
    throw new ApiError(res.status, error.message);
  }
  return res.json();
}
```

---

## SECTION 5 — Validation Rules

```
Validation exists at TWO levels. Both are required.

Frontend validation (UX layer):
  - Purpose: instant feedback, reduce server round trips
  - Scope: format checks, required fields, length limits
  - Tool: Zod + react-hook-form / Valibot
  - Rule: never trust frontend validation for security — it is UX only

Backend validation (security layer):
  - Purpose: authoritative — cannot be bypassed by the client
  - Scope: all the above + business rules + permission checks
  - Rule: backend must validate even if frontend already validated

IF there is a conflict between frontend and backend validation:
  → Backend wins. Always.
  → Display the backend error in the UI — do not silently ignore it.
```

**Frontend validation example:**

```typescript
const schema = z.object({
  email: z.string().email('Enter a valid email'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

// This validates format — the backend validates everything else
```

---

## SECTION 6 — Performance Basics

```
[ ] Images use lazy loading (loading="lazy" or next/image)
[ ] Large lists are virtualized if > 100 items (react-virtual / tanstack-virtual)
[ ] Bundle size checked before launch (no unintentional 1MB dependencies)
[ ] Code-split by route (not one giant bundle)
[ ] Fonts preloaded if custom fonts used
[ ] No layout shift on load (reserve space for images and async content)
```

---

## SECTION 7 — Accessibility Minimum

```
[ ] All interactive elements reachable by keyboard
[ ] Form inputs have associated labels (not just placeholder text)
[ ] Error messages linked to their input (aria-describedby)
[ ] Color is not the only indicator of state (use icons or text too)
[ ] Page has one H1 per route
```

These are the minimum. Not the full WCAG standard — just the items that affect the most users.

---

## EXTENSION COMPLETE

Apply these rules during:
- Architecture step (UI rendering decision)
- Code generation step (state handling, API client, validation)
- Pre-launch checklist (UI states, accessibility, performance)
