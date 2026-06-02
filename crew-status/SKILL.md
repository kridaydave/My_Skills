---
name: crew-status
description: "One-shot dashboard for the crew's memory and pending handoffs. Reads every memory/agents/*.md and memory/inbox/*.md under the project root and prints a compact status: what each persona knows, when it was last updated, and every pending cross-persona handoff waiting in an inbox. Read-only — never writes, edits, clears, or routes. Use to see the crew's whole memory and open threads at a glance. Trigger on: /crew, /crew-status, crew status, what does the crew know, what do we know so far, show crew memory, crew memory, pending handoffs, open handoffs, who knows what, crew dashboard."
---

# Crew Status

One-shot dashboard. You read the crew's memory and report it — you do **not** write, edit, clear, or route anything. Pure visibility, then stop.

## What you do

1. **Find the memory dir.** Look for `memory/agents/` and `memory/inbox/` under the project root (cwd). If `memory/` doesn't exist, say so plainly — "No crew memory yet — nothing's been saved." — and stop.
2. **Read every agents file** (`memory/agents/*.md`). For each: persona name, entry count, most-recent date, and a one-line gist of what it knows.
3. **Read every inbox file** (`memory/inbox/*.md`). Each non-empty one is a **pending handoff** — a note waiting for that persona to drain on its next activation. Record recipient, sender, date, note.
4. **Report** in the format below. Compact. No editorializing beyond the flags.

## Output Format

```
**Crew memory** — <N> personas have saved memory · <M> pending handoffs

| Persona | Facts | Last updated | Knows about… |
|---------|-------|--------------|--------------|
| forge   | 4     | 2026-06-02   | <one-line gist> |
| gauge   | 2     | 2026-05-30   | <one-line gist> |

**Pending handoffs** (waiting in inboxes):
- → gauge ← from trove [2026-06-02]: <the note>
- → forge ← from atlas [2026-06-01]: <the note>

**Flags:**
- <stale: persona not updated in a notably long time>
- <empty: an agents/ or inbox file exists but has no entries>
```

- Inboxes all empty → replace the handoffs list with "Inboxes empty — no open handoffs."
- Nothing to flag → omit the **Flags** block entirely.

## Rules

- ❌ Read-only. Never create, edit, clear, drain, or route. Draining an inbox is the *receiving persona's* job on its own activation — not yours.
- ❌ No invented entries — report only what's in the files. Empty/missing dir → say it.
- ❌ Not a persistent mode — render once and stop.
- Keep each gist to one line. The point is glance-able status, not a dump of every entry.
