---
name: echo
description: "Search everything the crew knows — across every persona's memory/agents/ file and every pending handoff in memory/inbox/ — without reading each file by hand. Spawns blind parallel subagents, each searching a slice of memory files for the query, then unions and deduplicates the results. Unlike /swarm (exhaustive find over a corpus), echo is scoped to THE CREW'S OWN KNOWLEDGE — what past-you and other personas already learned, decided, or handed off. Use to check 'do we already know X?', 'what did we decide about Y?', 'has anyone looked into Z?', or to gather context before starting a new task. Read-only — never edits, drains, or routes. Trigger on: /echo, search memory, what does the crew know about, do we already know, what did we decide, has anyone looked into, search agents, search inbox, dig through memory, remind me what we know about, crew knowledge search."
---

# Echo

You send **blind parallel searchers across every crew memory file and return a unified, deduped answer to a knowledge question** — so the user can query what the crew collectively knows without reading every `agents/*.md` and `inbox/*.md` by hand. Echo is read-only: it searches, reports, and stops. It never edits, drains, or routes.

**The crew accumulates facts across sessions. Without echo, "what do we know about X" means reading N files manually or guessing.** One persona's memory holds what it learned; another's inbox holds a pending handoff about the same topic. A single parallel sweep across all of them, deduped and ranked by relevance, answers the question in one shot instead of N reads.

Echo is to crew memory what `/swarm` is to a codebase — but scoped to memory files and tuned for *recall relevance*, not exhaustive enumeration. Swarm hunts every bug in a corpus; echo hunts every relevant memory entry.

---

## The core mechanic: fan-out searchers over memory files + union

Built on the **Agent tool**, same blind-parallel discipline as swarm, but over text files instead of code.

1. **Slice memory files across searchers.** Every `memory/agents/*.md` and `memory/inbox/*.md` under the project root (cwd) gets searched. Divide the files across N subagents so no file is searched twice and no searcher gets more than ~3–4 files (to keep each focused). Each searcher is a blind subagent that receives its file slice and the query.

2. **One batch, parallel.** Emit all searcher `Agent` calls in **a single message**. They run concurrently. Never serialize — the whole point is answering in one round.

3. **Each searcher returns structured matches.** Use the `Agent` `schema` so every searcher returns identical fields: `file` (the memory file path), `matched_line` (the raw line), `relevance` (high/medium/low), `why` (one-sentence reason it matches the query). Matches are per-line or per-entry, not whole-file dumps.

4. **Dedup in the main thread.** Searchers work independently; they may return overlapping facts if the same fact propagated across personas. **You** dedup by content (normalized line text) and rank by relevance. Report unique matches only.

5. **One round, then stop.** Echo answers in one parallel batch — no loop, no tail convergence. If the top matches don't satisfy, the user can refine the query and re-run. Unlike swarm, echo is fast recall, not exhaustive mining.

### Dispatch shape (illustrative)

```
# ONE message — searchers run in parallel, blind, each a slice of memory:
Agent(schema: MATCH_SCHEMA, prompt: "Search THESE files for facts matching the query.
       Files: memory/agents/forge.md, memory/agents/gauge.md
       Query: <Q>. Return every line that matches, with relevance and why. Isolated.")
Agent(schema: MATCH_SCHEMA,
      prompt: "Files: memory/agents/atlas.md, memory/agents/anchor.md, memory/inbox/forge.md
       Query: <Q>. Return matches only. Isolated.")
Agent(schema: MATCH_SCHEMA,
      prompt: "Files: memory/inbox/*.md, memory/agents/sage.md, memory/agents/vera.md
       Query: <Q>. Return matches only. Isolated.")
# ...one Agent per file slice, all this message.
```

---

## Pick the search scope

| Scope | Files searched | Use when… |
|-------|----------------|-----------|
| **Full memory** (default) | all `agents/*.md` + all `inbox/*.md` | vague query — let echo find where it lives |
| **Agents only** | all `memory/agents/*.md` | you want *settled knowledge*, not pending handoffs |
| **Inboxes only** | all `memory/inbox/*.md` | you want *pending handoffs* about a topic |
| **One persona** | `memory/agents/<name>.md` + `memory/inbox/<name>.md` | you only care what e.g. Forge or Gauge knows/is being told |
| **Several personas** | named slice, e.g. `agents/forge.md`, `agents/gauge.md` | targeted check without scanning everyone |

Default to **full memory**. Let the user narrow only if they want.

---

## How you run it

1. **Restate the query.** Sharpen it so every searcher answers the same question — a fuzzy query yields scattered results that look like poor coverage.
2. **Scope.** Decide which files to search. Announce it. "Echo: full memory sweep."
3. **Dispatch.** One message, N searchers (one per file slice), blind, structured schema.
4. **Dedup + rank.** Collapse duplicate line content. Rank by relevance (high > medium > low). Within same relevance, by file (agents/ before inbox/ — settled knowledge before pending).
5. **Report.** Show matches + coverage (which files had hits, which had none). If zero matches, say so — "No memory entries match that query." — rather than fabricating.

---

## Output Format

```
**Echo:** [the query — sharpened]

**Searchers:** [N] agents over [file count] files · scope: [full / agents-only / inboxes / named]

**Matches (deduped, by relevance):**

High relevance:
| File | Entry | Why |
|------|-------|-----|
| agents/forge.md | [YYYY-MM-DD] <fact> | [the match reason] |

Medium relevance:
| File | Entry | Why |
|------|-------|-----|

Low relevance:
| File | Entry | Why |
|------|-------|-----|

**Coverage:**
- Files searched: [N] — [list of files actually read]
- Files with matches: [N]
- Files with zero matches: [N] — [list, if any]
- Errors: [any searcher that errored or a file that couldn't be read — say which]

**Not found:** [anything the query might imply that echo found zero evidence of — prevents silent "no results" from being read as "we definitely don't know"]
```

If zero matches across all files:
```
**Echo:** [query]

**Result:** No memory entries match that query across [N] files searched ([list of files]).

**Not found:** [the topic — explicitly flag it as absent from crew memory]
```

No coverage table when there are zero matches — just the terse result.

---

## Rules

- ❌ **Read-only.** Never edit, drain, route, or clear a memory entry. Echo searches and reports — that's all.
- ❌ **No serial searchers.** One parallel batch per query. Serializing wastes wall-clock and leaks context between files.
- ❌ **Dedup in the main thread.** Searchers don't dedup each other — that's your job. Return raw matches, collapse duplicates.
- ❌ **Rank by relevance, not volume.** A file with 20 low-relevance matches is not "more knowledge" than a file with 1 high-relevance match. Relevance determines display order.
- ❌ **No fabricated matches.** If a searcher returns a line that doesn't actually match, drop it before reporting — don't pad results.
- ❌ **Report empty coverage honestly.** A file with zero hits is reported as searched-without-match, never silently omitted from the coverage list. Silence reads as "not searched."
- ❌ **Not a persistent mode.** Search once, report, stop.
- Always report **coverage** and **what wasn't found**. Echo's value isn't just the hits — it's knowing the crew has nothing on that topic.
