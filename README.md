# IssuesSkill

A [Claude Code](https://claude.com/claude-code) skill for managing a project's `issues/` folder — a lightweight, repo-local bug tracker built on plain markdown files.

## Overview

Each project keeps an `issues/` folder containing zero-padded `NNNN.md` files (`0001.md`, `0002.md`, …) alongside an `issues/Issues.md` workflow guide and an `issues/project.json` file naming the project and its repo. When you describe a bug to Claude — verbally, or by pasting a screenshot or screen recording — Claude files it as a markdown file so the conversation can move on without the bug being lost. When you ask to update status, attach an artifact, generate a status report, or summarize what's open, Claude works on the same files.

The skill lives entirely on the conventions: file format, status vocabulary, the rule that nothing gets closed without your explicit confirmation, and the macOS screenshot/screen-recording copy gotcha (filenames contain a U+202F narrow no-break space). It bundles a few small templates and a handful of load-on-demand references so projects stay consistent across sessions.

## How it fits with the Mac app

The companion [Issues.app](https://github.com/brennanMKE/Issues) is a native macOS viewer that watches the `issues/` folder and renders the current state — swimlane, timeline, and list views, with status / module / platform filters. Markdown files are the source of truth; the Mac app re-reads them on every change.

The two projects are complementary and independent:

- **This skill** is how Claude Code (or a subagent) authors and edits the markdown.
- **Issues.app** is how you read and triage them.

You don't need the Mac app to use the skill, and you don't need the skill to use the Mac app — but together they cover both sides of the workflow.

## Installation

### Quick install (recommended) — `npx skills`

Installs across supported AI coding agents (Claude Code, Codex, Cursor, and others)
with one command:

```bash
npx skills add brennanMKE/IssuesSkill --skill issues
```

Target specific tools, or run non-interactively:

```bash
npx skills add brennanMKE/IssuesSkill --skill issues -a claude-code -a codex
npx skills add brennanMKE/IssuesSkill --skill issues -a claude-code -g -y
```

> **Claude Code note:** if the skill installs but Claude Code doesn't see it, it
> likely landed in `~/.agents/skills/` without a `~/.claude/skills/` symlink. Fix:
> ```bash
> ln -s ~/.agents/skills/issues ~/.claude/skills/issues
> ```

### Manual install — `install.sh`

Clone the repo and run the installer. It links the skill into the global skills dir
of every detected tool (Claude Code, Codex):

```bash
git clone https://github.com/brennanMKE/IssuesSkill.git
cd IssuesSkill
./install.sh
```

Cursor has no global skills directory, so install it per project:

```bash
./install.sh --project /path/to/your/project   # links into <project>/.cursor/skills
```

Install straight from git without a manual clone (caches and updates on re-run):

```bash
REPO_URL=https://github.com/brennanMKE/IssuesSkill.git ./install.sh
```

Because `install.sh` uses symlinks, edits to the skill files are picked up without
reinstalling.

### Where skills live

| Tool        | Global               | Project            |
|-------------|----------------------|--------------------|
| Claude Code | `~/.claude/skills/`  | `.claude/skills/`  |
| Codex CLI   | `~/.codex/skills/`   | `.codex/skills/`   |
| Cursor      | *(project only)*     | `.cursor/skills/`  |

## Removal

```bash
rm ~/.claude/skills/issues
rm ~/.codex/skills/issues
rm /path/to/project/.cursor/skills/issues
```

Removing a symlink leaves the source repo untouched.

## Usage

Once installed, the skill triggers automatically when you describe a bug, ask to update an issue, or query what's open. You don't need to invoke it explicitly.

Examples of prompts that trigger it:

- *"the like button doesn't persist when I tap a post — file this"*
- *"mark 0046 as in progress, I'm starting on it"*
- *"what's still open in BlueskyFeed?"*
- *"set up issue tracking for this project"*

When you describe a bug for the first time in a project, Claude will create `issues/Issues.md` from the bundled template and then file the issue. From that point on, every issue lives as `issues/NNNN.md` with a metadata table (Status, Module, Platform, First seen) followed by Description, Steps to reproduce, etc.

Status values: `open` · `in-progress` · `resolved` · `closed` · `wontfix`. The `resolved` → `closed` distinction is deliberate — `resolved` says "work landed", `closed` says "user confirmed". Claude (and any subagent it spawns) will never set `closed` on its own.

## Project layout

```
IssuesSkill/
├── CONTRIBUTING.md                  # how to file issues / send PRs against this skill
├── install.sh                       # symlinks the skill into ~/.claude/skills/
├── LICENSE                          # MIT
├── README.md                        # this file
└── issues/                          # the skill itself
    ├── SKILL.md                     # workflow + rules, always loaded by Claude
    ├── assets/                      # templates copied verbatim into target projects
    │   ├── issue-template.md        # template for a new NNNN.md
    │   ├── Issues-md-template.md    # template for a project's issues/Issues.md
    │   └── project.json             # template for a project's issues/project.json
    └── references/                  # docs loaded on demand by Claude
        ├── cost-tracking.md         # per-issue token usage, model, and cost recording
        ├── issue-format.md          # canonical issue file format spec
        ├── parsing.md               # Mac app regex details (debug-only)
        ├── project-config.md        # schema and workflow for project.json
        ├── status-reports.md        # snapshot reports of the issue queue
        └── video-attachments.md     # how to attach videos (.mov, .mp4, etc.)
```

## Maintaining the skill

When the workflow described in `issues/SKILL.md` changes, mirror the change in `issues/assets/Issues-md-template.md`. The two files intentionally duplicate the workflow content so projects can use `Issues.md` as a self-contained guide even when the skill isn't installed. Without that discipline, the project-local guide drifts from the skill and the two start contradicting each other.

The other bundled files are canonical for their narrower concerns and don't need to be mirrored:

- **Templates** (`assets/issue-template.md`, `assets/project.json`) — copied verbatim into target projects.
- **References** (`references/*.md`) — loaded on demand by Claude when a relevant topic comes up.

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for how to file issues and send PRs against this skill.

## License

[MIT](LICENSE) © Brennan Stehling
