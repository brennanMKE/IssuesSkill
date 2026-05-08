---
name: issues
description: Manage a project's issues/ folder of NNNN.md markdown files: file new bugs, update status, attach screenshots, and run the claim→fix→build→commit→resolve workflow. Use this skill whenever the user or a subagent describes a bug or broken thing, says "log this" / "make a ticket" / "track this", asks to work the next open issue or fix issue NNNN, or wants a summary of what's open — even when they don't explicitly say the word "issue". Triggers in any project that has (or should have) an issues/ folder.
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
├── Issues.md          # project config + local guide for managing issues
├── 0001.md            # one file per issue
├── 0001/              # optional sibling folder for screenshots, crash logs, etc.
│   └── screenshot.png
├── 0002.md
└── …
```

That's it. No `generate.py`, no `index.html`, no `issues.json`. The Mac app reads the folder directly.

## Bundled files

This skill ships with templates and a parser reference. Use them rather than reconstructing from prose:

- **`assets/issue-template.md`** — the literal `NNNN.md` template. Read this with the `Read` tool when you need the exact structure for a new issue.
- **`assets/Issues-md-template.md`** — the template for `issues/Issues.md`. This is the project-local guide an agent walking into the project will read; it should be self-contained. Copy it verbatim into a new project's `issues/` folder, then customize the project name, description, and module conventions.
- **`references/issue-format.md`** — canonical spec for issue file structure: filename, title, metadata table, sections, and the **attachment relative-path rule** (link target must include the `NNNN/` folder prefix, e.g. `1335/screenshot.png`, not `screenshot.png`). Read this when you're unsure how a file should be laid out.
- **`references/video-attachments.md`** — how to attach `.mov`/`.mp4`/etc. videos: generating a poster frame with `qlmanage` and emitting the `[![alt](poster)](video)` image-inside-a-link form. Read this whenever a user hands over a screen recording or any video file.
- **`references/parsing.md`** — exact regex patterns the Mac app uses. Read this only if you're debugging why something isn't appearing or rendering correctly. Not needed for normal filing.
- **`references/status-reports.md`** — how to generate snapshot reports of the issue queue (counts by status, chart, delta vs. a baseline). Read this when the user asks for a status report, snapshot, or "what's changed since…".

## First, orient yourself

Before filing or updating, take a few seconds to check the project's state. This is cheap and prevents drift.

1. **Find the issues folder.** Usually `issues/` at the repo root.
2. **Read `issues/Issues.md`** if it exists — it's the project's local guide and defines the canonical status vocabulary, module conventions, build/verify command, commit conventions, and any project-specific rules. **`Issues.md` is authoritative for its project**: if anything there contradicts this skill, follow `Issues.md`.
3. **Read `CLAUDE.md`** at the repo root if it exists — project-wide guidance for Claude that may include code conventions, restricted areas, or workflow tweaks that affect issue work. Treat its instructions as binding.
4. **If `Issues.md` is missing**, create it from `assets/Issues-md-template.md` before filing the first issue. Fill in the project name and a real one-paragraph description.
5. **Glance at one or two existing issue files** to absorb the project's tone (how detailed are descriptions, what platforms appear, how modules are named).
6. **Check whether `issues/` is tracked by git.** Run `git rev-parse --is-inside-work-tree` and `git check-ignore -q issues/`. The result determines whether lifecycle events below produce commits or are working-copy-only edits — see the "Git tracking" section.

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

The `resolved` → `closed` distinction is deliberate: `resolved` says "work landed", `closed` says "user confirmed". A subagent that finishes a fix may set `resolved`; only the user moves an issue to `closed`.

## Critical rule: never close without explicit confirmation

The single most important rule of this skill: an issue must **never** be marked `resolved`, `closed`, or `wontfix` based on inference. Only when the user has said so in plain language. Specifically, do not infer resolution from:

- a code change you (or a subagent) just made
- a commit message
- the filing of a related issue
- the user saying "thanks, that looks better" or "nice"

Always leave status at `open` (or `in-progress` if work has started) until the user confirms in words like "close this", "this is fixed", "mark resolved", or "won't fix". When in doubt, ask. The cost of asking is one turn; the cost of wrongly closing a real bug is that it disappears from the open list and gets forgotten.

The deliberate exception is the subagent resolve: a subagent that finishes a fix may set status to `resolved` (work-is-done-but-not-confirmed). It must not set `closed` — that's the user's call. This separation is the entire reason `resolved` and `closed` are different states.

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
| Filing a new issue | the new `NNNN.md` (and `Issues.md` if newly created) | `#NNNN <issue title>` |
| Resolving a bug — code commit | code changes only | `#NNNN <verb> <title>` (the substantive commit) |
| Resolving a bug — resolution commit | markdown update (status `resolved` + Closed + Commit + summary) | `#NNNN Resolve: <title>` |
| Subagent bail (notes added, status reverted to open) | markdown update | `#NNNN Notes: <brief>` |
| User-confirmed close | markdown update (status `closed`) | `#NNNN Close` |
| Marking won't-fix (after user decision) | markdown update | `#NNNN Won't fix` |

### Working-copy-only changes (no commit by the skill)

- Setting status to `in-progress` at the start of subagent work — this is transient and gets superseded by the resolve commit. Committing every status flip would create excessive churn.
- Any change the user explicitly says they'll commit themselves.

The Mac app reflects working-copy state regardless of whether anything is committed yet, so an uncommitted in-progress flip is still visible to the user.

### Why two commits to resolve, not one

The **Commit** metadata row records the hash of the code-fix commit. That hash isn't known until *after* the code commit lands, so the row can't appear in the same commit it points to. Splitting resolution into a code commit and a follow-up resolution commit keeps each commit single-purpose — "fix the code" and "document the fix" — and lets the resolution commit reference the hash cleanly.

## Filing a new issue

1. **Make sure `issues/Issues.md` exists.** If not, create it from `assets/Issues-md-template.md` first.
2. **Pick the next number.** List `issues/`, find the highest existing `NNNN.md`, increment. Start at `0001` if empty. Skip past reserved high numbers like `8888`/`9999` (used for test issues).
3. **Read `assets/issue-template.md`** and copy it to `issues/NNNN.md`, filling in the placeholders.
4. **Title** is a single declarative sentence describing the bug ("Reply button not functional on post cells"), not a question or a fix description.
5. **Status** starts at `open`, always. Even if a fix is already in flight, file as `open` and update status separately.
6. **First seen** is today's date (your `currentDate` context). Format `YYYY-MM-DD`.
7. **If `issues/` is tracked by git**, commit the new file immediately so the issue enters git history with its `open` status. Stage `issues/NNNN.md` (and `issues/Issues.md` if you just created it) and commit with message `#NNNN <issue title>`. If `issues/` is ignored or there's no git repo, skip this step.
8. **Confirm to the user** in one line — issue number and title. Don't paste the whole file back.

### Format details that always apply

- Title separator is an em-dash (U+2014, `—`), not a hyphen — the Mac app's regex won't match a hyphen.
- Field rows in the metadata table must keep the field name in `**bold**` exactly.
- `Module` can list multiple modules separated by ` / `.
- `Platform` is `iOS`, `macOS`, `iPadOS`, `All`, or any string. `All` matters — the Mac app's platform filter treats it as matching every platform.

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

When an issue is moved to `resolved` via the standard workflow below, additional sections (`## Root cause`, `## Fix`, `## Files changed`, optional `## Gotchas`) get added to capture what was wrong and what landed. See "Resolving an issue" for the full structure.

For any move toward `resolved`, `closed`, or `wontfix`, the "Critical rule" at the top of this skill applies — those transitions require explicit user confirmation, not inference.

## Resolving an issue (the standard workflow)

The standard way of working through open issues: each issue is handled by a fresh subagent. The orchestrator picks the issue; the subagent does the work in isolation and returns when done. This keeps subagent context small and lets the orchestrator coordinate without losing focus.

### Orchestrator: pick and dispatch

When the user says "work through the open issues", "pick up the next bug", or "fix the next one":

1. List `issues/*.md` (skip `Issues.md`). Find the lowest-numbered file whose status is `open`.
2. Spawn a fresh subagent with the issue id and a brief task description. Tell the subagent explicitly to: read `issues/Issues.md` and `CLAUDE.md` first to absorb project conventions, then read `issues/NNNN.md` for the issue itself, then follow the project's resolve workflow and return when done. The fresh context is a feature — the subagent loads the project's rules cleanly each time, so the user can adjust them via `Issues.md` or `CLAUDE.md` and the next subagent will pick the changes up automatically.
3. When the subagent returns, repeat for the next open issue — unless the user asked for just one, or the user wants to review before continuing.

If the user asks to work on a specific id ("fix 0046"), skip the picking step and dispatch to that id directly.

### Subagent: claim → fix → build → commit → resolve

When you've been dispatched to handle `NNNN`, you start with fresh context — so the first step is loading the project's conventions before touching anything.

1. **Orient in the project.** Read these in order, every time, before making any edit:
   - **`issues/Issues.md`** — the project's local guide. It defines the status vocabulary, module conventions, build/verify command, commit message conventions, and any project-specific rules. **If `Issues.md` contradicts this skill, follow `Issues.md`** — it's the project owner's chance to adjust the workflow per project, and it wins.
   - **`CLAUDE.md`** at the repo root, if it exists — project-wide guidance for Claude. Often defines code conventions, restricted areas, build/test commands, and project-specific dos and don'ts. Treat its instructions as binding.
   - **`issues/NNNN.md`** — the issue you're working on, in full, including any screenshots or logs in `issues/NNNN/`. If anything in the report is unclear, read related code before guessing.

   If `Issues.md` and `CLAUDE.md` disagree, prefer `CLAUDE.md` for code/repo-wide conventions and `Issues.md` for issue-tracking specifics. They usually cover different ground.

2. **Set status to `in-progress`** — edit the Status row and save. This is a working-copy edit only; do not commit. The Mac app reflects it within ~1s, signaling that the issue is claimed.
3. **Make the code changes** required to fix the bug.
4. **Run the project's build / test command** and confirm it passes. Fix any failures caused by your changes. If the build was already broken when you started (failures unrelated to your work), do not fix unrelated breakage — note it on the issue and bail (see below).

5. **Make the code commit.** Stage *only the code changes* — do not stage the issue markdown yet. The commit message starts with `#NNNN` and a short, declarative title — pick the verb that actually fits (`Fix`, `Add`, `Refactor`, `Update`, `Remove`, etc.). Not every issue is a bug fix; missing features, design refinements, and audits each get the verb that matches. After the title, leave a blank line, then add a paragraph or two of details. Example:

   ```
   #0046 Add navigation from avatar tap to profile

   The avatar tap on PostCardView was not wired to any NavigationLink.
   Threaded the author DID through the cell and connected onTapGesture
   to push ProfileView. Verified on both feed and thread views.
   ```

   If the project's `CLAUDE.md` or recent `git log` defines a different convention, follow that instead.

6. **Capture the commit hash** with `git rev-parse --short HEAD` immediately after the code commit lands. You'll record it on the issue in the next step.

7. **Update the issue markdown** to mark it resolved. Edit the metadata table:
   - Change the Status row to `resolved`.
   - Add a `**Closed**` row with today's date.
   - Add a `**Commit**` row with the short hash from step 6.

   Then add a structured summary in this order so the issue becomes a primary-source record of why the change happened:

   - **`## Root cause`** — one paragraph on what was actually wrong (often different from what the original report suggested).
   - **`## Fix`** — one paragraph on the approach you took.
   - **`## Files changed`** — a bulleted list, one bullet per file you touched, each with a short note describing what changed in that file.
   - **`## Gotchas`** *(optional)* — surprises, dead ends you tried, non-obvious behavior, or anything a future engineer working on similar code should know. Skip the section entirely if there's nothing notable. Be specific — across many issues these notes accumulate into docs about common pitfalls, and a vague "be careful with X" doesn't help future readers.

   Mentioning the commit hash inline in `## Fix` is fine but optional; the `**Commit**` metadata row is the canonical record.

8. **If `issues/` is tracked by git, make the resolution commit.** Stage `issues/NNNN.md` and commit with message `#NNNN Resolve: <issue title>`. Body of the commit should briefly describe what the resolution captures (e.g. "Records resolution of #NNNN; code change is in <hash>."). This second commit pairs with the code commit from step 5 — together they encode "fix landed, fix documented."

   **If `issues/` is ignored or there's no git repo**, skip this step. The markdown change from step 7 is the entire record.

Together, the metadata table (Status, Module, Platform, First seen, Closed, Commit) plus the summary sections plus the original Description/Steps/Expected/Actual give a complete record of the bug and its resolution. Future Claude sessions can grep across resolved issues to surface common gotchas and feed them into project-wide documentation.

Status moves are `open` → `in-progress` → `resolved`. **Never set `closed`** — that's the user's transition after they verify the fix in the Mac app.

The project's build command lives in the project's docs (`Issues.md`, `CLAUDE.md`, `README.md`), not this skill. Look there before assuming a default like `make`, `xcodebuild`, or `npm test`.

### When you can't finish

If the bug is unreproducible, out of scope, or the build won't pass after reasonable effort:

1. **Discard or stash any partial code changes.** The working copy needs to be clean before the bail commit so it doesn't accidentally include half-done work.
2. **Revert the status to `open`** in the issue markdown so it goes back into the queue. Don't leave it at `in-progress` — that signals active work and blocks the next iteration.
3. **Add a `## Notes` section** describing what you tried and why you stopped. Be specific: which approaches were attempted, what the failure mode was, what you'd try next. The next subagent (or the user) will start from your notes.
4. **If `issues/` is tracked by git**, commit just the markdown update with message `#NNNN Notes: <one-line summary of the bail>`. If `issues/` is ignored, skip the commit.
5. Return to the orchestrator with a one-line summary explaining the bail.

Never use `wontfix` or `closed` as an escape hatch for a stuck issue — those are the user's decisions.

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
- **Don't reformat existing issues** while updating them. Touch only the rows or sections that changed — diff-friendly edits matter when the user is reviewing through the Mac app.
- **Don't skip numbers.** Use the next sequential 4-digit id. Reserved high numbers (8888, 9999) for test/dummy issues are intentional — leave them alone.
- **Don't paraphrase the user's bug report into something cleaner.** Their words are usually closer to the truth than your interpretation. Quote text from screenshots verbatim where it appears.
- **Don't close-and-refile** to "clean up" an issue's history. Edit in place. The file *is* the history.

### Legacy projects

If the project still has `generate.py`, `issues.json`, `issues.js`, `index.html`, or a master Index table at the top of `Issues.md`, that's the older web-dashboard workflow. The Mac app has replaced it entirely. Leave the legacy files in place — don't update them, don't run them, don't add to them.
