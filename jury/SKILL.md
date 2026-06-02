---
name: jury
description: "Stress-test ONE claim by spawning many independent skeptics whose job is to REFUTE it — the claim survives only if too few of them can. Where /council diffs opinions and /swarm unions finds, jury runs adversarial verification: N subagents, blind to each other, each prompted to try its hardest to break the claim (default to 'refuted' when uncertain), then a refute-vote decides. Survives if fewer than the majority refute; killed otherwise. Use to verify a finding before acting on it, fact-check a claim, confirm a bug is real, check 'is this actually true', gate an expensive run on a shaky premise, or filter a pile of candidate findings down to the ones that withstand attack. Optionally give each skeptic a distinct lens (correctness / does-it-reproduce / counterexample / security) so they fail it in different ways. Trigger on: /jury, jury this, verify this, is this real, fact-check, confirm the bug, refute this, try to break this, adversarial check, does this hold up, stress-test the claim, vote on it, prove me wrong."
---

# Jury

You put **one claim on trial before a panel of independent skeptics, each spawned to refute it**, and you let the vote decide. Not "is this plausible" — "can a determined skeptic break it." A claim that survives N honest attempts to kill it is one you can act on. One that doesn't was going to fail later, more expensively.

**The cheap place to kill a wrong claim is before you build on it.** Plausible-but-wrong findings survive when nobody's *job* is to refute them. Jury makes refutation the job, spreads it across independent skeptics so one skeptic's blind spot isn't fatal, and converts "I'm fairly sure" into a vote you can read.

Where council asks "what should we do" (diff opinions) and swarm asks "what's all in here" (union finds), jury asks **"does this specific claim hold up"** (refute-vote). It's the natural verify-stage after either: council surfaces a contested call → jury the load-bearing claim under it; swarm surfaces candidate bugs → jury each before anyone fixes them.

---

## The core mechanic: blind parallel skeptics + refute-vote

Built on the **Agent tool**, same blind-parallel discipline as council, tuned for adversarial verification.

1. **One batch, parallel.** Emit all juror `Agent` calls in **a single message**. They run concurrently and blind. Serial jurors leak context and bias the vote.
2. **Every juror's mandate is to REFUTE.** Not "evaluate," not "give a balanced take" — *try to break it*. The prompt must say: find the counterexample, the flaw, the unstated assumption, the case where it fails; and **default to `refuted: true` when you cannot establish it holds.** Skewing the burden toward refutation is the safeguard — it's why surviving means something.
3. **Blind isolation.** Each juror sees only the claim (and the minimum context to test it) — never another juror's verdict, never the running tally, never that a panel exists. A fresh `Agent` context per juror.
4. **Optional: distinct lenses.** For a claim that can fail multiple ways, give each juror a different attack surface — correctness, reproducibility, counterexample, security, does-it-actually-measure-that. Diverse lenses catch failure modes that N identical skeptics all miss together. For a one-dimensional claim, identical skeptics are fine (then it's a pure independence check).
5. **Structured verdict, forced.** Use the `Agent` `schema` so each juror returns the same shape: `refuted` (bool), `confidence`, `attack` (how it tried to break it), and `killing_evidence` (the counterexample/flaw if refuted) or `why_it_held` (if not). The `attack` field is what lets you audit a suspiciously unanimous verdict.
6. **You tally — the vote lives in the main thread.** Jurors never see the count or each other. **You** collect verdicts and apply the rule: **survives iff `refuted` count < majority** (default: needs majority-refute to kill; tune the bar to the stakes — see below). Report the tally AND the strongest attack even when the claim survives.

### Dispatch shape (illustrative)

```
# ONE message — jurors run in parallel, blind, each trying to REFUTE:
Agent(schema: VERDICT_SCHEMA, prompt: "CLAIM: <the claim>. Your job: REFUTE it. Find the
      counterexample / flaw / unstated assumption. Default to refuted=true if you cannot
      establish it holds. Correctness lens. You work alone. Return the schema.")
Agent(... prompt: "CLAIM: <same>. REFUTE via reproducibility — would this actually rerun /
      replicate? Default refuted if unsure. ...")
Agent(... prompt: "CLAIM: <same>. REFUTE by counterexample — construct the input where it
      breaks. ...")
# ...one Agent per juror, all this message.
```

Inside a workflow, prefer `parallel()` of `agent()` calls with the shared verdict schema; survives iff `votes.filter(v => !v.refuted).length >= threshold`.

---

## Set the bar to the stakes

The kill threshold is a dial, not a constant. State it up front.

| Bar | Survives if… | Use for |
|-----|--------------|---------|
| **Lenient** | < majority refute (default) | filtering a pile of candidate findings — keep what mostly holds |
| **Strict** | zero refute (unanimous "holds") | gating an expensive/irreversible run on a load-bearing premise |
| **Supermajority** | ≤ ⅓ refute | high-stakes single claim, but one contrarian juror shouldn't veto |

Jury size: 3 for a quick check, 5 for a real verification, 7 when the cost of a wrong "survives" is high. Odd numbers avoid ties. **Every juror is a full subagent — size to the stakes**, not to comfort.

---

## How you run it

1. **State the claim — precisely.** A vague claim gets refuted on a technicality or survives on ambiguity. Pin it to something falsifiable. If it's really three claims, jury the load-bearing one (or run three).
2. **Set the bar + size.** Announce threshold and juror count and why. "5 jurors, strict bar — this gates a training run."
3. **Dispatch.** One message, N refuting jurors, shared schema, blind.
4. **Tally.** Count `refuted`. Apply the bar. A juror that errored → missing vote, note it; don't assume its verdict.
5. **Report the verdict + the attacks.** Even a survivor's strongest attack is intel — it's the claim's weak point. A killed claim's `killing_evidence` is the actionable part.

---

## Output Format

```
**Jury:** [the claim — sharpened to something falsifiable]
**Panel:** [N] jurors · bar: [lenient / strict / supermajority] · [lenses, if diverse]

**Verdict: HOLDS ✅ / REFUTED ❌  ([refuted_count]/N voted to refute)**

**Votes**
| Juror | Lens | Refuted? | Conf | The attack it ran |
|-------|------|----------|------|-------------------|
| 1 | correctness | no | 0.7 | tried X, claim held because… |
| 2 | repro | YES | 0.8 | breaks when… (counterexample) |
| 3 | counterex | no | 0.6 | … |

**Strongest attack [even if it held]:** [the single best refutation attempt — the claim's weakest point]
**Killing evidence [if refuted]:** [the counterexample / flaw that sinks it]

**Call:** [act on it / don't / fix-then-rejury] — given the bar and the spread.
**If split near the bar:** [what one fact would settle it — get that before betting on this claim]
```

---

## Rules

- ❌ **Jurors refute, they don't balance.** "Give a fair assessment" defeats the mechanism. The mandate is attack + default-to-refuted-when-unsure. Surviving that bias is the signal.
- ❌ **No serial jurors, no shared tally.** One parallel batch; no juror sees the count or another's verdict. Independence is the whole point of a vote.
- ❌ **No self-jury.** Spawn real subagents — you don't get to imagine the skeptics. Verification you simulate isn't verification.
- ❌ **State the bar before the vote, not after.** Picking the threshold once you see the tally is rigging it.
- ❌ **Report the attack even on survivors.** "Held" without showing what was tried is unfalsifiable cheerleading. The strongest attack is intel regardless of outcome.
- ❌ **No fabricated or averaged-away verdicts.** Report each real vote; a lone high-confidence refuter on a strict-bar claim is a stop sign, not noise.
- ❌ **Not persistent.** Try the claim, tally, deliver the call, stop.
- Always report the **tally** and the **strongest attack**. A jury that returns only "yes it's fine" wasted the spawn.
