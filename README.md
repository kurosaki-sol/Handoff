# `handoff` — evidence-based context persistence for Claude Code

> Save the state of a Claude Code session so the next session can resume in
> under 60 seconds — without relying on stale info that silently drifted
> while you were away.

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that
runs at session end (or context-cap time) and persists three kinds of value
separately, with **evidence-based staleness tracking** instead of a
calendar-based TTL.

- **Reusable learnings** → extracted into the Claude Code memory system so
  they auto-load into every future session in the same project directory.
- **Project work state** → written to a single dense `HANDOFF.md` at the
  project root.
- **Durable team facts** → optionally appended to the project's existing
  `CLAUDE.md` when (and only when) the insight is high-signal enough.

---

## Why this exists

Long Claude Code sessions accumulate value that dies on `/clear`. Two
predecessors tried to fix this:

1. **`brain-dump`** (the original version of this repo) — a two-phase
   skill that extracted reusable skills to `~/.claude/skills/` (Phase A)
   and persisted session state to `CLAUDE.md` + `/docs/*.md` (Phase B).
   Honest, well-structured, and the intellectual starting point for this
   work.

2. **The memory system Anthropic ships with Claude Code** — a per-project
   memory directory (`~/.claude/projects/<slug>/memory/`) whose `MEMORY.md`
   index is automatically loaded into every new session in that directory.
   Typed entries (`user` / `project` / `feedback` / `reference`) let
   Claude build up a useful picture of the collaboration over time.

`handoff` is the merge of both, plus one thing neither addressed:

> **Memories rot.** A learning that was true six months ago may describe a
> reality that no longer exists — but Claude has no way to tell. Worse, it
> will confidently rely on the stale info and give you recommendations
> based on a codebase state that doesn't exist anymore.

`brain-dump` had no staleness signal. The Anthropic memory system has
no automatic freshness check either. A naive fix — "expire memories after
N days" — is wrong: a project dormant for six months is still accurate; a
project that churned 200 commits in two days may have invalidated half its
memories overnight. **Time-on-the-clock is the wrong axis.**

`handoff` fixes this by recording **evidence** at write-time (git
commits, file hashes, TTLs for external facts, or explicit
"user-override-only") and by defining a clear state machine
(`fresh → suspect → contested → stale → archived`) that future sessions
walk when a memory is recalled. A dormant project's memories stay
`fresh`; a fast-moving project's memories hit `suspect` the moment git
shows drift on the paths they pinned.

---

## What it does

When you invoke it (typing `handoff`, `brain-dump`, `save state`, `bilan`,
or close to any of those, or when your context is at ~85%), Claude Code
runs three phases:

### Phase A — Persistent memory extraction

For each non-obvious learning that passes quality gates (reusable,
specific, verified, not a duplicate), it writes a memory file with full
staleness metadata:

```yaml
---
name: Helius webhook authentication model
description: Helius signs outgoing webhooks via verbatim-echo of the authHeader
type: reference
volatility: low
state: fresh
created_at: 2026-04-21
last_verified_at: 2026-04-21
pin:
  type: ttl
  expires: 2026-10-21
  reason: "Reconfirm every 6 months in case Helius adds HMAC."
domain_tags: [helius, webhook, auth, third-party]
---

{the actual learning, with Why and How-to-apply sections}
```

Seven pin types cover every scenario:

| `pin.type` | Staleness signal |
|---|---|
| `git-commit` | Claim is still valid if `git log <commit>..HEAD -- <paths>` is empty. |
| `git-path` | Same, scoped to a subdirectory of a monorepo. |
| `git-multi` | Multi-repo project (e.g. backend + frontend); stale if any repo drifts. |
| `file-hash` | No git (or git forbidden); re-hash files and compare. |
| `ttl` | External fact with a known expiry (API sunset, roadmap milestone). |
| `user-override-only` | User preferences, collaboration rules. Never invalidated by code. |
| `semantic` | Cross-project general knowledge. Trust-on-first-use; invalidated by contradiction. |

### Phase B — HANDOFF.md

A single dense file at your project root, scannable cold in under a
minute. Template includes:

- Status at a glance (branches, PRs, current phase, blockers)
- PRs in flight (table with commit hashes + deploy notes)
- What was done this session (terse bullets)
- Decisions made (**append-only** — never overwritten)
- Open questions to validate with the team
- TODOs prioritized P0 / P1 / P2 / Won't-do (with reasons)
- Known deploy risks (DB migrations, env var dependencies, etc.)
- Resume script for the next session

### Phase C — CLAUDE.md (optional)

Only runs when a durable, high-signal insight genuinely belongs in the
team-facing `CLAUDE.md` at the project root (architecture pattern
validated by the team, threat model clarification, adopted convention).
Skipped otherwise — it won't pollute your team's diff for the sake of
pollution.

---

## How memory gets checked in future sessions

The skill doesn't run a check at write-time or at every session start
(both would be wasteful). Instead, memories are **checked lazily when
recalled**. This is the behavior Claude is expected to follow when it
loads a memory into its reasoning:

```
For each recalled memory:
  if pin.type == "user-override-only":   use as fresh
  if pin.type == "git-commit":
    git log <commit>..HEAD -- <paths>
    if zero commits:                     stays fresh
    else:                                → suspect, re-read files
  if pin.type == "ttl":
    if expires < today:                  → suspect
  if pin.type == "semantic":             trust, watch for contradictions
  ...
```

On contradiction (user correction, grep-disproven claim, file hash
mismatch), the memory transitions `fresh → contested` or `fresh → stale`,
and Claude flags the conflict **before** acting on the old value.

**Hard overrides always win.** A user saying "we don't use Mongo anymore"
invalidates every Mongo-tagged memory regardless of other signals.

---

## Installation

```bash
git clone https://github.com/YOUR_USER/claude-code-handoff
mkdir -p ~/.claude/skills/handoff
cp claude-code-handoff/SKILL.md ~/.claude/skills/handoff/SKILL.md
```

Claude Code picks up skills from `~/.claude/skills/*/SKILL.md`
automatically. No config needed. Next session, type `handoff` or let
Claude invoke it at ~85% context.

---

## Usage examples

See [`examples/`](./examples/) for:

- [`stealf-session.md`](./examples/stealf-session.md) — a real handoff
  after shipping 5 PRs + auditing threat model + writing 8 memories. Shows
  `HANDOFF.md` + memory files + how phase C was deliberately skipped.
- [`dormant-project-return.md`](./examples/dormant-project-return.md) —
  what returning to a project 6 months later looks like when every memory
  stays `fresh` because the code hasn't moved on the pinned paths.
- [`fast-moving-project.md`](./examples/fast-moving-project.md) — the
  opposite: a project with 200 commits in 2 days, multiple memories
  transitioning to `suspect` on first recall.

---

## Lineage and attribution

This skill is a direct descendant of two earlier designs:

- **`brain-dump`** by community contributors — the original two-phase
  (skill-extraction + state-persistence) pattern. Structure and Phase A/B
  separation come from there. The original version of this repo (when it
  was titled "BrainDump - Claude Context Persistence") shipped that
  skill. See [git history](../../commits/main) for the transition.

- **Anthropic's memory system** for Claude Code — the typed
  (`user`/`project`/`feedback`/`reference`) per-project memory at
  `~/.claude/projects/<slug>/memory/` with `MEMORY.md` auto-loaded every
  session. The `handoff` skill writes into this system rather than to
  `~/.claude/skills/` global skills, so memories stay scoped to the
  project where they were learned.

The novel contribution in `handoff` is the **staleness schema**
(`volatility` × `pin.type` × `state` lifecycle) and the lazy
evidence-based verification rule. Credit to `brain-dump` for the
two-phase skeleton; credit to Anthropic for the memory substrate; the
staleness layer is new.

---

## Design decisions

A handful of choices that might read as surprising:

- **No calendar TTL by default.** `ttl` is one pin type among seven, not
  the backbone. A six-month-dormant project's `git-commit`-pinned
  memories stay `fresh` — there's literally no evidence they've rotted.
  Forcing them to expire on a calendar would produce false suspicion.

- **Lazy verification, not eager.** Loading all memories into context and
  running `git log` on each at session start would cost tokens for
  memories we never use that session. We pay the verification cost only
  on recall.

- **Fail-closed on conflict.** When evidence contradicts a memory,
  Claude is expected to surface the conflict *before* acting on the old
  value, not after. The state machine has no path from `contested` to
  `fresh` that bypasses user or code re-confirmation.

- **Decision log is append-only.** The "Decisions made" section in
  `HANDOFF.md` never rewrites prior entries. A decision that's later
  reversed produces a **new** entry that supersedes — the history is
  preserved for audit.

- **Phase C is pessimistic by default.** Editing a team's `CLAUDE.md`
  pollutes their diffs, so the skill only touches it when the insight is
  genuinely durable and high-signal. Most sessions end with phase C
  skipped, and that's the right default.

- **No git writes.** The skill only writes files. Commits, pushes,
  branch ops are all left to the user — the skill is compatible with
  arbitrary git workflows including those with strict hooks.

---

## FAQ

**How is this different from CLAUDE.md?**
CLAUDE.md is a flat team-facing doc, read once at session start. `handoff`
memories are typed, scoped to a project directory, and carry staleness
metadata. CLAUDE.md doesn't rot less than memories do — it just hides the
rot. `handoff` surfaces it.

**How is this different from `brain-dump`?**
`brain-dump` writes Phase A as **global** skills in `~/.claude/skills/`.
`handoff` writes Phase A as **project-scoped** memories in
`~/.claude/projects/<slug>/memory/`. The latter auto-loads in the project
it was learned in and stays out of unrelated projects. Also, `brain-dump`
has no staleness tracking; `handoff`'s core contribution is exactly that.

**What happens if I move the project directory?**
The memory path encodes the absolute project path (e.g.
`-home-user-Documents-MyProject`). Moving the project orphans its
memories. Workaround: rename the memory dir to match. A future skill
version could detect this via a git remote origin fingerprint.

**Does it work without git?**
Yes. Use `pin.type: file-hash` (recompute hashes) or
`pin.type: user-override-only` (only invalidated by user). Multiple
git-less projects can coexist. The skill explicitly documents edge cases
for monorepo, multi-repo, read-only mirror, detached workspace, and
no-git-allowed scenarios.

**How big can memories grow before they hurt context?**
`MEMORY.md` (the index) auto-truncates at ~200 lines, which is a hard
ceiling of ~200 memories per project. Individual files aren't loaded
unless recalled. In practice a well-curated project lives under 20-30
memories; beyond that, invoke `prune memory` (not yet a skill — tracked
as a follow-up here).

---

## Contributing

Issues and PRs welcome. The skill's design hits many orthogonal concerns
(git semantics, state machines, file formats, LLM reasoning boundaries)
so please prefix your issue with the dimension you're attacking.

Before opening a PR that changes the schema, check:

- Does the change preserve backwards compatibility with v1 / v2 memories?
  The skill has an explicit migration path in `SKILL.md`.
- Does it keep lazy verification as the default? Eager checking is fine
  as an opt-in flag but must not be forced.
- Does it keep the "no git writes" rule? The skill is usable inside any
  workflow exactly because it stays out of git state mutation.

---

## License

[MIT](./LICENSE) — use it, fork it, redistribute it. Attribution
appreciated (link back to this repo) but not required.
