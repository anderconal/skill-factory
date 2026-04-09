# Temporal Coupling — Exact Extraction Commands

Replace `BRANCHES` with the confirmed branch scope (e.g., `develop master training`).
Replace file extension patterns with those relevant to the project (`.cs`, `.vue`, `.js`, `.ts`, etc.).

---

## Co-Commit Query

For each pair of ACTIVE CRISIS files, count commits that touched both simultaneously:

```bash
git log BRANCHES --format="%H" | while read hash; do
  files=$(git show --name-only --format="" $hash | grep -E "\.(cs|vue|js|ts)$" | sort)
  echo "$files"
done | sort | uniq -c | sort -rn | head -50
```

Report per pair: co-commit count AND percentage of the smaller file's total commit history.

---

## Seasonal Gap Query

For each ACTIVE CRISIS file, extract defect commit dates and compute inter-incident gaps:

```bash
git log BRANCHES --format="%ad" --date=short \
  --grep="fix\|bug\|error\|hotfix" -i -- <file> \
| sort | awk 'NR>1 {
    prev=$0;
    split(prev,a,"-");
    split($0,b,"-");
    days=(mktime(b[1]" "b[2]" "b[3]" 0 0 0") - mktime(a[1]" "a[2]" "a[3]" 0 0 0")) / 86400;
    if (days > 200 && days < 500) print days, prev, $0
  }'
```

Gaps in the 200–500 day range are seasonal candidates. Verify both incidents share the same error class before labeling seasonal.

---

## Temporal Chain Query

Find on-call incidents affecting multiple files in the same 15-day window:

```bash
awk -F'|' '{print $1}' /tmp/oncall_commits.txt | while read hash; do
  git show --name-only --format="%ad" --date=short $hash
done
```

Cross-reference with defect commit lists from hotspot-analysis. A chain = minimum 3 fixes across 2+ files; a single multi-file commit is not a chain.

---

## Bug Class by Year

For a cluster of ACTIVE CRISIS files, group defect commits by year:

```bash
git log BRANCHES --format="%ad|%H|%s" --date=short \
  --grep="fix\|bug\|error\|hotfix" -i -- <file1> <file2> <file3> \
| sort | awk -F'|' '{year=substr($1,1,4); print year, $3}' | sort
```

Identify: same root structural cause across multiple years, recurrence count, what was patched each time vs what structural condition remained.
