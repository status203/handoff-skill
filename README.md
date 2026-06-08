# handoff

Yet another Claude Code skill for capturing session state so a fresh session can continue without re-discovery. Write a handoff at the end of a session, pick one up at the start of the next.

## How it compares

There are many handoff/context-passing scripts for Claude Code. Here are the decisions that were made for this one:

**Single skill, no framework.** Plain markdown — a `SKILL.md` entry point plus two flow references, no MCP servers, no external dependencies, no companion scripts. Install it and `/handoff` works.

**Context-aware default behaviour.** `/handoff` with no arguments scans for active handoffs and decides what to do: pick one up if you haven't done work yet, offer a choice if you have, or write a new one if none exist. You can also hand it a focus string (`/handoff "auth refactor"`) to write a new handoff directly — less ceremony when you already know you're writing one.

**Handoffs live in the project.** Handoffs go to `.handoffs/` in the project root by default — plain markdown with YAML frontmatter. They're version-controllable, greppable, and readable without tooling. No external state, no database. The location is overridable per project (see [Configuration](#configuration)).

**Location-encoded lifecycle.** Each handoff lives in `<base>/active/` (waiting for pickup) or `<base>/consumed/` (picked up). Pickup *moves* the file between the two — an atomic rename, not a frontmatter edit — so state is encoded by location, with no `status` field to drift. Consumed handoffs stay on disk; nothing is silently deleted. There's no prune command: to reclaim space in the rare case it matters, delete `<base>/consumed/` by hand.

**Ledgers for durable invariants.** A handoff is a snapshot, consumed on pickup — so across a long multi-session thread, conclusions settled early decay and a later session rederives them, or loses a fact already surfaced. A *ledger* is a persistent store of an effort's durable invariants — ruled-out approaches, framing decisions, surfaced facts — that every session reads and maintains as a small living set. Where it lives and what's in it are per-project judgments; the skill describes what belongs there and relays the *pointer*, not the contents, in each handoff's Ledgers section. A later session finds a ledger because the handoff it picked up named it, and reads it as live state before orienting. This replaces the old `Continues from:` chain line — across a long thread, the ledger is what actually needs carrying.

**Session-bounded content.** The skill explicitly guards against over-researching: a handoff captures what the session produced, not what it could produce with more investigation. For unstarted ideas, it asks the user for enough narrative to be legible later rather than exploring the codebase to populate sections.

**Sensitive-data guard (soft, LLM-interpreted).** Two best-effort guards, both LLM judgment rather than a deterministic scanner: handoffs are authored to *reference* sensitive values rather than embed them (unless you ask), and at write time, if the handoff will be git-tracked, the content is reviewed for likely secrets/PII and you're warned. It *describes* what counts as sensitive rather than prescribing patterns, so treat the warning as a prompt to check, not a guarantee.

**Structured sections with empty-section honesty.** Six mandatory sections (Goal, State, Decisions, Ruled out, Next steps, Key files) plus two optional (Context, Ledgers). Empty sections say "None this session." rather than being omitted — the receiver knows what was considered, not just what had content.

## Load profile

A skill's instructions load into the context window when it fires, so word count is a fair thing to weigh when picking one — it's the cost you pay every time. But it isn't a single number here: the skill keeps a small shared entry point and pulls in only the flow you actually take:

| Invocation | Loads | Words |
|---|---|---|
| Pick up a handoff | entry + pickup flow | ~1,050 |
| Write a handoff | entry + write flow | ~2,600 |
| Both (pick up, then write a continuation) | entry + both flows | ~2,900 |

The split is also deliberate about *where in a session* the cost lands. Pickup tends to fire near the **start** — context is near-empty, and whatever loads stays resident for the rest of the session, so the branch that's live longest is the lighter one. Writing fires at the **end** (or at least later, for genuine asides) — context is tighter by then, but less of the session remains to carry the heavier branch.

## When you'd use it

**A session ran out of room.** You're mid-refactor, the agent's starting to repeat "I am a fish" or contradict earlier decisions. The context window is the wall. Write a handoff, start a fresh session, pick it up — the new session has the state without the noise. (If the work's continuing and the context just needs trimming, `/compact` may be the lighter call — see [When /compact is enough](#when-compact-is-enough).)

**Work spans more than one session.** A multi-evening auth refactor: each evening's handoff uses focus `auth refactor`, and a shared ledger carries the effort's settled invariants across them — so tomorrow's session won't rederive what tonight's ruled out.

**An aside would derail the main thread.** A spike or open-ended side-question surfaces while you're deep in something else — *"would approach A or B be cleaner?"* — the kind where you'll want to poke at it and let early results steer the next question. Branch it: write a spike handoff, spin up a fresh session, do the investigation, hand back curated findings. The main thread stays focused; the return handoff is the filter — main picks up the distilled answer, not the side session's raw exploration. (If the question's bounded enough to specify up front, a subagent is the lighter call — see [When a subagent is enough](#when-a-subagent-is-enough).)

**An idea surfaces that you don't want to lose.** A thought, half-formed, unrelated to what you're doing right now — you don't want to commit it to todos, don't want to context-switch into another app, don't want to derail what you're working on. Write a handoff, keep going. *Caveat: this skill stores handoffs in the current project's `.handoffs/`. Thoughts tangential to this project resurface naturally next session. Thoughts about something else still get captured frictionlessly — but resurfacing is hit-and-miss.*

## When /compact is enough

`/compact` summarises the current session in-place; the handoff workflow writes a doc and starts fresh. They overlap. Two dimensions decide it:

**Mid-session vs. closing.** If the work is continuing and the context just needs trimming, `/compact` wins — pickup ceremony is friction. If the thread is breaking (end of day, task shift, context exhausted, side-question branching), the handoff wins; you can refresh yours and the LLM's contexts at the same time.

**Linear vs. "excessively deliberative".**

*It's late. The session has been... let's say, "excessively deliberative" — many options weighed, changes of mind, dead ends explored, you've had to come up with 5 different @#$%ing! ways of expressing the approach, you're both tired, there's dents in the monitor... &lt;ahem&gt;. I mean: the working context is full of reasoning that's been superseded.*

Both relieve that pressure — lossily; neither hands you the whole context back — so shedding bloat isn't where they differ. `/compact [focus]` is the lowest-ceremony move: one command, in place, no file, no pickup. But it regenerates a summary from the exhausted context that you never see, and even with a focus the superseded reasoning tends to ride along compressed, still subtly live. The handoff is written from that same tired context — but it's a durable artifact: you can read and correct it before pickup, and its `Decisions` / `Ruled out` structure forces the dead ends to be recorded as explicit-and-not-live rather than silently folded in. Pay its extra ceremony when it's the superseded reasoning — not just the volume — that's the problem.

The same gap applies to native session-forking. `/branch` (and the `--fork-session` flag) copies the whole conversation into a new session — raw, uncurated. `/rewind` does more than roll context back: in place, it can restore the conversation and/or the code to an earlier message (edits are checkpointed), or summarize the context from a chosen point. But that summarize is the same lossy auto-compression as `/compact`, and branch/fork copy verbatim — none of them give you a record you write, review, and correct. Forking is the native shape of branching an aside; the handoff is what makes it clean. The same test: fork or rewind when you want the full context to travel with you; handoff when carrying the superseded reasoning along is itself the problem.

## When a subagent is enough

A researcher subagent — the kind most agent-roster frameworks ship — keeps main's context clean the same way a branch does: the digging happens elsewhere, only the distilled answer comes back. So a branch overlaps with a dispatch much like it overlaps with `/compact`. The line is steering.

**Delegation vs. relocation.** A subagent is *delegation*: you specify the question up front, it runs autonomously, you collect its report — and you never leave main, so it can run in parallel while you keep working. A branch is *relocation*: you go *be* the branch session and drive it. The tell is whether you can frame the question well enough up front. If you can, dispatch — it's strictly cheaper on your attention. If the value is in the back-and-forth — early results reshape what you ask next, or it's a judgment call you'd rather make than trust delegated — branch, because that's the only version where you're in the loop while the work happens.

**Sub-task vs. peer session.** A subagent is a bounded sub-task: it returns once and it's gone. A branch is a peer session — it can span a break, and it can spawn subagents of its own. When the aside is big enough to want delegation *within* it, you've outgrown a single dispatch; branch.

The honest concession: a bounded lookup — *"what does this dependency actually do?"* — is better served by a subagent. Reach for the branch when the investigation needs your hand on it.

## Install

Download or clone this repository, then either copy or symlink the skill directory into your global skills location (typically `~/.claude/skills/handoff/`). The skill needs only `SKILL.md` and `references/`.

## Usage

```
/handoff                    # scan and decide: pickup, write, or ask
/handoff "focus text"       # write a handoff scoped to that focus
```

## Storage

```
.handoffs/                              (or the path in .claude/handoff.conf)
  active/
    2026-05-24T16-00-auth-refactor.md
    2026-05-25T09-00-continuation.md
  consumed/
    2026-05-24T14-30-auth-refactor.md
```

Files are `<ISO-datetime>-<focus-slug>.md`. Frontmatter tracks focus and creation time; state is the subdirectory (`active/` vs `consumed/`), and any ledgers relevant to the work are listed by path in the body's Ledgers section.

## Configuration

The handoff location is configured per project, via a `.claude/handoff.conf` file at the project root. Without one, handoffs go to `.handoffs/`.

Its first non-blank, non-comment (`#`) line is read as a directory path relative to the project root, for example:

```
docs/handoffs
```

Surrounding whitespace is trimmed and a trailing slash is fine. The path is always relative to the project root; `~` and environment variables are not expanded. Pointing it at a tracked-but-not-shipped directory lets handoffs travel with the repo.

## What's included

| File | Purpose |
|---|---|
| `SKILL.md` | Entry point — invocation, directory resolution, the no-arg routing decision, lifecycle |
| `references/writing.md` | Write flow — document format, sections, style rules, write steps (loaded when writing) |
| `references/pickup.md` | Pickup flow — selection, presentation, consume-on-pickup (loaded when picking up) |
