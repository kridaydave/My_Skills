---
name: council
author: "Council <council@alethia.local>"
description: "Put ONE question to several crew personas at once as independent parallel subagents, blind to each other, then diff their answers to expose where the experts disagree — because disagreement is where your risk lives. Unlike /debate (two personas argue inside one context), council spawns N real subagents via the Agent tool in a single batch; each embodies one persona, answers in isolation with no sight of the others, and returns a structured verdict. You synthesize: where they converge (settled, low-risk), where they diverge and why (the worry surface), and the one fact that would resolve the split. Use for high-stakes calls, to triangulate a decision across lenses, get a confidence read, or surface blind spots no single persona sees. Name the panel or let it auto-pick. Trigger on: /council, convene the council, ask the crew, poll the personas, get multiple opinions, panel review, what does everyone think, parallel opinions, triangulate this, second opinion, third opinion, where do the experts disagree, confidence check, blind panel."
---

# Council

You convene a **panel of crew personas on one question, each as an independent subagent running in parallel and blind to the others**, then you diff their answers. The product is not a consensus — it's a **disagreement map**. Where independent experts converge, you can relax. Where they split, that's exactly where the decision is unsafe, and that's what you surface.

**Agreement reached by talking it out can be groupthink; agreement reached blind, in separate rooms, is signal.** Debate lets one persona's framing infect the other. Council forbids that by construction — each persona answers in its own isolated subagent context, never seeing the question's other answers. The split between them is uncontaminated, so it means something.

This is the key difference from `/debate`: debate is *one context, two voices, they argue*. Council is *N isolated subagent contexts, one voice each, they never meet* — you are the only one who sees all the answers.

---

## The core mechanic: spawn blind parallel subagents

This skill is built on the **Agent tool**. You do not role-play the personas yourself — you *dispatch* them as real subagents and let them think independently. Get this part right or the council is worthless.

1. **One batch, parallel.** Emit **all panelist `Agent` calls in a single message** (multiple tool_use blocks in one response). They run concurrently — the wall-clock cost is one agent, not N. Never spawn them one-at-a-time across turns: that wastes time *and* risks leaking context between them.
2. **Blind isolation is the whole point.** Each subagent gets **only the question + its persona assignment**. It must NOT receive any other panelist's answer, nor know who else is on the panel, nor that a panel exists. A fresh `Agent` call starts with a clean context — exploit that. If you ever find yourself pasting one agent's output into another's prompt, you've broken the council; that's a pipeline, not a panel.
3. **Each subagent embodies exactly one persona.** The prompt tells it which persona's lens to adopt and points it at that persona's skill so it answers in-character with that discipline's evidence standard. One persona per agent — never ask a single agent to "consider all viewpoints" (that just averages them and destroys the disagreement signal you're hunting).
4. **Force structured returns.** Use the `Agent` tool's `schema` option (or instruct the agent precisely) so every panelist returns the *same shape*. Identical shape across answers is what makes the diff mechanical instead of vibes. Minimum fields: `position` (the call), `confidence` (0–1 or low/med/high), `key_reason`, `top_risk`, `what_would_change_my_mind`.
5. **You are the only synthesizer.** Subagents return raw verdicts to you and stop. They do not talk to each other, do not vote, do not reconcile. **You** collect all returns, align them by field, and compute the agreement/disagreement map. The synthesis lives in the main thread, never in a subagent.
6. **Independence > harmony.** Tell each panelist explicitly: answer honestly from your lens, do not hedge toward a safe middle, you are not being graded against anyone. A panel of identical safe answers has zero information.

### Dispatch shape (illustrative)

```
# In ONE message — these run in parallel, blind to each other:
Agent(subagent_type: "general-purpose", schema: COUNCIL_SCHEMA,
      prompt: "Embody the ATLAS persona (load its skill; architecture lens — scale,
               failure modes, will-this-hold). Answer ONLY from that lens, in isolation.
               You are not coordinating with anyone. Question: <Q>. Return the schema.")
Agent(... prompt: "Embody FORGE (ship-it / YAGNI / what breaks in practice) ...")
Agent(... prompt: "Embody GAUGE (is the gain real / how would we even measure this) ...")
# ...one Agent call per panelist, all in this single message.
```

If the user invoked `council` *inside* a workflow context, prefer `parallel()` of `agent()` calls with the shared schema — same principle, same blindness.

---

## Pick the panel

3–5 personas is the sweet spot. Fewer than 3 → just use `/debate`. More than ~6 → diminishing signal, rising cost (every panelist is another full subagent). Auto-pick by question type, or honor a user-named panel:

| Question | Panel | Why these lenses |
|----------|-------|------------------|
| "Should we build / adopt X?" | Beacon · Atlas · Forge · Compass | novel? · sound? · shippable? · cuttable & realistic? |
| "Is this result / claim trustworthy?" | Gauge · Vera · Quill | real gain? · what data says · where it breaks |
| "Which architecture / stack?" | Atlas · Forge · Anchor | scale · build cost · will-it-reproduce |
| "Is this paper ready to submit?" | Quill · Helix · Ethos · Scribe | holes · methodology · consent/bias · framing |
| "Is this plan realistic?" | Compass · Forge · Lemma | schedule · effort · the math checks |
| Open / "what am I missing?" | Quill · Sage · Atlas · Vera | skeptic · first-principles · structure · evidence |

State the panel and the lens each brings **before** dispatching. (*Panel comes from the actual crew roster only — cross-check against GUIDE.md / Aleth's table; never invent a persona to fill a seat.*)

---

## How you run it

1. **Frame.** Restate the exact question. Sharpen it so every panelist answers the *same* thing — a fuzzy question yields fake disagreement (they answered different questions). If it's genuinely two questions, run two councils or pick the live one.
2. **Pick + announce the panel.** Name each persona and its lens. "Convening Atlas, Forge, Gauge — blind, in parallel."
3. **Dispatch (the batch).** One message, N `Agent` calls, shared schema, full blindness. (See mechanic above.)
4. **Collect.** Wait for all returns. A panelist that errors or is skipped → note it as a missing seat, don't fabricate its answer.
5. **Diff — the actual work.** Align answers field-by-field. Cluster the `position`s. Find: unanimous points (settled), the axis they split on, *why* they split (different evidence? different priority? different read of the same fact?), and any lone dissenter worth heeding.
6. **Synthesize.** Emit the disagreement map (format below). End with the **decisive question** — the single fact that, if known, collapses the disagreement.

---

## Output Format

```
**Council:** [the question — sharpened]
**Panel:** [Persona] (lens) · [Persona] (lens) · [Persona] (lens)   — blind, parallel
**Returns:** N/N seats answered [· note any missing/errored seat]

**Per-persona verdict**
| Persona | Position | Conf | Key reason | Top risk it flags |
|---------|----------|------|-----------|-------------------|
| Atlas   | …        | 0.8  | …         | …                 |
| Forge   | …        | 0.6  | …         | …                 |
| Gauge   | …        | 0.4  | …         | …                 |

**Where they AGREE (settled — low risk):**
- [unanimous / near-unanimous points]

**Where they SPLIT (the worry surface):**
- **[axis of disagreement]** — A says X because…, B says Y because…  → they split on [the underlying difference, not the surface]
- [lone dissent worth heeding, if any — a low-confidence outlier can be the only one seeing the real risk]

**Confidence read:** [tight cluster = trust it · wide spread = decision is not ready, here's why]

**The call:** [your synthesis given the spread + user's constraints]
**Decisive question:** [the one fact that would collapse the disagreement — go get this before committing]
```

---

## Cost & when NOT to use

- **Each panelist is a full subagent** — a 4-persona council costs ~4× a single answer in tokens/compute. Scale the panel to the stakes. A reversible one-way-door decision doesn't need a council; a hard-to-undo one does.
- **One question, not a task.** Council answers *a question* ("should we?", "is this real?"). It does not *do work* — it won't write the code or run the eval. Route execution through Aleth afterward.
- **Not for things with a known right answer.** "What's the syntax for X" → just answer it. Council is for genuinely contested / high-uncertainty calls where lenses legitimately differ.
- **Tight cluster on a trivial question = wasted spawn.** If you can predict all panelists will agree, skip it.

---

## Rules

- ❌ **No serial dispatch.** All panelist Agent calls go in ONE message, parallel. Spawning them across turns leaks context and defeats blindness.
- ❌ **No cross-contamination.** A panelist never sees another's answer, the panel roster, or that a council exists. Break this and the disagreement is meaningless.
- ❌ **No self-roleplay.** You dispatch real subagents; you do not "imagine" what Atlas would say. The independence has to be real to be signal.
- ❌ **No forced consensus, no forced split.** Report the agreement honestly — sometimes they genuinely all agree (great, high confidence). Don't manufacture a disagreement for drama, or paper over a real one for comfort.
- ❌ **No inventing personas.** Panel comes from the actual crew roster only.
- ❌ **No omitting the dissenter.** The lone low-confidence outlier gets reported, not averaged away — it's often the only seat seeing the real failure mode.
- ❌ **Not a persistent mode.** Convene once, diff, deliver the call + decisive question, stop.
- Always end with the **confidence read** and the **decisive question**. A council that just lists opinions without mapping the split did half the job.
```
