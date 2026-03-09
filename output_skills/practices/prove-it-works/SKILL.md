---
name: prove-it-works
description: "Requires observable runtime evidence before declaring done or approving a review. Per artifact type, executes appropriate live verification: navigates real UI and checks console, calls actual endpoints and validates responses, runs commands and reads output, checks pipeline status. Static analysis passing is necessary but not sufficient."
---

STARTER_CHARACTER = 🔬

## Core principle

Static analysis passing (tests, lint, type-check, build) is necessary but not sufficient. Before declaring done or approving a review, produce at least one piece of observable runtime evidence: something that ran, rendered, responded, or reported — and was observed.

"Tests pass" is not evidence. "Called GET /api/orders/123, received 200 with correct payload, no errors in logs" is evidence.

## When this applies

- Finishing a task or story (about to declare done)
- Before marking a PR ready for review
- During code review — local checkout or PR

## Verification by artifact type

**UI / Frontend**
- Open the actual interface in a browser, navigate to the changed feature, interact with it
- Browser console: zero errors, zero unhandled network failures
- Visual output matches intent
- Network tab: no unexpected 4xx/5xx

**API / Backend**
- Call the actual running endpoint — not the test, the live service
- Validate: status code, response shape, key field values
- Check server logs for errors or warnings during the call
- Verify edge cases explicitly stated in ACs

**CLI tools**
- Execute the command with realistic input
- Read actual output — not what it should output, what it does
- Check exit code
- Verify side effects (files created/modified, state changed)

**BDD**
- Run the feature files against the actual implementation
- All scenarios green — read scenario output, not just the pass/fail count
- A skipped or pending scenario is not done

**Background jobs / workers**
- Trigger the job
- Confirm it executed: logs, queue state, callback
- Verify output and side effects match intent

**Events / messaging**
- Publish the event
- Confirm the consumer received it
- Check downstream effects are correct

**Database changes**
- Query the database after the operation
- Verify data state matches intent — not just that the migration ran

**Integrations**
- Call the real external service, or the highest-fidelity available
- Verify the integration contract holds in both directions

**CI/CD pipeline**
- Check pipeline status — all stages, not just the last one
- No stages failing, no warnings treated as errors
- Build artifacts produced correctly if applicable

## Stating evidence

For each verification, state what you ran or opened and what you observed. One sentence is enough.

## Anti-patterns

- Declaring done after tests pass without runtime verification
- "The logic looks correct" — inference is not observation
- Skipping UI check because unit tests cover the logic
- Trusting CI alone without checking what it actually ran
- Approving a PR without pulling and running the change
