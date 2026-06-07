---
name: crew-init
author: "Crew-Init <crew-init@alethia.local>"
description: "Scaffold the crew's memory structure in the current project. Creates memory/agents/ and memory/inbox/ (the per-persona memory + handoff layers) plus a memory/README.md explaining the convention, so personas have somewhere to read and write from the first session. Idempotent — never clobbers existing memory, just reports and fills the gaps. Use once per project before the crew starts accumulating memory. Trigger on: /crew-init, crew init, set up crew memory, initialize crew, scaffold memory, bootstrap the crew, create memory folder, set up persona memory."
---

# Crew Init

One-shot scaffolder. Creates the memory layout the crew reads and writes, under the project root (cwd). Idempotent: if something already exists, leave it untouched and report it — never overwrite a populated memory file.

## What you create

```
memory/
  README.md          <- explains the layout (write the template below)
  agents/
    .gitkeep         <- so git tracks the empty dir
  inbox/
    .gitkeep
```

## Steps

1. **Check first.** If `memory/` already exists, list what's there and only create the missing pieces. Never overwrite a non-empty `agents/*.md`, `inbox/*.md`, or an existing README.
2. **Create the dirs** `memory/agents/` and `memory/inbox/`, each with a `.gitkeep` so the empty dirs are committable.
3. **Write `memory/README.md`** using the template below.
4. **Report** what you created vs. what already existed. Tell the user memory is now live and that `/crew` shows status once personas start saving.

Do NOT pre-create per-persona files — personas create their own lazily on first real fact. Init only lays the empty structure.

## memory/README.md template

```markdown
# Crew Memory

Persistent memory for the crew of personas. Three layers:

- **`agents/<persona>.md`** — what each persona learned. A persona reads and writes ONLY its own file. Durable, role-specific facts (preferences, constraints, corrections, defaults). One atomic dated entry per line. Loaded only when that persona is active, so memory never bloats context.
- **`inbox/<persona>.md`** — handoffs. To pass a fact to another persona, append `- [date] from <you>: <note>` to their inbox. They drain it on their next activation.
- *(filenames are stable persona names, never dates — keeps the folder tidy and greppable.)*

Run `/crew` (the `crew-status` skill) for a read-only dashboard: who knows what + pending handoffs.

Commit this folder to share crew knowledge with the team, or add `memory/` to `.gitignore` to keep it local — your call.
```

## Rules

- ❌ Never overwrite or truncate a populated memory file — preserve existing knowledge.
- ❌ Don't seed fake/example entries into `agents/` or `inbox/` — they start empty.
- ❌ Not a persistent mode — scaffold once, report, stop.
- Mention the git choice (commit vs. ignore) once; don't decide it for the user.
