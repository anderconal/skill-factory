---
name: generate-backlog
description: Generates structured Epics and User Stories from a spec, PRD, or feature list, with optional issue creation. Use when turning requirements into a ready-to-develop backlog.
disable-model-invocation: true
argument-hint: <requirements-file> [--dry-run] [--lang=<code>] [--release=<name>] [--type=web|mobile|ai] [--greenfield]
---

STARTER_CHARACTER = 🗂️

Uses a PM agent + QA validation loop to produce INVEST-quality Epics and Stories. Optionally creates issues if a project management MCP is available.

## Arguments

```
/generate-backlog <requirements-file> [--dry-run] [--lang=<code>] [--release=<name>] [--type=web|mobile|ai] [--greenfield]
```

- `<requirements-file>` — path to the requirements doc (default: `requisitos-de-proyecto.md`)
- `--dry-run` — generate and validate without creating issues
- `--lang=<code>` — language for all generated content (default: `en`)
- `--release=<name>` — assign all stories to this release/fix version
- `--type=web|mobile|ai` — include stack-specific technical foundation stories
- `--greenfield` — new project: include full technical foundation (all base + type-specific stories)

**Language rule**: all generated content (titles, descriptions, ACs) must be in a single language. No mixing under any circumstances.

## Execution Flow

### Step 1: Read requirements

Read the file provided in `$ARGUMENTS`. If not provided, use `requisitos-de-proyecto.md`.

### Step 2: Load technical story templates (if --greenfield or --type)

Look for templates in this order (first found wins):
1. Project-local: `.claude/templates/technical-stories/`
2. Skill default: `references/templates/` (always available)

Load `base.md` (or `_base.md`) plus the stack-specific file if `--type` is specified (`web.md`, `mobile.md`, `ai.md`).

If `--greenfield` is absent and `--type` is absent: assume an existing project. Skip this step. Generate functional stories only.

### Step 3: Run PM Agent

Read [references/pm-agent.md](references/pm-agent.md) for the full agent prompt and output schema.

Output: JSON with `metadata`, `epics[]`, and `stories[]`.

### Step 4: Run QA Agent

Read [references/qa-agent.md](references/qa-agent.md) for the full agent prompt and validation rules.

Input: PM agent JSON. Output: enriched JSON with `approved` flags and `investScore` per story.

### Step 5: Create issues (unless --dry-run)

**If `--dry-run`** or no issue tracker MCP is available: display the final JSON and a structured summary. Done.

**Otherwise**, discover project config via MCP before creating anything:
1. Get accessible resources to find the workspace/cloud ID
2. Get visible projects — if multiple exist, ask the user which one to use
3. Get issue type metadata for that project to identify Epic and Story type IDs

**Creation strategy — two sequential phases, parallel within each:**

```
Phase 1: Create ALL approved Epics in parallel
         ↓ wait for all to complete, collect keys
Phase 2: Create ALL approved Stories in parallel
         (each links to its parent Epic key from Phase 1)
```

If the MCP reports rate limiting, batch into groups of 5–10.

For each Story: set story points if the field is available, assign release if `--release` was specified.

**Show summary when done:**
```
## Created
- [EPIC-KEY] Epic Title (N stories)
  - [STORY-KEY] Story Title

## Summary
Epics: X created | Stories: Y created
```
