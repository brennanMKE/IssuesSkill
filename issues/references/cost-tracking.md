# Token usage and cost tracking

How to record, per issue, how many tokens each work session consumed, which model did the work, and what it cost. Usage is recorded by the **orchestrator** after a subagent returns — the subagent can't measure its own totals (its transcript is still growing while it works, and it doesn't know its own transcript filename).

The standard workflow dispatches a subagent in **three phases** — planning (Fable) at issue creation, implementation (Sonnet), and review (Opus) — so a single issue normally accumulates at least three work-log rows, plus one for every bounce or retry. **The first row is the planning subagent, recorded at filing time**, not when the issue is later worked. Record usage after *each* subagent returns, whichever phase it was; the recipe below is identical for all three. See `workflow.md` for the phases.

Two artifacts are involved:

- **`issues/model-pricing.json`** — a cached copy of Anthropic's published per-MTok prices, refreshed at most once per day.
- **`## Work log`** — a section in each issue file, one table row per work session, with a running total.

## The pricing cache (`issues/model-pricing.json`)

Anthropic does not expose pricing through an API endpoint — prices are published on the docs site. Fetch them once per day and cache the result so issue-resolution loops don't re-fetch per issue.

### Schema

```json
{
  "fetched": "2026-06-03",
  "source": "https://docs.claude.com/en/docs/about-claude/pricing",
  "currency": "USD per MTok",
  "models": {
    "claude-fable-5": {
      "input": 5.00,
      "output": 25.00,
      "cache_write_5m": 6.25,
      "cache_read": 0.50
    },
    "claude-opus-4-8": {
      "input": 5.00,
      "output": 25.00,
      "cache_write_5m": 6.25,
      "cache_read": 0.50
    },
    "claude-sonnet-4-6": {
      "input": 3.00,
      "output": 15.00,
      "cache_write_5m": 3.75,
      "cache_read": 0.30
    }
  }
}
```

All rates are **USD per million tokens**. The four rates map directly onto the four token counts in transcript usage records (see below). The numbers above are examples — never trust them over a fresh fetch.

### Daily refresh

Before dispatching the first subagent of a session — **including the planning subagent fired when a new issue is filed**, which is often the very first dispatch:

1. Read `issues/model-pricing.json`. If it exists and `fetched` equals today's date, use it as-is — done.
2. Otherwise, fetch `https://docs.claude.com/en/docs/about-claude/pricing` with WebFetch and extract, for each current model: input, output, cache-write (5-minute), and cache-read rates per MTok. Rewrite the cache file with today's date.
3. **If the fetch fails** (offline, page moved), keep using the stale cache and append ` (pricing as of <fetched date>)` to the cost cell of any work-log rows you write. A stale price is an estimate; say so. If there's no cache at all, record tokens and model but put `—` in the cost column.
4. **If `issues/` is tracked by git**, commit a refreshed cache with message `Update model pricing`.

Only include models that actually appear (or are likely to appear) in this project's work. Under the standard three-phase workflow that's the **top model used for planning (currently Fable), plus the Sonnet and Opus generation** used for implementation and review. If a work-log entry uses a model missing from the cache, add that model on the next refresh.

## Getting exact token counts for a subagent

When Claude Code spawns a subagent, the subagent's full transcript is written to:

```
~/.claude/projects/<project-slug>/<session-id>/subagents/agent-<id>.jsonl
```

`<project-slug>` is the project's working directory with `/`, `.`, and `_` each replaced by `-` (e.g. `/Users/brennan/Developer/MyApp` → `-Users-brennan-Developer-MyApp`). Every assistant turn in the transcript carries a `message.usage` object with exact counts:

| Usage field | Meaning | Priced at |
|---|---|---|
| `input_tokens` | uncached input | `input` rate |
| `output_tokens` | generated output | `output` rate |
| `cache_read_input_tokens` | input served from prompt cache | `cache_read` rate |
| `cache_creation_input_tokens` | input written into the cache | `cache_write_5m` rate |

`message.model` on the same lines is the exact model id (e.g. `claude-opus-4-8`).

**Critical: dedupe by `requestId`.** One API response can be written as several JSONL lines (one per content block), each repeating the same `usage` object. Summing every line over-counts — count each `requestId` once.

### Recipe

Right after the subagent returns, locate its transcript — the most recently modified agent file whose contents mention the issue id — and sum the usage:

```bash
SLUG=$(pwd | tr '/._' '---')
FILE=$(grep -l "NNNN" $(ls -t ~/.claude/projects/$SLUG/*/subagents/agent-*.jsonl 2>/dev/null | head -10) 2>/dev/null | head -1)
python3 - "$FILE" <<'EOF'
import json, sys
seen, models = {}, set()
for line in open(sys.argv[1]):
    try:
        obj = json.loads(line)
    except json.JSONDecodeError:
        continue
    msg = obj.get("message", {})
    u = msg.get("usage")
    if not u:
        continue
    models.add(msg.get("model", "?"))
    seen[obj.get("requestId")] = u   # dedupe: last line per request wins
t = {"input": 0, "output": 0, "cache_read": 0, "cache_write": 0}
for u in seen.values():
    t["input"] += u.get("input_tokens", 0)
    t["output"] += u.get("output_tokens", 0)
    t["cache_read"] += u.get("cache_read_input_tokens", 0)
    t["cache_write"] += u.get("cache_creation_input_tokens", 0)
print(json.dumps({"models": sorted(models), "requests": len(seen), **t}))
EOF
```

Replace `NNNN` in the `grep` with the actual issue id. The `ls -t … | head -10` bounds the search to recent agents; `grep -l` preserves that ordering, so `head -1` picks the newest match.

### Cost formula

```
cost = (input × input_rate
      + output × output_rate
      + cache_read × cache_read_rate
      + cache_write × cache_write_5m_rate) / 1,000,000
```

Round to the cent. Cache reads usually dominate the token count but cost a tenth of the input rate — don't be alarmed by multi-million cache-read counts.

### When transcripts aren't available

The transcript layout is a Claude Code implementation detail and the skill also runs under other harnesses (Cursor, Codex). If no matching `agent-*.jsonl` exists:

1. If the harness reported a token total when the subagent returned (e.g. a "Done (… tokens …)" summary in the tool result), record that total in the `Output` column with a `(total, breakdown unavailable)` note and leave cost as `—` or a rough estimate clearly marked `~`.
2. If nothing is available, still record the date and model with `—` for tokens and cost. A row with a model and no numbers is better than no row — it shows work happened.

Never fabricate counts. `—` is the honest value.

## The `## Work log` section

Each issue accumulates one table row per work session, conventionally as the **last section** of the file (always after `## Description` — never above it, where the Mac app's frontmatter parser eats content). Create the section on first work; append on later sessions.

```markdown
## Work log

| Date | Phase | Model | Input | Output | Cache read | Cache write | Cost |
|---|---|---|---|---|---|---|---|
| 2026-06-03 | plan | claude-fable-5 | 84 | 6,102 | 512,400 | 41,200 | $0.58 |
| 2026-06-04 | implement | claude-sonnet-4-6 | 120 | 18,530 | 2,904,110 | 98,400 | $1.12 |
| 2026-06-04 | review | claude-opus-4-8 | 96 | 9,240 | 1,331,200 | 44,800 | $1.05 |

**Total: $2.75**
```

Rules:

- One row per subagent dispatch, including **bails** and review **bounces** — a failed or rejected attempt still burned tokens and the queue's true cost should reflect it.
- The optional **Phase** column (`plan` / `implement` / `review`) makes the three-phase breakdown legible. It's recommended but not required — a project that only ever ran one phase per issue can omit it. If you add it, keep it on every row for that issue.
- **The `plan` row is appended at filing time**, when the planning subagent returns — long before the issue is worked. The `implement` and `review` rows are appended later, as each of those subagents returns.
- Token cells use thousands separators for readability.
- The `**Total: $X.XX**` line is the running sum of the Cost column; update it whenever a row is appended.
- If a session used more than one model (rare), list both in the Model cell separated by ` / ` and price each portion at its own rate if you have per-model splits; otherwise price at the more expensive model and note `~`.
- Don't reformat existing rows when appending — diff-friendly edits, as everywhere in this skill.

## Git commits

When `issues/` is tracked:

| Event | What's committed | Commit message |
|---|---|---|
| Daily pricing refresh | `model-pricing.json` | `Update model pricing` |
| Work-log row appended | `issues/NNNN.md` | `#NNNN Work log: <model>, <total tokens>, $<cost>` |

The work-log commit lands *after* the phase's own commit (the plan commit, the resolution commit, a bounce, or a bail), since the orchestrator can only measure usage once the subagent has returned. In practice the row is usually folded into that same markdown edit — e.g. the planning subagent's row rides along with the `## Plan` in the filing commit, and the review subagent's row rides along with the resolution commit. Folding usage into a markdown edit the orchestrator already owns is fine — don't split hairs.

## Anti-patterns

- **Don't sum every JSONL line.** Dedupe by `requestId` or totals over-count (verified: ~30% of usage lines can be duplicates).
- **Don't hardcode prices in prose or code.** Prices change; the dated cache file is the only place numbers live.
- **Don't price cache reads at the input rate.** They're ~10× cheaper; conflating them inflates costs dramatically on cache-heavy sessions.
- **Don't let the subagent self-report usage.** It can't see its own totals; the orchestrator measures after return.
- **Don't skip the row on a bail.** Failed attempts cost real money and belong in the tally.
