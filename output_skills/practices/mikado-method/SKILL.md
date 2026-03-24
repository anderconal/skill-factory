---
name: mikado-method
description: Navigates cascading refactoring dependencies by drawing a prerequisite graph, reverting to safety, and attacking leaf nodes. Use when a refactoring attempt breaks many things at once or dependencies block progress.
---

# The Mikado Method

STARTER_CHARACTER = 🎋

When starting, announce: "🎋 Using MIKADO-METHOD skill".

## The Core Principle

If a change requires too many cascading changes to compile or pass tests, you are working in the wrong order.

The Mikado Method assumes that in a coupled legacy system, the order of changes matters more than the changes themselves. Instead of pushing through the cascade, you map the prerequisites first and work from the outside in.

Named after the Mikado stick game: you cannot move a stick that depends on others without toppling the pile. Find the sticks you can move freely and start there.

## The Five Steps

### 1. Set a Goal

Define one concrete, verifiable objective:
- "Extract `UserRepository` out of `GodController`"
- "Replace `LegacyPaymentModule` with `StripeGateway`"
- "Remove direct DB access from `OrderService`"

Write it at the top of your Mikado graph (a sheet of paper or a simple text file).

### 2. Try It Naively

Make the change directly in the codebase. Do not be careful. Break things.

Compile. Run tests. Let the errors surface.

### 3. Draw Prerequisites and Revert

For each compilation error or test failure, ask: "What would need to be true first for this change to work?"

Write each prerequisite as a child node in the graph:

```
Goal: Extract UserRepository from GodController
  ├── UserRepository must not depend on HttpContext
  │     └── SessionData must be passed as parameter, not read from context
  ├── GodController must use an IUserRepository interface
  │     └── IUserRepository interface must exist
  └── Tests for GodController must not use DB directly
        └── Characterization tests must be in place first
```

Then: **`git reset --hard`**. This is not failure — this is the method. You keep the graph, not the code.

### 4. Attack the Leaf Nodes

A leaf node is a prerequisite with no further prerequisites below it — it can be done right now.

Pick the smallest leaf. Make only that change. Compile. Run tests. If it passes: commit.

```bash
git add -p  # stage only the leaf change
git commit -m "prereq: add IUserRepository interface"
```

### 5. Repeat

Return to the goal. Try the change again.

New errors may surface — add them to the graph. New leaf nodes become available — attack them. Each iteration either succeeds or teaches you the next prerequisite.

The graph shrinks from the leaves inward until the goal becomes trivial to implement.

## The Mikado Graph in Practice

Keep it simple. A text file or whiteboard is enough:

```
[DONE] IUserRepository interface created
[DONE] SessionData passed as parameter to UserRepository
[IN PROGRESS] GodController uses IUserRepository
  └── [TODO] Remove direct new UserRepository() from GodController
[TODO] Extract UserRepository from GodController  ← GOAL
```

Nodes move from TODO → IN PROGRESS → DONE. The goal is always the last node to be completed.

## Key Rules

- **Revert ruthlessly.** A `git reset --hard` after discovering prerequisites is correct usage, not wasted work. The graph is your deliverable at that step.
- **One leaf at a time.** Do not batch multiple prerequisite changes. Each completed leaf is a deployable commit.
- **Prerequisites go down, goals go up.** The graph reads "to achieve X, I first need Y and Z."
- **The graph is a living document.** New prerequisites discovered during leaf work get added immediately.

## When Not to Use It

If a change compiles cleanly and tests stay green: you do not need Mikado. Mikado is for the situations where you pull one thread and the whole sweater unravels.

## Credits

Ola Ellnestam & Daniel Brolund, *The Mikado Method* (Manning, 2014).
