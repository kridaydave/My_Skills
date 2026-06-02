---
name: debate
description: "Adversarial dialectic between two crew personas to stress-test a plan, design, claim, or decision before you commit to it. Pits two opposing lenses against one artifact in structured rounds — opening positions, rebuttal, then a neutral synthesis verdict: what survived, what's settled, what's still open, and the call. Use to pressure-test a decision, surface blind spots, steelman both sides, or when one persona's take isn't enough and you want the crew to argue it out. Name the two personas, or let it auto-pick the natural opposing pair. Trigger on: /debate, debate this, debate mode, argue both sides, stress-test this, pressure-test this decision, red-team vs defend, steelman both sides, devil's advocate, X vs Y on this, have the crew argue it out."
---

# Debate

You stage a structured adversarial debate between **two crew personas** over one artifact — a plan, design, claim, result, or decision. The goal is not to "win"; it's to surface what survives real opposition and reach a sharper call than any single persona gives alone.

**A decision agreed to without friction is untested — and the cheapest place to find a flaw is an argument, not production.** One persona's take has one persona's blind spots. Two opposing lenses, each arguing its honest strongest case, expose what either misses alone.

You orchestrate both sides as independent subagents and then judge neutrally. You do not let either persona strawman the other, and you do not force a tie for harmony.

---

## Pick the pair

If the user names two personas, use them. Otherwise auto-pick the natural opposition for the artifact:

| Artifact / question | Pair | The tension |
|---------------------|------|-------------|
| Architecture / tech / stack choice | **Atlas vs Forge** | scale & correctness vs. ship-it & YAGNI |
| Paper / claim / framing | **Quill vs Scribe** | find the holes vs. defend the contribution |
| Experiment design | **Helix vs Forge** | rigor & controls vs. feasibility & cost |
| Result / "it improved" | **Gauge vs Vera** | is the gain real vs. what the data says it means |
| Plan / timeline / scope | **Compass vs Forge** | what's cuttable & realistic vs. the ambition |
| Using some data | **Ethos vs Trove** | consent & risk vs. get-the-data |
| Math claim / bound | **Lemma vs the proposer** | prove / counterexample vs. the intuition |
| Build it or not | **Beacon vs Forge** | already exists / not novel vs. let's build |

No clean fit → default to **Quill (skeptic) vs the natural owner** of the artifact. State the pair and why before you start.

---

## Subagent Architecture

Each debate round is a **batch of parallel API calls** — both personas run simultaneously, with no awareness of each other's output within the same round. This enforces genuine independence: neither persona can anchor to or soften against the other mid-round.

The outputs from one batch become input context for the next batch. You are the **orchestrator**: you dispatch batches, collect outputs, and produce the final synthesis.

```
Round 0 (setup)        → orchestrator decides pair, frames artifact & question
Round 1 (Opening)      → BATCH: [Persona A subagent] || [Persona B subagent]
Round 2 (Rebuttal)     → BATCH: [Persona A subagent] || [Persona B subagent]
                           each receives the other's Opening output
Round 3 (Counter)      → BATCH (only if unresolved): same pattern
Round N (Synthesis)    → orchestrator synthesises from all prior outputs
```

Bounded — two rounds by default, three max. Converge to a verdict; don't loop.

---

## Implementation

### Step 0 — Frame (orchestrator)

Before dispatching any batch:
- Restate the artifact and the precise question being debated
- Name the two personas and the lens each brings
- Announce that subagents are being launched in parallel

Output this framing to the user before the first batch fires.

---

### Step 1 — Opening batch

Dispatch **two API calls in parallel** (do not await one before launching the other).

**System prompt for every persona subagent:**
```
You are [PERSONA_NAME], a crew persona with this lens: [PERSONA_LENS].
You are participating in a structured adversarial debate. Your job is to argue the strongest honest case for your position — no hedging, no pre-emptive concessions.
Do NOT strawman the opposing position. Do NOT drift from your persona's discipline.
Respond only as [PERSONA_NAME]. Do not break character or meta-comment on the debate format.
```

**User prompt for Persona A:**
```
Artifact under debate:
[ARTIFACT]

Question being debated:
[QUESTION]

Your role: [PERSONA_A] — [LENS_A]
Your opponent: [PERSONA_B] — [LENS_B]

This is the OPENING round. State your position — the strongest honest case from your lens.
No hedging. No pre-emptive concessions. Argue it fully.
```

**User prompt for Persona B:** identical structure, roles swapped.

Collect both outputs before proceeding.

---

### Step 2 — Rebuttal batch

Dispatch **two API calls in parallel**, each receiving the other's Opening output.

**User prompt for Persona A:**
```
Artifact under debate:
[ARTIFACT]

Question being debated:
[QUESTION]

Your role: [PERSONA_A] — [LENS_A]
Your opponent: [PERSONA_B] — [LENS_B]

YOUR opening position:
[PERSONA_A_OPENING]

OPPONENT'S opening position:
[PERSONA_B_OPENING]

This is the REBUTTAL round.
1. First, steelman your opponent's strongest point — present it in its best form.
2. Then attack that steelmanned version. Find the real weakness, not a weak version.
Do not concede unless the argument forces it.
```

**User prompt for Persona B:** identical structure, roles swapped.

Collect both outputs before proceeding.

---

### Step 3 — Counter batch (conditional)

**Only dispatch if the rebuttals left a genuine unresolved tension.** Skip if the key question is already settled.

Dispatch **two API calls in parallel**, each receiving the full prior context.

**User prompt for Persona A:**
```
Artifact under debate:
[ARTIFACT]

Question being debated:
[QUESTION]

Your role: [PERSONA_A] — [LENS_A]
Your opponent: [PERSONA_B] — [LENS_B]

Prior exchange:
Opening — [PERSONA_A]: [PERSONA_A_OPENING]
Opening — [PERSONA_B]: [PERSONA_B_OPENING]
Rebuttal — [PERSONA_A]: [PERSONA_A_REBUTTAL]
Rebuttal — [PERSONA_B]: [PERSONA_B_REBUTTAL]

This is the COUNTER round — only one more exchange after this, then synthesis.
The unresolved tension is: [STATE_THE_CRUX].
Focus only on the crux. Don't relitigate settled ground.
```

**User prompt for Persona B:** identical structure, roles swapped.

---

### Step 4 — Synthesis (orchestrator, not a subagent)

Step out of both personas entirely. You are the neutral judge. Synthesise from all outputs.

The synthesis prompt you give yourself:
```
You have orchestrated a debate between [PERSONA_A] and [PERSONA_B] over:
[ARTIFACT] / [QUESTION]

Full debate transcript:
[ALL_OUTPUTS_IN_ORDER]

Now produce the neutral synthesis verdict. You are not either persona.
Identify:
1. What both sides agree on (settled ground)
2. What remains genuinely contested and why
3. The recommendation, given the user's constraints
4. The flip condition — the one fact or result that would change the call
```

---

## Output Format

Present the debate as it unfolds — frame first, then each batch's outputs as they arrive, then the synthesis.

```
**Debate:** [the artifact / decision]
**Question:** [the precise thing being argued]
**Pair:** [Persona A] (lens) vs [Persona B] (lens)
*Launching Opening batch — both subagents running in parallel...*

---

**Opening**
- **[A]:** [output from A subagent]
- **[B]:** [output from B subagent]

*Launching Rebuttal batch — each subagent receives the other's Opening...*

---

**Rebuttal**
- **[A] → B:** [steelman B's point, then the counter]
- **[B] → A:** [steelman A's point, then the counter]

[*Launching Counter batch — crux: [STATE_CRUX]...*  ← only if unresolved]

[**Counter**
- **[A]:** [...]
- **[B]:** [...]]

---

**Verdict (neutral synthesis)**
- **Settled:** [what survived from both sides — the agreed ground]
- **Open:** [what's genuinely contested, and on what it depends]
- **Call:** [the recommendation, given the user's constraints]
- **Flip condition:** [the fact or result that would change the call]
```

---

## Rules

- ❌ **No strawman** — each subagent's prompt requires steelmanning the other before countering. A debate against a weak version proves nothing.
- ❌ **No conceding for harmony** — each side argues its honest strongest case; agreement only when the argument earns it.
- ❌ **No persona drift** — the system prompt locks each subagent to its discipline. Atlas argues like Atlas; Quill hunts holes; Compass calls out optimism.
- ❌ **No parallel peeking** — within a batch, neither subagent sees the other's output. Independence is the point.
- ❌ **No endless loop** — two rounds, three max, then a verdict. The point is a decision, not a transcript.
- ❌ **Synthesis is the judge, not either persona** — the orchestrator names what survived, not who "won."
- **Always end with a call and the flip condition.** A debate with no decision wasted the friction.
- **Announce each batch before it fires.** The user should know when subagents are running in parallel and what they've each been asked to do.
