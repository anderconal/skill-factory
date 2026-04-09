---
name: hotspot-analysis
description: Scores and classifies git hotspots by defect density, churn, on-call commits, and fix-chain clusters, then delivers a quadrant-classified HTML report with function-level drill-down. Use when prioritizing refactoring, identifying risky files, or preparing a codebase health review.
allowed-tools: Bash, Write, Read
---

STARTER_CHARACTER = 🔬

When starting, announce: "🔬 Using HOTSPOT-ANALYSIS skill".

Identifies WHICH files break most — scored, quadrant-classified, acceleration-tracked, on-call-annotated, and drilled to function level. Writes structured CSVs to `.claude/hotspots/` that `/temporal-coupling` reads. Produces a scored HTML report.

## Step 0: Declare configuration

Before extracting any data, ask or infer:

1. **Maintenance mode?** (no active feature development, only incident response) — changes all ACTIVE WORK quadrant interpretations
2. **Branch scope** — which branches to include (e.g., `develop master training`). Required: do not assume.
3. **Time window** — default: **all history**, extracted with monthly granularity. Recent months are weighted higher in scoring; this is the default and should not be shortened unless the repo history is too large to process.
4. **Output target** — HTML file path

Gate: If maintenance mode is not explicitly confirmed, emit a visible WARNING block in the final report. Never silently assume normal-development mode.

## Step 1: Extract raw signals

Run all extraction commands from [references/methodology.md](references/methodology.md). Do not substitute your own — branch scope, date ranges, and grep patterns are exact and their consistency across runs matters.

Required per-file signals:
- churn_total, churn_6m, churn_7_12m
- defect_total, defect_6m, defect_7_12m
- oncall_count (weekends + 22:00–05:59 local time)
- unique_author_count
- revert_count
- null_commits, sql_commits, notification_commits, concurrency_commits, auth_commits (see [references/methodology.md — Error Class Extraction](references/methodology.md))

Gate: Every file in the final matrix must have values for all signals. If a file has only partial data, flag it explicitly — do not default to zero.

## Step 2: Compute the hotspot score

Exact formula — do not simplify, do not substitute:

```
hotspot_score = 0.50 × defect_score
              + 0.25 × churn_score
              + 0.20 × oncall_score
              + 0.05 × author_score
```

The input to log-normalization is not raw churn_total but a **time-weighted sum** that gives more importance to recent history:

```
weighted_churn  = churn_6m × 3.0  +  churn_712m × 2.0  +  churn_older × 0.5
weighted_defect = defect_6m × 3.0 + defect_712m × 2.0 + defect_older × 0.5
```

where `churn_older = churn_total − churn_6m − churn_712m`.

All weighted values are then log-normalized: `score(x) = log(1 + x) / log(1 + max_x_in_dataset)`

Log normalization prevents a single outlier from compressing all other scores toward zero.

Additional derived metrics:
- **Acceleration**: `defect_6m / defect_712m`. If `defect_712m == 0`, mark as `"new"` — do not divide by zero.
- **recency_ratio_defect**: `defect_6m / defect_total`
- **peak_month**: month with the highest defect commit count (from monthly breakdown in file_monthly.csv — ACTIVE CRISIS files only)
- **On-call weight**: `log(1 + oncall_count) / log(1 + max_oncall) × 3`

Gate: Show the formula, intermediate normalized values (norm_defect, norm_churn, norm_oncall), and the final score together. A number without component breakdown cannot be challenged or verified.

## Step 3: Classify quadrant

Use percentile rank within the dataset. Fixed thresholds (e.g., ≥ 0.5) break on well-maintained codebases where every score falls below the cutoff and nothing is classified.

- **ACTIVE CRISIS** — defect_score in top 25% of dataset
- **DORMANT DEBT** — defect_score in top 25%, churn_score in bottom 50%
- **ACTIVE WORK** — churn_score in top 25%, defect_score in bottom 50%
- **SAFE ZONE** — all remaining files

**Maintenance mode override**: ACTIVE WORK does NOT mean safe. Reclassify as **MAINTENANCE INCIDENT RESPONSE** — churn in maintenance mode equals repeated emergency patching with non-standard commit messages, not feature development.

## Step 4: Detect fix-chain clusters

For each ACTIVE CRISIS file, extract all defect commits and compute inter-commit gaps. Use the exact command in [references/methodology.md — Fix-Chain Extraction](references/methodology.md).

A fix-chain cluster = ≥3 defect commits to the same file within ≤15 days.

Report per cluster: start date, end date, length in days, commit count, whether any commit occurred in on-call hours.

Gate: Count clusters exactly from raw commit dates. "Approximately 3 clusters" is not acceptable.

## Step 5: Function-level drill-down (ACTIVE CRISIS files only)

For each ACTIVE CRISIS file, identify which functions accumulated the most defect commits in the last 2 years. Use the git log + grep approach in [references/methodology.md — Function Drill-Down](references/methodology.md) when lizard is unavailable.

Dominant error class per function is derived from whichever of null_commits, sql_commits, notification_commits, concurrency_commits, auth_commits is highest for that file's defect commits.

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

Write results to `.claude/hotspots/workflow_deviations.csv` using the schema in [references/methodology.md — CSV Schemas](references/methodology.md).

## Step 7: Generate the HTML report

Use the dark CSS theme defined in [references/methodology.md — HTML Theme](references/methodology.md). Both hotspot-analysis and temporal-coupling must use the same theme so linked reports are visually consistent.

All sections are required. Do not omit any.

1. **Methodology box** — formula, weights, data sources, time window, commit count, branch scope
2. **Top N matrix table** — all files above threshold, all component scores visible
3. **Quadrant scatter chart** — ASCII or rendered, defect_score vs churn_score axes
4. **Acceleration callouts** — files with acceleration > 1.5 or marked "new", with trend direction
5. **ACTIVE CRISIS cards** — one card per crisis file: function-level table, fix-chain count, on-call incidents, last incident date, dominant error class
6. **Fix-chain timeline** — top 3 files: chronological fix clusters with dates and inter-cluster gaps
7. **Workflow deviation table** — all direct-to-master or on-call commits found
8. **Maintenance mode banner** — if set: all ACTIVE WORK reclassifications made explicit

Gate: Do not deliver the report if any ACTIVE CRISIS file is missing its function-level table. Run the drill-down first.

## Step 8: Persist structured CSVs

After generating the report, write all intermediate data to `.claude/hotspots/` so `/temporal-coupling` can read it without re-extracting.

Files to write — use the exact column schemas in [references/methodology.md — CSV Schemas](references/methodology.md):

- `.claude/hotspots/data/develop-backend.csv` — per-file signals for backend files (from Step 1)
- `.claude/hotspots/data/develop-frontend.csv` — per-file signals for frontend files (from Step 1)
- `.claude/hotspots/matrix.csv` — scored matrix with quadrant (from Steps 2–3)
- `.claude/hotspots/fix_chains.csv` — fix-level records, one row per defect commit per file (from Step 4)
- `.claude/hotspots/oncall_commits.csv` — classified on-call commits, full 13-column schema (from Step 1)
- `.claude/hotspots/file_monthly.csv` — per-file monthly defect counts for ACTIVE CRISIS files (from Step 1)
- `.claude/hotspots/workflow_deviations.csv` — deviation records (from Step 6)

Gate: If any file cannot be written, halt and report the error before proceeding. `/temporal-coupling` will fail its prerequisite gate if any file is missing.

## Anti-patterns (enforced, not advisory)

- Never report a hotspot score without the component breakdown. A number without weights is not verifiable.
- Never say "approximately N commits" — use the exact git command and report the exact count.
- Never classify maintenance-mode churn as feature work or safe.
- Never omit on-call commits — these are the highest-signal data points in the dataset.
- Never claim a DI or architecture change fixes a performance problem without verifying the actual incident root cause. Dependency injection patterns do not fix N+1 queries or connection scope errors — those are query-structure problems independent of instantiation pattern.

## Hard constraints

1. Every claim cites a commit SHA or a specific data row. "Approximately" and "seems to" are not citations.
2. When raw data and derived reports conflict, cite the raw data.
3. Maintenance mode must be declared before interpretation begins.
4. Architecture claims are bounded to what they actually fix — no scope inflation.
5. Output in English regardless of session language.

## Supporting files

- [references/methodology.md](references/methodology.md) — exact git extraction commands, error class patterns, CSV schemas, HTML theme, function drill-down, acceleration guard, workflow deviation query
