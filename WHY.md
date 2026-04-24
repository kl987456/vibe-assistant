# Why This Exists

AI writes code.

But it does NOT:
- ask the right questions
- design systems
- think about scale
- handle production failures

It just writes code. Fast. Confidently. Often wrong.

---

## The Gap

**Code that runs:**
- passes the demo
- works on localhost

**A system that works:**
- survives real traffic
- fails gracefully when dependencies go down
- can be debugged at 2am
- doesn't have secrets in git history

AI closes the first gap instantly.

This repo closes the second one.

---

## How

It forces AI to behave like a senior engineer:

```
Without this repo:   Idea → Code
With this repo:      Idea → Questions → PRD → Architecture → Credentials → Checks → Code
```

Not as suggestions. As enforced gates that block code generation until each step is done.

---

## The Result

10 minutes of planning before coding.

In exchange: production-ready output instead of a demo that breaks the first time a real user touches it.

That trade is always worth it.
