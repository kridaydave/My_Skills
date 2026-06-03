---
name: crew-sync
description: "Audit the crew's memory layer for consistency, structure, and hygiene — finds orphan inbox notes, stale entries, duplicate facts across personas, malformed timestamps, empty files, and references to personas that no longer exist. Read-only diagnostic: it reports issues and their severity, never edits, drains, or repairs. Use to catch memory drift before it causes confusion ('who is this handoff even for?'), to clean up after a persona was removed or renamed, or as a periodic health check for long-running projects. Trigger on: /crew-sync, crew sync, audit memory, check memory health, memory consistency, memory hygiene, validate crew memory, check for orphans, find stale entries, clean up memory, memory lint."
---

# Crew Sync

You **audit the entire crew memory layer for consistency, structure, and hygiene** — then report every issue you find, ranked by severity. You never edit, drain, or repair. You surface the problems and let the user (or the owning persona) decide what to fix.

**Memory that isn't maintained becomes noise, then rot.** An inbox note addressed to a deleted persona sits forever. A duplicate fact across two agent files silently contradicts itself when one gets updated and the other doesn't. A stale-agent file with 47 entries from six months ago bloats context on every activation of that persona. Crew sync finds these silently compounding problems before they cost trust in the memory layer.

This is a **read-only lint**. You check, you report, you stop. No writes, no edits, no drains.

---

## What you check

Every check falls into one of three severity levels. Report everything you find at the appropriate level; don't stop after the first problem.

### 🔴 Errors (must-fix)
| Check | What you look for |
|-------|-------------------|
| **Missing memory root** | `memory/` directory doesn't exist at project root |
| **Missing agents dir** | `memory/agents/` doesn't exist (but `memory/inbox/` does, or vice versa — asymmetric) |
| **Missing inbox dir** | `memory/inbox/` doesn't exist |
| **Bad file format** | An `agents/*.md` file contains lines that don't match the expected entry format (`- [YYYY-MM-DD] <fact>`) and aren't blank or comments |
| **Future timestamp** | An entry dated after today + 1 day |
| **Orphan inbox sender** | An inbox note references a sender persona (`from <name>:`) that is not in the current crew roster (aleth, anchor, atlas, beacon, charter, cipher, codex, compass, ethos, forge, gauge, helix, lemma, orator, prism, quill, sage, scribe, trove, vera) — or a custom persona the user has confirmed exists |
| **Orphan inbox recipient** | An inbox file exists for a persona not in the crew roster (inbox note for nobody) |

### 🟡 Warnings (should-fix)
| Check | What you look for |
|-------|-------------------|
| **Stale agent file** | An `agents/*.md` where the most recent entry is >90 days old |
| **Stale inbox note** | An `inbox/*.md` entry dated >30 days ago — a handoff that's been waiting too long |
| **Duplicate fact across personas** | The same substantive fact (normalized) appears in two or more `agents/*.md` files — suggests cross-persona duplication that should have been a handoff instead |
| **Entry without date** | A line in an `agents/*.md` file that matches the entry format but has no parsable date |
| **Empty agent file** | An `agents/*.md` exists but has zero entries (just whitespace or a header) |
| **Empty inbox file** | An `inbox/*.md` exists with no pending notes |

### 🔵 Info (for awareness)
| Check | What you look for |
|-------|-------------------|
| **Agent file count** | Total entries per persona — flag if > 30 (suggests it's bloated and needs pruning) |
| **Inbox backlog** | Total pending handoffs per recipient — flag if > 5 (someone's inbox is piling up) |
| **No agents at all** | `memory/agents/` is empty (crew has never saved anything) |
| **No inboxes at all** | `memory/inbox/` is empty (no handoffs have ever been sent) |

---

## How you run it

1. **Check crew root.** Does `memory/` exist? If not, emit the single error and stop — nothing to audit.
2. **Read all agent files.** `memory/agents/*.md` — parse every entry, check timestamps, check format, check for duplicates across files.
3. **Read all inbox files.** `memory/inbox/*.md` — parse every note, check sender against crew roster, check staleness.
4. **Cross-reference.** For duplicate facts, compare normalized content across agent files. For orphan senders/recipients, reference the current crew roster (listed below).
5. **Classify + report.** Group findings by severity (🔴 Errors → 🟡 Warnings → 🔵 Info). Within each severity, order by impact (most actionable first).
6. **Stop.** Read-only, one pass, no edits. Report and done.

### Current crew roster (for sender/recipient validation)

Base crew: `aleth`, `anchor`, `atlas`, `beacon`, `charter`, `cipher`, `codex`, `compass`, `ethos`, `forge`, `gauge`, `helix`, `lemma`, `orator`, `prism`, `quill`, `sage`, `scribe`, `trove`, `vera`.

Meta-skills (`debate`, `council`, `swarm`, `jury`, `tournament`, `crew-init`, `crew-status`, `echo`) do NOT have their own `agents/` or `inbox/` files — they spawn, operate, and stop without persistent memory. If you find a file for one of these, flag it as an orphan.

Custom personas added by the user beyond the base 20 are valid — flag an orphan only when the sender/recipient is neither in the base roster nor a user-confirmed custom persona. When in doubt, list the unrecognized name and let the user confirm it.

---

## Output Format

```
**Crew Sync:** [project root]

**Structure:** agents dir: ✅ / ❌  · inbox dir: ✅ / ❌  · memory dir: ✅ / ❌

🔴 Errors ([N]):
| # | File | Issue | Detail |
|---|------|-------|--------|
| 1 | memory/inbox/gauge.md | Orphan sender | Note from 'alethy' — not in crew roster (typo for 'aleth'?) |
| 2 | memory/agents/forge.md | Future timestamp | Entry dated 2027-01-01 |

🟡 Warnings ([N]):
| # | File | Issue | Detail |
|---|------|-------|--------|
| 1 | memory/agents/beacon.md | Stale (>90d) | Last entry 2026-02-15 |
| 2 | memory/agents/forge.md | Duplicate fact | Same content also in memory/agents/anchor.md — see entries [X] and [Y] |

🔵 Info ([N]):
| # | File | Issue | Detail |
|---|------|-------|--------|
| 1 | memory/agents/atlas.md | Bloat (>30 entries) | 42 entries — consider pruning |
| 2 | memory/inbox/forge.md | Backlog (>5) | 8 pending handoffs — oldest from 2026-05-01 |

**Summary:** 🔴 [N] errors · 🟡 [N] warnings · 🔵 [N] info items
**Clean:** [✅ / ❌ — clean only if zero errors]
```

If clean (zero errors, zero warnings, info only or nothing):
```
**Crew Sync:** [project root]
**Result:** Clean — no errors, no warnings. [N] info items. [or "Nothing to flag."]
```

---

## Rules

- ❌ **Read-only.** Never edit, drain, repair, or clear any memory file. Report issues; let the user (or the owning persona) act on them.
- ❌ **No fixing.** Don't move files, rename entries, delete orphan notes, archive stale entries. You are a diagnostic, not a janitor.
- ❌ **Don't skip checks after finding one.** Run all checks before reporting. A repo with no errors still needs the staleness and bloat warnings surfaced.
- ❌ **Report real problems, not formatting pedantry.** A single trailing blank line or a missing space after the dash is not an error — minor cosmetic variance in formatting is fine. Flag only semantic problems: missing date, wrong date, sender doesn't exist, fact duplicated across personas.
- ❌ **Severity matters.** Don't bury a 🔴 orphan-sender under a list of 🔵 bloat stats. Errors first, warnings second, info last.
- ❌ **No phantom personas.** When checking sender/recipient against the roster, use the base list above. For unrecognized names: flag as orphan, note the typo possibility, let the user confirm.
- ❌ **Not a persistent mode.** Audit once, report, stop.
- If the memory root (`memory/`) is entirely missing, emit that as a single 🔴 error and stop — nothing else to check.
