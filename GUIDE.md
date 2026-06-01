# The Crew — User Guide

A team of single-purpose expert personas covering the full research + engineering lifecycle. Each is a skill that activates on what you ask. You don't have to memorize them — **Aleth auto-routes** any task to the right one(s).

---

## The 15 Personas

| Persona | Ask it for… |
|---------|-------------|
| **Aleth** | "who handles this / what's the workflow" — the auto-router and front door |
| **Beacon** | survey a field — prior art, SOTA, the gap, "is this novel" (many papers) |
| **Codex** | deeply read ONE paper — contribution, method, claim-vs-evidence |
| **Helix** | design an experiment — hypothesis, controls, sample size, what would falsify it |
| **Ethos** | ethics/compliance — consent, PII, bias, IRB, "can we use/publish this" |
| **Vera** | research a question OR analyze data — evidence, fact-check, EDA, stats |
| **Lemma** | the math — prove/disprove, derive, check a proof, find a counterexample |
| **Atlas** | architecture before code — stack/DB choice, will-this-scale, system design |
| **Forge** | write/debug/refactor code, build pipelines, fix CI/build failures |
| **Anchor** | make a result reproducible — pin env/seeds/data, build the replication package |
| **Prism** | charts and figures — what plot, publication-quality, fix a misleading graphic |
| **Scribe** | writing — paper, abstract, grant, whitepaper, README, design doc |
| **Quill** | adversarial peer review — red-team a paper/claim, "will this get rejected" |
| **Orator** | talks — conference deck, defense, poster, demo script, pitch |
| **Sage** | learn/teach — explain, build intuition, learning path, "help me understand" |
| **Compass** | plan a project — timeline, milestones, critical path, "what do we cut" |

---

## How to Use It

**Two ways to invoke:**

1. **Just ask.** Describe the task in plain language — the matching persona activates on its own. "Debug why my eval is 0/50" → Forge. "Is this paper novel?" → Beacon. "Prove this bound" → Lemma.

2. **Ask Aleth when the owner isn't obvious.** Multi-step work, or you're not sure who does what:
   ```
   /aleth I have an idea for a caching algorithm and want to publish it
   ```
   Aleth returns the full chain (who → who → who), the handoffs, where it runs in parallel, and where to stop early. You then run that chain.

**Rule of thumb:** one clear task → ask the persona directly. A goal that spans research + build + write → ask Aleth to route it.

---

## How Aleth Routes (so you can predict it)

- **Endpoint first.** The deliverable picks the last step: paper→Quill, talk→Orator, working code→Forge, reproducible artifact→Anchor, proven claim→Lemma, understanding→Sage.
- **Symptom names the owner.** Broken code→Forge · won't reproduce→Anchor · wrong math→Lemma · what-does-data-say→Vera · one paper→Codex. (Diagnosis does NOT default to Vera.)
- **Mandatory gates** (never skipped): Ethos before touching people-data · Helix before collecting data · Quill before submitting · a cheap check (Lemma/Anchor) before an expensive run.
- **Starts where you are.** Have data already → starts at analysis, not survey. Have a draft → starts at review, not design.
- **Stops early.** Names the kill-switch ("if not novel → stop, don't build").

---

## Workflow Recipes

Common chains. Run them yourself, or paste the goal to `/aleth` and it builds the exact one.

### 1. Idea → published paper
```
Beacon (novel? prior art)  →[stop if not novel]
  → Helix (design the experiment)
  → Forge (build + run)
  → Anchor (pin env/seed/data — make it rerun)
  → Vera (analyze results)
  → Prism (figures)
  → Scribe (write IMRaD)
  → Quill (red-team before submission)
```

### 2. Broken model / pipeline (debug)
```
Forge (debug — find root cause)
  → Lemma (verify the fix math BEFORE a costly eval run)   [gate]
  → Forge (apply fix, re-run)
  → Anchor (pin so the bug can't recur)
  → Vera (analyze fresh metrics vs baseline)
  → Quill (no regression?)
Stop early if: the fix doesn't move the metric → root cause is wrong, back to Forge.
```

### 3. I have data, what's going on?
```
Ethos (consent scope — ok to analyze this?)   [gate, if people-data]
  → Vera (EDA — find the pattern, separate correlation from cause)
  → Helix (design the follow-up that would confirm cause)
  → Prism (visualize the finding)
```

### 4. Get funding (grant)
```
Beacon (the gap + position vs prior work)   →[stop if crowded]
  → Helix (feasible methodology)
  → Compass (timeline + milestones + budget)
  → Ethos (compliance plan)
  → Scribe (write — sell the significance)
  → Quill (will a panel fund it?)
```

### 5. Understand a paper, then build on it
```
Codex (deep read — contribution, method, claim-vs-evidence, repro gaps)
  → [if reproducing] Anchor (pin the reimplementation)
  → [if building new] Helix (design what beats it)
```

### 6. Prepare a talk / defense
```
[work is done]
  → Orator (arc, slides, say-vs-show, Q&A prep)
  → Prism (only if a key figure needs rework for the room)
```

### 7. Review a draft before submitting (parallel)
```
Parallel:  Quill (holes/overclaims) + Helix (methodology sound?) + Ethos (bias/consent disclosed?)
  → synthesize → route fixes (stats→Helix/Lemma, prose→Scribe) → re-Quill
```

### 8. Design a system before coding
```
Atlas (requirements → options → recommendation → failure modes)
  → Forge (build it)
  → Anchor (if it's a pipeline that must reproduce)
```

### 9. Learn a topic from zero
```
Sage (explain, build intuition, learning path)
  → Beacon (only if you then need the SOTA / what to read next)
```

### 10. Plan a multi-week / multi-person project
```
Compass (decompose → dependencies → critical path → timeline → cut lines)
  → [Compass holds the schedule; the other personas execute their steps against it]
```

---

## Tips

- **Don't over-route.** A one-liner ("explain a Bloom filter") is a single hop to Sage — Aleth will say "skip the crew, go straight to X."
- **Trust the gates.** If Aleth inserts Ethos or a Lemma check before your big run, it's saving you a wasted GPU pass or a rejected paper.
- **Tell Aleth what you already have.** "I have the data / a draft / the results" changes the entry point and skips redundant steps.
- **The personas push back.** Atlas fights over-engineering, Quill hunts holes, Compass calls out optimistic estimates. That friction is the point — it's cheaper than finding out later.
