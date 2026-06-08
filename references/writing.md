# Handoff — write flow

Loaded from `SKILL.md` once the routing selects writing. `<base>` is already resolved (see `SKILL.md` § Handoffs directory) — don't re-resolve.

## Document format

### Filename

```
<base>/active/<ISO-datetime>-<focus-slug>.md
```

Timestamp for sort order, kebab-case focus slug for identification (max ~40 chars). Examples:
- `<base>/active/2026-05-24T14-30-continuation.md`
- `<base>/active/2026-05-24T14-30-auth-refactor.md`

### Frontmatter

````yaml
---
focus: "auth refactor"
created: 2026-05-24T14:30:00Z
---
````

- `focus`: the focus argument, or `"continuation"` if none given.
- `created`: the real UTC timestamp (see Write flow step on `date -u`).

No `status` field — location (`active/` vs `consumed/`) encodes that.

### Sections

Six mandatory sections, two optional. Mandatory sections always appear — if nothing applies, write "None this session."

1. **Goal** — what the session set out to accomplish. One or two sentences.
2. **State** — what's done, in-progress, blocked. Prose, not a checklist.
3. **Decisions** — choices made with rationale. "We chose X because Y."
4. **Ruled out** — approaches tried or considered and abandoned, with *why*.
5. **Next steps** — prioritised actions for the receiver. Most important first. Cite paths and functions the session actually touched; for unstarted work, orient ("the change probably goes in X") without investigating to confirm.
6. **Key files** — paths the session read, edited, or discussed, with brief annotation per file. Don't list files discovered while writing the handoff.
7. **Ledgers** *(optional, omit if none relevant)* — persistent stores this effort keeps its durable invariants in (see § Ledgers), each as `path — what it holds`. By **stable path**, not filename: a ledger doesn't move between `active/` and `consumed/` the way a handoff does, so the filename-not-path rule that governs handoff links inverts here. List every ledger relevant to the task — carried forward from whatever this session picked up, plus any created this session. This pointer is the only thing relayed across the thread; the ledger's contents are never restated here.
8. **Context** *(optional, omit if empty)* — anything relevant that doesn't fit the above. Each item gets a one-line lead-in explaining why it matters. Last resort, not overflow.

### Style

- **Session-bounded.** A handoff preserves the session's state — it does not advance the work. If filling a section would require reading files, exploring the codebase, or doing design work the session didn't do, that section is "None this session." or carries only what's already in hand from conversation context. The receiving session does its own discovery. For unstarted work, "the change probably goes in X" is enough orientation — don't read source, cite line numbers, or draft a multi-step plan to fill a section; that's advancing the work, not capturing it. Exception: when the handoff captures a standalone idea with minimal session context, ask the user for enough narrative that the idea makes sense cold — what it is, why it matters, what prompted it (elicitation, not investigation). A bare line-item rots; a sentence or two of motivation keeps it legible a month later.
- **Prose, not bullets.** Write coherent sentences, not fragment lists — fragments lose the rationale. "Tried Redis — too slow" doesn't say what was measured; "we tried Redis but p99 latency was 12ms against our 5ms budget, so we switched to in-memory snapshotting" does.
- **Specificity discipline.** Exact paths, exact errors, exact commands — for work that was done. "Fixed null user in `src/auth/middleware.ts:42` by initializing before guard check" — not "fixed the auth bug." For unstarted work, record what's already known; don't investigate to fill gaps.
- **Empty-section honesty.** "None this session." — not section omission. The receiver can't tell "considered and empty" from "never considered," so the explicit note is informative where silence is ambiguous.

## Ledgers

A handoff is a snapshot — picked up once, then consumed. That holds until a thread runs long: a conclusion ruled out in an early session lives only in its consumed handoff, and the session-bounded rule (above) tells each later handoff not to restate it, so across a multi-handoff thread the settled conclusions decay and a later session rederives them, or loses a fact already surfaced. A **ledger** is the fix — a persistent store of an effort's durable invariants that every session in the thread reads and maintains, so the invariants don't have to survive the relay.

**What belongs in a ledger** is whatever would cost *re-derivation* to lose, not merely *re-orientation*. Re-orientation ("where was I?") — state, next steps, files touched — stays in the handoff and expires on pickup. Re-derivation ("did we already settle this?") — ruled-out approaches, framing decisions, surfaced invariant facts — belongs in the ledger, because its job is to bind sessions you can't name yet. Promote the conclusion, not the journey; the constraint, not the task. The conclusions most worth promoting are the *attractors* — the ones a fresh, capable session would plausibly re-propose; those are exactly what gets rederived.

**A ledger is a living set, not a log.** When an invariant stops being true or a later decision supersedes it, edit or strike it — don't append a correction. It's re-read on every pickup, so it has to stay small; that smallness is what makes it work.

**Where a ledger lives and what it's called are per-project judgments** — a file in the repo, a page of the artifact you're building, a doc beside the handoffs. There is no fixed location and no prescribed format. The skill doesn't choose for you; it only says what belongs in one and how to carry it forward.

**Carrying it forward is mechanical: the pointer, never the contents.** Each handoff records the ledgers relevant to its task by path, in the Ledgers section. A later session finds the ledger because the handoff it picked up named it — and re-lists it, so the next handoff does too. The judgment of whether, where, and what is paid once, when the ledger is identified or created; from then on the pointer relays that decision without re-judging it. Never relay a ledger's *contents* through handoffs — that's heavy, lossy, and the very thing the ledger exists to avoid.

## Sensitive data

Handoffs are conversational, so they're more likely than code to capture a secret, token, or third-party detail seen in passing — and git **history** keeps it even if a later publish/wipe step strips the working copy. Handle it at authorship (prevent), backed by a check at write time (detect). Both refer to the one definition below.

### What counts as sensitive

A judgment call, not a fixed list — err toward flagging. The categories:

- **Credentials and secrets** — API keys, access tokens, private keys, passwords, session cookies, connection strings with embedded credentials. Often surface as long opaque/high-entropy strings or `KEY=value` / `password: …` shapes.
- **Third-party PII** — real names, addresses, emails, phone numbers, or account identifiers belonging to people other than the user.
- **Private infrastructure** — internal hostnames, private URLs, or IPs not meant to be public.
- **Anything the user flagged confidential in-session.**

Ignore obvious placeholders as values: `xxx`, `<your-key>`, `REDACTED`, `changeme`, `example`.

### Authoring (prevent)

A handoff's job is to let a future session *continue* — which almost never needs a literal sensitive value. Reference it by location or description ("the API key is in `.env`", "credentials in the password manager", "the customer's email from the ticket") rather than embedding the value. Include a literal value only if the user **specifically asks**, or continuation genuinely can't work without it (rare).

The same applies to a ledger: it's persistent and usually git-tracked, so reference sensitive values by location rather than embedding them there too — more so, since a ledger outlives the handoff.

## Write flow

1. Determine focus: the argument if provided, otherwise `"continuation"`. (`<base>` is already resolved — see `SKILL.md` § Handoffs directory.)
2. Create `<base>/active/` if it doesn't exist.
3. **Ledgers — give durable artifacts a home, then record the pointers.** Apply the promotion test (§ Ledgers) to what this session produced: is there content that would cost *re-derivation* to lose — a ruled-out approach, a framing decision, a surfaced invariant fact? This check runs on every handoff; it simply finds nothing on a pure aside or a one-off, and most often finds something on a long multi-session effort. For each durable artifact:
   - **A ledger already exists** (a handoff this session picked up named one, or you created one) → make sure the artifact is *in* it, editing the ledger as a living set — supersede stale entries rather than appending.
   - **No ledger yet** → *recommend* creating one (propose where it should live and what goes in it) and **ask before creating the file**. If the user declines, fold the artifact into the handoff as usual and say plainly that this re-accepts the decay a ledger would prevent — not an equivalent outcome.
   Then record every task-relevant ledger in the **Ledgers** section by path (carried-forward plus any new one). Omit the section if no ledger is relevant.
4. Walk the session context. Extract content into the eight sections. Apply style rules: prose, specific, honest about empty sections. Don't reproduce sensitive values — reference them per § *Sensitive data* (Authoring).
5. Get the real timestamp: run `date -u +%Y-%m-%dT%H:%M:%SZ` in its own step, before writing. Never guess the time, reuse a stale reading from earlier in the session, or stamp local time as UTC. Derive both fields from this one value — the `created:` frontmatter is the output verbatim (e.g. `2026-05-24T14:30:00Z`); the filename `<ISO-datetime>` is the same instant to minute precision with `:` replaced by `-` (e.g. `2026-05-24T14-30`).
6. Write the file to `<base>/active/<ISO-datetime>-<focus-slug>.md`.
7. **Leak check.** After writing, check whether git will carry the file:
   ```sh
   git rev-parse --is-inside-work-tree >/dev/null 2>&1   # is this a git work tree?
   git check-ignore -q "<base>/active/<file>"            # exit 0 = ignored, 1 = will be tracked
   ```
   If there's no git work tree, or `check-ignore` exits **0** (ignored), skip — git won't carry the file. If `check-ignore` exits **1**, the file will be tracked (newly added, whitelisted, or already-tracked — the one call covers all three, including the whitelist-under-an-excluded-dir asymmetry, so no `git ls-files` is needed); scan its text against § *Sensitive data* — flag anything in those categories (judgment; err toward flagging).
   If this session created or edited a ledger, run the same check against that file — it's persistent and usually tracked, so the sensitive-data discipline matters there at least as much. (A ledger may live outside `<base>`; check wherever it actually is.)
8. Report to the user: the **full path** of the file as written — the complete `<base>/active/<file>`, from the project root (e.g. `.handoffs/active/2026-05-24T14-30-auth-refactor.md`), not a bare filename or partial path. Handoffs are project-local, so this project-relative path uniquely identifies the file. Report a one-line summary of what was captured. If the leak check matched, append a warning (warn, don't block): name the location, that git will track it, the find with its line/snippet, and the options — redact, gitignore the handoffs dir, or confirm it's fine. Note that git history retains it even if a later publish step strips the file. If a ledger was created or updated, name it (path) and what was recorded.
