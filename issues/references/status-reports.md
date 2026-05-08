# Status reports

Periodic snapshots of the project's issue queue. A report captures the counts by status at a point in time and the delta against a baseline (the previous report, or a user-specified period).

Reports are for humans (and future Claude sessions reading the project's history). The Mac app does not currently parse `issues/reports/` — keep reports readable as plain markdown without worrying about parser regex compatibility.

## When to generate a report

Trigger when the user asks for a snapshot of the queue. Common phrasings:

- "give me a status report"
- "snapshot where we are"
- "report on the last week / month"
- "what's changed since the last report"
- "summarize progress"

If the user doesn't specify a baseline, default to *most recent prior report*. If there are no prior reports, default to *initial report* (no delta section).

## Folder layout

Reports live in a `reports/` subfolder of `issues/`:

```
issues/
├── Issues.md
├── 0001.md
├── 0002.md
├── …
└── reports/
    ├── report-04-30-2026.md
    └── report-05-07-2026.md
```

Create `issues/reports/` the first time it's needed; otherwise leave it alone.

## Filename format

```
report-<TIMESTAMP>.md
```

`<TIMESTAMP>` is the localized short-date format for the user's locale, with hyphens between components (no slashes — slashes are path separators). For the US locale, that's `MM-dd-YYYY`; e.g. May 7, 2026 → `report-05-07-2026.md`. Other common locales:

| Locale | Pattern | Example for May 7, 2026 |
|---|---|---|
| US | `MM-dd-YYYY` | `report-05-07-2026.md` |
| UK / most EU | `dd-MM-YYYY` | `report-07-05-2026.md` |
| ISO / JP / KR | `YYYY-MM-dd` | `report-2026-05-07.md` |

If you don't know the user's locale, ask once or fall back to ISO. Whatever pattern you pick, stay consistent within a project — the project's prior reports are the source of truth for the convention.

If two reports land on the same day, append `-<n>`: `report-05-07-2026-2.md`.

## Baseline selection

A report is always a delta against a baseline. Pick the baseline in this order:

1. **User-specified period.** "past week" → cutoff is `today - 7 days`. "past month" → `today - 30 days` (or the same day-of-month one month back, whichever the user means — ask if ambiguous). "since April 1" → that exact date.
2. **Most recent prior report.** Sort `issues/reports/report-*.md` by the date in the filename (parsing per the project's convention). Tie-break by mtime.
3. **No prior report and no period.** It's the initial report — note `Baseline: none (initial report)` and omit the delta section.

When the baseline is a period, also note in the report which dated range you covered, so a future reader can reproduce the calculation.

## Report content

Structure each report in this order. Skip sections that don't apply.

### 1. Header

```markdown
# Status report — May 7, 2026

- **Baseline**: `report-04-30-2026.md` (7 days ago)
- **Generated**: 2026-05-07
- **Total issues**: 63
```

For an initial report, replace the Baseline line with `**Baseline**: none (initial report)`.

For a period-based report, the Baseline line names the period: `**Baseline**: past week (2026-04-30 → 2026-05-07)`.

### 2. Counts

Top-level breakdown by status. Always include every status row even if zero — the zeros are informative.

```markdown
## Counts

| Status | Count |
|---|---|
| Open | 12 |
| In Progress | 3 |
| Resolved | 5 |
| Closed | 41 |
| Won't Fix | 2 |
| **Total** | **63** |
```

### 3. Chart

A horizontal bar chart in a fenced code block, one block (`█`) per issue. The point is at-a-glance comparison — keep it ASCII and embedded in the markdown.

```markdown
## Distribution

​```
Open         ████████████ 12
In Progress  ███ 3
Resolved     █████ 5
Closed       █████████████████████████████████████████ 41
Won't Fix    ██ 2
​```
```

If the longest bar would exceed ~50 blocks, scale: pick a round factor (e.g. 1 block = 2 issues) and note it on a line above the chart. Right-pad the labels so the bars start in the same column.

### 4. Changes since baseline

Skip this whole section if it's the initial report. Otherwise list what moved, with issue numbers.

```markdown
## Changes since baseline

- **Opened** (4): #0060, #0061, #0062, #0063
- **Resolved** (3): #0057, #0058, #0059
- **Closed** (2): #0049, #0052
- **Reopened** (1): #0044
- **Won't Fix** (0): —
```

Categories to consider:

- **Opened** — issue file did not exist (or was not yet `open`) at baseline; is `open`/`in-progress` now.
- **Resolved** — moved from any non-`resolved` status to `resolved` since baseline.
- **Closed** — moved to `closed` since baseline.
- **Reopened** — was `resolved`/`closed`/`wontfix` at baseline; is `open`/`in-progress` now.
- **Won't Fix** — moved to `wontfix` since baseline.

Omit any category whose count is zero rather than padding with em-dashes — keep the list tight.

### 5. Notable items (optional)

A short bulleted list when something stands out: a long-running `open` issue, a regression, a `wontfix` worth revisiting, an in-progress that's been sitting too long. Skip the section entirely if nothing is notable. This is the one place to editorialize — everywhere else stays factual.

## How to compute the delta

1. **Snapshot the current state.** Glob `issues/*.md` (skip `Issues.md`), read each metadata table to extract Status. Read partially — the table is always near the top.
2. **Reconstruct the baseline state.**
   - **Prior-report baseline**: parse the prior report's `## Counts` table and `## Changes since baseline` lists to derive the per-issue status as of that report. Or, more reliably, walk back through every prior report in order and replay the deltas — but the simpler "current files − changes since prior report" subtraction usually suffices when reports run on a regular cadence.
   - **Period baseline (git tracked)**: `git log --since=<cutoff> -- issues/` to find which files changed, and `git show <hash>:issues/NNNN.md` to read the prior version of any changed file. The pre-period status is whatever the file said before the first commit in the window.
   - **Period baseline (git not tracked / no repo)**: fall back to file mtimes for "opened in window" (mtime > cutoff and Status is `open`). For status moves, you don't have history — note this limitation in the report ("Period reports are best-effort without git tracking; status moves prior to today aren't reconstructable").
3. **Diff.** Compute the five categories above by comparing baseline status to current status per issue id.

## Git tracking

Reports follow the same rules as the rest of `issues/`. Check once before writing:

```bash
git rev-parse --is-inside-work-tree 2>/dev/null
git check-ignore -q issues/
```

- **Tracked**: commit the new report file with message `Report: <date>` (e.g. `Report: 2026-05-07`). Stage only the new report — don't bundle other changes.
- **Ignored or no repo**: write the file; no commit.

## Anti-patterns to avoid

- **Don't mutate prior reports.** A report is a snapshot of a moment; if it's wrong, write a new report rather than rewriting history.
- **Don't change issue files** as a side-effect of generating a report. The report observes the queue; it doesn't reshape it.
- **Don't over-editorialize.** The factual sections (Counts, Distribution, Changes) are the spine. Save commentary for the optional Notable items section.
- **Don't generate a report on every interaction.** Reports are user-triggered. The Mac app already shows live state.
