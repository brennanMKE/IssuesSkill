# Issue file format

Canonical spec for what an `issues/NNNN.md` file looks like. Use this when you need the rules in one place — the templates in `assets/` show the shape; this file explains the why and the edge cases.

## Filename

- Pattern: `^\d{4}\.md$` — exactly four digits, then `.md`. `0001.md`, `0042.md`, `1335.md`.
- Anything else (`001.md`, `0001a.md`, `0001.markdown`) is silently skipped by the Mac app.
- Reserved high numbers `8888.md` and `9999.md` are for test/dummy issues — leave them alone.

## Title line

```markdown
# 1335 — Reply button not functional on post cells
```

- Starts with `# ` (one hash, one space).
- Then the 4-digit id.
- Then a space, an **em-dash** (U+2014, `—`), a space.
- Then the title — a single declarative sentence describing the bug, not a question or a fix description.

A hyphen (`-`) or en-dash (`–`) won't match the Mac app's regex and the issue will render without a title. The em-dash is non-negotiable.

## Metadata table

```markdown
| | |
|---|---|
| **Status** | open |
| **Module** | BlueskyFeed |
| **Platform** | iOS |
| **First seen** | 2026-05-07 |
```

Rules:

- **Field name in `**bold**`** — the parser scans for `\|\s*\*\*<NAME>\*\*\s*\|`, so the asterisks are required.
- **Status** — one of `open`, `in-progress`, `resolved`, `closed`, `wontfix`. Lowercase, hyphenated.
- **Module** — one or more module names separated by ` / ` (space-slash-space). The first segment is the primary module.
- **Platform** — `iOS`, `macOS`, `iPadOS`, `All`, or any string. `All` matches every platform filter.
- **First seen** — `YYYY-MM-DD`. Other date formats parse as missing.
- **Header/separator rows** are not required by the parser but the templates include them for readability — keep them.

When the issue moves to `resolved` or `closed`, two more rows get appended:

```markdown
| **Closed** | 2026-05-08 |
| **Commit** | a1b2c3d |
```

- **Closed** — `YYYY-MM-DD`. Add when status moves to `resolved` or `closed`.
- **Commit** — short hash from `git rev-parse --short HEAD` of the code-fix commit. Add only when the resolution was driven by a code commit.

## Sections

In order:

1. `## Description` — what is wrong. **First paragraph leads with the punchline** — the Mac app's summary view shows only that paragraph.
2. `## Steps to reproduce` — numbered list. Optional for design/feature issues.
3. `## Expected behavior` — one paragraph. Optional.
4. `## Actual behavior` — one paragraph. Optional.
5. `## Attachments` — image/file links. See below for path rules. Omit the section if there are no attachments.
6. `## Notes` — any extra context, guesses at root cause, related code locations. Optional.

When an issue is resolved via the standard workflow, additional sections get appended after the original ones:

7. `## Root cause` — what was actually wrong.
8. `## Fix` — the approach taken.
9. `## Files changed` — bulleted list, one bullet per file with a short note.
10. `## Gotchas` — optional. Surprises, dead ends, non-obvious behavior.

The Mac app currently only parses the metadata table and `## Description`. Other sections render in the detail panel but aren't filtered/searched against — keep them readable for humans.

## Attachment paths — the relative-path rule

Attachments live in a **sibling folder** named after the issue's id:

```
issues/1335.md           ← the markdown
issues/1335/             ← the attachments folder
issues/1335/screenshot.png
issues/1335/crash.log
```

The link in the markdown is **relative to the `.md` file itself**, which means the folder prefix is part of the path:

```markdown
## Attachments

![Reply button does nothing when tapped](1335/screenshot.png)
![Crash log](1335/crash.log)
```

Three forms to remember — only the first is correct:

| Link target | Resolves to | Correct? |
|---|---|---|
| `1335/screenshot.png` | `issues/1335/screenshot.png` | ✅ |
| `screenshot.png` | `issues/screenshot.png` (doesn't exist) | ❌ |
| `issues/1335/screenshot.png` | `issues/issues/1335/screenshot.png` (doesn't exist) | ❌ |

Why this matters: Markdown image/link paths are interpreted relative to the file containing them. The issue file is at `issues/1335.md`, so its working directory for resolving links is `issues/`. From there, the screenshot is at `1335/screenshot.png` — the folder prefix is required.

Other rules:

- **Filenames are descriptive** — `before.png`, `after.png`, `console.png`, `crash.log`. Not just `screenshot.png` when there are multiple.
- **Caption is the alt text** — describes what's in the file ("Reply button does nothing when tapped"), not its filename.
- **Folder is created on demand.** If `issues/NNNN/` doesn't exist when the first attachment is added, `mkdir -p issues/NNNN` first.
- **Don't link to files outside `issues/NNNN/`.** No `../`, no absolute paths, no Desktop links — the issue should be self-contained so the Mac app can resolve attachments without leaving the folder.
- **Videos use a different shape.** `.mov`/`.mp4`/etc. can't load as `<img>`; they're emitted as `[![alt](NNNN/file.poster.png)](NNNN/file.mov)`. The relative-path rule still applies — both poster and video live in `issues/NNNN/`. See `video-attachments.md` for the full procedure.

### macOS screenshot copy

macOS screenshot filenames contain a narrow no-break space (U+202F) before AM/PM. Glob past it:

```bash
mkdir -p issues/1335
cp ~/Desktop/Screenshot\ 2026-05-07\ at\ 3.42*PM.png issues/1335/screenshot.png
```

Once the file is in place, reference it as `1335/screenshot.png` in the markdown — the on-disk filename and the link target should match.

## A complete minimal example

```markdown
# 1335 — Reply button not functional on post cells

| | |
|---|---|
| **Status** | open |
| **Module** | BlueskyFeed |
| **Platform** | iOS |
| **First seen** | 2026-05-07 |

## Description

Tapping the reply button on a post cell in the timeline does nothing. No
sheet appears, no navigation occurs, no console output.

## Steps to reproduce

1. Open the app and sign in.
2. Scroll the timeline until any post cell is visible.
3. Tap the reply button.

## Expected behavior

A reply composer sheet appears.

## Actual behavior

Nothing happens.

## Attachments

![Reply button does nothing when tapped](1335/screenshot.png)

## Notes

Likely a missing action wiring on `PostCardView`. The repost button on the
same cell works, so the cell itself receives taps.
```

## Anti-patterns

- **Don't strip the folder prefix from attachment paths.** `screenshot.png` looks tidy but resolves to the wrong location.
- **Don't paraphrase the user's bug report into something cleaner.** Their words are usually closer to the truth than your interpretation.
- **Don't reformat existing issues** when updating them. Diff-friendly edits matter when the user is reviewing through the Mac app.
- **Don't skip the em-dash** in the title — a hyphen breaks Mac app rendering.
