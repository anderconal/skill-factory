# Lens: Conditionals & Boolean Logic

Branching logic that obscures the main path, booleans that require mental gymnastics, or type-switching scattered across the codebase.

## The Question

Can the reader see the main path at a glance? Is branching logic as simple as it can be?

## How to Spot

- Deep nesting: the happy path is buried under layers of if/else
- Complex booleans: `if a && !b || c && d` — conditions that lack a name for what they mean
- Repeated switching: the same conditional pattern appears in multiple places
- Null checks scattered everywhere instead of handling the absence once

## Process

Start with the lightest tools:
1. Flatten nesting with guard clauses — handle edge cases early, return immediately, let the main path stand un-indented
2. Name complex booleans — extract conditions into explaining variables or predicate functions
3. Consolidate — if multiple conditions lead to the same result, they are one concept, name it
4. Replace value-mapping conditionals with a lookup table or map
5. Consider polymorphism only when the same type-switching repeats across multiple locations

## Trade-off

A simple conditional is not a smell. Guard clauses and if/else both have their place — use guards when one path is clearly exceptional, if/else when both paths are equally important. Polymorphism is the heaviest tool in this toolkit — reach for it when simpler options are not enough, not as the first response.

## Go Deeper

What other conditionals are hiding complexity? Where do boolean expressions lack a name? Where is the same switching logic repeated?
