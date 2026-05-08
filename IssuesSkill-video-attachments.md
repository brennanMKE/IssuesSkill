# IssuesSkill update: video attachments

Hand-off doc for the Claude session updating the [IssuesSkill](https://github.com/brennanMKE/IssuesSkill) repo. Describes how the skill should handle video attachments (`.mov`, `.mp4`, `.m4v`, etc.) when filing or updating an issue.

## Goal

Today the skill copies any user-provided file into `issues/NNNN/` and references it from the issue's `## Attachments` section as a plain inline image:

```markdown
![Sidebar resize jitter](0072/sidebar-resize-jitter.mov)
```

That works for images, but every standard markdown renderer (GitHub, Marked, the companion Issues.app on macOS) treats `![…](path)` as an `<img>`. A `.mov` can't load as an image, so the attachment renders as broken — even though the file is present and correct.

This update teaches the skill to recognize video files and emit a markdown form that:

1. Renders correctly on GitHub and other plain markdown renderers (a clickable poster image that opens the video).
2. Gives Issues.app a clean signal to upgrade the click into an in-app video player.
3. Adds zero runtime dependencies — uses macOS-built-in `qlmanage` to extract the poster.

## The markdown form

For video attachments, emit **image-inside-a-link**:

```markdown
[![Sidebar resize jitter](0072/sidebar-resize-jitter.poster.png)](0072/sidebar-resize-jitter.mov)
```

- The image (`![…](…)`) is the auto-generated poster frame.
- The surrounding link (`[…](…)`) targets the video file.
- This is standard CommonMark — every renderer handles it. GitHub shows the poster as a clickable thumbnail that opens the `.mov`. Issues.app intercepts the click on `.movie`-conforming targets to play in-app.

For image attachments, **keep today's behavior unchanged**:

```markdown
![Filter bar where the new selector should go](0042/filter-bar.png)
```

The link wrapper is reserved for videos only. Don't apply it to images.

## File naming convention

For a video file `<basename>.<ext>`, place both files in `issues/NNNN/`:

```
issues/
└── 0072/
    ├── sidebar-resize-jitter.mov           ← original video
    └── sidebar-resize-jitter.poster.png    ← generated poster frame
```

Rules:

- Poster filename is `<basename>.poster.png` — same basename as the video, plus `.poster.png`. Predictable so future tooling (and the app's parser) can pair them without parsing the markdown.
- One poster per video. If the video is replaced, regenerate the poster.
- `.png` always — `qlmanage` outputs PNG by default; don't convert.

## Detecting "this is a video"

Use the file extension. Treat these as video (lowercase comparison):

- `.mov` · `.mp4` · `.m4v` · `.qt` · `.avi` · `.mkv` · `.webm`

If you want a more robust check, on macOS you can ask the system:

```sh
mdls -name kMDItemContentTypeTree -raw "$file" | tr ',' '\n' | grep -q public.movie && echo "video"
```

But the extension list is good enough for the skill's purposes — users hand over files with sensible extensions, and false positives/negatives here are recoverable.

## Generating the poster

Use `qlmanage` — ships with macOS, no install:

```sh
# General form
qlmanage -t -s 1280 -o "issues/NNNN" "issues/NNNN/<basename>.<ext>"

# qlmanage writes "<basename>.<ext>.png" — rename to the convention
mv "issues/NNNN/<basename>.<ext>.png" "issues/NNNN/<basename>.poster.png"
```

- `-t` — generate a thumbnail.
- `-s 1280` — max edge in pixels. 1280 is enough for the inline 360×240 thumbnail in Issues.app and still reasonable when rendered full-width on GitHub.
- `-o <dir>` — output directory. Use the issue's attachment folder so both files end up adjacent.

`qlmanage` succeeds silently. To verify, check the renamed file exists; if not, fall back gracefully (see "When poster generation fails" below).

### Worked example

Filing an issue with a screen recording at `~/Desktop/Sidebar resizing jitter.mov`:

```sh
mkdir -p issues/0072
cp ~/Desktop/Sidebar\ resizing\ jitter.mov issues/0072/sidebar-resize-jitter.mov

qlmanage -t -s 1280 -o issues/0072 issues/0072/sidebar-resize-jitter.mov >/dev/null 2>&1
mv issues/0072/sidebar-resize-jitter.mov.png issues/0072/sidebar-resize-jitter.poster.png
```

Then in `issues/0072.md`, the Attachments section becomes:

```markdown
## Attachments

[![Sidebar resize jitter](0072/sidebar-resize-jitter.poster.png)](0072/sidebar-resize-jitter.mov)
```

That's all that changes from today's flow.

## When poster generation fails

`qlmanage` can fail on corrupt files, unsupported codecs, or sandboxed environments. If the rename step doesn't produce `<basename>.poster.png`:

1. Skip the poster — fall back to the plain inline image form (`![alt](video.mov)`). The attachment will still be present and openable; it just won't preview.
2. Add a one-line note inside the `## Attachments` section so the user knows the poster is missing:
   ```markdown
   ![Sidebar resize jitter](0072/sidebar-resize-jitter.mov)

   <!-- poster generation failed; original video is the attachment -->
   ```
3. Don't loop or retry. Don't abort filing the issue — the issue itself is still valuable without the poster.

## Multiple attachments

Each video gets its own poster. Order in the markdown follows the order the user provided files:

```markdown
## Attachments

[![Before fix](0072/before.poster.png)](0072/before.mov)
[![After fix](0072/after.poster.png)](0072/after.mov)
![Console output](0072/console.png)
```

Posters and originals are listed/discoverable as siblings in `0072/`. No index file, no metadata, no JSON.

## Updating existing issues

When a user adds a video to an issue that already exists:

1. Copy the video into `issues/NNNN/`.
2. Generate the poster (above).
3. Append the `[![alt](poster)](video)` line to the existing `## Attachments` section. Don't reformat what's already there.
4. If the issue had no `## Attachments` section yet, add one — same format as a new issue.

## Replacing a video

If the user replaces a video with a corrected version:

1. Overwrite the `.mov` in place.
2. Re-run the poster generation step. Overwrite the `.poster.png`.
3. Don't change the markdown — the references stay the same.

## What NOT to do

- Don't store posters anywhere other than `issues/NNNN/`. They are an attachment, not a generated artifact, and they live with the video.
- Don't `.gitignore` posters. They're checked in alongside videos so GitHub renders cleanly without runtime regeneration.
- Don't strip the link wrapper on subsequent edits. Once an attachment is `[![…](poster)](video)`, leave that shape alone.
- Don't add the link wrapper to plain image attachments — it changes click behavior in Issues.app for no reason.
- Don't generate posters for animated GIFs. GIFs render natively in markdown; treat as images.

## Companion Mac app behavior (FYI, not the skill's concern)

For context only — the parallel issue tracked in the Issues.app repo (`brennanMKE/Issues`) is `#0073`. The app will:

1. Extend `InlineImageMarkdown.split` (or the equivalent extractor) to recognize the surrounding link.
2. When the link target's UTI conforms to `.movie`, render the existing thumbnail with a play-button overlay.
3. On click, present an `AVPlayerView` popover/sheet inside the app instead of opening the file in QuickTime.
4. When the link target is anything else (or absent), behave as today.

The skill doesn't need to know any of this — its job is to emit the markdown shape and put the files on disk. The app's upgrade is independent and degrades gracefully (a non-upgraded app shows the poster image with a click-to-open-externally fallback).

## Summary checklist for the skill update

- [ ] Detect video files by extension on attachment.
- [ ] For videos, run `qlmanage -t -s 1280 -o <dir> <video>` and rename output to `<basename>.poster.png`.
- [ ] Emit `[![alt](NNNN/<basename>.poster.png)](NNNN/<basename>.<ext>)` in `## Attachments`.
- [ ] Fall back to plain `![alt](NNNN/<basename>.<ext>)` if poster generation fails, with a `<!-- poster generation failed -->` comment.
- [ ] Update the skill's instructions to describe the new behavior alongside the existing screenshot guidance (including the macOS U+202F filename gotcha for `Screen Recording …` filenames, which has the same narrow-no-break-space issue as screenshots).
- [ ] Add an example to `assets/Issues-md-template.md` showing both image and video attachment forms.
