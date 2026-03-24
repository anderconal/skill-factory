---
name: finish-work
description: Completes work on the active story: verifies git is clean, confirms all ACs, runs quality gates, transitions to done, and clears local state. Use when a story is ready to ship.
disable-model-invocation: true
---

STARTER_CHARACTER = ✅

Definitive completion flow. Each step is a gate — do not proceed past a failing gate without explicit user authorization.

## Step 1: Find the active story

Check `.claude/state/.current-story` first. If not present, query the issue tracker for stories in an active/in-progress status assigned to the current user.

If no active story: show an error and stop.

## Step 2: Verify git state (BLOCKING)

```bash
git status --porcelain       # must be empty
git log origin/HEAD..HEAD --oneline  # must be empty
```

Show results:
```
Git state:
- [ ] Working tree clean
- [ ] All commits pushed
```

**Uncommitted changes**: list the files. Ask if you should prepare the commit, or stop and let the user handle it.

**Unpushed commits**: list them. The user must push before continuing.

**Do not proceed until git is fully clean and in sync.** This gate is non-negotiable.

## Step 3: Verify acceptance criteria

Show the ACs from the active story and ask: are all of them met?

```
Acceptance Criteria for [ISSUE-KEY]:
- [ ] AC1
- [ ] AC2
```

If any AC is unmet: offer to use `/sync-story` to document the current state, then stop. Do not transition a story to done with unmet ACs.

If the user wants to close with partial ACs (PO authorization): ask explicitly, document what was completed and what remains, and offer to create a follow-up story for the remainder.

## Step 4: Run quality gates

Detect the project's tooling from the repository structure, then run appropriate checks:

- `package.json` → look for `test`, `lint`, `type-check`, `build` scripts and run them
- `*.csproj` / `*.sln` → `dotnet test` and `dotnet format --verify-no-changes`
- `Makefile` → check for `test`, `lint`, `check` targets
- `pyproject.toml` / `pytest.ini` → `pytest`
- CI config (`.github/workflows/`, `azure-pipelines.yml`) → use as reference for what the project considers required checks

Show results per check. If any gate fails: show the specific error and stop. Fix before proceeding — do not allow skipping a failing gate.

## Step 5: Build closure comment

```
✅ Story completed

Acceptance Criteria verified:
- ✅ AC1: [how it was verified]
- ✅ AC2: [how it was verified]

Quality gates:
- ✅ Tests: passing
- ✅ Lint/format: clean
- ✅ Build: success

Changes summary:
- [key files or features modified]
```

## Step 6: Add comment and transition

Add the closure comment to the issue tracker, then get available transitions and move the story to a done/completed state.

## Step 7: Clear local state

```bash
rm -f "$CLAUDE_PROJECT_DIR/.claude/state/.current-story"
```

## Step 8: Confirm and suggest next

```
🎉 Done: [ISSUE-KEY] Story title

What's next?
/start-work — pick the next story from the backlog
/create-story — create a new story
```

Optionally show the top 3 priority stories still in the backlog.
