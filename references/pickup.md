# Handoff — pickup flow

Loaded from `SKILL.md` once the routing selects pickup. `<base>` is already resolved (see `SKILL.md` § Handoffs directory) — don't re-resolve.

## Selection

Scan `<base>/active/`:
- **Single active handoff** → auto-select.
- **Multiple active handoffs** → present a short list (date, focus, first line of Goal). User chooses.

## Presentation

Read the selected handoff in full. Present a distilled summary — not the raw file.

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
- **Chain is consulted only on demand, and ranks below the handoff you picked up.** If the handoff has a `Continues from:` line, you may mention the predecessor(s) exist — but do **not** read or follow automatically. Open one only if the user asks, or the current handoff explicitly defers a specific fact to it; then read *that one* predecessor for *that fact* — one hop, never a transitive walk. A predecessor is **superseded background**: where it conflicts with the handoff you picked up, the picked-up one wins (it's later and supersedes by construction). Read it for context the current handoff assumes, not for current direction.
- Next steps are suggestions, not assignments.
