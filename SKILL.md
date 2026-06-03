---
name: handoff
description: Use when needing to capture session state for continuation in another session, branch work to a parallel session, or pick up a previous handoff
---

# Handoff

## Overview

Write and pick up session handoff documents. A handoff captures what happened, what was decided, what failed, and what's next — enough for a fresh session to continue without re-discovery.

**Core principle:** Specific enough to act on, honest about what's empty, self-contained enough to stand alone.

**Announce at start:** "I'm using the handoff skill to [write a handoff / pick up a handoff]."

## Invocation

Single entry point: `/handoff`. Behaviour depends on arguments.

| Invocation | Behaviour |
|---|---|
| `/handoff` | Context-aware: resolve the directory, scan for active handoffs, then decide (see flow below) |
| `/handoff "focus text"` | Always write a handoff scoped to that focus → load and follow `references/writing.md` |

## Handoffs directory

**Resolving the base directory is the first thing every `/handoff` invocation does — once, before any branching — and the resolved paths are reused throughout.** Never reference a literal `.handoffs/` past this step, and don't re-resolve inside the write or pickup flows (a no-arg invocation that branches into pickup has already resolved). This single point of resolution is what makes the location reliably overridable.

**Resolution — run this, don't eyeball the file** (the same discipline as *fetching* the timestamp rather than guessing it):

```sh
base="$(grep -vE '^[[:space:]]*(#|$)' .claude/handoff.conf 2>/dev/null | head -n1 | xargs)"
: "${base:=.handoffs}"
```

That is: `<base>` is the first non-blank, non-comment (`#`) line of `.claude/handoff.conf`, trimmed, taken as a path relative to the project root; if the file is absent or has no such line, `<base>` is `.handoffs`. Running the command keeps resolution identical across flows and invocations.

Derive the two working paths once — `<base>/active/` (waiting for pickup) and `<base>/consumed/` (picked up). State is encoded by **which subdirectory** a file sits in; there is no `status` field.

### No-arg flow

````dot
digraph handoff_noarg {
    resolve [label="Resolve <base>"];
    scan [label="Scan <base>/active/"];
    any [label="Any active?", shape=diamond];
    work [label="Substantive\nwork done?", shape=diamond];
    write [label="Write continuation"];
    pickup [label="Pickup flow"];
    ask [label="Ask: pickup\nor write?"];

    resolve -> scan;
    scan -> any;
    any -> write [label="  none"];
    any -> work [label="  found"];
    work -> pickup [label="  no / early"];
    work -> ask [label="  yes"];
}
````

**Scan:** read frontmatter (focus, created) plus the first line of the Goal section from each file in `<base>/active/`. Everything in `active/` is active by definition — no status filter.

**"Substantive work done"** is a judgment call based on session context — file edits, implementation decisions, research findings. The cost of a wrong guess is one exchange to correct, and `/handoff "focus"` always bypasses the heuristic.

**"Ask: pickup or write?"** — present the choice neutrally. Pickup is the default; writing is the alternative. Use AskUserQuestion with these options:
1. **Pick up the handoff** — load the existing handoff and orient to its next steps (default).
2. **Write a new handoff** — capture this session's work as a separate handoff.
3. **Both** — pick up the existing handoff, then write a continuation that merges its context with this session's work.

Frame the question around what the user wants to do, not around the handoff's content. Do not summarise the handoff's findings, evaluate its next steps, or propose actions on its content before the user chooses — that analysis belongs in the pickup flow, not the ask step.

**Branch → reference.** Once the flow lands on an action, load the matching reference and follow it:
- **Write** (no active handoffs, or the user chose "Write") → `references/writing.md`.
- **Pickup** (active handoff and no substantive work, or the user chose "Pick up") → `references/pickup.md`.
- **Both** (the user chose "Both") → `references/pickup.md` first, then `references/writing.md` for the continuation.

## Lifecycle

```
write  →  <base>/active/<file>
pickup →  mv to <base>/consumed/<file>
```

- Pickup never deletes; it moves. The scan only ever globs `<base>/active/`, so consumed handoffs drop out of view automatically.
- Consumed handoffs accumulate harmlessly in `<base>/consumed/`. There is **no prune flow** — to reclaim space in the rare case it matters, `rm -rf <base>/consumed/` by hand.
- Handoffs can carry conversational content that shouldn't be shared. The write flow keeps literal secrets/PII out unless you ask for them, and warns if a handoff will be git-tracked *and* still looks sensitive (see `references/writing.md` § Sensitive data). Where handoffs are stored otherwise — gitignored, or a publish-stripped folder — is the project's choice; no-leak is the only requirement.
