---
name: swarm
author: "Swarm <swarm@alethia.local>"
description: "Throw many independent subagents at one SEARCH task — find all the bugs, all the prior art, all the edge cases, all the broken links — then union and dedup what they surface. Where /council spawns DIFFERENT personas to diff opinions, swarm spawns MANY finders (same or mixed lens) at one corpus, each blind to the others, each searching a different slice or angle, so coverage compounds instead of overlapping. Runs in rounds: spawn a batch in parallel, collect, dedup against everything seen, and loop until K consecutive rounds turn up nothing new (the tail is where the rare bug hides). Use for exhaustive hunts: audit a codebase, sweep for a class of bug, enumerate failure modes, comprehensive prior-art search, find every X. Trigger on: /swarm, swarm this, find all, exhaustive search, comprehensive audit, sweep for, hunt for every, enumerate all, fan out and find, parallel search, leave no stone unturned, what bugs are in here, find every edge case."
---

# Swarm

You launch **many independent subagents at a single search problem and union their finds**. The bet: one searcher has one searcher's blind spots and quits early; N blind searchers, each working a different slice or angle, cover ground no single pass reaches — and the rare find that one misses, another catches.

**Coverage is a union, not a max.** A lone agent that finds 7 of 10 bugs looks thorough until you see the 3 it never looked for. Five agents pointed at different slices, then deduped, get you closer to all 10 — and tell you when you've stopped finding new ones.

This is council's spawn engine aimed at a different target. **Council = different personas → diff their *opinions*. Swarm = many finders → union their *findings*.** Council answers "what should we do"; swarm answers "what's all in here."

---

## The core mechanic: fan-out finders + loop-until-dry

Built on the **Agent tool**, same blind-parallel discipline as council, plus a convergence loop.

1. **One batch per round, parallel.** Emit all finder `Agent` calls for a round in **a single message** (multiple tool_use blocks). They run concurrently. Never serialize finders across turns within a round.
2. **Blind, but each on a different slice or angle.** Every finder gets a clean context and a *distinct* assignment so they don't all re-find the same obvious thing. Slice by **area** (these files / this module / this section), or by **angle** (search by control-flow, by data-flow, by naming, by error-handling, by the spec). Each is blind to the others' finds — overlap is waste, distinct slices are coverage.
3. **Force a structured find shape.** Use the `Agent` `schema` so every find returns identically — minimum: `id`/`location` (file:line, URL, section), `what` (the finding), `severity`/`relevance`, `evidence`. Identical shape is what makes dedup mechanical.
4. **Dedup in the main thread — against EVERYTHING seen, not just this round.** Keep a running `seen` set keyed on location+kind. Each round, drop finds already in `seen`; the rest are *fresh*. (Dedup against all-seen, never against just the last round, or the tail loops forever.)
5. **Loop until dry.** After each round: if fresh finds > 0, add them to `seen` and run another round (optionally re-slicing toward where finds clustered). If a round adds **zero** fresh finds, increment a dry-counter. Stop when **K consecutive dry rounds** (default K=2 — one empty round can be luck; two means the well's dry).
6. **You own the union.** Finders return raw lists and stop. They don't dedup each other, don't rank globally. **You** merge, dedup, severity-sort, and report. The synthesis is the main thread's job.

### Dispatch shape (illustrative, one round)

```
# ONE message — finders run in parallel, blind, each a different slice/angle:
Agent(schema: FIND_SCHEMA, prompt: "Audit ONLY src/auth/** for security bugs.
      Search by data-flow: trace untrusted input to sinks. List every finding. Isolated.")
Agent(schema: FIND_SCHEMA, prompt: "Audit ONLY src/auth/** by error-handling: every
      catch/throw/unhandled path. List findings. You are working alone.")
Agent(schema: FIND_SCHEMA, prompt: "Audit src/auth/** against the spec in docs/auth.md —
      every deviation. List findings.")
# ...one Agent per slice/angle, all this message.
```

Inside a workflow context, prefer the loop-until-dry pattern with `parallel()` of `agent()` calls and a `seen` Set across rounds.

---

## Pick the finders

Two knobs: **how many** and **what lens**.

- **Slice count** — split the corpus so finders don't collide. 3–6 slices per round is typical. Big codebase → slice by module. One file, many angles → slice by analysis type (control-flow / data-flow / naming / error paths / spec-conformance).
- **Lens** — usually one finding-type per swarm (all hunting *security* bugs, or all hunting *prior art*). Mix lenses only if the target is genuinely multi-kind, and label each find with its lens.

| Hunt | Slice by | Finder lens |
|------|----------|-------------|
| Bug audit of a codebase | module / dir | correctness + the bug class you care about |
| One class of bug everywhere | file batches | that specific class (e.g. injection, race, off-by-one) |
| Enumerate failure modes | subsystem | "how does THIS break" |
| Comprehensive prior art | venue / sub-area / keyword cluster | relevance to the claim |
| Dead links / stale docs | doc section | broken refs + outdated facts |
| Edge cases for a function | input dimension | boundary / malformed / adversarial inputs |

Match finders to the actual crew lens where one fits (Forge for code bugs, Beacon for prior art, Quill for holes, Sentry for security if it exists) — point each subagent at that persona's skill.

---

## How you run it

1. **Frame the hunt.** What exactly are we finding, in what corpus, and what counts as a find (severity bar)? A fuzzy target floods you with noise.
2. **Slice + announce.** State the slices/angles and round-1 plan. "Round 1: 4 finders, sliced by module, hunting security bugs."
3. **Round loop.** Dispatch batch (one message) → collect → dedup vs `seen` → fresh>0? add & maybe re-slice & repeat : dry++ → stop at K dry rounds.
4. **Report the union.** Deduped, severity-sorted findings + coverage honesty (below).
5. **Hand off.** Swarm *finds*; it doesn't *fix*. Route fixes through Aleth / the owning persona.

---

## Output Format

```
**Swarm:** [the hunt — what + where]
**Finders:** [N per round] sliced by [axis] · [lens]
**Rounds:** [R rounds run · stopped after K dry] · [total raw finds → deduped to M]

**Findings (deduped, severity-sorted):**
| # | Location | Severity | What | Evidence |
|---|----------|----------|------|----------|
| 1 | file:line | high | … | … |
| 2 | … | … | … | … |

**Coverage:**
- Searched: [slices/angles actually covered]
- NOT searched: [anything left out — say it plainly, silence reads as "all clear"]
- Convergence (per round, raw → fresh after dedup against all-seen): `R1: <raw1> → <fresh1> | R2: <raw2> → <fresh2> | R3: <raw3> → <fresh3> | …` — e.g. `R1: 12 → 12 | R2: 9 → 4 | R3: 3 → 0 | R4: 0 → 0 (dry)`. The fresh column is what matters: dropping to zero = likely exhausted, still finding at stop = NOT exhausted, more rounds would help.

**Next:** [route fixes to owner via Aleth · or "more rounds recommended — not yet dry"]
```

---

## Rules

- ❌ **No serial finders within a round.** One message, parallel batch. Serializing wastes wall-clock and tempts context leakage.
- ❌ **No identical slices.** Two finders on the same slice with the same lens just re-find the same things — that's cost without coverage. Distinct slice or distinct angle, always.
- ❌ **Dedup against ALL seen, not the last round.** Otherwise the tail never converges and you loop forever.
- ❌ **No silent cap.** If you stop for budget/time before dry, SAY the hunt is incomplete and what's unsearched. A truncated sweep that reads as exhaustive is worse than no sweep.
- ❌ **No fabricated finds, no dropped dissent.** Report only what finders actually returned, evidence attached. A single finder's lone high-severity find still gets reported — don't average it away.
- ❌ **Swarm finds, doesn't fix.** Enumerate and hand off; don't start editing mid-sweep.
- ❌ **Not persistent.** Run the rounds to convergence, report the union, stop.
- Always report the **convergence curve** and **what wasn't searched**. Coverage honesty is the deliverable as much as the finds.
