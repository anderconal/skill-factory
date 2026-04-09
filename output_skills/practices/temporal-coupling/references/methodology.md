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

Portable POSIX awk — no mktime, no GNU date, no Python. Run for all ACTIVE CRISIS files in one loop:

```bash
while IFS=',' read -r file _; do
  echo "=== $file ==="
  git log BRANCHES --format="%as" \
    --grep="fix\|bug\|error\|hotfix" -i -- "$file" \
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
done < <(awk -F',' 'NR>1 && $NF == "ACTIVE CRISIS" {print}' .claude/hotspots/matrix.csv)
```

`file` is column 1 of `matrix.csv`; `$NF` is the `quadrant` column (last). The `NR>1` skips the header row. Gaps in the 200–500 day range are seasonal candidates. Verify both incidents share the same error class before labeling seasonal.

---

## Temporal Chain Query

Read on-call incident hashes from `.claude/hotspots/oncall_commits.csv` (column 1 = hash). Do NOT read from `/tmp/oncall_commits.txt` — the column formats are incompatible.

```bash
# Single-pass: pipe all hashes to one git log invocation (O(1) subprocess spawns)
tail -n +2 .claude/hotspots/oncall_commits.csv \
| cut -d',' -f1 \
| tr -d '\r' \
| xargs git log --no-walk --name-only --format="COMMIT:%H|%ad" --date=short 2>/dev/null
```

Performance note: this is O(1) subprocess spawns vs O(n) for the `while read hash; do git show...` pattern — the same reason the Co-Commit Query bans that pattern. `oncall_commits.csv` is typically small (50–100 rows), but the approach is consistent and avoids the ban.

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
