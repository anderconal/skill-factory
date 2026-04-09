# Temporal Coupling — Extraction Commands

Replace `BRANCHES` with the confirmed branch scope (e.g., `develop master training`).
Replace the extension filter (`\.(cs|vue|js|ts)$`) with the extensions relevant to the project.

All commands are portable across Windows Git Bash and Linux. No Python required. No GNU awk extensions (mktime, etc.).

---

## Co-Commit Query

Single-pass awk — reads git log once, no per-commit subprocess spawning:

```bash
git log BRANCHES --name-only --format="COMMIT:%H" \
| awk '
  /^COMMIT:/ {
    if (n > 1) {
      for (i=1; i<=n; i++)
        for (j=i+1; j<=n; j++)
          pairs[files[i] SUBSEP files[j]]++
    }
    n=0; delete files; next
  }
  NF && /\.(cs|vue|js|ts)$/ { files[++n]=$0 }
  END {
    for (p in pairs) {
      if (pairs[p] >= 5) {
        split(p, a, SUBSEP)
        print pairs[p], a[1], a[2]
      }
    }
  }
' | sort -rn | head -50
```

Performance note: this is O(1) subprocess spawns vs O(n) for the `while read hash; do git show...` pattern. On a 4,354-commit repo, the difference is seconds vs minutes.

---

## Seasonal Gap Detection

Portable POSIX awk — no mktime, no GNU date, no Python:

```bash
git log BRANCHES --format="%as" \
  --grep="fix\|bug\|error\|hotfix" -i -- <file> \
| sort \
| awk '
  function to_days(date,    y,m,d,i,days) {
    y=substr(date,1,4)+0; m=substr(date,6,2)+0; d=substr(date,9,2)+0
    days = y*365 + int(y/4) - int(y/100) + int(y/400) + d
    for (i=1; i<m; i++) {
      if (i==1||i==3||i==5||i==7||i==8||i==10||i==12) days+=31
      else if (i==4||i==6||i==9||i==11) days+=30
      else days += (y%400==0 || (y%4==0 && y%100!=0)) ? 29 : 28
    }
    return days
  }
  NR==1 { prev=$0; next }
  {
    gap = to_days($0) - to_days(prev)
    if (gap > 200 && gap < 500)
      print gap " days: " prev " -> " $0
    prev=$0
  }
'
```

Gaps in the 200–500 day range are seasonal candidates. Verify both incidents share the same error class before labeling seasonal.

---

## Temporal Chain Query

Read on-call incident hashes from `.claude/hotspots/oncall_commits.csv` (column 1 = hash). Do NOT read from `/tmp/oncall_commits.txt` — the column formats are incompatible.

```bash
# Extract hashes from the CSV (skip header line)
tail -n +2 .claude/hotspots/oncall_commits.csv \
| cut -d'|' -f1 \
| while read hash; do
    git show --name-only --format="%H|%ad" --date=short "$hash" 2>/dev/null
  done
```

Cross-reference the dates and file lists to find clusters: ≥3 fixes across 2+ files within 15 days. A single multi-file commit is not a chain — it is a large changeset.

---

## Bug Class by Year

For a cluster of ACTIVE CRISIS files, group defect commits by year:

```bash
git log BRANCHES --format="%ad|%H|%s" --date=short \
  --grep="fix\|bug\|error\|hotfix" -i -- <file1> <file2> <file3> \
| sort \
| awk -F'|' '{year=substr($1,1,4); print year, $3}' \
| sort
```

Identify: same root structural cause across multiple years, recurrence count, what was patched vs what structural condition remained.

---

## Two-Mechanism Percentage

The report requires a "two-mechanism summary" with percentage estimates backed by data.

`fix_chains.csv` stores fix-level records (one row per defect commit). `days_since_previous_fix` is empty for the first fix of each file and an integer for all subsequent rows — a non-empty value means the commit was a repeat fix to an already-known problem.

**Symptom-patching rate** — on-call repeat fixes to already-known problems:

```
# Count on-call rows in fix_chains.csv that are NOT the first fix for their file.
# "Not the first" = days_since_previous_fix is non-empty.
symptom_patching_count = count of rows in fix_chains.csv
  where was_oncall == 1
  AND days_since_previous_fix != ""

total_oncall_rows = count of all data rows in oncall_commits.csv (exclude header)

symptom_patching_pct = (symptom_patching_count / total_oncall_rows) * 100
```

No hash cross-reference is required. Both values come directly from the CSVs.

**Coupling-induced rate** — on-call incidents that are part of a confirmed temporal chain:

```
coupling_count = count of on-call hashes (from oncall_commits.csv)
  that appear in a confirmed temporal chain
  (≥3 fixes across 2+ files within 15 days, identified in Step 3)

coupling_pct = (coupling_count / total_oncall_rows) * 100
```

These can overlap: a commit can be both a repeat fix AND part of a temporal chain. Report both percentages independently; note the overlap if it exists.

---

## HTML Theme

Use the same CSS as hotspot-analysis. Copy from `output_skills/practices/hotspot-analysis/references/methodology.md — HTML Theme`. Both reports must use identical CSS variables and tag classes so linked reports are visually consistent.
