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

**Otherwise**, start with a single upfront prompt to the user before any MCP call:

> "To create issues in Jira I need:
> - **Project key** (e.g. `ABC`)
> - **Issue type names**: I'll default to **Epic** and **User Story**. If your project uses different names, tell me now."

Then run all discovery calls in parallel:
1. Get accessible resources to find the workspace/cloud ID
2. Get issue type metadata for the specified project → match names to get Epic and Story type IDs
3. Get field metadata for the **User Story** issue type → extract field IDs for:
   - Story points (key/name containing `story_point`)
   - Acceptance criteria (key/name containing `acceptance`)
   - Fix Versions (key/name `fixVersions` or name "Fix versions")
4. Get available issue link types → identify the one representing "is blocked by" / "depends on" (look for name containing `block` or `depend`)

**Fallback rule for missing fields:** if a custom field for story points, acceptance criteria, or fix versions is not found in the project, include the value in the description instead of omitting it. All data must appear somewhere — never silently dropped.

**Description format — always ADF:**

All descriptions must be Atlassian Document Format (ADF). Never send plain text with `\n` — it renders literally.

Build the description ADF document in this order:
1. User story sentence (`As a [role]...`)
2. Story points line — only if no dedicated field found (e.g. `Story Points: 3`)
3. Fix versions line — only if no dedicated field found (e.g. `Fix Versions: v1.0`)
4. Acceptance Criteria bullet list — only if no dedicated field found

```json
{
  "version": 1,
  "type": "doc",
  "content": [
    { "type": "paragraph", "content": [{ "type": "text", "text": "As a [role], I want [action], so that [benefit]." }] },
    { "type": "paragraph", "content": [{ "type": "text", "text": "Acceptance Criteria:" }] },
    {
      "type": "bulletList",
      "content": [
        { "type": "listItem", "content": [{ "type": "paragraph", "content": [{ "type": "text", "text": "Given X, when Y, then Z" }] }] }
      ]
    }
  ]
}
```

**Creation strategy — batch, dependency-aware:**

```
Phase 1: Bulk-create ALL approved Epics → collect Epic keys

Phase 2: Topologically sort stories by dependsOn:
  Level 0: stories with no dependsOn → bulk-create, collect their keys
  Level N: stories whose dependsOn are all already created → bulk-create,
           then add the discovered link type to each dependency pair
  Repeat until all stories are created
```

**Show summary when done:**
```
## Created
- [EPIC-KEY] Epic Title (N stories)
  - [STORY-KEY] Story Title

## Summary
Epics: X created | Stories: Y created
```
