---
name: project-done
description: "Defines project Definition of Done: ACs verified, git clean and pushed, quality gates passing (dotnet test + format, pnpm test:unit, lint, type-check, build), and ethical gate (features cannot ship if usage increases without value). Use when checking if a story is done or a feature can be shipped."
---

STARTER_CHARACTER = ✅

Work is done when all four gates pass, in order. Each must clear before moving to the next.

## Gate 1: Git Clean

```bash
git status --porcelain        # must return nothing
git log origin/main..HEAD --oneline  # must return nothing
```

Uncommitted changes or unpushed commits block everything else.

## Gate 2: Acceptance Criteria

Every AC from the active story (`.claude/state/.current-story`) must be demonstrably met. Walk through each one — don't self-certify without checking.

If an AC is ambiguous, clarify before closing, not after.

## Gate 3: Quality Gates

Run only for layers with changes:

**Backend** (if `backend/` changed):
```bash
cd backend && dotnet test && dotnet format --verify-no-changes
```

**Frontend** (if `frontend/` changed):
```bash
cd frontend && pnpm test:unit && pnpm lint && pnpm type-check && pnpm build
```

All must pass. No exceptions, no skipping.

## Gate 4: Ethical Gate

Before closing, verify the feature does not:
- Increase usage without increasing value
- Reduce user autonomy or control
- Introduce social pressure or comparison
- Depend on dark patterns or designed re-engagement

If uncertain: don't ship. Use `/sync-story` to flag it and escalate.

## Partial Completion

If some ACs are met but not all:
- Use `/sync-story` to document state and what remains
- Create a new story for the outstanding work
- Never close a story with open ACs without explicit authorization

## Not Done Until Pushed

Work that exists only locally does not exist. A story is not closeable until code is on the remote.
