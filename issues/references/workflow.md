# The standard workflow — plan → implement → review

The canonical way issues move from filed to resolved. It runs as a **three-phase pipeline**, and every phase runs in a **fresh subagent** dispatched by the orchestrator (the main session). The orchestrator never does the plan/fix/review work itself — it picks the issue, dispatches, and records usage. Keeping the work in subagents is what keeps the orchestrator's context small enough to coordinate a whole queue.

| Phase | When | Model | Subagent does | Status after |
|---|---|---|---|---|
| **1. Planning** | Right after a new issue is filed | **Fable** (top available model) | Reads conventions + issue, writes a `## Plan` section. No code. | `open` |
| **2. Implementation** | When the issue is worked | **Sonnet** | Follows the plan, fixes, builds + verifies, makes the code commit, drafts resolution sections. | `in-progress` |
| **3. Review** | After implementation returns | **Opus** | Independently re-verifies the diff, then approves or bounces. | `resolved` (approve) or `open` (bounce) |

## Why fresh subagents, and why these models

- **Fresh context per phase is a feature.** Each subagent loads `Issues.md` and `CLAUDE.md` cleanly at the start, so the moment the user edits those files the *next* subagent picks up the change — no stale conventions carried over from an earlier phase.
- **The orchestrator stays small.** Plan, fix, and review transcripts each grow large. Isolating them in subagents means the orchestrator only ever sees a one-line return summary per phase, so it can drive a long queue without its own context ballooning.
- **Model tiers match the work.** Planning is the highest-leverage, most-reasoning-heavy step, so it gets the top model (**Fable**). Implementation is well-scoped once a plan exists, so **Sonnet** handles it. Review is an adversarial correctness check, where **Opus** earns its cost.
- **Fable runs *only* in a subagent.** Never invoke the top model in the orchestrator's own context — the whole point is to spend that model's tokens against a fresh, minimal context, not to inflate the coordinating session.

## Phase 1 — Planning (Fable), at issue creation

Planning happens as part of **filing a new issue**, not when work later begins. Once the `NNNN.md` file exists with the user's report:

1. **Orchestrator dispatches a fresh Fable subagent** with the issue id and instructions to produce an implementation plan (not to write code).
2. **The planner orients**, reading in order:
   - `issues/Issues.md` — status vocabulary, module conventions, build/verify command, commit conventions, project-specific rules.
   - `CLAUDE.md` at the repo root, if present — binding project-wide guidance.
   - `issues/NNNN.md` — the issue in full, including any attachments in `issues/NNNN/`.
   - Relevant code — enough to ground the plan in what actually exists. The planner reads code but does **not** modify it.
3. **The planner writes a `## Plan` section** into `issues/NNNN.md`, placed *after* `## Description` (never above it — the Mac app's frontmatter parser eats anything between the metadata table and the first `##`). A good plan states the suspected root cause, the files/functions likely involved, the approach in a few steps, and how the fix should be verified. It is guidance for the implementation subagent, not a contract — the implementer may deviate if reality differs, and should say so.
4. **Status stays `open`.** Planning does not claim the issue. A planned issue is a normal open issue with a head start.
5. **Orchestrator records the planner's usage** as a `## Work log` row (model = the Fable id). See `cost-tracking.md`.
6. **If `issues/` is tracked by git**, the plan can be folded into the filing commit (`#NNNN <title>`) if it lands before that commit, or committed separately as `#NNNN Plan`. If `issues/` is ignored or there's no repo, skip the commit.

If a planning subagent can't produce a useful plan (issue too vague, needs user input), it writes what it can into `## Plan` with an explicit note about what's unclear, and leaves the issue `open` for the user to clarify. It never blocks filing.

## Phase 2 — Implementation (Sonnet)

### Orchestrator: pick and dispatch

When the user says "work through the open issues", "pick up the next bug", or "fix the next one":

1. **Refresh the pricing cache if stale.** Read `issues/model-pricing.json`; if missing or its `fetched` date isn't today, fetch current prices and rewrite it (once per day, not per issue). See `cost-tracking.md`.
2. List `issues/*.md` (skip `Issues.md`). Pick the lowest-numbered file whose status is `open`.
3. **Spawn a fresh Sonnet subagent** with the issue id and instructions to follow the implementation steps below. The issue already carries a `## Plan` from phase 1 — the subagent follows it.
4. **When the subagent returns, record its usage** (model = the Sonnet id) as a `## Work log` row. Bails get a row too — a failed attempt still spent tokens.
5. Proceed to phase 3 (review) for the same issue before moving to the next one.

If the user names a specific issue ("fix 0046"), dispatch to that id directly.

### Implementation subagent: claim → fix → build → commit

You start with fresh context, so orient before touching anything.

1. **Orient in the project.** Read, every time, in order:
   - `issues/Issues.md` — authoritative for issue-tracking workflow; if it contradicts this skill, follow it.
   - `CLAUDE.md` at the repo root, if it exists — binding code/repo conventions.
   - `issues/NNNN.md` — the issue in full, **including its `## Plan`**, plus attachments in `issues/NNNN/`.

   If the two project guides disagree, prefer `CLAUDE.md` for code/repo conventions and `Issues.md` for issue-tracking specifics.

2. **Set status to `in-progress`** in the markdown — working-copy edit only, no commit. The Mac app reflects it within ~1s, signaling the issue is claimed.
3. **Make the code changes** required to fix the bug, following the `## Plan`. If you deviate from the plan, that's fine — note why in the resolution `## Fix` section so the reviewer understands the divergence.
4. **Build *and* run the project's verification command, and confirm tests actually executed and passed.** Mandatory; cannot be skipped or shortcutted.
   - **Compilation is not verification.** "It builds" / "no type errors" does not count. The verification command must actually *execute* — unit tests run, UI tests run on a simulator, the app launches, whatever the project defines as proof. A green build with zero tests run is a failure of this step.
   - **If you wrote or modified tests, you MUST execute those specific tests and observe them pass.** Confirm the test names appear in the run, the counts increased, and the result was success. A test that compiles but never ran proves nothing.
   - **Read the output, don't just check the exit code.** "0 tests run", "skipped", "no tests found", or "build succeeded" with no test summary are red flags even at exit code 0. iOS in particular reports `xcodebuild` success when the test target didn't run.
   - **If verification can't run in your environment** (no simulator, missing credentials, hardware required, sandbox), you have not verified the fix. Bail per "When you can't finish" and name the step you couldn't run.
   - **If the build was already broken when you started**, note it and bail — don't fix unrelated breakage.
5. **Make the code commit.** Stage *only the code changes* — do not stage the issue markdown yet. The message starts with `#NNNN` and a short, declarative title with the verb that fits (`Fix`, `Add`, `Refactor`, `Update`, `Remove`, …). Blank line, then a paragraph of detail. Example:

   ```
   #0046 Add navigation from avatar tap to profile

   The avatar tap on PostCardView was not wired to any NavigationLink.
   Threaded the author DID through the cell and connected onTapGesture
   to push ProfileView.
   ```

   If `CLAUDE.md` or recent `git log` defines a different convention, follow that.
6. **Capture the commit hash** with `git rev-parse --short HEAD`.
7. **Draft the resolution sections in the markdown — but do NOT set `resolved`.** Setting `resolved` is the reviewer's transition in phase 3. What you do here:
   - Leave **Status** at `in-progress`.
   - Add a `**Commit**` row with the short hash from step 6.
   - Add the resolution sections, all *after* `## Description`:
     - **`## Root cause`** — what was actually wrong (often different from the original report).
     - **`## Fix`** — the approach taken; call out any divergence from the plan.
     - **`## Verification`** — the exact command(s) you ran and what you observed (e.g. "`xcodebuild test -scheme MyAppUITests` — 14 tests passed including the 3 new tests in `ReplyButtonUITests`"). Name any new tests and confirm they ran. Mandatory — it's the audit trail the reviewer checks against.
     - **`## Files changed`** — one bullet per file, with a short note on what changed.
     - **`## Gotchas`** *(optional)* — surprises, dead ends, non-obvious behavior worth knowing. Skip if nothing's notable; be specific when present.
8. **This markdown draft is a working-copy edit — do not commit it.** The reviewer makes the single resolution commit in phase 3 once the work is approved. Return to the orchestrator with a one-line summary of what you changed.

**No dangling follow-ups.** If your fix carves out scope you're deliberately not addressing, **file that follow-up as its own ticket before you finish** and link it bidirectionally (parent's Resolution notes / `## Relation` point at the child; child's `## Relation` says it was carved out of the parent). A "follow-up" sentence with no ticket behind it disappears the moment the issue closes.

### When you can't finish

If the bug is unreproducible, out of scope, or the build won't pass after reasonable effort:

1. **Discard or stash any partial code changes** so the bail doesn't include half-done work.
2. **Revert status to `open`** so the issue re-enters the queue. Don't leave it `in-progress`.
3. **Add a `## Notes` section** describing what you tried, the failure mode, and what you'd try next. The next subagent starts from your notes.
4. **If `issues/` is tracked by git**, commit just the markdown with message `#NNNN Notes: <one-line bail summary>`. If ignored, skip.
5. Return with a one-line summary of the bail.

Never use `wontfix` or `closed` as an escape hatch — those are the user's decisions.

## Phase 3 — Review (Opus)

An independent Opus subagent is the gate between "code landed" and "issue resolved." It owns the `resolved` transition; the implementation subagent never sets it. This separation is what makes `resolved` mean *verified by a second party*, not *the author thinks it's done*.

### Orchestrator: dispatch the reviewer

After the implementation subagent returns (with a code commit and drafted resolution sections, status still `in-progress`), **spawn a fresh Opus subagent** to review it. When the reviewer returns, record its usage (model = the Opus id) as a `## Work log` row, exactly as for the other phases.

### Review subagent: verify → resolve or bounce

1. **Orient**, same as the other phases: `issues/Issues.md`, then `CLAUDE.md`, then `issues/NNNN.md` in full — including its `## Plan` and the drafted `## Root cause` / `## Fix` / `## Verification` / `## Files changed` sections.
2. **Inspect the code commit.** Read the diff of the implementation commit (`git show <hash>`). Check it against the plan and the issue: does it actually address the reported bug? Is the scope right? Any obvious correctness, security, or regression risk?
3. **Re-run verification independently.** Don't trust the `## Verification` section — run the project's verification command yourself and read the output. Confirm tests actually executed and passed (the same standard as phase 2, step 4). This independent run is the core of the review.
4. **Decide:**
   - **Approve** — the fix is correct and verification passed:
     - Change **Status** to `resolved`.
     - Add a `**Closed**` row with today's date; ensure the `**Commit**` row (short hash) is present.
     - Add a `## Resolution notes` blockquote summary at the top of the resolution sections: `> 🟢 Resolved YYYY-MM-DD — <one sentence>.` This is what the user reads first on the Mac app's detail view; keep it terse.
     - **If `issues/` is tracked by git**, make the resolution commit: stage `issues/NNNN.md` and commit with `#NNNN Resolve: <title>`. The body notes the code commit it pairs with (the hash). This second commit pairs with the implementation's code commit — "fix landed, fix verified & documented." If `issues/` is ignored or there's no repo, skip the commit; the markdown change is the record.
   - **Bounce** — verification failed, the fix is wrong, or scope is off:
     - Revert **Status** to `open` so the issue re-enters the queue.
     - Add a `## Review notes` section stating exactly what failed (which check, what you observed) and what the next implementation pass must address. Be specific — this is the brief for the next Sonnet subagent.
     - Leave the code commit in place; the next implementation pass amends or builds on it. If the commit must be discarded, say so explicitly in `## Review notes`.
     - **If `issues/` is tracked by git**, commit the markdown with `#NNNN Review: <one-line reason for bounce>`. If ignored, skip.
     - Return to the orchestrator, which may re-dispatch phase 2 for another attempt.

**Never set `closed`.** Even an approving reviewer stops at `resolved` — `closed` is the user's transition after they confirm the fix in the Mac app. See the Critical rule in `SKILL.md`.

## Status flow, end to end

```
file issue ──▶ open (with ## Plan, Fable)
                 │  orchestrator dispatches Sonnet
                 ▼
            in-progress ──▶ code commit + drafted resolution (Sonnet)
                 │  orchestrator dispatches Opus
                 ▼
              review ──┬──▶ resolved   (Opus approves, resolution commit)
                       └──▶ open       (Opus bounces, ## Review notes) ──▶ re-dispatch Sonnet
                                          │
                                          ▼
                                    user confirms ──▶ closed
```

`closed` and `wontfix` are always the user's call. Bails from any phase land back at `open`.

## Related references

- **`cost-tracking.md`** — recording each phase's token usage and cost. The planning subagent's usage is recorded at filing time; implementation and review usage after each returns. Every issue can accumulate three or more work-log rows (plan, implement, review, plus any bounces/retries).
- **`issue-format.md`** — where `## Plan`, `## Resolution notes`, and the resolution sections sit relative to `## Description`.
