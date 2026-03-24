---
name: sync-story
description: Adds a structured progress comment to the active story in the issue tracker. Supports progress, blocked, and pause update types. Use when reporting status or handing off work mid-session.
argument-hint: [message]
---

STARTER_CHARACTER = 🔄

Adds a structured comment to the active story. Invokable by the user, by Claude when context is running low, or via hooks on session end or pre-compaction.

## Step 1: Find the active story

First check `.claude/state/.current-story` for cached state. If present, use it.

If not present and an issue tracker MCP is available, query for stories assigned to the current user that are in an active/in-progress status. If exactly one found, use it. If multiple, ask the user to pick.

If no active story found: show an error and suggest `/start-work`.

## Step 2: Determine update type

If `$ARGUMENTS` contains a message: use it directly as a free-form comment, skip to Step 4.

Otherwise ask:
- **Progress** — I've advanced on the implementation
- **Blocked** — I'm blocked and need help
- **Pause** — leaving the work for later (generates handoff comment)
- **Other** — free-form comment

## Step 3: Collect details

**Progress**: ask what was completed (tests written, partial implementation, feature complete) and what the next steps are.

**Blocked**: ask for the reason (technical dependency, product decision needed, technical problem) and what has been tried.

**Pause**: generate automatically from context — summarize completed ACs, pending ACs, and enough context for the next session to resume without re-reading the conversation.

## Step 4: Build comment

**Progress:**
```
✅ Progress update

Completed:
- [items]

Next steps:
- [inferred or provided]
```

**Blocked:**
```
🚫 Blocked

Reason: [type]
Details: [description]

Tried:
- [what was attempted]

Needs:
- [what would unblock]
```

**Pause (handoff):**
```
⏸️ Session paused — handoff state

Acceptance Criteria progress:
- ✅ AC1: [how it was verified]
- ⬜ AC2: [pending]

Context to resume:
[compact summary of what was done and what remains]

```json agent-session-state
{"completedACs":[0],"pendingACs":[1],"context":"[compact summary for agent]"}
` ``
```

The `agent-session-state` block is machine-readable — it allows the next session to restore context without parsing prose.

## Step 5: Add comment and confirm

Post the comment to the issue tracker via MCP.

```
✅ Comment added to [ISSUE-KEY]

[brief summary of what was logged]
```
