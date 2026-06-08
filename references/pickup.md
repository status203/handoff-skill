# Handoff — pickup flow

Loaded from `SKILL.md` once the routing selects pickup. `<base>` is already resolved (see `SKILL.md` § Handoffs directory) — don't re-resolve.

## Selection

Scan `<base>/active/`:
- **Single active handoff** → auto-select.
- **Multiple active handoffs** → present a short list (date, focus, first line of Goal). User chooses.

## Presentation

Read the selected handoff in full. **If it has a Ledgers section, read each ledger it names before distilling anything.** A ledger is *live state*, not superseded background: it holds the invariants the thread has already settled, so it must frame your orientation rather than arrive after you've re-proposed something it rules out. If a named ledger is missing, note it and carry on. Then present a distilled summary — not the raw file — and let the ledger's invariants shape the Next steps you surface.

**Fresh session** (full orientation):
- Goal — what the work is about
- State — where it left off
- Next steps — what to do first
- File path for full detail

**Mid-session** (merge framing):
- What the handoff covers (goal, one sentence)
- What's directly relevant to current work
- What's new context this session didn't have
- File path for full detail

## After pickup

- **Consume by moving the file** immediately, as part of pickup: `mv <base>/active/<file> <base>/consumed/<file>` (create `<base>/consumed/` first if absent). Consuming on pickup *claims* the handoff — it leaves the active pool so an overlapping or parallel session won't pick up the same one. Location encodes consumed; do not edit frontmatter. The move is atomic (power-cut safe), and a mistaken consume is cheap to undo — `mv` it back to `active/`.
- Next steps are suggestions, not assignments.
