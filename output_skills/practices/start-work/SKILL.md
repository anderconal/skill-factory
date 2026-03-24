---
name: start-work
description: Starts work on a story: transitions it to in-progress, shows its acceptance criteria, and persists state for sync-story and finish-work. Without an argument, shows prioritized backlog to choose from.
disable-model-invocation: true
argument-hint: [issue-key]
---

STARTER_CHARACTER = 🛠️

Transitions a story to in-progress and writes local state so sync-story and finish-work can operate without re-fetching.

## Step 1: Determine the story

**With argument** (`$ARGUMENTS` contains an issue key): skip to Step 2.

**Without argument**: fetch the backlog from the issue tracker — stories in a "to do" or equivalent status, ordered by priority descending. Show up to 5, let the user pick. If no MCP is available, ask for the issue key directly.

If a story is already active (`.claude/state/.current-story` exists), show it and ask: switch to a different one, or continue with the current one?

**Invalid argument**: show usage and stop.

## Step 2: Fetch story details

Get the story from the issue tracker via MCP. If it doesn't exist, show an error and suggest `/create-story`.

## Step 3: Show summary and confirm

Display before doing anything:

```
[ISSUE-KEY] Story title
Status: current status
Labels: ...

Acceptance Criteria:
- [ ] AC1
- [ ] AC2
```

Ask: start working on this story, or cancel?

## Step 4: Transition to in-progress

Get available transitions from the MCP. Find the one that moves the story to an active/in-progress state — match by name (look for "in progress", "in development", "en curso", or equivalent). Execute it.

If already in-progress, skip the transition.

## Step 5: Add start comment

Add a brief comment to the issue: work session started.

## Step 6: Persist local state

Write `.claude/state/.current-story` so sync-story and finish-work can reference the active story without hitting the API:

```json
{
  "issueKey": "PROJ-123",
  "issueId": "<numeric id>",
  "title": "<story title>",
  "status": "in-progress",
  "startedAt": "<ISO timestamp>",
  "acceptanceCriteria": ["AC1", "AC2"],
  "labels": ["label1"]
}
```

This file is a disposable cache — delete it to reset, recreate by running `/start-work` again.

```bash
mkdir -p "$CLAUDE_PROJECT_DIR/.claude/state"
# write the JSON above to $CLAUDE_PROJECT_DIR/.claude/state/.current-story
```

## Step 7: Confirm

```
✅ Started: [ISSUE-KEY] Story title

Acceptance Criteria to meet:
- [ ] AC1
- [ ] AC2

/sync-story — add a progress update
/finish-work — complete the story when done
```
