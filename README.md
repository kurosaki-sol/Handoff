# handoff

> **Because fck /compact — and fck stale memories.** Evidence-based session
> persistence for Claude Code, Codex CLI, OpenCode, and any other skills-
> compatible agent. Git-pinned, hash-pinned, or TTL-pinned learnings that
> stay fresh when your project is dormant and get flagged the moment the
> code moves — no calendar TTLs, no false confidence.

This skill follows the [Agent Skills specification](https://agentskills.io/specification)
so it works with any skills-compatible agent, including **Claude Code**,
**Codex CLI**, **OpenCode**, and **Vercel Agent**. Same `SKILL.md`, same
behavior across tools — the install path is the only thing that changes.

---

## The problem

Long agent sessions accumulate value that dies on `/clear` or `/compact`.
Two predecessors tried to fix this:

1. **[`brain-dump`](https://github.com/kurosaki-sol/BrainDump---Claude-Context-Persistence)** (the original version of this
   repo) — a two-phase skill that extracted reusable skills globally
   (Phase A) and persisted session state to `CLAUDE.md` + `/docs/*.md`
   (Phase B). Honest, well-structured, and the intellectual starting
   point for this work.

2. **Anthropic's memory system** in Claude Code — a per-project memory
   directory whose `MEMORY.md` index auto-loads into every new session
   in that directory. Typed entries (`user` / `project` / `feedback` /
   `reference`) let the agent build up a picture of the collaboration
   over time.

`handoff` is the merge of both, plus one thing neither addressed:

> **Memories rot.** A learning that was true six months ago may describe a
> reality that no longer exists — but the agent has no way to tell. Worse,
> it will confidently rely on the stale info and give recommendations
> based on a codebase state that doesn't exist anymore.

`brain-dump` had no staleness signal. The Anthropic memory system has no
automatic freshness check either. A naive fix — "expire memories after N
days" — is wrong: a project dormant for six months is still accurate; a
project that churned 200 commits in two days may have invalidated half
its memories overnight. **Time-on-the-clock is the wrong axis.**

`handoff` fixes this by recording **evidence** at write-time (git
commits, file hashes, TTLs for external facts, or explicit
"user-override-only") and by defining a clear state machine
(`fresh → suspect → contested → stale → archived`) that future sessions
walk when a memory is recalled. A dormant project's memories stay
`fresh`; a fast-moving project's memories hit `suspect` the moment git
shows drift on the paths they pinned.

---

## What it does

When you invoke it (typing `handoff`, `brain-dump`, `save state`,
`bilan`, or when your context is at ~85%), the agent runs three phases:

### Phase A — Persistent memory extraction

For each non-obvious learning that passes quality gates (reusable,
specific, verified, not a duplicate), it writes a memory file with full
staleness metadata. Seven pin types cover every scenario:

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
minute. Template includes status-at-a-glance, PRs in flight, decisions
made (append-only log), open questions, prioritized TODOs, deploy
risks, and a resume script for the next session.

### Phase C — CLAUDE.md / AGENTS.md (optional)

Only runs when a durable, high-signal insight genuinely belongs in the
team-facing context file. Skipped otherwise — it won't pollute the
team's diff for the sake of pollution.

---

## Installation

### Via the Agent Skills CLI (recommended — cross-agent)

If you have the [`skills` CLI](https://skills.sh) installed, a one-liner
installs into whichever of your agents is active:

```bash
npx skills add kurosaki-sol/Handoff
```

Use `-g` for a global install, or install to a specific project:

```bash
npx skills add -g kurosaki-sol/Handoff        # global
npx skills add kurosaki-sol/Handoff           # project scope
```

### Manually, per agent

The `SKILL.md` is the only file the agent needs. Drop it in the right
place for your tool:

#### Claude Code

```bash
git clone https://github.com/kurosaki-sol/Handoff
mkdir -p ~/.claude/skills/handoff
cp Handoff/SKILL.md ~/.claude/skills/handoff/SKILL.md
```

Claude Code picks up the skill automatically on next session. No
restart needed.

#### Codex CLI

```bash
git clone https://github.com/kurosaki-sol/Handoff
mkdir -p ~/.codex/skills/handoff
cp Handoff/SKILL.md ~/.codex/skills/handoff/SKILL.md
```

Or, if you'd rather let Codex fetch and wire it up itself:

```bash
# From inside a Codex session, ask it to clone the repo into its skills dir:
clone https://github.com/kurosaki-sol/Handoff into ~/.codex/skills/handoff
# then `handoff` triggers the skill in subsequent turns
```

#### OpenCode

```bash
git clone https://github.com/kurosaki-sol/Handoff ~/.opencode/skills/handoff
```

OpenCode watches its skills directory for new entries on the next agent
turn.

#### Vercel Agent / v0

Install via the skills CLI as above, or point Vercel Agent at the
GitHub URL directly from the skill picker UI.

#### Any other skills-compatible agent

Drop `SKILL.md` into the agent's documented skills directory. The
frontmatter is the open [agentskills.io](https://agentskills.io/specification)
format — no agent-specific fields are required.

---

## Usage examples

See [`examples/`](./examples/) for:

- [`stealf-session.md`](./examples/stealf-session.md) — a real handoff
  after shipping 5 PRs + auditing a threat model + writing 8 memories.
  Shows `HANDOFF.md` + memory file + how phase C was deliberately
  skipped.
- [`dormant-project-return.md`](./examples/dormant-project-return.md) —
  returning to a project 6 months later and finding every memory stays
  `fresh` because the code hasn't moved on the pinned paths.
- [`fast-moving-project.md`](./examples/fast-moving-project.md) — the
  opposite: a project with 200 commits in 2 days, multiple memories
  transitioning to `suspect` on first recall.

---

## How memory gets verified in future sessions

The skill doesn't run a check at write-time or at every session start
(both would be wasteful). Memories are checked **lazily on recall**:

```
For each recalled memory:
  if pin.type == "user-override-only":  use as fresh
  if pin.type == "git-commit":
    git log <commit>..HEAD -- <paths>
    if zero commits:                    stays fresh
    else:                                → suspect, re-read files
  if pin.type == "ttl":
    if expires < today:                 → suspect
  if pin.type == "semantic":            trust, watch for contradictions
  ...
```

On contradiction (user correction, grep-disproven claim, file hash
mismatch), the memory transitions `fresh → contested` or `fresh →
stale`, and the agent flags the conflict **before** acting on the old
value.

**Hard overrides always win.** A user saying "we don't use Mongo
anymore" invalidates every Mongo-tagged memory regardless of other
signals.

---

## Lineage and attribution

This skill is a direct descendant of two earlier designs:

- **[`brain-dump`](https://github.com/kurosaki-sol/BrainDump---Claude-Context-Persistence)** — the
  original two-phase (skill-extraction + state-persistence) pattern.
  Structure and Phase A/B separation come from there.

- **Anthropic's memory system** for Claude Code — the typed per-project
  memory at `~/.claude/projects/<slug>/memory/` with `MEMORY.md`
  auto-loaded every session. `handoff` writes into this system rather
  than to `~/.claude/skills/` global skills, so memories stay scoped to
  the project where they were learned.

The novel contribution in `handoff` is the **staleness schema**
(`volatility` × `pin.type` × `state` lifecycle) and the lazy
evidence-based verification rule. Credit to `brain-dump` for the
two-phase skeleton; credit to Anthropic for the memory substrate; the
staleness layer is new.

---

## Design decisions

A handful of choices that might read as surprising:

- **No calendar TTL by default.** `ttl` is one pin type among seven,
  not the backbone. A six-month-dormant project's `git-commit`-pinned
  memories stay `fresh` — there's literally no evidence they've rotted.
  Forcing them to expire on a calendar would produce false suspicion.

- **Lazy verification, not eager.** Loading all memories into context
  and running `git log` on each at session start would cost tokens for
  memories we never use that session. Verification cost is paid only on
  recall.

- **Fail-closed on conflict.** When evidence contradicts a memory, the
  agent is expected to surface the conflict *before* acting on the old
  value, not after. The state machine has no path from `contested` to
  `fresh` that bypasses user or code re-confirmation.

- **Decision log is append-only.** The "Decisions made" section in
  `HANDOFF.md` never rewrites prior entries. A decision that's later
  reversed produces a **new** entry that supersedes — the history is
  preserved for audit.

- **Phase C is pessimistic by default.** Editing a team's `CLAUDE.md`
  pollutes their diffs, so the skill only touches it when the insight
  is genuinely durable and high-signal. Most sessions end with phase C
  skipped, and that's the right default.

- **No git writes.** The skill only writes files. Commits, pushes,
  branch ops are all left to the user — the skill is compatible with
  arbitrary git workflows including those with strict hooks.

---

## FAQ

**How is this different from `CLAUDE.md` / `AGENTS.md`?**
Those are flat team-facing docs, read once at session start. `handoff`
memories are typed, scoped to a project directory, and carry staleness
metadata. Flat context files don't rot less than memories do — they
just hide the rot. `handoff` surfaces it.

**How is this different from `brain-dump`?**
`brain-dump` writes Phase A as **global** skills in `~/.claude/skills/`.
`handoff` writes Phase A as **project-scoped** memories in
`~/.claude/projects/<slug>/memory/`. The latter auto-loads in the
project it was learned in and stays out of unrelated projects. Also,
`brain-dump` has no staleness tracking; `handoff`'s core contribution
is exactly that.

**Does it work for agents that don't auto-load skills directories?**
Yes. For agents where the skill format isn't natively consumed, point
the agent at the repo and ask it to read `SKILL.md` as part of its
context. The instructions inside are agent-agnostic — they describe a
workflow, not API calls.

**Does it work without git?**
Yes. Use `pin.type: file-hash` (recompute hashes) or
`pin.type: user-override-only` (only invalidated by user). Multiple
git-less projects can coexist. The `SKILL.md` explicitly documents
edge cases for monorepo, multi-repo, read-only mirror, detached
workspace, and no-git-allowed scenarios.

**How big can memories grow before they hurt context?**
`MEMORY.md` (the index) auto-truncates at ~200 lines for Claude Code,
a hard ceiling of ~200 memories per project. Individual files aren't
loaded unless recalled. In practice a well-curated project lives
under 20-30 memories.

**What happens if I move the project directory?**
For Claude Code the memory path encodes the absolute project path.
Moving the project orphans its memories — rename the memory dir to
match, or re-run `handoff` to regenerate. A future skill version could
detect this via a git-remote-origin fingerprint.

---

## Contributing

Issues and PRs welcome. The skill's design hits many orthogonal
concerns (git semantics, state machines, file formats, LLM reasoning
boundaries) so please prefix your issue with the dimension you're
attacking.

Before opening a PR that changes the schema, check:

- Does the change preserve backwards compatibility with v1 / v2
  memories? The skill has an explicit migration path in `SKILL.md`.
- Does it keep lazy verification as the default? Eager checking is
  fine as an opt-in flag but must not be forced.
- Does it keep the "no git writes" rule? The skill is usable inside
  any workflow exactly because it stays out of git state mutation.

---

## License

[MIT](./LICENSE) — use it, fork it, redistribute it. Attribution
appreciated (link back to this repo) but not required.
