---
name: issues
description: Manage a project's issues/ folder of NNNN.md markdown files — file new bugs, update status, attach screenshots, find the next number, keep issues/Issues.md in sync, and run the standard claim→fix→build→commit→resolve workflow when working through the queue. Use this skill whenever the user (or a subagent doing work on their behalf) describes a bug, regression, "broken" thing, or asks to "log this", "track this", "write this up", "make a ticket", "add to issues" — even if they don't explicitly say the word "issue". Also use it when the user asks to "work on the next open issue", "pick up the next bug", "fix issue NNNN", "work through the open issues", or to update status, attach a screenshot to an existing issue, or summarize what's open / in-progress for a project that has (or should have) an issues/ folder.
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
- **`references/parsing.md`** — exact regex patterns the Mac app uses. Read this only if you're debugging why something isn't appearing or rendering correctly. Not needed for normal filing.

## First, orient yourself

Before filing or updating, take a few seconds to check the project's state. This is cheap and prevents drift.

1. **Find the issues folder.** Usually `issues/` at the repo root.
2. **Read `issues/Issues.md`** if it exists — it's the project's local guide and defines the canonical status vocabulary, module conventions, and any project-specific rules. **`Issues.md` is authoritative for its project**: if anything there contradicts this skill, follow `Issues.md`.
3. **If `Issues.md` is missing**, create it from `assets/Issues-md-template.md` before filing the first issue. Fill in the project name and a real one-paragraph description.
4. **Glance at one or two existing issue files** to absorb the project's tone (how detailed are descriptions, what platforms appear, how modules are named).

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

## Filing a new issue

1. **Make sure `issues/Issues.md` exists.** If not, create it from `assets/Issues-md-template.md` first.
2. **Pick the next number.** List `issues/`, find the highest existing `NNNN.md`, increment. Start at `0001` if empty. Skip past reserved high numbers like `8888`/`9999` (used for test issues).
3. **Read `assets/issue-template.md`** and copy it to `issues/NNNN.md`, filling in the placeholders.
4. **Title** is a single declarative sentence describing the bug ("Reply button not functional on post cells"), not a question or a fix description.
5. **Status** starts at `open`, always. Even if a fix is already in flight, file as `open` and update status separately.
6. **First seen** is today's date (your `currentDate` context). Format `YYYY-MM-DD`.
7. **Confirm to the user** in one line — issue number and title. Don't paste the whole file back.

### Format details that always apply

- Title separator is an em-dash (U+2014, `—`), not a hyphen — the Mac app's regex won't match a hyphen.
- Field rows in the metadata table must keep the field name in `**bold**` exactly.
- `Module` can list multiple modules separated by ` / `.
- `Platform` is `iOS`, `macOS`, `iPadOS`, `All`, or any string. `All` matters — the Mac app's platform filter treats it as matching every platform.

## Updating an existing issue

1. Edit `issues/NNNN.md` in place. The Mac app picks up the change automatically.
2. If status changed, update the **Status** row.
3. If status moved to `resolved` or `closed`, add a `**Closed**` row with today's date. If the move to `resolved` was triggered by a fix commit, also add a `**Commit**` row with the short hash (`git rev-parse --short HEAD`).
4. Touch only what changed — don't reformat the rest of the file. Diff-friendly edits matter when the user reviews through the Mac app.

After a bug is closed, you may add `## Root cause` and `## Fix` sections explaining what was wrong and what landed. These are useful for future readers tracing why a fix happened.

### CRITICAL: do not close issues without explicit confirmation

An issue must **never** be marked `resolved`, `closed`, or `wontfix` unless the user has said so. Do not infer resolution from:

- a code change you (or a subagent) just made
- a commit message
- the filing of a related issue
- the user saying "thanks, that looks better" or "nice"

Always leave status as `open` (or `in-progress` if work has started) until the user confirms in plain language ("close this", "this is fixed", "mark resolved", "won't fix"). When in doubt, ask. The cost of asking is one turn; the cost of wrongly closing a real bug is that it disappears from the open list and gets forgotten.

A subagent that fixes a bug may set status to `resolved` (work-is-done-but-not-confirmed). It must not set `closed` — that's the user's call. This separation is the whole reason the two states exist.

## Resolving an issue (the standard workflow)

The standard way of working through open issues: each issue is handled by a fresh subagent. The orchestrator picks the issue; the subagent does the work in isolation and returns when done. This keeps subagent context small and lets the orchestrator coordinate without losing focus.

### Orchestrator: pick and dispatch

When the user says "work through the open issues", "pick up the next bug", or "fix the next one":

1. List `issues/*.md` (skip `Issues.md`). Find the lowest-numbered file whose status is `open`.
2. Spawn a fresh subagent with the issue id and a brief task description: read `issues/NNNN.md`, fix the bug, follow the project's resolve workflow, return when done.
3. When the subagent returns, repeat for the next open issue — unless the user asked for just one, or the user wants to review before continuing.

If the user asks to work on a specific id ("fix 0046"), skip the picking step and dispatch to that id directly.

### Subagent: claim → fix → build → commit → resolve

When you've been dispatched to handle `NNNN`:

1. **Read `issues/NNNN.md`** in full, including any screenshots or logs in `issues/NNNN/`. If something is unclear, read related code before guessing.
2. **Set status to `in-progress`** — edit the Status row and save. The Mac app reflects this within ~1s, signaling that the issue is claimed and avoiding double-work.
3. **Make the code changes** required to fix the bug.
4. **Run the project's build / test command** and confirm it passes. Fix any failures caused by your changes. If the build was already broken when you started (failures unrelated to your work), do not fix unrelated breakage — note it on the issue and bail (see below).
5. **Make a single git commit.** The commit message starts with `#NNNN` and a short, declarative title describing what this commit does — choose the verb that actually fits (`Fix`, `Add`, `Refactor`, `Update`, `Remove`, etc.). Not every issue is a bug fix; missing features, design refinements, and audits each get the verb that matches. After the title, leave a blank line, then add a paragraph or two of details. Example:

   ```
   #0046 Add navigation from avatar tap to profile

   The avatar tap on PostCardView was not wired to any NavigationLink.
   Threaded the author DID through the cell and connected onTapGesture
   to push ProfileView. Verified on both feed and thread views.
   ```

   This is the canonical format for issue-linked commits. If the project's `CLAUDE.md` or recent `git log` defines a different convention, follow that instead.
6. **Capture the commit hash** with `git rev-parse --short HEAD` immediately after the commit lands. You'll record it in the next step.
7. **Set status to `resolved`** and update the metadata table in one edit:
   - Change the Status row to `resolved`.
   - Add a `**Closed**` row with today's date.
   - Add a `**Commit**` row with the short hash from step 6.

   Then add `## Root cause` and `## Fix` sections summarizing what was wrong and what landed — these are the future-reader's view of why the change happened. Mentioning the commit hash inline in `## Fix` is fine but optional; the metadata row is the canonical record.

Status moves are `open` → `in-progress` → `resolved`. **Never set `closed`** — that's the user's transition after they verify the fix in the Mac app.

The project's build command lives in the project's docs (`Issues.md`, `CLAUDE.md`, `README.md`), not this skill. Look there before assuming a default like `make`, `xcodebuild`, or `npm test`.

### When you can't finish

If the bug is unreproducible, out of scope, or the build won't pass after reasonable effort:

1. **Revert the status to `open`** so the issue goes back into the queue. Don't leave it at `in-progress` — that signals active work and blocks the next iteration.
2. **Add a `## Notes` section** describing what you tried and why you stopped. Be specific: which approaches were attempted, what the failure mode was. The next subagent (or the user) will start from your notes.
3. **Don't commit partial changes.** Discard or stash your working copy so the next attempt starts clean.
4. Return to the orchestrator with a one-line summary explaining the bail.

Never use `wontfix` or `closed` as an escape hatch for a stuck issue — those are the user's decisions.

## Attaching screenshots and other artifacts

Screenshots, crash logs, console output, sample data — anything related to an issue — live in a sibling folder `issues/NNNN/`. Reference them from the issue's Attachments section with relative paths:

```markdown
## Attachments

![Reply button does nothing when tapped](screenshot.png)
![Crash log](crash.log)
```

Filenames are descriptive (`before.png`, `after.png`, `console.png`, `crash.log`) — not just `screenshot.png` when there are multiple.

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

- **Don't auto-close.** See the warning above. This is the single most important rule of the skill.
- **Don't create or maintain `generate.py`, `issues.json`, `issues.js`, or `index.html`.** Those are vestiges of an older workflow. The Mac app has replaced them. If they exist in a legacy project, leave them alone but don't update them.
- **Don't maintain an Index table** in `Issues.md` or anywhere else. The Mac app enumerates the folder directly. An Index table is extra state that goes stale.
- **Don't reformat existing issues** while updating them. Touch only the rows or sections that changed.
- **Don't skip numbers.** Use the next sequential 4-digit id. Reserved high numbers (8888, 9999) are intentional — leave them alone.
- **Don't paraphrase the user's bug report into something cleaner.** Their words are usually closer to the truth than your interpretation. Quote text from screenshots verbatim where it appears.
- **Don't close-and-refile** to "clean up" an issue's history. Edit in place. The file *is* the history.
