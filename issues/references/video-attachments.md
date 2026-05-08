# Video attachments

How to attach `.mov`, `.mp4`, and other video files to an issue. Videos need a different markdown shape than images — read this whenever a user hands over a screen recording or any video file.

## Why videos are different

Image attachments use the standard inline form:

```markdown
![Filter bar where the new selector should go](1335/filter-bar.png)
```

Every markdown renderer (GitHub, Marked, Issues.app) treats `![…](path)` as an `<img>` tag. A `.mov` can't load as an image, so the same form renders as a broken image even though the file is present and correct.

The fix is to render a **poster image inside a link** that targets the video. The poster loads as an image; the click opens the video.

## The markdown form

```markdown
[![Sidebar resize jitter](1335/sidebar-resize-jitter.poster.png)](1335/sidebar-resize-jitter.mov)
```

- The inner `![…](…)` is the auto-generated poster frame.
- The outer `[…](…)` link targets the video file.
- This is standard CommonMark — every renderer handles it. GitHub shows the poster as a clickable thumbnail that opens the `.mov`. Issues.app intercepts the click on `.movie`-conforming targets to play in-app.

For image attachments, **keep the existing form**:

```markdown
![Filter bar where the new selector should go](1335/filter-bar.png)
```

The link wrapper is reserved for videos only. Don't apply it to images.

## Detecting "this is a video"

Check the file extension (case-insensitive). Treat any of these as a video for the purposes of choosing the markdown shape:

| Extension | Container | qlmanage poster on stock macOS? |
|---|---|---|
| `.mov` | QuickTime | ✅ reliable (AVFoundation) |
| `.mp4` | MPEG-4 | ✅ reliable (AVFoundation) |
| `.m4v` | MPEG-4 (Apple) | ✅ reliable (AVFoundation) |
| `.qt` | QuickTime | ✅ reliable |
| `.avi` | AVI | ⚠️ usually works (legacy QuickTime path); some codecs fail |
| `.mkv` | Matroska | ❌ generally fails without a third-party Quick Look generator (VLC's, etc.) |
| `.webm` | WebM | ❌ generally fails without a third-party Quick Look generator |
| `.mpeg`, `.mpg` | MPEG | ⚠️ usually works for MPEG-2 program streams |
| `.3gp`, `.3g2` | 3GPP | ⚠️ usually works (AVFoundation) |
| `.flv` | Flash Video | ❌ generally fails on stock macOS |
| `.wmv` | Windows Media | ❌ generally fails on stock macOS |

The first four (`.mov`, `.mp4`, `.m4v`, `.qt`) are by far the most common — screen recordings and exports from QuickTime, ScreenFlow, OBS, mobile phones, and most editors land in one of those. Treat the rest as "try `qlmanage`, fall back if it fails."

For everything in the table above, **the link-around-image markdown shape applies regardless of whether the poster generation succeeds.** If poster generation fails, the markdown becomes `![alt](NNNN/file.ext)` with a `<!-- poster generation failed -->` comment, but the file is still attached and openable.

If you want a more robust runtime check on macOS:

```bash
mdls -name kMDItemContentTypeTree -raw "$file" | tr ',' '\n' | grep -q public.movie && echo "video"
```

But the extension list is the default path — users hand over files with sensible extensions, and edge cases here degrade to the same fallback.

**Animated GIFs are not videos** for this purpose. GIFs render natively as `<img>` in markdown — treat them as images and use `![alt](NNNN/file.gif)`.

## File layout

For a video file `<basename>.<ext>`, both the video and its poster live in `issues/NNNN/`:

```
issues/
└── 1335/
    ├── sidebar-resize-jitter.mov           ← original video
    └── sidebar-resize-jitter.poster.png    ← generated poster frame
```

Rules:

- **Poster filename is `<basename>.poster.png`** — same basename as the video, plus `.poster.png`. Predictable so future tooling (and Issues.app's parser) can pair them without parsing the markdown.
- **One poster per video.** If the video is replaced, regenerate the poster.
- **`.png` always.** `qlmanage` outputs PNG by default; don't convert.
- **Both files live under `issues/NNNN/`** — same relative-path rule as any other attachment (see `references/issue-format.md`).

## Generating the poster

Use `qlmanage` — it ships with macOS, no install needed:

```bash
qlmanage -t -s 1280 -o "issues/NNNN" "issues/NNNN/<basename>.<ext>"

# qlmanage writes "<basename>.<ext>.png" — rename to the convention
mv "issues/NNNN/<basename>.<ext>.png" "issues/NNNN/<basename>.poster.png"
```

Flags:

- `-t` — generate a thumbnail.
- `-s 1280` — max edge in pixels. Enough for the inline 360×240 thumbnail in Issues.app and still reasonable when GitHub renders full-width.
- `-o <dir>` — output directory. Use the issue's attachment folder so both files end up adjacent.

`qlmanage` succeeds silently. To verify, check that the renamed file exists; if not, fall back gracefully (see "When poster generation fails").

## Worked example

User hands over a screen recording at `~/Desktop/Sidebar resizing jitter.mov` for issue 1335:

```bash
mkdir -p issues/1335
cp ~/Desktop/Sidebar\ resizing\ jitter.mov issues/1335/sidebar-resize-jitter.mov

qlmanage -t -s 1280 -o issues/1335 issues/1335/sidebar-resize-jitter.mov >/dev/null 2>&1
mv issues/1335/sidebar-resize-jitter.mov.png issues/1335/sidebar-resize-jitter.poster.png
```

Then in `issues/1335.md`:

```markdown
## Attachments

[![Sidebar resize jitter](1335/sidebar-resize-jitter.poster.png)](1335/sidebar-resize-jitter.mov)
```

## When poster generation fails

`qlmanage` can fail on corrupt files, unsupported codecs, or sandboxed environments. If the rename step doesn't produce `<basename>.poster.png`:

1. **Skip the poster** — fall back to the plain inline form `![alt](NNNN/<basename>.<ext>)`. The file is still attached and openable; it just won't preview.
2. **Add a one-line HTML comment** in the Attachments section so the user knows the poster is missing:

   ```markdown
   ![Sidebar resize jitter](1335/sidebar-resize-jitter.mov)

   <!-- poster generation failed; original video is the attachment -->
   ```

3. **Don't loop or retry.** Don't abort filing the issue — the bug report is still valuable without the preview.

## macOS Screen Recording filename gotcha

macOS Screen Recording filenames have the same narrow no-break space (U+202F) gotcha as screenshots — there's a U+202F before AM/PM that visually looks like a regular space but breaks literal `cp`. Glob past it:

```bash
mkdir -p issues/1335
cp ~/Desktop/Screen\ Recording\ 2026-05-07\ at\ 3.42*PM.mov issues/1335/sidebar-resize-jitter.mov
```

The `*` matches the U+202F. List `~/Desktop/Screen\ Recording*` by mtime if you don't know which recording the user means, or ask.

## Multiple attachments in one issue

Each video gets its own poster. Order in the markdown follows the order the user provided the files:

```markdown
## Attachments

[![Before fix](1335/before.poster.png)](1335/before.mov)
[![After fix](1335/after.poster.png)](1335/after.mov)
![Console output](1335/console.png)
```

Posters and originals are siblings in `1335/`. No index file, no metadata, no JSON.

## Updating existing issues

When a user adds a video to an issue that already has attachments:

1. Copy the video into `issues/NNNN/`.
2. Generate the poster (see above).
3. Append the `[![alt](poster)](video)` line to the existing `## Attachments` section. Don't reformat what's already there.
4. If the issue had no `## Attachments` section yet, add one — same format as a new issue.

## Replacing a video

If the user hands over a corrected version of an existing video:

1. Overwrite the `.mov` in place.
2. Re-run the poster generation step. Overwrite the `.poster.png`.
3. **Don't change the markdown** — the link targets stay the same.

## Companion Mac app behavior (FYI)

The Mac app (Issues.app) recognizes the link-around-image shape: when the link target's UTI conforms to `.movie`, it renders the existing thumbnail with a play-button overlay and presents an in-app `AVPlayerView` on click instead of opening the video in QuickTime. When the link target is anything else (or absent), it falls back to today's image rendering.

The skill doesn't need to know any of this — its job is to emit the right markdown shape and put the files on disk. The app's behavior degrades gracefully on a non-upgraded build (poster image with click-to-open-externally fallback).

## Anti-patterns

- **Don't store posters anywhere other than `issues/NNNN/`.** They live with the video, not in a generated/build folder.
- **Don't `.gitignore` posters.** Check them in alongside videos so GitHub renders cleanly without runtime regeneration.
- **Don't strip the link wrapper on subsequent edits.** Once an attachment is `[![…](poster)](video)`, leave that shape alone.
- **Don't add the link wrapper to plain image attachments.** It changes click behavior in Issues.app for no benefit.
- **Don't generate posters for animated GIFs.** GIFs render natively in markdown; treat them as images.
- **Don't strip the folder prefix.** Same relative-path rule as images: `1335/file.mov`, not `file.mov` and not `issues/1335/file.mov`.

## Quick checklist

- [ ] Detect video by extension (case-insensitive).
- [ ] Copy the video into `issues/NNNN/`.
- [ ] `qlmanage -t -s 1280 -o issues/NNNN <video>` then rename to `<basename>.poster.png`.
- [ ] Emit `[![alt](NNNN/<basename>.poster.png)](NNNN/<basename>.<ext>)` in `## Attachments`.
- [ ] Fall back to plain `![alt](NNNN/<basename>.<ext>)` with a `<!-- poster generation failed -->` comment if poster generation didn't produce the file.
