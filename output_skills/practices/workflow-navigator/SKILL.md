---
name: workflow-navigator
description: Watches the development session and proactively suggests the right command when signals emerge: ambiguity or blockers → /sync-story, completion → /finish-work, scope creep → /create-story, session ending → /sync-story pause, no active story → /start-work. Always active during development sessions.
---

STARTER_CHARACTER = 🧭

Watches the conversation for signals that indicate the workflow needs attention. Surfaces suggestions without interrupting flow — one observation, concrete, only when it matters.

## Signal Map

**No active story + code change requested**
User wants to implement something without an active story (`/.claude/state/.current-story` absent).
→ Pause before any code. Remind about Regla #1. Suggest `/start-work` first.

**Fresh start / "what should I work on?"**
Session begins, user is orienting, no direction yet.
→ Suggest `/start-work` to see the prioritized backlog.

**Ambiguity during implementation**
User asks "should I do X or Y?", finds conflicting requirements, or an AC doesn't cover the current case.
→ Surface as `PENDING DECISION: X or Y?`. If it needs PO input, suggest `/sync-story` to document it before guessing.

**Blocker**
User is stuck for multiple turns, a failure persists, or an external dependency is needed (waiting for a decision, API key, clarification).
→ Suggest `/sync-story` with the blocked type and a concrete blocker description. Don't keep trying indefinitely.

**Scope creep / discovery**
Mid-implementation, user says "I also need to..." or realizes a new requirement exists.
→ Don't expand the current story. Flag it: "That's a new story. Let's finish this one first and use `/create-story` for that."

**Completion signals**
Tests passing, user reviewing ACs, "I think this is done", all ACs checked.
→ Suggest `/finish-work` and remind: git clean + pushed, quality gates, ethical gate — all must pass in order.

**Session ending / context getting long**
User says "let's stop", "that's enough for today", conversation has many turns, or context is approaching limits.
→ Suggest `/sync-story` pause before the session ends. Context in Jira outlasts context windows.

**Requirements document available**
User mentions a requirements file or wants to plan a full feature set at once.
→ Suggest `/generate-backlog <file> [--dry-run] [--lang=es]`.

## How to Surface Suggestions

Keep it brief. One line alongside the response, not a separate interruption:

> "Side note: this sounds like a blocker worth documenting — `/sync-story` before we keep trying?"

> "Heads up: that's a new scope item. Should we `/create-story` for it after we close this story?"

Only surface when the signal is clear. A quick question mid-task is not a blocker. A passing comment about another feature is not scope creep. Apply judgment.

## Anti-patterns

- Suggesting a command on every other turn
- Suggesting `/finish-work` because one test passed (wait for all ACs)
- Treating minor uncertainty as a blocker requiring `/sync-story`
- Interrupting to suggest `/start-work` when a story is already active
- Suggesting `/sync-story` when the user just asked a quick clarification question
