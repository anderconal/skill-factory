# Lens: Conditionals & Boolean Logic (Reviewer Guide)

## When to Apply

After semantic clarity. Now that the code tells its story, look at branching logic — is the main path visible, or buried under conditions?

## What to Look For in Diffs

- Nested if/else flattened into guard clauses with early returns
- Complex boolean expressions extracted into named variables or predicate functions
- Multiple conditions with the same outcome consolidated into one named check
- Type-switching replaced with lookup tables or maps
- Scattered null checks replaced with a special case or default object

## First Pass

Worker should find the obvious: deep nesting that can be flattened with guard clauses, unnamed boolean expressions, repeated null checks.

## Second Pass

If conditional complexity remains:
- Look for boolean expressions the worker left inline — do they have a name in the domain?
- Check for the same switching pattern appearing in multiple places — that is the signal for polymorphism, not a single conditional
- Look for guard clauses that could be consolidated — if three guards return the same thing, they are one concept

## Push Back On

- Premature polymorphism: if the worker introduces a class hierarchy for a conditional that only appears once, push back — a guard clause or lookup table is likely simpler
- Too many guard clauses: if a function has 7+ guards, the problem is not the conditionals but the function doing too much

## When Done

Move on when the main path is visible at a glance and branching logic is as simple as it can be.
