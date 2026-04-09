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

# For all candidate files at once:
git log BRANCHES --format="%H %aN" \
  | sort -u \
  | awk '{print $2, $1}' > /tmp/commit_authors.txt
# Then cross-reference with per-file commit lists.
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
      is_fix  = ($5 ~ /fix|bug|error|hotfix|patch/) ? 1 : 0;
      is_rev  = ($5 ~ /^[Rr]evert/) ? 1 : 0;
      if (dow==0 || dow==6 || h>=22 || h<6)
        print $1 "," $2 "," $5 "," $6 "," dow "," $3 ",direct," is_fix ",0,0,," is_rev ",unknown"
    }' > .claude/hotspots/oncall_commits.csv
```

Column order: `hash,date,message,author,day_of_week,hour,workflow_type,is_fix,files_touched,lines_changed,cherry_picked_to,is_revert,error_class`

`files_touched` and `lines_changed` require a per-commit `git show --stat`. For large repos, compute only for the most recent 200 on-call commits.

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
FILE="<file>"
git log BRANCHES --format="%H|%ad|%s" --date=format:"%Y-%m-%d|%H|%w" \
  --grep="fix\|bug\|error\|hotfix\|patch" -i -- "$FILE" \
| sort -t'|' -k2 \
| awk -F'|' -v file="$FILE" '
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
    is_oncall = (dow==0 || dow==6 || hour>=22 || hour<6) ? 1 : 0
    gap = (prev_date == "") ? "" : (to_days(date) - to_days(prev_date))
    print file, date, hash, is_oncall, gap, "BRANCHES", "normal", msg
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
| grep "^+.*function\|^+.*def \|^+.*public.*(" \
| sort | uniq -c | sort -rn | head -20
```

---

## Workflow Deviations

```bash
git log master --format="%H|%ad|%s|%aN" --date=short | grep -v "Merge" | head -30
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
`was_oncall` — 1 if the commit fell on a weekend or between 22:00–05:59 local time, 0 otherwise.
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

Both hotspot-analysis and temporal-coupling must use the same CSS:

```css
:root {
  --dark:   #1a1a2e;
  --panel:  #252542;
  --border: #3d3d5c;
  --text:   #e0e0e0;
  --red:    #c0392b;
  --orange: #e67e22;
  --yellow: #f1c40f;
  --green:  #27ae60;
  --blue:   #2980b9;
}

.tag.causal      { background: var(--red); color: #fff; }
.tag.seasonal    { background: var(--orange); color: #fff; }
.tag.structural  { background: var(--yellow); color: #000; }
.tag.coincidental{ background: var(--panel); color: var(--text); }
.chain-node.oncall { border: 2px solid var(--red); }
```

Apply these to verdict badges, chain nodes, and quadrant labels throughout both reports.
