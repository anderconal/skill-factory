# Hotspot Analysis — Extraction Commands and Schemas

Replace `BRANCHES` with the confirmed branch scope (e.g., `develop master training`).

All history is extracted by default. Time-weighted scoring (Step 2) gives recent months higher importance; extracting all history lets the monthly breakdown in file_monthly.csv reveal multi-year patterns that would otherwise be invisible.

---

## Step 1: Churn Extraction

```bash
# All history
git log BRANCHES --format=format: --name-only \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/churn_total.txt

# Last 6 months
git log BRANCHES --since="6 months ago" --format=format: --name-only \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/churn_6m.txt

# 7–12 months ago
git log BRANCHES --since="12 months ago" --until="6 months ago" --format=format: --name-only \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/churn_712m.txt
```

`churn_older = churn_total − churn_6m − churn_712m` (computed in Step 2 — do not extract separately).

---

## Step 1: Defect Extraction

```bash
DEFECT_GREP="fix\|bug\|error\|hotfix\|patch\|revert\|crash\|broken\|null\|fail"

# All history
git log BRANCHES --format=format: --name-only --grep="$DEFECT_GREP" -i \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/defects_total.txt

# Last 6 months
git log BRANCHES --since="6 months ago" --format=format: --name-only --grep="$DEFECT_GREP" -i \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/defects_6m.txt

# 7–12 months ago
git log BRANCHES --since="12 months ago" --until="6 months ago" \
  --format=format: --name-only --grep="$DEFECT_GREP" -i \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/defects_712m.txt
```

---

## Step 1: Error Class Extraction

Run one command per class. These produce the null_commits, sql_commits, etc. columns in the matrix.

```bash
# Null / undefined errors
git log BRANCHES --format=format: --name-only \
  --grep="null\|undefined\|NullReference\|unloaded\|LoadWith" -i \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/error_null.txt

# SQL / performance errors
git log BRANCHES --format=format: --name-only \
  --grep="sql\|query\|performance\|N+1\|slow\|timeout\|connection\|index" -i \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/error_sql.txt

# Notification / email errors
git log BRANCHES --format=format: --name-only \
  --grep="notif\|email\|smtp\|notification\|SendNotif" -i \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/error_notification.txt

# Concurrency errors
git log BRANCHES --format=format: --name-only \
  --grep="deadlock\|race\|concurrent\|thread\|async\|lock\|mutex" -i \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/error_concurrency.txt

# Auth / permission errors
git log BRANCHES --format=format: --name-only \
  --grep="auth\|permission\|unauthorized\|forbidden\|access\|401\|403" -i \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/error_auth.txt
```

Dominant error class per file = the class with the highest count. Report all five values in the matrix.

---

## Step 1: Dominant Class Lookup

Run this immediately after the five error-class greps above, before generating `file_monthly.csv`. It builds a per-file lookup from the already-computed `/tmp/error_*.txt` files — no new git log queries.

```bash
# Label each error class file with its class name and merge
awk '{print $1, $2, "null"}' /tmp/error_null.txt > /tmp/all_classes.txt
awk '{print $1, $2, "sql"}' /tmp/error_sql.txt >> /tmp/all_classes.txt
awk '{print $1, $2, "notification"}' /tmp/error_notification.txt >> /tmp/all_classes.txt
awk '{print $1, $2, "concurrency"}' /tmp/error_concurrency.txt >> /tmp/all_classes.txt
awk '{print $1, $2, "auth"}' /tmp/error_auth.txt >> /tmp/all_classes.txt

# For each file, keep only the class with the highest count
sort -k2,2 -k1,1rn /tmp/all_classes.txt \
  | awk '!seen[$2]++ {print $2, $3}' > /tmp/dominant_class.txt
```

Output: one line per file — `<file> <dominant_class>`. Files absent from all five error-class outputs default to `unknown`.

---

## Step 1: Monthly Breakdown (file_monthly.csv)

Requires: Dominant Class Lookup completed above (`/tmp/dominant_class.txt` must exist).

Covers all history. One row per file × month. Run for each ACTIVE CRISIS file (peak_month is only reported in ACTIVE CRISIS cards, so full-history extraction for non-crisis files is not required).

```bash
# For each ACTIVE CRISIS file, look up its dominant class and get monthly defect commit counts
for FILE in <ACTIVE_CRISIS_FILE_1> <ACTIVE_CRISIS_FILE_2>; do
  DOM=$(awk -v f="$FILE" '$1==f {print $2; exit}' /tmp/dominant_class.txt)
  DOM=${DOM:-unknown}
  git log BRANCHES --format="%ad" --date=format:"%Y-%m" \
    --grep="$DEFECT_GREP" -i -- "$FILE" \
    | sort | uniq -c \
    | awk -v f="$FILE" -v dom="$DOM" '{print f "," $2 "," $1 "," dom}' >> /tmp/file_monthly_raw.txt
done
```

After running for all ACTIVE CRISIS files, write the result to `.claude/hotspots/file_monthly.csv` with the header `file,year_month,defect_count,dominant_bug_class`.

`dominant_bug_class` is the file-level dominant error class — the same value for every month row of that file. It comes from the Dominant Class Lookup above, not from per-month per-file git greps (which would require 5 × N × M commands and are not needed).

peak_month = the year_month row with the highest defect_count for that file.

---

## Step 1: Unique Authors per File

```bash
# For a specific file:
git log BRANCHES --format="%aN" -- <file> | sort -u | wc -l
```

---

## Step 1: Reverts per File

```bash
git log BRANCHES --format=format: --name-only \
  --grep="^[Rr]evert" \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/reverts_by_file.txt
```

---

## Step 1: On-Call Commits (13-column CSV)

```bash
git log BRANCHES --format="%H|%ad|%s|%aN" --date=format:"%Y-%m-%d|%H|%w" \
  | awk -F'|' '
    BEGIN { print "hash,date,message,author,day_of_week,hour,workflow_type,is_fix,files_touched,lines_changed,cherry_picked_to,is_revert,error_class" }
    {
      h=$3+0; dow=$4;
      gsub(/,/, " ", $5)
      is_fix  = ($5 ~ /fix|bug|error|hotfix|patch/) ? 1 : 0;
      is_rev  = ($5 ~ /^[Rr]evert/) ? 1 : 0;
      if (dow==0 || dow==6 || h>=22 || h<6)
        print $1 "," $2 "," $5 "," $6 "," dow "," $3 ",direct," is_fix ",0,0,," is_rev ",unknown"
    }' > .claude/hotspots/oncall_commits.csv
```

Column order: `hash,date,message,author,day_of_week,hour,workflow_type,is_fix,files_touched,lines_changed,cherry_picked_to,is_revert,error_class`

`files_touched` and `lines_changed` require a per-commit `git show --stat`. For large repos, compute only for the most recent 200 on-call commits.

```bash
# Extract the most recent 200 on-call hashes
tail -n +2 .claude/hotspots/oncall_commits.csv | cut -d',' -f1 | head -200 > /tmp/oncall_200.txt

# For each hash: git show --stat --format="" <hash> | tail -1
# Output: " N files changed, M insertions(+), K deletions(-)"
> /tmp/oncall_stat.txt
while IFS= read -r hash; do
  git show --stat --format="" "$hash" 2>/dev/null | tail -1 \
  | awk -v h="$hash" '{
      files=0; lines=0
      for (i=1; i<=NF; i++) {
        if ($i~/^[0-9]+$/ && $(i+1)~/^file/)      files=$i+0
        if ($i~/^[0-9]+$/ && $(i+1)~/^insertion/) lines+=$i+0
        if ($i~/^[0-9]+$/ && $(i+1)~/^deletion/)  lines+=$i+0
      }
      print h, files, lines
    }' >> /tmp/oncall_stat.txt
done < /tmp/oncall_200.txt

# Patch files_touched (col 9) and lines_changed (col 10) in-place
awk 'BEGIN{FS=OFS=","}
     FILENAME=="/tmp/oncall_stat.txt" {split($0,a," "); ft[a[1]]=a[2]; lc[a[1]]=a[3]; next}
     FNR==1  {print; next}
     {if ($1 in ft) {$9=ft[$1]; $10=lc[$1]}; print}
' /tmp/oncall_stat.txt .claude/hotspots/oncall_commits.csv \
> /tmp/oncall_statted.csv \
&& mv /tmp/oncall_statted.csv .claude/hotspots/oncall_commits.csv
```

After writing the CSV, update `error_class` for each row by cross-referencing each commit's touched files against `/tmp/dominant_class.txt` (built in the Dominant Class Lookup step above). Requires `/tmp/dominant_class.txt` to exist.

```bash
# Extract on-call hashes (skip header)
tail -n +2 .claude/hotspots/oncall_commits.csv | cut -d',' -f1 > /tmp/oncall_hashes.txt

# For each hash, resolve error_class from the files it touched
> /tmp/oncall_hash_class.txt
while IFS= read -r hash; do
  cls=$(git show --name-only --format="" "$hash" 2>/dev/null \
    | awk 'NR==FNR {dom[$1]=$2; next} ($0 in dom) && dom[$0]!="unknown" {print dom[$0]; exit}' \
         /tmp/dominant_class.txt -)
  echo "$hash ${cls:-unknown}"
done < /tmp/oncall_hashes.txt > /tmp/oncall_hash_class.txt

# Patch error_class column (column 13) in-place
awk 'BEGIN{FS=OFS=","}
     NR==FNR {cls[$1]=$2; next}
     FNR==1 {print; next}
     {$13 = ($1 in cls ? cls[$1] : "unknown"); print}
' /tmp/oncall_hash_class.txt .claude/hotspots/oncall_commits.csv \
> /tmp/oncall_updated.csv \
&& mv /tmp/oncall_updated.csv .claude/hotspots/oncall_commits.csv
```

---

## Step 2: Acceleration Guard

```
if defect_712m == 0:
    acceleration = "new"   # file had no defects 7–12 months ago — flag separately
else:
    acceleration = defect_6m / defect_712m
```

Files marked "new" are in active escalation and should appear in the acceleration callouts regardless of their absolute score.

---

## Fix-Chain Extraction

One row per defect commit, sorted by date ascending. Run once per ACTIVE CRISIS file and append to `.claude/hotspots/fix_chains.csv`.

```bash
# Write header first — overwrites any existing file to prevent doubled rows on re-run
echo "file,fix_date,commit_hash,was_oncall,days_since_previous_fix,branch,workflow_deviation,notes" \
  > .claude/hotspots/fix_chains.csv
FILE="<file>"
git log BRANCHES --format="%H|%ad|%s" --date=format:"%Y-%m-%d|%H|%w" \
  --grep="fix\|bug\|error\|hotfix\|patch" -i -- "$FILE" \
| sort -t'|' -k2 \
| awk -F'|' -v file="$FILE" -v branch="$BRANCHES" '
  function to_days(d,   y,m,dd,i,tot,ml) {
    y=substr(d,1,4)+0; m=substr(d,6,2)+0; dd=substr(d,9,2)+0
    split("31,28,31,30,31,30,31,31,30,31,30,31",ml,",")
    if (y%400==0 || (y%4==0 && y%100!=0)) ml[2]=29
    tot = y*365 + int(y/4) - int(y/100) + int(y/400) + dd
    for (i=1; i<m; i++) tot += ml[i]
    return tot
  }
  BEGIN { prev_date = ""; OFS = "," }
  {
    hash=$1; date=$2; hour=$3+0; dow=$4+0; msg=$5
    gsub(/,/, " ", msg)
    is_oncall = (dow==0 || dow==6 || hour>=22 || hour<6) ? "true" : "false"
    gap = (prev_date == "") ? "" : (to_days(date) - to_days(prev_date))
    print file, date, hash, is_oncall, gap, branch, "normal", msg
    prev_date = date
  }
' >> .claude/hotspots/fix_chains.csv
```

Cluster = ≥3 consecutive rows for the same file where all gaps ≤15 days. Count exactly; do not estimate.

---

## Function Drill-Down

When lizard is not available:

```bash
git log --since="2 years ago" --format=format: -p -- <file> \
| grep "^+.*function\|^+.*def \|^+.*public.*(\|^+[[:space:]]*[a-zA-Z][a-zA-Z0-9]*[[:space:]]*(" \
| sort | uniq -c | sort -rn | head -20
```

---

## Workflow Deviations

```bash
git log master --first-parent --no-merges --format="%H|%ad|%s|%aN" --date=short | head -30
```

Classify each: `direct_to_master`, `on_call_dtm`, `training_only`. Write to `workflow_deviations.csv`.

---

## CSV Schemas

### develop-backend.csv / develop-frontend.csv

```
file, churn_total, churn_6m, churn_712m, churn_older,
defect_total, defect_6m, defect_712m, defect_older,
oncall_count, unique_authors, revert_count,
null_commits, sql_commits, notification_commits, concurrency_commits, auth_commits,
peak_month, recency_ratio_defect
```

### matrix.csv

All columns from above, plus:

```
weighted_churn, weighted_defect,
norm_churn, norm_defect, norm_oncall, norm_author,
defect_score, churn_score, oncall_score, author_score,
hotspot_score, acceleration, quadrant
```

### fix_chains.csv

One row per defect commit (fix-level, not cluster summaries). Clusters are groups of ≥3 consecutive rows for the same file with all gaps ≤15 days — count them from the rows; they are not stored as separate records.

```
file, fix_date, commit_hash, was_oncall, days_since_previous_fix, branch, workflow_deviation, notes
```

`days_since_previous_fix` — empty string for the first fix of each file; integer days for all subsequent rows.
`was_oncall` — `"true"` if the commit fell on a weekend or between 22:00–05:59 local time, `"false"` otherwise.
`workflow_deviation` — `normal`, `direct_to_master`, or `on_call_dtm`.

### oncall_commits.csv (13 columns)

```
hash, date, message, author, day_of_week, hour, workflow_type,
is_fix, files_touched, lines_changed, cherry_picked_to, is_revert, error_class
```

`workflow_type` values: `direct`, `merge`, `cherry-pick`
`error_class` values: `null`, `sql`, `notification`, `concurrency`, `auth`, `unknown`

### file_monthly.csv

```
file, year_month, defect_count, dominant_bug_class
```

One row per ACTIVE CRISIS file × month, covering all history. Used for peak_month computation and seasonal gap detection.

Note: this is distinct from any project-wide monthly summary file (e.g., `seasonal.csv`) that may already exist in the repo. Do not overwrite such a file — write only to `.claude/hotspots/file_monthly.csv`.

### workflow_deviations.csv

```
hash, date, author, message, classification
```

---

## HTML Theme

Both hotspot-analysis and temporal-coupling must use the same CSS. Embed the full block below in every generated report's `<style>` tag — do not abbreviate it.

```css
/* ── Reset & base ──────────────────────────────────────────── */
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;
     background:#0f1117;color:#e2e8f0;line-height:1.7;font-size:15px}
.wrap{max-width:960px;margin:0 auto;padding:2.5rem 2rem 4rem}

/* ── Typography ────────────────────────────────────────────── */
h1{font-size:2rem;color:#f7fafc;font-weight:700;margin:2.5rem 0 0.5rem;
   border-bottom:2px solid #4a5568;padding-bottom:0.5rem}
h2{font-size:1.5rem;color:#f7fafc;font-weight:600;margin:2.5rem 0 0.75rem;
   border-bottom:1px solid #2d3748;padding-bottom:0.4rem}
h3{font-size:1.15rem;color:#e2e8f0;font-weight:600;margin:2rem 0 0.6rem}
h4{font-size:1rem;color:#cbd5e0;font-weight:600;margin:1.5rem 0 0.5rem}
p{margin:0.6rem 0 1rem}
a{color:#63b3ed}
strong{color:#f7fafc}
em{color:#a0aec0}
hr{border:none;border-top:1px solid #2d3748;margin:2rem 0}
ul,ol{padding-left:1.6rem;margin:0.5rem 0 1rem}
li{margin:0.3rem 0}

/* ── Code & pre ────────────────────────────────────────────── */
code{font-family:'Cascadia Code','Fira Code','Consolas',monospace;
     background:#1a1f2e;color:#90cdf4;padding:0.15em 0.4em;
     border-radius:4px;font-size:0.875em}
pre{background:#0d1117;border:1px solid #2d3748;border-radius:8px;
    padding:1.25rem;margin:1rem 0;overflow-x:auto}
pre code{background:none;color:#e2e8f0;padding:0;font-size:0.85rem;line-height:1.6}

/* ── Blockquote (methodology / data-sources box) ───────────── */
blockquote{border-left:3px solid #4a5568;padding:0.5rem 1rem;margin:1rem 0;
           background:#1a1f2e;border-radius:0 6px 6px 0;color:#a0aec0}
blockquote p{margin:0.25rem 0}

/* ── Tables ────────────────────────────────────────────────── */
table{border-collapse:collapse;width:100%;margin:1rem 0;font-size:0.875rem}
thead{background:#1a1f2e}
th{color:#90cdf4;font-weight:600;padding:0.65rem 0.85rem;
   text-align:left;border-bottom:2px solid #2d3748}
td{padding:0.6rem 0.85rem;border-bottom:1px solid #1a1f2e;vertical-align:top}
tr:nth-child(even) td{background:#111622}
tr:hover td{background:#1e2535}

/* ── Cards (ACTIVE CRISIS callouts) ────────────────────────── */
.card{background:#1a1f2e;border:1px solid #2d3748;border-radius:8px;
      padding:1.25rem 1.5rem;margin:1rem 0}
.card.red   {border-left:4px solid #c0392b;background:#1f1215}
.card.orange{border-left:4px solid #e67e22;background:#1f1a12}
.card.yellow{border-left:4px solid #f1c40f;background:#1f1e12}

/* ── Two-column layout ──────────────────────────────────────── */
.cols2{display:grid;grid-template-columns:1fr 1fr;gap:1rem;margin:1rem 0}

/* ── Verdict tags ───────────────────────────────────────────── */
.verdict{display:inline-block;font-size:0.75rem;font-weight:700;
         letter-spacing:0.05em;text-transform:uppercase;
         padding:0.2em 0.6em;border-radius:4px;margin:0 0.25rem 0.25rem 0}
.verdict.causal      ,.tag.causal      {background:#c0392b;color:#fff}
.verdict.seasonal    ,.tag.seasonal    {background:#e67e22;color:#fff}
.verdict.structural  ,.tag.structural  {background:#f1c40f;color:#000}
.verdict.coincidental,.tag.coincidental{background:#252542;color:#e0e0e0;
                                        border:1px solid #3d3d5c}

/* ── Year badge (seasonal coupling section) ─────────────────── */
.year-badge{display:inline-block;font-size:0.7rem;font-weight:700;
            background:#2d3748;color:#90cdf4;padding:0.15em 0.5em;
            border-radius:12px;margin-right:0.4rem;letter-spacing:0.04em}

/* ── Bar chart (logical coupling) ───────────────────────────── */
.bar-row  {display:flex;align-items:center;gap:0.6rem;margin:0.35rem 0}
.bar-label{width:260px;font-size:0.8rem;color:#a0aec0;
           white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
.bar-track{flex:1;background:#1a1f2e;border-radius:4px;height:14px;overflow:hidden}
.bar-fill {height:100%;border-radius:4px;transition:width 0.3s ease}
.bar-fill.high  {background:#c0392b}
.bar-fill.medium{background:#e67e22}
.bar-fill.low   {background:#27ae60}

/* ── Temporal chains ────────────────────────────────────────── */
.chain      {display:flex;align-items:center;flex-wrap:wrap;gap:0;margin:0.75rem 0}
.chain-node {background:#1a1f2e;border:1px solid #3d3d5c;border-radius:6px;
             padding:0.4rem 0.75rem;font-size:0.8rem;color:#e2e8f0;white-space:nowrap}
.chain-node.oncall{border:2px solid #c0392b;background:#1f1215}
.chain-arrow{color:#4a5568;padding:0 0.3rem;font-size:1rem;user-select:none}
.chain-gap  {font-size:0.7rem;color:#718096;padding:0 0.2rem}
```

Apply this block verbatim. Do not strip the component classes — reports rendered without them will have broken bar charts, unstyled chain nodes, and invisible verdict badges.
