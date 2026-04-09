---
name: temporal-coupling
description: Classifies file pairs as causal, seasonal, structural-recurrence, or coincidental using co-commit frequencies and inter-incident gap analysis. Run after /hotspot-analysis to explain WHY hotspot files keep breaking together.
disable-model-invocation: true
allowed-tools: Bash
---

STARTER_CHARACTER = 🔗

When starting, announce: "🔗 Using TEMPORAL-COUPLING skill".

Answers WHY ACTIVE CRISIS files keep breaking — distinguishing accidental co-occurrence from structural coupling from seasonal reactivation. Every verdict requires specific commit SHAs. Runs after `/hotspot-analysis`.

## Prerequisite

Requires hotspot data in `.claude/hotspots/data/`. If not present, run `/hotspot-analysis` first.

Required CSV files:
- `develop-backend.csv` / `develop-frontend.csv`
- `fix_chains.csv`
- `oncall_commits.csv`
- `seasonal.csv`

Gate: Verify all files exist and are non-empty before proceeding. If any are missing, halt with a clear error message. Do not substitute ad-hoc git log queries — the CSVs contain pre-classified data that git log alone cannot reconstruct.

Also confirm branch scope (e.g., `develop master training`) before running any queries.

## Step 1: Logical coupling — co-commit frequencies

For each pair of ACTIVE CRISIS files, count commits that touched both simultaneously. Use the exact command in [references/methodology.md — Co-Commit Query](references/methodology.md).

Report per pair:
- Co-commit count (exact)
- Percentage of the smaller file's history that includes the pair
- Direction of dependency — which file changes first when they co-change

Coupling threshold: ≥10 co-commits OR ≥30% of either file's commit history = structurally coupled. Below = coincidental.

Gate: Report both the raw count AND the percentage. 10 co-commits in a 200-commit file (5%) is different from 10 co-commits in a 15-commit file (67%). Both numbers are required.

## Step 2: Seasonal coupling — the 300–400 day pattern

For each ACTIVE CRISIS file, extract defect commit dates and compute inter-incident gaps. Use the exact awk command in [references/methodology.md — Seasonal Gap Query](references/methodology.md).

Flag gaps in the 200–500 day range as seasonal candidates.

Report per gap:
- Length in days
- The two bracketing incidents (dates + commit SHAs)
- Bug message for each incident
- Whether they share the same bug class

Gate: Do not label a gap "seasonal" without verifying the two incidents share the same structural cause — same error class, same function, same fix pattern. A 369-day gap between unrelated bugs is coincidental, not seasonal.

## Step 3: Extract confirmed temporal chains

A temporal chain = a sequence of fixes across different files within ≤15 days where each fix exposes a new problem revealed by the previous fix.

Look for on-call window overlaps across ACTIVE CRISIS files. Use the command in [references/methodology.md — Temporal Chain Query](references/methodology.md).

Report per chain:
- Files in sequence order
- Date and time of each fix
- Gap between consecutive fixes
- On-call indicator per commit

Gate: A chain requires minimum 3 fixes across 2+ files. A single multi-file commit is a large changeset, not a temporal chain. Do not conflate them.

## Step 4: Classify each coupling

For every file pair from Steps 1–2, produce an explicit verdict with evidence:

**CAUSAL**
Criteria: high co-commit rate + confirmed temporal chains + architectural dependency.
Evidence required: co-commit %, chain with SHAs, the specific interface or contract linking the files.

**SEASONAL**
Criteria: low co-commit rate + 200–500 day gap + same bug class reactivation.
Evidence required: gap in days, two incident dates, same error class confirmed.

**STRUCTURAL RECURRENCE**
Criteria: low co-commit rate between files, but same bug recurs independently from a shared structural flaw (missing ORM filter, absent shared utility, etc.).
Evidence required: the structural condition + at least 2 dated instances.

**COINCIDENTAL**
Criteria: low co-commit rate + no temporal chains + different bug classes.
Evidence required: explicit absence of evidence for the three categories above.

Gate: Every verdict must cite specific commit SHAs or CSV rows. "Files seem related" is not a verdict. "121/176 co-commits (70%), notification contract change confirmed in commit 810c2d67" is a verdict.

## Step 5: Identify recurring bug classes

For each ACTIVE CRISIS file cluster, extract the bug class pattern across years using the command in [references/methodology.md — Bug Class by Year](references/methodology.md).

Group by year. Identify:
- Same root structural cause across multiple years
- How many times it recurred
- What was patched each time vs what structural condition remained unchanged

Gate: Do not claim a structural cause without demonstrating at least 2 separate year-instances of the same bug class. Two instances are coincidence. Three or more with the same bug class in the same month range is a pattern.

## Step 6: Root cause structural analysis

Only after all coupling types are classified, identify the structural conditions not yet eliminated.

For each STRUCTURAL RECURRENCE bug class, answer three questions with data:
1. How many instances? — count from git history, provide dates
2. What was patched each time? — from commit messages, be specific
3. What structural condition remains unchanged? — the terrain that keeps producing the bug

Gate: The root cause section must be grounded in the bug class table from Step 5. It cannot introduce new claims not supported by the temporal chains and co-commit data. Architecture inferences (DI patterns, shared utilities) must be bounded to what the incident data actually shows they would fix.

## Step 7: Generate the HTML report

All sections are required. Do not omit any.

1. **The central question** — stated explicitly (e.g., "Does fixing X cause Y to break later?")
2. **Data sources box** — which CSV files, which git log range, commit count
3. **Two-mechanism summary** — dominant cause vs secondary cause, both with percentage estimates backed by data
4. **Logical coupling bar chart** — co-commit counts for top pairs, percentage of history shown
5. **Seasonal coupling section** — per-year breakdown, gaps in days, confirmed recurrences
6. **Confirmed temporal chains** — one visual chain per incident: node per file, gap label between nodes, on-call indicator
7. **Recurring bug class table** — all years, root structural cause per class
8. **Verdict table** — causal/seasonal/structural/coincidental per pair, with evidence column
9. **Root cause section** — ordered by confidence, backed by post-2023 evidence only if confirmed
10. **Raw counts appendix** — commit totals per file, on-call pattern breakdown, quarterly trend table

## Anti-patterns (enforced, not advisory)

- Never say correlation implies causation without checking co-commit rates. Two files both breaking in fire season with 0 co-commits are coincidental, not coupled.
- Never claim architecture improvements fix performance problems without checking the actual incident root cause. DI patterns do not fix N+1 queries or connection scope errors — those are query-structure problems.
- Never report a seasonal pattern from a single year-pair. Two instances are coincidence; three or more with the same bug class in the same month range is a pattern.
- Never combine logical-coupling and seasonal-coupling verdicts in one finding. They are different mechanisms with different evidence requirements.

## Hard constraints

These constraints derive from explicit session corrections and are not negotiable:

1. Every claim cites a commit SHA or a specific CSV row. "Approximately" and "seems to" are not citations.
2. Raw CSVs take precedence over derived reports — if they conflict, cite the CSV.
3. Maintenance mode must be declared before interpretation begins.
4. Architecture claims are bounded to what they actually fix — no scope inflation.
5. Output in English regardless of session language.
6. Causal, seasonal, structural-recurrence, and coincidental are distinct categories with different evidence requirements. Do not mix them in one verdict.

## Workflow position

Runs after `/hotspot-analysis`. Hotspot analysis identifies WHICH files are risky; temporal coupling explains WHY they keep breaking together. Running temporal coupling without a prior hotspot score produces coupling observations without risk-weighted context — the results are unanchored.

## Supporting files

- [references/methodology.md](references/methodology.md) — exact git commands for co-commit analysis, seasonal gap detection, temporal chain extraction, and bug class grouping
