---
name: user-story-craft
description: "Shapes User Story quality: INVEST validation, specific ACs (Given/When/Then), vertical slicing, state separation (happy/empty/error). Accepts User Story or Job Story formats. Applies PM-with-ownership (not requirements transcription). Use when writing, reviewing, or discussing stories, features, or acceptance criteria."
---

STARTER_CHARACTER = 📋

Applies PM judgment to features — not transcription, but product decisions.

## Story Format

Both formats are valid. What must be present regardless of format: **who/situation**, **what they need**, **why it matters concretely**.

**User Story:** As a [role], I want [action], so that [concrete outcome]

**Job Story:** When [situation], I want to [motivation], so I can [concrete outcome]

Reject abstract benefits — "for a better experience" is not a benefit. Push for concrete outcomes: "so I can close the app knowing I won't miss anything important" ✓

## Acceptance Criteria

Given/When/Then when possible. Non-negotiable: every AC must be verifiable without interpretation.

Reject these patterns:
- "shows post info" → which fields? sizes? format?
- "ordered" → by what field? ASC or DESC?
- "shows empty message" → what text? any CTA?
- "content can expand" → when does it truncate? at how many chars? what triggers expansion?

Require specifics: field names, character limits, exact copy, timeouts, px dimensions when layout-critical.

## INVEST

**Small** is the most violated principle — check it explicitly. Ask for each criterion: "Does this AC deliver verifiable value on its own?"

- Yes → separate story
- No → keep as criterion

**Size heuristic**: a story must be shippable to production in 1 day or less. 2 days is an exception. 3 days maximum for genuinely complex work. If it can't ship tomorrow, it needs splitting before writing ACs.

**Detect while writing, don't write big then suggest splitting.** If a criterion triggers "this could stand alone", stop and split before continuing.

The other five:
- **Independent**: no dependency on unstarted work
- **Negotiable**: scope open to discussion
- **Valuable**: user-visible outcome
- **Estimable**: team can size it
- **Testable**: every AC has a clear pass/fail

## Vertical Slicing

Every story cuts all layers. Never split frontend/backend into separate stories. A story that only touches the API with no visible outcome is not a story.

## State Separation

Think in states, split when each delivers independent value:
- Happy path — always first
- Empty state — next
- Error state — after that
- Loading state — fold into happy path unless it's complex enough to stand alone

## PM-with-ownership

Don't transcribe requirements. Take decisions:
- Vague requirement → decide and document the reasoning
- Multiple valid approaches → pick one, explain why
- Genuinely unclear → mark inline as `PENDING DECISION: X or Y?`

Specify everything that could be interpreted multiple ways: data fields, limits, copy, behaviors, timeouts. If it's ambiguous, it's not done.

## Anti-patterns

- Story mixes happy path + empty state + error handling
- AC says "works" or "is configured" without specifying what that means
- Separate frontend and backend stories for the same feature
- Benefit is abstract ("better UX", "improved performance" without a number)
- Story cannot ship to production in 1 day (needs splitting first)
