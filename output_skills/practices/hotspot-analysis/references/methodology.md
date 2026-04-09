# Hotspot Analysis — Exact Extraction Commands

Replace `BRANCHES` with the confirmed branch scope (e.g., `develop master training`).

---

## Step 1: Extraction Commands

```bash
# 1. Churn — all history across branches
git log BRANCHES --format=format: --name-only | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/churn_total.txt

# 2. Churn — last 6 months
git log BRANCHES --since="6 months ago" --format=format: --name-only | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/churn_6m.txt

# 3. Churn — 7–12 months ago (prior window for acceleration)
git log BRANCHES --since="12 months ago" --until="6 months ago" --format=format: --name-only | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/churn_712m.txt

# 4. Defect commits — all history
git log BRANCHES --format=format: --name-only \
  --grep="fix\|bug\|error\|hotfix\|patch\|revert\|crash\|broken\|null\|fail" -i \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/defects_total.txt

# 5. Defects — last 6 months
git log BRANCHES --since="6 months ago" --format=format: --name-only \
  --grep="fix\|bug\|error\|hotfix\|patch\|revert\|crash\|broken\|null\|fail" -i \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/defects_6m.txt

# 6. Defects — 7–12 months ago
git log BRANCHES --since="12 months ago" --until="6 months ago" --format=format: --name-only \
  --grep="fix\|bug\|error\|hotfix\|patch\|revert\|crash\|broken\|null\|fail" -i \
  | grep -v '^$' | sort | uniq -c | sort -rn > /tmp/defects_712m.txt

# 7. On-call commits — weekends (dow 0 or 6) and overnight (22:00–05:59)
git log BRANCHES --format="%H|%ad|%s" --date=format:"%Y-%m-%d %H:%M:%S %w" \
  | awk -F'|' '{
      split($2,d," ");
      h=substr(d[2],1,2)+0;
      dow=d[3];
      if (dow==0 || dow==6 || h>=22 || h<6) print $0
    }' > /tmp/oncall_commits.txt

# 8. Reverts
git log BRANCHES --format="%H %s" --grep="^[Rr]evert" | head -20 > /tmp/reverts.txt
```

---

## Fix-Chain Extraction

For each ACTIVE CRISIS file, run:

```bash
git log BRANCHES --format="%H|%ad|%s" --date=short \
  --grep="fix\|bug\|error\|hotfix\|patch" -i -- <file> \
| awk -F'|' '{print $2, $1, $3}' | sort
```

Cluster = ≥3 defect commits within ≤15 days. Count them; do not estimate.

---

## Function Drill-Down

When lizard is not available, use git log + grep to find which functions received the most defect-related changes in the past 2 years:

```bash
git log --since="2 years ago" --format=format: -p -- <file> \
| grep "^+.*function\|^+.*def \|^+.*public.*(" \
| sort | uniq -c | sort -rn | head -20
```

Report per function: count, fix% (defect commits to that function / total commits), dominant error class.

---

## Workflow Deviations

```bash
# Direct-to-master commits (not arriving via merge)
git log master --format="%H|%ad|%s|%an" --date=short | grep -v "Merge" | head -30
```

Classify each:
- `direct_to_master` — commit on master not from a merge
- `on_call_dtm` — direct-to-master AND timestamp falls in on-call window
- `training_only` — commit on training branch with no equivalent on master
