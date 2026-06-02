---
name: tournament
description: "Generate many DIFFERENT solutions to one open problem in parallel, then run them through a judged bracket to find the best — and synthesize a winner that grafts the best ideas of the runners-up. Where /council diffs opinions, /swarm unions finds, and /jury refute-votes a claim, tournament is the GENERATIVE primitive: K subagents each produce a distinct attempt (different angle/strategy/constraint), blind to each other, then parallel judge-pairs score them head-to-head over rounds until one stands. Use when the solution space is WIDE and one-attempt-then-iterate would lock in early on a mediocre idea: design proposals, API/schema shapes, naming, algorithm choices, copy/messaging, architecture options, 'give me the best version of X'. Trigger on: /tournament, tournament this, bracket it, generate options and pick, best of N, competing designs, explore the design space, multiple approaches then choose, run a bake-off, bake-off, pit approaches against each other, find the best version."
---

# Tournament

You **generate several genuinely different solutions to one open problem in parallel, then judge them down to a winner in a bracket** — and the final answer grafts the best parts of the losers onto the champion. The bet: a single attempt-then-polish converges fast on whatever you thought of first; many divergent attempts, then ruthless comparison, find the idea you wouldn't have reached by iterating.

**One draft iterated is a local maximum; many drafts judged is a search.** When the space is wide — a design, a schema, a name, an algorithm — the first plausible answer is rarely the best, and polishing it just makes a mediocre idea shiny. Tournament spends parallel spawns to explore breadth first, then spends judgment to exploit the best of it.

It's the **generative** member of the spawn family. Council/swarm/jury all *evaluate what exists*; tournament *creates options* and then evaluates. Reach for it when the problem is "what's the best X" and you don't yet have the candidates.

---

## The core mechanic: divergent generate → judged bracket → graft

Built on the **Agent tool**. Two distinct phases: a generation fan-out, then judging rounds.

### Phase 1 — Generate (divergent, parallel, blind)

1. **One batch, parallel.** Emit all generator `Agent` calls in **a single message**. Each returns one complete candidate solution.
2. **Force divergence — this is the whole game.** If generators aren't pushed apart they all produce the same safe answer and the bracket is theater. Give each a **distinct seed**: a different strategy ("MVP-first" / "scale-first" / "user-first" / "weirdest idea that could work"), a different constraint ("assume 10× load" / "assume zero budget" / "assume one engineer"), or a different prior-art anchor. Blind to each other — a fresh `Agent` context each, no shared draft.
3. **Structured candidate shape.** Use the `Agent` `schema` so each candidate returns identically: `approach` (the one-line strategy), `solution` (the actual artifact), `tradeoffs`, `best_idea` (its single strongest move — this is what gets grafted later). Identical shape makes judging mechanical.

### Phase 2 — Judge (bracket, parallel pairs, blind)

4. **Define the rubric BEFORE judging.** State 3–5 weighted criteria for *this* problem (e.g. correctness 40% / simplicity 30% / extensibility 30%). Picking criteria after seeing candidates rigs the outcome.
5. **Head-to-head, parallel.** Each bracket round pairs candidates; spawn **one judge subagent per pair in a single batch**, each blind to other pairs, each returning a winner + scored reasons against the fixed rubric. Winners advance; emit the next round's judge batch. Pairwise (A-vs-B) beats scoring-in-isolation — comparison surfaces differences a lone score blurs.
6. **Odd counts / byes.** Odd field → one bye to the next round; never invent a phantom opponent. ~4–8 candidates is the sweet spot (1–3 judging rounds).
7. **Optional semifinal cross-check.** For high stakes, have the final-round judge see both finalists' full text (not just the verdict) to catch a bracket artifact where a strong candidate lost early to an even stronger one.

### Phase 3 — Synthesize (you, main thread)

8. **You own the graft.** Generators and judges return raw; **you** take the champion and graft the `best_idea` of the runners-up where they strengthen it without breaking its coherence. The deliverable is *champion + grafts*, not a raw winner — the runner-up that lost overall may still own the single best move.
9. **Show the bracket.** Report the path so the choice is auditable, not a black-box verdict.

### Dispatch shape (illustrative)

```
# Phase 1 — ONE message, divergent generators, blind:
Agent(schema: CANDIDATE_SCHEMA, prompt: "Design <X>. Strategy: SIMPLEST thing that works,
      ruthless YAGNI. Return the schema. You work alone.")
Agent(... prompt: "Design <X>. Strategy: assume 100× scale from day one. ...")
Agent(... prompt: "Design <X>. Strategy: optimize for the END USER's mental model. ...")
Agent(... prompt: "Design <X>. Strategy: the unconventional approach nobody picks. ...")

# Phase 2 — ONE message per round, judge pairs, blind, fixed rubric:
Agent(schema: JUDGE_SCHEMA, prompt: "Compare candidate A vs B on rubric [c1 40% / c2 30% /
      c3 30%]. Pick a winner, score both, justify. Isolated.")
Agent(... prompt: "Compare candidate C vs D on the SAME rubric. ...")
# winners → next round's batch.
```

Inside a workflow, this maps cleanly to the judge-panel pattern: `parallel()` generators → `parallel()` pairwise judges per round → synthesize.

---

## Pick the field

- **Generator count** — 4 default, up to ~8 for a big-stakes open design. Each is a full subagent; cost scales with the field. Below 3, just answer it directly.
- **Generator lens** — usually one persona generating under varied seeds (e.g. Atlas proposing 4 architectures), or mixed personas when the problem spans disciplines (Atlas vs Forge vs a from-scratch take). Point each at the relevant crew skill.
- **Judge** — neutral and consistent across all pairs: same rubric, same standard. Use Quill's skeptical eye or the problem's natural owner as the judge lens, but the rubric is fixed regardless of who judges.

| Problem | Generator seeds (force these apart) | Rubric leans on |
|---------|-------------------------------------|-----------------|
| System / architecture | simple · scale-first · user-first · unconventional | correctness · scale · build-cost |
| API / schema shape | minimal · extensible · convention-matching · ergonomic | clarity · evolvability · misuse-resistance |
| Algorithm choice | brute-simple · optimal-O · cache-friendly · approximate | speed · memory · correctness bounds |
| Naming / copy / messaging | literal · evocative · short · domain-native | clarity · memorability · fit |
| Refactor strategy | minimal-diff · clean-slate · incremental-migration | risk · payoff · reversibility |

---

## How you run it

1. **Frame the problem + rubric.** State what's being designed and the weighted criteria — *before* generating. A tournament with no rubric can't crown anyone honestly.
2. **Announce the field.** Generator count + each seed. "4 candidates: simple / scale-first / user-first / wildcard."
3. **Generate.** One parallel batch, divergent, blind.
4. **Bracket.** Parallel pairwise judges per round, fixed rubric, winners advance, byes for odd counts.
5. **Synthesize.** Champion + grafted best-ideas of runners-up. Show the bracket path.
6. **Hand off.** Tournament *chooses*; building it is Forge's job — route via Aleth.

---

## Output Format

```
**Tournament:** [the problem]
**Rubric:** [c1 w% · c2 w% · c3 w%] — fixed before judging
**Field:** [N] candidates · seeds: [list]

**Candidates**
| # | Approach (seed) | One-line solution | Best single idea |
|---|-----------------|-------------------|------------------|
| A | simplest | … | … |
| B | scale-first | … | … |

**Bracket**
- R1: A vs B → **A** (won on c1, c3) · C vs D → **D** (won on c2)
- Final: A vs D → **D** (edge on c2 outweighed A's c1 lead under the weights)

**Winner: [D] — [why it took the rubric]**

**Synthesized answer (champion + grafts):**
- Base: D
- Grafted: A's [best idea] → strengthens [c-axis] without breaking D's coherence
- Dropped: [any runner-up idea considered and rejected, + why]

**Runner-up worth remembering:** [the loser whose single idea may matter later]
**Next:** [route the build to Forge / the owner via Aleth]
```

---

## Rules

- ❌ **No false divergence.** Generators MUST be seeded apart — four near-identical candidates make the bracket meaningless. If they come back too similar, re-seed harder and regenerate, don't judge clones.
- ❌ **Rubric before candidates.** Fix the weighted criteria before generation/judging. Choosing them after you've seen the field is rigging the result.
- ❌ **No serial generation or judging within a phase.** One parallel batch per phase / per round. Serializing leaks context and biases.
- ❌ **No self-generation or self-judging.** Real subagents both phases — you don't imagine the candidates or the verdicts.
- ❌ **Pairwise, not lone scores, for the bracket.** Judge A-vs-B head-to-head; isolated scores blur the comparison the tournament exists to make.
- ❌ **The deliverable is champion + grafts, not a raw winner.** A loser's single best idea often survives into the answer — don't discard the whole candidate with the bracket.
- ❌ **No phantom opponents.** Odd field → bye, never a fabricated candidate to fill the slot.
- ❌ **Not persistent.** Run generate → bracket → synthesize once, deliver, stop.
- Always show the **bracket path** and the **grafts**. A winner with no visible bracket is just an opinion wearing a trophy.
