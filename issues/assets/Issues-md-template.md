# {Project Name}

{One-paragraph description: what app or repo this tracker covers, what platforms it targets, what kinds of issues belong here.}

This file is the local guide for managing issues in this project. The companion Mac app (Issues.app) watches the `issues/` folder and renders the current state. Markdown files are the source of truth — there is no JSON, generated artifact, or index to keep in sync.

## Folder layout

```
issues/
├── Issues.md          # this file
├── 0001.md            # one file per issue
├── 0001/              # optional sibling folder for screenshots, crash logs, etc.
│   └── screenshot.png
├── 0002.md
└── …
```

## Status values

| File value | Display name | Meaning |
|---|---|---|
| `open` | Open | Filed but not yet started |
| `in-progress` | In Progress | Actively being worked on |
| `resolved` | Resolved | Work is done; awaiting user confirmation |
| `closed` | Closed | User has confirmed the fix |
| `wontfix` | Won't Fix | Acknowledged but won't be addressed |

Use the **file value** (lowercase, hyphenated) in the issue's metadata table. The Mac app converts to the display name when rendering.

## Issue file format

Each issue is `NNNN.md` (4-digit zero-padded) with this structure:

```markdown
# NNNN — Title

| | |
|---|---|
| **Status** | open |
| **Module** | <module name(s)> |
| **Platform** | iOS · macOS · iPadOS · All |
| **First seen** | YYYY-MM-DD |

## Description

What is wrong. Lead with the punchline — the first paragraph shows in the Mac app summary.

## Steps to reproduce

1. …
2. …

## Expected behavior

What should happen.

## Actual behavior

What actually happens.

## Attachments

![caption](screenshot.png)

## Notes

Any additional context, guesses at root cause, related code locations.
```

### Format details that matter

- **Title separator** is an em-dash (U+2014, `—`), not a hyphen.
- **Metadata field rows** must keep the field name in `**bold**` exactly.
- **Dates** are `YYYY-MM-DD`.
- **Module** can list multiple modules separated by ` / ` (e.g. `BlueskyFeed / BlueskyDataStore`).
- **Platform** is `iOS`, `macOS`, `iPadOS`, `All`, or any other string. `All` is treated as matching every platform filter.
- When status moves to `resolved` or `closed`, add a `**Closed**` row with today's date. When the move to `resolved` is the result of a fix commit, also add a `**Commit**` row with the short hash (`git rev-parse --short HEAD`).
- Steps / Expected / Actual / Attachments / Notes are conventional but not all required — for design-refinement or feature-gap issues, Description alone is fine.

## Filing a new issue

1. Find the highest existing `NNNN.md` and increment. Start at `0001` if the folder is empty. Skip past reserved high numbers (e.g. `8888`, `9999` for test issues).
2. Create `issues/NNNN.md` from the template.
3. Set status to `open`.
4. Use today's date for First seen.
5. Phrase the title as a single declarative sentence describing the bug, not a question or a fix description.

## Updating an issue

Edit the file in place. The Mac app picks up changes automatically — no follow-up command. Touch only the rows or sections that changed; don't reformat the rest.

When status moves to `resolved` or `closed`, add a `**Closed**` row with the date. When the move to `resolved` was driven by a fix commit, also add a `**Commit**` row with the short hash.

## CRITICAL: do not close issues without explicit confirmation

An issue must **never** be marked `resolved`, `closed`, or `wontfix` unless the user has explicitly said so. Do not infer resolution from:

- a code change you (or a subagent) just made
- a commit message
- the filing of a related issue
- the user saying "thanks, that looks better"

When in doubt, ask. Leave status at `open` or `in-progress` until the user confirms in plain language ("close this", "this is fixed", "mark resolved", "won't fix").

A subagent that finishes implementing a fix may set status to `resolved` (the work-is-done-but-not-confirmed state). It must not set `closed` — that's the user's call.

## Resolving an issue (the standard workflow)

Each open issue is handled by a fresh subagent. The orchestrator picks the issue; the subagent does the work in isolation and returns when done.

### Orchestrator: pick and dispatch

1. List `issues/*.md` (skip `Issues.md`). Pick the lowest-numbered file whose status is `open`.
2. Spawn a fresh subagent with the issue id and instructions to follow the resolve workflow below.
3. When the subagent returns, move on to the next open issue (or stop if only one was requested).

If the user names a specific issue ("fix 0046"), dispatch to that id directly.

### Subagent: claim → fix → build → commit → resolve

1. **Read `issues/NNNN.md`** in full, including any attachments in `issues/NNNN/`.
2. **Set status to `in-progress`** and save. This claims the issue.
3. **Make the code changes** required to fix the bug.
4. **Run the project build / verification command** and confirm it passes. Fix failures caused by your changes. If the build was already failing before you started, note it on the issue and bail — don't fix unrelated breakage.
5. **Make a single git commit.** The message starts with `#NNNN` and a short, declarative title — pick the verb that actually fits the change (`Fix`, `Add`, `Refactor`, `Update`, `Remove`, etc.); not every issue is a bug fix. Leave a blank line after the title, then add a paragraph of details. Example:

   ```
   #0046 Add navigation from avatar tap to profile

   The avatar tap on PostCardView was not wired to any NavigationLink.
   Threaded the author DID through the cell and connected onTapGesture
   to push ProfileView.
   ```
6. **Capture the commit hash** with `git rev-parse --short HEAD`.
7. **Set status to `resolved`** and update the metadata table:
   - Change Status to `resolved`.
   - Add a `**Closed**` row with today's date.
   - Add a `**Commit**` row with the short hash from step 6.

   Add `## Root cause` and `## Fix` sections summarizing what was wrong and what landed.

Status flow: `open` → `in-progress` → `resolved`. **Never set `closed`** — the user does that after verifying the fix.

### Build / verify command for this project

<!-- Replace this with the actual command(s) used to verify a fix in this project. Delete if the project has these documented elsewhere (e.g. CLAUDE.md). -->

- e.g. `xcodebuild -scheme MyApp build` or `npm test`

### When the subagent can't finish

If the bug is unreproducible, out of scope, or the build won't pass after reasonable effort:

1. **Revert status to `open`** so the issue goes back into the queue.
2. **Add a `## Notes` section** describing what was tried and why work stopped. Be specific.
3. **Discard or stash partial changes** — don't commit them.
4. Return with a one-line summary of why work stalled.

Never use `wontfix` or `closed` to escape a stuck issue.

## Attachments

Screenshots, crash logs, console output, sample data, etc. live in a sibling folder `issues/NNNN/`. Reference them with relative paths from the issue file:

```markdown
## Attachments

![Reply button does nothing when tapped](screenshot.png)
![Crash log](crash.log)
```

### macOS screenshot filename gotcha

macOS screenshot filenames use a **narrow no-break space** (U+202F) before AM/PM, visually identical to a regular space. A literal `cp` of the quoted filename will fail with "No such file or directory". Use a glob to skip past it:

```bash
mkdir -p issues/NNNN
cp ~/Desktop/Screenshot\ YYYY-MM-DD\ at\ H.MM.SS*PM.png issues/NNNN/screenshot.png
```

The `*` matches the U+202F. Substitute the actual timestamp; if you don't know which screenshot the user means, list `~/Desktop/Screenshot*` by mtime and pick the most recent.

## Module conventions for this project

<!-- List the canonical module / area names so issues stay consistent. Delete this section if not relevant. -->

- e.g. `BlueskyFeed`, `BlueskyAuth`, `Bluesky-SwiftUI`
