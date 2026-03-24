---
name: create-story
description: Guides creation of a single User Story interactively: checks for duplicates, validates INVEST and ACs, then optionally creates in an issue tracker. Use when adding one story to the backlog.
disable-model-invocation: true
argument-hint: [brief description]
---

STARTER_CHARACTER = 📝

Guided single-story creation. Interactive by design — ask, validate, confirm before creating anything.

## Step 1: Check for duplicates

If an issue tracker MCP is available, extract keywords from `$ARGUMENTS` or ask the user for them, then search for similar existing stories.

If similar stories are found, show them and ask: use an existing one or create new? If the user picks an existing story, suggest `/start-work <key>` and stop.

If no MCP is available, skip this step.

## Step 2: Collect story details (Card)

Ask for the three elements — who, what, and why:

- **Who**: the user role (e.g., registered user, visitor, admin)
- **What**: the action or capability they need
- **Why**: the concrete benefit — not abstract ("better experience" is not a benefit). Push for specific outcomes: "so I can close the app knowing I won't miss anything important" ✓

Build the story title and description from these answers:
`As a [who], I want [what], so that [why]`

## Step 3: Define acceptance criteria

Collect at least 2–3 ACs. For each one, validate specificity before accepting it.

Reject and push back on vague ACs:
- "shows posts ordered" → which field? ASC or DESC? how many?
- "shows empty message" → what text? any CTA?
- "content can expand" → when truncated? at how many chars?

Require: field names, character limits, copy, timeouts, order direction — anything that could be interpreted multiple ways must be specified.

Format: `Given [context], when [action], then [result]`

## Step 4: Validate INVEST

Check all six principles. Flag any that fail before proceeding.

**Small is the most commonly violated** — check explicitly: does any AC deliver verifiable value on its own? If yes, it should be a separate story.

Size heuristic: shippable to production in 1 day or less. If it can't ship tomorrow, split before continuing.

State separation: happy path, empty state, and error handling each deliver independent value — keep them as separate stories unless genuinely trivial.

If the story fails Small: propose the specific split and restart from Step 2 with the first piece.

## Step 5: Estimate

Ask for story points using Fibonacci: 1, 2, 3, 5, 8, 13.

If the estimate is 13+, the story is too large — propose splitting before creating.

## Step 6: Preview and confirm

Show the full story before creating anything:

```
Title: [title]
As a [who], I want [what], so that [why]

Acceptance Criteria:
- [ ] Given..., when..., then...
- [ ] ...

INVEST: I✓ N✓ V✓ E✓ S✓ T✓
Estimate: X points
```

Ask: create in issue tracker, edit something, or cancel?

## Step 7: Create in issue tracker

If the user confirms and an issue tracker MCP is available:
- Discover workspace and project via MCP (ask user if multiple projects exist)
- Create the story with title, description, ACs, and story points
- Show the created issue key and link

If no MCP is available, output the story as markdown for manual entry.

## Step 8: Offer next step

After creation, ask: start working on this story now? If yes, proceed with the `/start-work` flow.
