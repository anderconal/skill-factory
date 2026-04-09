---
name: hotspot-analysis
description: Scores and classifies git hotspots by defect density, churn, on-call commits, and fix-chain clusters, then delivers a quadrant-classified HTML report with function-level drill-down. Use when prioritizing refactoring, identifying risky files, or preparing a codebase health review.
disable-model-invocation: true
allowed-tools: Bash
---

STARTER_CHARACTER = 🔬

When starting, announce: "🔬 Using HOTSPOT-ANALYSIS skill".

Identifies WHICH files break most — scored, quadrant-classified, acceleration-tracked, on-call-annotated, and drilled to function level. Output is a scored HTML report. Precedes `/temporal-coupling`.

## Step 0: Declare configuration

Before extracting any data, ask or infer:

1. **Maintenance mode?** (no active feature development, only incident response) — changes all ACTIVE WORK quadrant interpretations
2. **Branch scope** — which branches to include (e.g., `develop master training`). Required: do not assume.
3. **Time window** — default: last 12 months, split as 6m recent + 7–12m prior (for acceleration)
4. **Output target** — HTML file (path) or inline markdown summary

Gate: If maintenance mode is not explicitly confirmed, emit a visible WARNING block in the final report. Never silently assume normal-development mode.

## Step 1: Extract raw signals

Run all extraction commands from [references/methodology.md](references/methodology.md). Do not substitute your own — branch scope, date ranges, and defect grep patterns are exact and their consistency matters across runs.

Required per-file signals:
- churn_total, churn_6m, churn_7_12m
- defect_total, defect_6m, defect_7_12m
- oncall_count (weekends + 22:00–05:59 local time)
- unique_author_count

Gate: Every file in the final matrix must have values for all signals. If a file has only partial data, flag it explicitly — do not default to zero.

## Step 2: Compute the hotspot score

Exact formula — do not simplify, do not substitute:

```
hotspot_score = 0.50 × defect_score
              + 0.25 × churn_score
              + 0.20 × oncall_score
              + 0.05 × author_score
```

All component scores are log-normalized: `score(x) = log(1 + x) / log(1 + max_x_in_dataset)`

Log normalization prevents a single outlier (a file with 318 churn commits) from compressing all other scores toward zero.

Additional metrics:
- **Acceleration**: `defect_6m / defect_712m` — files where this > 1.5 are in active escalation regardless of absolute score
- **On-call weight**: `log(1 + oncall_count) / log(1 + max_oncall) × 3` — the ×3 balances typically small raw oncall counts against churn/defect volume

Gate: Show the formula, intermediate normalized values (norm_defect, norm_churn, norm_oncall), and the final score together. A number without component breakdown cannot be challenged or verified.

## Step 3: Classify quadrant

Based on defect_score and churn_score:

- **ACTIVE CRISIS** — defect_score ≥ 0.5, any churn_score
- **DORMANT DEBT** — defect_score ≥ 0.5, churn_score < 0.3
- **ACTIVE WORK** — defect_score < 0.3, churn_score ≥ 0.5
- **SAFE ZONE** — defect_score < 0.3, churn_score < 0.3

**Maintenance mode override**: ACTIVE WORK does NOT mean safe. Reclassify as **MAINTENANCE INCIDENT RESPONSE** — churn in maintenance mode equals repeated emergency patching with non-standard commit messages, not feature development.

## Step 4: Detect fix-chain clusters

For each ACTIVE CRISIS file, extract all defect commits and compute inter-commit gaps. Use the exact command in [references/methodology.md — Fix-Chain Extraction](references/methodology.md).

A fix-chain cluster = ≥3 defect commits to the same file within ≤15 days.

Report per cluster: start date, end date, length in days, commit count, whether any commit occurred in on-call hours.

Gate: Count clusters exactly from raw commit dates. "Approximately 3 clusters" is not acceptable.

## Step 5: Function-level drill-down (ACTIVE CRISIS files only)

For each ACTIVE CRISIS file, identify which functions accumulated the most defect commits in the last 2 years. Use the git log + grep approach in [references/methodology.md — Function Drill-Down](references/methodology.md) when lizard is unavailable.

Report per top function:
- Defect commit count
- Fix% — defect commits as share of total commits to that function
- Dominant error class: Null, SQL/Perf, Notification, Concurrency, or Auth

## Step 6: Detect workflow deviations

Find direct-to-master commits that bypass the hotfix branch. See [references/methodology.md — Workflow Deviations](references/methodology.md) for the exact command.

Classify each commit as:
- `direct_to_master` — commit on master not arriving via merge
- `on_call_dtm` — direct-to-master during on-call hours
- `training_only` — commit on training branch not mirrored to master

## Step 7: Generate the HTML report

All sections are required. Do not omit any.

1. **Methodology box** — formula, weights, data sources, time window, commit count, branch scope
2. **Top N matrix table** — all files above threshold, all component scores visible
3. **Quadrant scatter chart** — ASCII or rendered, defect_score vs churn_score axes
4. **Acceleration callouts** — files with acceleration > 1.5, with trend direction
5. **ACTIVE CRISIS cards** — one card per crisis file: function-level table, fix-chain count, on-call incidents, last incident date, dominant error class
6. **Fix-chain timeline** — top 3 files: chronological fix clusters with dates and inter-cluster gaps
7. **Workflow deviation table** — all direct-to-master or on-call commits found
8. **Maintenance mode banner** — if set: all ACTIVE WORK reclassifications made explicit

Gate: Do not deliver the report if any ACTIVE CRISIS file is missing its function-level table. Run the drill-down first.

## Anti-patterns (enforced, not advisory)

- Never report a hotspot score without the component breakdown. A number without weights is not verifiable.
- Never say "approximately N commits" — use the exact git command and report the exact count.
- Never classify maintenance-mode churn as feature work or safe.
- Never omit on-call commits — these are the highest-signal data points in the dataset.
- Never claim a DI or architecture change fixes a performance problem without verifying the actual incident root cause. Dependency injection patterns do not fix N+1 queries or connection scope errors — those are query-structure problems independent of instantiation pattern.

## Hard constraints

These constraints derive from explicit session corrections and are not negotiable:

1. Every claim cites a commit SHA or a specific data row. "Approximately" and "seems to" are not citations.
2. When raw data and derived reports conflict, cite the raw data.
3. Maintenance mode must be declared before interpretation begins.
4. Architecture claims are bounded to what they actually fix — no scope inflation.
5. Output in English regardless of session language.

## Supporting files

- [references/methodology.md](references/methodology.md) — exact git extraction commands, function drill-down approach, workflow deviation query
