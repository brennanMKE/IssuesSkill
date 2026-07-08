---
name: issues
description: Manage a project's issues/ folder of NNNN.md markdown files: file new bugs, update status, attach screenshots, and run the standard plan→implement→review workflow (planning subagent on the top model at filing time, then implementation and review subagents). Use this skill whenever the user or a subagent describes a bug or broken thing, says "log this" / "make a ticket" / "track this", asks to work the next open issue or fix issue NNNN, or wants a summary of what's open — even when they don't explicitly say the word "issue". Triggers in any project that has (or should have) an issues/ folder.
---

# Issues — markdown-based bug tracking

A lightweight, repo-local issue tracker. Each project keeps an `issues/` folder containing zero-padded `NNNN.md` files (`0001.md`, `0002.md`, …) plus an `issues/Issues.md` config file. The user reports bugs verbally or with a screenshot; you record them as markdown so the conversation can move on without losing the bug.

A separate native Mac app (Issues.app) watches the folder and renders the current state for the user. There is no JSON or generated artifact to keep in sync — the markdown files *are* the source of truth, and the Mac app re-reads them on every change. This skill is solely about producing well-formed markdown.

## When to use this skill

Trigger this skill whenever the user is working in a project that has (or should have) an `issues/` folder and they:

- describe a bug, a regression, something that's "broken", "not working", "weird", "wrong"
- attach or paste a screenshot of something that's broken
- say "file this", "log this", "track this", "write this up", "make a ticket", "add to issues"
- ask to update an issue's status, add a note, attach a screenshot, or close one out
- ask "what's still open?", "what's in progress?", "show me the bugs in module X"

The user often won't say the word "issue" — they'll just describe the bug. Default to filing it.

This skill is also intended for **subagents**: when a subagent is doing work on the user's behalf and discovers a bug, regression, or design problem, it should file an issue rather than fix-and-forget. The user reviews the queue through the Mac app.

## Folder layout

```
issues/
├── project.json       # canonical project name + repo URL (see references/project-config.md)
├── Issues.md          # local guide for managing issues
├── model-pricing.json # daily-refreshed model price cache (see references/cost-tracking.md)
├── 0001.md            # one file per issue
├── 0001/              # optional sibling folder for screenshots, crash logs, etc.
│   └── screenshot.png
├── 0002.md
└── …
```

That's it. No `generate.py`, no `index.html`, no `issues.json` index. The Mac app reads the folder directly.

`project.json` is the canonical source for the project's name and URL — don't infer either from the parent folder path.

## Bundled files

This skill ships with templates and a parser reference. Use them rather than reconstructing from prose:

- **`assets/issue-template.md`** — the literal `NNNN.md` template. Read this with the `Read` tool when you need the exact structure for a new issue.
- **`assets/Issues-md-template.md`** — the template for `issues/Issues.md`. This is the project-local guide an agent walking into the project will read; it should be self-contained. Copy it verbatim into a new project's `issues/` folder, then customize the project name, description, and module conventions.
- **`assets/project.json`** — minimal template for `issues/project.json` (name + url). Copy when setting up a new project, then fill in the values.
- **`references/workflow.md`** — the canonical spec for the standard **plan → implement → review** workflow: the three-phase pipeline, which model runs each phase (Fable / Sonnet / Opus), what each subagent does, and the end-to-end status flow. Read this whenever you're planning a new issue, dispatching an implementation or review subagent, or working through the open queue.
- **`references/project-config.md`** — schema and workflow for `issues/project.json` (the canonical source for the project's name and repo URL). Read this when creating or updating that file.
- **`references/issue-format.md`** — canonical spec for issue file structure: filename, title, metadata table, sections, and the **attachment relative-path rule** (link target must include the `NNNN/` folder prefix, e.g. `1335/screenshot.png`, not `screenshot.png`). Read this when you're unsure how a file should be laid out.
- **`references/video-attachments.md`** — how to attach `.mov`/`.mp4`/etc. videos: generating a poster frame with `qlmanage` and emitting the `[![alt](poster)](video)` image-inside-a-link form. Read this whenever a user hands over a screen recording or any video file.
- **`references/cost-tracking.md`** — how to record token usage, model, and cost per work session: the `issues/model-pricing.json` daily price cache, locating the subagent's transcript to get exact token counts, the cost formula, and the `## Work log` section format. Each issue accumulates a row per phase — planning (Fable), implementation (Sonnet), review (Opus) — plus any bounces or retries. Read this whenever you dispatch any subagent (including the planning subagent at issue creation), or when the user asks what issue work has cost.
- **`references/parsing.md`** — exact regex patterns the Mac app uses. Read this only if you're debugging why something isn't appearing or rendering correctly. Not needed for normal filing.
- **`references/status-reports.md`** — how to generate snapshot reports of the issue queue (counts by status, chart, delta vs. a baseline). Read this when the user asks for a status report, snapshot, or "what's changed since…".

## First, orient yourself

Before filing or updating, take a few seconds to check the project's state. This is cheap and prevents drift.

1. **Find the issues folder.** Usually `issues/` at the repo root.
2. **Read `issues/project.json`** if it exists — it's the canonical source for the project's name and repo URL. Don't infer either from the parent folder path. If it's missing, create it (see `references/project-config.md`) before continuing — it's a one-time setup step.
3. **Read `issues/Issues.md`** if it exists — it's the project's local guide and defines the canonical status vocabulary, module conventions, build/verify command, commit conventions, and any project-specific rules. **`Issues.md` is authoritative for its project**: if anything there contradicts this skill, follow `Issues.md`.
4. **Read `CLAUDE.md`** at the repo root if it exists — project-wide guidance for Claude that may include code conventions, restricted areas, or workflow tweaks that affect issue work. Treat its instructions as binding.
5. **If `Issues.md` is missing**, create it from `assets/Issues-md-template.md` before filing the first issue. Fill in the project name (matching `project.json`) and a real one-paragraph description.
6. **Glance at one or two existing issue files** to absorb the project's tone (how detailed are descriptions, what platforms appear, how modules are named).
7. **Check whether `issues/` is tracked by git.** Run `git rev-parse --is-inside-work-tree` and `git check-ignore -q issues/`. The result determines whether lifecycle events below produce commits or are working-copy-only edits — see the "Git tracking" section.

If `issues/` doesn't exist at all and the user is asking to file something, ask once: "I don't see an `issues/` folder yet — should I create one at the repo root?"

## Status values

| File value | Display name | Meaning |
|---|---|---|
| `open` | Open | Filed but not yet started |
| `in-progress` | In Progress | Actively being worked on |
| `resolved` | Resolved | Work is done; awaiting user confirmation |
| `closed` | Closed | User has confirmed the fix |
| `wontfix` | Won't Fix | Acknowledged but won't be addressed |

Use the **file value** (lowercase, hyphenated) in the issue's metadata table. The Mac app converts to the display name when rendering.

The `resolved` → `closed` distinction is deliberate: `resolved` says "work landed and was independently verified", `closed` says "user confirmed". The **review subagent** (Opus) sets `resolved` after re-verifying the fix; only the user moves an issue to `closed`.

## Critical rule: never close without explicit confirmation

The single most important rule of this skill: an issue must **never** be marked `resolved`, `closed`, or `wontfix` based on inference. Only when the user has said so in plain language. Specifically, do not infer resolution from:

- a code change you (or a subagent) just made
- a commit message
- the filing of a related issue
- the user saying "thanks, that looks better" or "nice"

Always leave status at `open` (or `in-progress` if work has started) until the user confirms in words like "close this", "this is fixed", "mark resolved", or "won't fix". When in doubt, ask. The cost of asking is one turn; the cost of wrongly closing a real bug is that it disappears from the open list and gets forgotten.

The deliberate exception is the review-subagent resolve: after the Opus review subagent independently re-verifies the fix (phase 3 of the standard workflow), it may set status to `resolved` (work-is-done-but-not-confirmed). The implementation subagent never sets `resolved` — it leaves the issue `in-progress` for review. And no subagent ever sets `closed` — that's the user's call. This separation is the entire reason `resolved` and `closed` are different states. See `references/workflow.md`.

## Git tracking

Some projects keep `issues/` in git so the bug-tracking history lives alongside the code. Others ignore it via `.gitignore` because they treat issues as ephemeral workflow state. The skill respects whichever the project chose — **check before doing anything that could cause an unwanted commit.**

### How to check

```bash
git rev-parse --is-inside-work-tree 2>/dev/null   # is this a git repo?
git check-ignore -q issues/                        # exit 0 = ignored, 1 = tracked
```

Three outcomes:

- **Not a git repo** (first command fails): edit files only; no commits ever.
- **Repo, `issues/` is ignored** (second command exits 0): edit files only; no commits. The Mac app still picks up changes.
- **Repo, `issues/` is tracked** (second command exits non-zero): every lifecycle event below produces a git commit.

### What gets committed when `issues/` is tracked

| Event | What's committed | Commit message |
|---|---|---|
| Initial setup | new `project.json` and `Issues.md` together | `Add issue tracker setup` (or bundle with the first `#NNNN` commit) |
| Filing a new issue | the new `NNNN.md` (and `project.json` / `Issues.md` if newly created) | `#NNNN <issue title>` |
| Planning subagent adds `## Plan` | markdown update (the `## Plan` section) | folded into `#NNNN <issue title>` if it lands first, else `#NNNN Plan` |
| Editing project config | `project.json` only | `Update project config` (or more specific: `Update project URL`) |
| Implementation — code commit | code changes only | `#NNNN <verb> <title>` (the substantive commit) |
| Review — resolution commit | markdown update (status `resolved` + Closed + Commit + summary), made by the Opus reviewer | `#NNNN Resolve: <title>` |
| Review — bounce back to open | markdown update (status reverted to `open` + `## Review notes`) | `#NNNN Review: <reason>` |
| Subagent bail (notes added, status reverted to open) | markdown update | `#NNNN Notes: <brief>` |
| Work-log row appended (post-subagent usage record) | markdown update | `#NNNN Work log: <model>, <total tokens>, $<cost>` |
| Daily pricing refresh | `model-pricing.json` only | `Update model pricing` |
| User-confirmed close | markdown update (status `closed`) | `#NNNN Close` |
| Marking won't-fix (after user decision) | markdown update | `#NNNN Won't fix` |

### Working-copy-only changes (no commit by the skill)

- Setting status to `in-progress` at the start of subagent work — this is transient and gets superseded by the resolve commit. Committing every status flip would create excessive churn.
- Any change the user explicitly says they'll commit themselves.

The Mac app reflects working-copy state regardless of whether anything is committed yet, so an uncommitted in-progress flip is still visible to the user.

### Why two commits to resolve, not one

The **Commit** metadata row records the hash of the code-fix commit. That hash isn't known until *after* the code commit lands, so the row can't appear in the same commit it points to. Splitting resolution into a code commit and a follow-up resolution commit keeps each commit single-purpose — "fix the code" and "document the fix" — and lets the resolution commit reference the hash cleanly.

## Filing a new issue

1. **Make sure `issues/project.json` and `issues/Issues.md` exist.** If `project.json` is missing, create it from `assets/project.json` and populate `name` + `url` per `references/project-config.md`. If `Issues.md` is missing, create it from `assets/Issues-md-template.md`.
2. **Pick the next number.** List `issues/`, find the highest existing `NNNN.md`, increment. Start at `0001` if empty. Skip past reserved high numbers like `8888`/`9999` (used for test issues).
3. **Read `assets/issue-template.md`** and copy it to `issues/NNNN.md`, filling in the placeholders.
4. **Title** is a single declarative sentence describing the bug ("Reply button not functional on post cells"), not a question or a fix description.
5. **Status** starts at `open`, always. Even if a fix is already in flight, file as `open` and update status separately.
6. **First seen** is today's date (your `currentDate` context). Format `YYYY-MM-DD`.
7. **Plan it (phase 1 of the standard workflow).** Dispatch a fresh subagent on the **top available model (currently Fable)** to read the project conventions and the new issue, then write a `## Plan` section into `issues/NNNN.md`. This runs at filing time so a planned issue is ready to work the moment someone picks it up. Fable runs *only* in a subagent — never in this orchestrator context — to keep the plan's large context isolated. Record the planner's usage as a `## Work log` row when it returns. Full procedure in `references/workflow.md`. (If the user is filing a quick note and doesn't want planning yet, skip this — the issue is still a normal `open` issue.)
8. **If `issues/` is tracked by git**, commit the new file so the issue enters git history with its `open` status. Stage `issues/NNNN.md` (and `issues/Issues.md` if you just created it); if the `## Plan` from step 7 has already landed, it rides along in this commit with message `#NNNN <issue title>` — otherwise commit the plan separately as `#NNNN Plan`. If `issues/` is ignored or there's no git repo, skip this step.
9. **Confirm to the user** in one line — issue number and title. Don't paste the whole file back.

### Format details that always apply

- Title separator is an em-dash (U+2014, `—`), not a hyphen — the Mac app's regex won't match a hyphen.
- Field rows in the metadata table must keep the field name in `**bold**` exactly.
- `Module` can list multiple modules separated by ` / `.
- `Platform` is `iOS`, `macOS`, `iPadOS`, `All`, or any string. `All` matters — the Mac app's platform filter treats it as matching every platform.
- **`## Description` MUST be the first `##` section after the metadata table.** The Mac app treats everything between the metadata table and the first `##` heading as part of the metadata frontmatter and does not render it. Resolution notes, blockquote summaries, status banners, scoping audits — none of those go above Description. They go in their own `##` sections *after* Description.
- **Description is the TL;DR.** Aim for 1–3 sentences (≤ ~12 non-blank lines). If you need more space to explain context, mechanism, or trade-offs, put that in a separate `## Long Description` section after Description. The Mac app's summary view shows only the first paragraph of Description; a wall-of-text Description is hostile to that surface.
- **`## Resolution notes`** is the conventional section for a quote-block summary when an issue is resolved (e.g. `> 🟢 Resolved YYYY-MM-DD — …`). It goes *after* Description, never above the metadata table.
- **`## Relation`** is the conventional section for bidirectional cross-links to parent / sibling / follow-on tickets. Cross-linking goes both directions: if A spawns B as a follow-on, A's Relation says `Follow-on: [#B](B.md)` and B's Relation says `Carved out of: [#A](A.md)`. Tickets that disappear from one side's Relation aren't actually linked.
- **`## Work log`** is the conventional **last section** of the file: one table row per subagent work session (date, model, token counts, cost) plus a running `**Total: $X.XX**` line. Written by the orchestrator after each dispatch — see `references/cost-tracking.md`.

## Updating an existing issue

For ad-hoc edits outside the standard resolve workflow — adding a note, attaching a screenshot, flipping status manually:

1. Edit `issues/NNNN.md` in place. The Mac app picks up the change automatically.
2. If status changed, update the **Status** row.
3. If status moved to `resolved` or `closed`, add a `**Closed**` row with today's date. If the move to `resolved` was triggered by a fix commit, also add a `**Commit**` row with the short hash (`git rev-parse --short HEAD`).
4. Touch only what changed — don't reformat the rest of the file. Diff-friendly edits matter when the user reviews through the Mac app.
5. **If `issues/` is tracked by git**, commit the change. Use a message that fits the edit:
   - `#NNNN Close` for a user-confirmed close
   - `#NNNN Won't fix` for a wontfix decision
   - `#NNNN Notes: <brief>` for added context or a screenshot
   - `#NNNN Update <field>` for other targeted edits

   The exception: setting status to `in-progress` at the start of subagent work is a transient working-copy edit and is *not* committed on its own — the resolve workflow's two commits supersede it.

When an issue is moved to `resolved` via the standard workflow below, additional sections (`## Root cause`, `## Fix`, `## Verification`, `## Files changed`, optional `## Gotchas`) get added to capture what was wrong and what landed. See "The standard workflow" below and `references/workflow.md` for the full structure.

For any move toward `resolved`, `closed`, or `wontfix`, the "Critical rule" at the top of this skill applies — those transitions require explicit user confirmation, not inference.

## The standard workflow (plan → implement → review)

Issues move from filed to resolved through a **three-phase pipeline**, and every phase runs in a **fresh subagent** dispatched by the orchestrator (this session). The orchestrator picks the issue, dispatches, and records usage — it never does the plan/fix/review work in its own context. Keeping the work in subagents is what keeps the orchestrator small enough to drive a whole queue.

| Phase | When | Model | The subagent | Status after |
|---|---|---|---|---|
| **1. Planning** | at issue creation | **Fable** (top model) | reads conventions + issue, writes `## Plan`; no code | `open` |
| **2. Implementation** | when the issue is worked | **Sonnet** | follows the plan, fixes, builds + verifies, code commit, drafts resolution sections | `in-progress` |
| **3. Review** | after implementation returns | **Opus** | independently re-verifies the diff, then approves or bounces | `resolved` or `open` |

Key rules that the rest of this skill depends on:

- **Planning happens at filing time** (see "Filing a new issue" above), not when work later starts. The `## Plan` gives whoever picks up the issue a head start. The planning subagent never writes code and leaves status at `open`.
- **The implementation subagent does not resolve.** It makes the code commit, drafts `## Root cause` / `## Fix` / `## Verification` / `## Files changed`, adds the `**Commit**` row, and leaves status at `in-progress`. Setting `resolved` is the reviewer's job.
- **The Opus reviewer owns the `resolved` transition.** It re-runs the verification command itself — it does not trust the drafted `## Verification` — and only then flips the issue to `resolved` and makes the single resolution commit. If the fix is wrong or verification fails, it reverts status to `open` with a `## Review notes` brief and the orchestrator re-dispatches implementation.
- **Verification is mandatory in phases 2 and 3.** Compilation is not verification: tests must actually *execute* and pass, and the output must be read (not just the exit code). A green build with zero tests run is a failure, not a pass. If verification can't run in the environment, the subagent bails instead of resolving.
- **No subagent ever sets `closed`.** `closed` is the user's transition after they confirm the fix. See the Critical rule above.
- **Fable runs only in subagents.** Never invoke the top model in this orchestrator context — the point is to spend its tokens against a fresh, minimal context.

If the user names a specific issue ("fix 0046"), dispatch phase 2 to that id directly (it should already carry a `## Plan`). Bails from any phase land back at `open` with a `## Notes` (implementation) or `## Review notes` (review) section explaining what happened.

**The full procedure — orchestrator dispatch steps, each subagent's checklist, the git-commit points, and the end-to-end status diagram — lives in `references/workflow.md`. Read it before planning, dispatching, or reviewing.** The project build/verify command lives in the project's own docs (`Issues.md`, `CLAUDE.md`, `README.md`), not this skill.

## Attaching screenshots and other artifacts

Screenshots, crash logs, console output, sample data — anything related to an issue — live in a sibling folder `issues/NNNN/`. Reference them from the issue's Attachments section with paths *relative to the issue's `.md` file*. That means the folder prefix `NNNN/` is part of the link target — `1335/screenshot.png`, not `screenshot.png` and not `issues/1335/screenshot.png`.

```
issues/1335.md               ← the markdown containing the link
issues/1335/screenshot.png   ← the file being linked

# inside 1335.md:
![caption](1335/screenshot.png)
```

Concrete Attachments section:

```markdown
## Attachments

![Reply button does nothing when tapped](1335/screenshot.png)
![Crash log](1335/crash.log)
```

Filenames are descriptive (`before.png`, `after.png`, `console.png`, `crash.log`) — not just `screenshot.png` when there are multiple. See `references/issue-format.md` for the full file-format spec.

### Videos (`.mov`, `.mp4`, etc.)

Videos can't be embedded as `![…](…)` — markdown renderers treat that as an `<img>` and a `.mov` won't load. Instead, generate a poster frame with `qlmanage` and emit an image-inside-a-link:

```markdown
[![Sidebar resize jitter](1335/sidebar-resize-jitter.poster.png)](1335/sidebar-resize-jitter.mov)
```

Quick recipe — copy the video into `issues/NNNN/`, then:

```bash
qlmanage -t -s 1280 -o issues/NNNN issues/NNNN/<basename>.<ext>
mv issues/NNNN/<basename>.<ext>.png issues/NNNN/<basename>.poster.png
```

If `qlmanage` doesn't produce the poster (corrupt file, unsupported codec — e.g. `.mkv`/`.webm` without a third-party Quick Look generator), fall back to the plain `![alt](NNNN/file.mov)` form with a `<!-- poster generation failed -->` comment. Don't apply the link wrapper to plain images, and don't generate posters for animated GIFs. See `references/video-attachments.md` for the full procedure, supported extensions, and edge cases.

### The macOS screenshot filename gotcha

macOS screenshot filenames look like `Screenshot 2026-05-03 at 3.42.17 PM.png`, but the space before "PM" is actually a **narrow no-break space** (U+202F), not a regular space. A literal `cp` of the quoted filename fails with "No such file or directory" because the byte sequence doesn't match.

Glob past it:

```bash
mkdir -p issues/NNNN
cp /Users/brennan/Desktop/Screenshot\ YYYY-MM-DD\ at\ H.MM.SS*PM.png issues/NNNN/screenshot.png
```

The `*` matches the U+202F without you typing it. Use the user's actual timestamp; if you don't know which screenshot they mean, list `~/Desktop/Screenshot*` by mtime and pick the most recent, or ask.

If you can't read the user's Desktop (sandbox, permissions, or running remotely), ask them to run the `cp` themselves with the `!` prefix — their shell has the permissions.

## Querying issues

When the user asks "what's still open in module X", "what's in progress", or similar:

1. Glob `issues/*.md` (skip `Issues.md`).
2. Read enough of each file to extract the metadata table — it's always near the top, so a partial read is enough.
3. Filter and report back with `#NNNN — Title` lines, grouped or sorted as asked.

For "give me a summary of open bugs", include the first paragraph of `## Description` for each. Don't dump full files at the user — that's what the Mac app is for.

## Anti-patterns to avoid

- **Don't auto-close.** The Critical rule above is the single most important constraint in this skill.
- **Don't put anything above `## Description`.** No resolution-status blockquote, no scoping audit, no banner. The Mac app eats whatever sits between the metadata table and the first `##` — the content is effectively invisible. If you have something to say at the top, say it inside `## Description` or in a dedicated section after it (see "Format details that always apply").
- **Don't leave dangling follow-ups in prose.** "Will be addressed in a future pass" / "remains a separate, lower-priority follow-up" / "out of scope here, will revisit" — each of those phrases is a real piece of scope that needs a real ticket. File it, link both ways, then mark the parent resolved. Dropping the follow-up into the parent's text alone means it disappears the moment the parent gets closed.
- **Don't confuse "compiles" with "verified".** Marking an issue `resolved` because the code built is the most common false-success failure mode. Tests must actually *run*. New tests in particular must execute and pass — a test that only compiles is not a test. If you can't run the verification step, you have not finished; bail with notes instead of resolving.
- **Don't reformat existing issues** while updating them. Touch only the rows or sections that changed — diff-friendly edits matter when the user is reviewing through the Mac app.
- **Don't skip numbers.** Use the next sequential 4-digit id. Reserved high numbers (8888, 9999) for test/dummy issues are intentional — leave them alone.
- **Don't paraphrase the user's bug report into something cleaner.** Their words are usually closer to the truth than your interpretation. Quote text from screenshots verbatim where it appears.
- **Don't close-and-refile** to "clean up" an issue's history. Edit in place. The file *is* the history.

## Umbrella + phase tickets (for big projects)

When a chunk of work is large enough that a single ticket would balloon — multi-week refactors, cross-cutting initiatives, anything with more than ~3 independently-shippable pieces — use the **umbrella pattern**:

- One **umbrella ticket** scopes the project: the why, the phases, the acceptance criteria for the whole. It doesn't ship code itself.
- Each phase / sub-piece is its own ticket, child of the umbrella. The child's Relation section says `Parent: [#NNNN](NNNN.md)`. The umbrella keeps a checklist (or table) of children and their status.
- As children resolve, **re-audit the downstream phases** before opening them — the work that already shipped often changes what's needed next, and an audit on the umbrella catches the drift before the next phase starts.

This is heavier than a single ticket but it pays for itself once a project crosses ~2 weeks: each phase has a focused conversation, the umbrella stays scannable, and "what's left?" is one glance at the children's statuses. The umbrella's status stays `in-progress` until every child is terminal (`resolved` / `closed` / `wontfix`); only then does the umbrella itself move to `resolved`.

### Legacy projects

If the project still has `generate.py`, `issues.json`, `issues.js`, `index.html`, or a master Index table at the top of `Issues.md`, that's the older web-dashboard workflow. The Mac app has replaced it entirely. Leave the legacy files in place — don't update them, don't run them, don't add to them.
