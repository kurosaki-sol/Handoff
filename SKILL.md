---
name: handoff
description: Three-phase session handoff for preserving reusable learnings, active project state, and durable project facts. Use when context is nearly full, the user is about to clear or close the session, says "handoff", "brain-dump", "save state", "bilan", or similar, or a meaningful work unit has just finished. Writes evidence-aware memories and HANDOFF.md without committing to git.
license: MIT
compatibility:
  - claude-code
  - codex-cli
  - opencode
  - vercel-agent
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
metadata:
  version: 2.0.0
  repository: https://github.com/kurosaki-sol/Handoff
  specification: https://agentskills.io/specification
---

# Handoff — Session End Persistence

## Problem

Long Claude Code sessions accumulate three kinds of value that die on `/clear`:

1. **Reusable learnings** — non-obvious debugging techniques, workarounds, gotchas that help across *any* future project.
2. **Project state** — branches in flight, open PRs, blockers, TODOs. Painful to reconstruct from git log.
3. **Durable project facts** — architecture decisions, threat model clarifications, team preferences the codebase can't express.

On top of that, memories **rot**: a memory written 6 months ago may describe a reality that no longer exists. The worst failure isn't a long prompt; it's Claude confidently using stale info as if it were current. This skill tracks enough metadata per memory that future sessions can distinguish `fresh` from `suspect` from `stale` **on evidence**, not on calendar age.

## Trigger conditions

Invoke when ANY of these is true:

- Context window at ~85%+ and the session has produced meaningful work
- User types `handoff`, `brain-dump`, `save state`, `bilan`, `bilan complet`, `handoff complet`
- User mentions they're about to `/clear`, close the terminal, or step away
- A coherent work unit just finished (PR batch shipped, audit complete, feature merged)
- Proactively, at natural transition points in very long sessions (e.g. after 3+ hours)

Don't invoke for trivial sessions, or mid-task when the user just needs context compression.

## Workflow

### 0. Preamble — identify the scope

Before writing anything:

- **Working directory + git state** — `pwd`, `git rev-parse --show-toplevel 2>/dev/null` (no git → non-repo handling applies), `git rev-parse HEAD 2>/dev/null` for each relevant repo.
- **Existing CLAUDE.md** — read if present.
- **Existing HANDOFF.md** — read; augment, don't overwrite.
- **Existing memory dir** — `~/.claude/projects/<slug>/memory/`; read `MEMORY.md` index.
- **Session summary** — what was done, decided, shipped, blocked.

Decide which phases apply: Phase A if non-obvious learnings exist; Phase B always if in-flight work; Phase C only if a team-owned CLAUDE.md can reasonably absorb a durable insight (never in a fork you'll PR against).

---

## Phase A — Extract reusable learnings to persistent memory

For each candidate learning, pass the quality gates:

- [ ] **Reusable** across future sessions — not a one-off bug fix
- [ ] **Non-obvious** — required discovery, not a docs lookup
- [ ] **Specific** — has concrete triggers, symptoms, file paths
- [ ] **Verified** — confirmed during the session
- [ ] **Not already in memory** — check `MEMORY.md` first; update in place if it exists

Files go in `~/.claude/projects/<project-slug>/memory/` (auto-loaded every future session in that directory). Two steps per entry: write the file, add an index pointer to `MEMORY.md`.

### A.1 — The memory schema (v2)

```yaml
---
name: {short title — what it is}
description: {one-line — specific enough to judge relevance from index alone}
type: feedback | project | reference | user
volatility: none | low | code-anchored | high
state: fresh
created_at: YYYY-MM-DD
last_verified_at: YYYY-MM-DD
pin:
  type: git-commit | git-path | git-multi | file-hash | ttl | user-override-only | semantic
  {type-specific fields — see A.2}
anchors:
  - path: src/models/FaucetClaim.ts
    sha256: <hash of file content at write-time>
    claim_excerpt: "FaucetClaim has no userId field on stealth path"
domain_tags: [faucet, privacy, mongoose-schema]
supersedes: <prior-memory-name if this replaces one>
---

{body — the actual learning, lead with the rule, then **Why:**, then **How to apply:**}
```

Only `name`, `description`, `type`, `volatility`, `state`, `created_at` are required. The rest are added when applicable.

### A.2 — Choosing a `pin.type`

The pin defines the **evidence** a future session can use to check whether the memory is still valid.

- **`git-commit`** — memory depends on code in one git repo. Store `commit: <HEAD-sha>` and `paths: [file1, file2]`. Future sessions run `git log <commit>..HEAD -- <paths>` → 0 commits = fresh; N commits = suspect, read the files to resolve.
- **`git-path`** — memory depends on a subdir of a monorepo. `root`, `path`, `commit`. Scope drift detection to that subdir.
- **`git-multi`** — memory spans multiple repos (like Stealf: backend + frontend). `repos: [{root, commit, paths}, ...]`. Stale if ANY repo drifts on its paths.
- **`file-hash`** — no git, or git forbidden, or memory lives in a non-repo dir. Store `files: [{path, sha256}]`. Future sessions hash the files and compare.
- **`ttl`** — memory describes an external fact with a known expiry (API sunset date, roadmap milestone). `expires: YYYY-MM-DD`. After expiry, memory becomes `suspect` automatically.
- **`user-override-only`** — memory about the user / collaboration / cross-project norms. Never invalidated by code drift. Only the user can kill it.
- **`semantic`** — general cross-project knowledge with no anchor (e.g. "in general, X API is idempotent"). Treated as trust-on-first-use: the first contradiction invalidates it.

### A.3 — Choosing `volatility`

Orthogonal to `pin.type` — describes how fast the memory decays in practice:

- **`none`** — meta-memories about the user / collaboration. Never auto-verified.
- **`low`** — durable project facts: threat model, repo topology, conventions. Re-verify on demand; default to fresh.
- **`code-anchored`** — depends on specific code state. Re-verify whenever recalled if `pin` says drift.
- **`high`** — ephemeral / roadmap state. Surface as suspect after short TTL or first contradiction.

Use `user` / `project` / `reference` / `feedback` (the `type` field) to drive **prioritization in the MEMORY.md index**; use `volatility` to drive **verification behavior**.

### A.4 — `state` lifecycle

A memory moves through states as evidence accumulates:

```
fresh ── recall + evidence ok ──> fresh (bump last_verified_at)
fresh ── recall + evidence drift ──> suspect
fresh ── in-session contradiction ──> contested
suspect ── re-read, claim still holds ──> fresh
suspect ── re-read, claim broken ──> stale
contested ── user confirms memory ──> fresh
contested ── user confirms new reality ──> stale (write replacement, set `supersedes`)
stale ──> archived/<file>.md (kept for audit, never loaded)
```

**No time-based transitions.** A memory goes `fresh → suspect` only when evidence says so.

**Resolution rules**:
- Code beats memory for code-anchored claims. If `pin` evidence contradicts the memory, the code wins.
- User beats memory for intent/preference claims. If the user says "we don't use X", memories mentioning X go to `contested`.
- Memory beats nothing — it's always the thing being challenged.

### A.5 — Quality gates per type

Don't extract:

- Things derivable from reading the current code
- One-off bug fixes with no reusable pattern
- Stuff already in the project's CLAUDE.md
- Ephemeral state (goes in Phase B, not memory)

Do extract:

- **`feedback`** (rules the user gave about collaboration): "prefer small PRs", "no Co-Authored-By"
- **`project`** (facts not in code): threat model decisions, deadline constraints, "we don't log IPs"
- **`reference`** (pointers to external systems): Linear board, deck URL, dashboard URLs
- **`user`** (the user's role, preferences, expertise)

### A.6 — Reading memories in future sessions (how they should be used)

The skill `handoff` writes; but the session that reads must do so safely. The rules I (Claude) follow when a memory is loaded:

1. **Filter by relevance** — only pull memories the current turn plausibly needs. Don't verify unused ones.
2. **For each recalled memory, check its pin**:
   - `user-override-only` → use as `fresh`, no check
   - `git-*` → run `git log <commit>..HEAD -- <paths>`. Zero commits on paths = fresh. Commits exist = suspect.
   - `file-hash` → recompute hashes. Match = fresh. Mismatch = suspect.
   - `ttl` → compare `expires` to today. Expired = suspect.
   - `semantic` → load as fresh; watch for contradictions in the response I'm forming.
3. **If evidence contradicts the memory** in-session (explicit user correction, grep shows the claimed file lacks the claimed symbol, etc.) → mark `contested` or `stale`, surface the conflict **before** acting on the old value.
4. **Communicate confidence**: cite the memory inline when it's load-bearing; flag `suspect` explicitly; refuse to act on `contested` without resolving.
5. **Absence of signal ≠ freshness**: if a memory has no anchors and no user-override-only flag, treat it as `unknown` — use with a flag, don't assume fresh.

Hard overrides always win over weighted signals: a user correction or a grep-contradicted claim invalidates regardless of other freshness indicators.

### A.7 — Non-git and edge cases

- **No git in working dir**: use `pin.type: file-hash` for anything referencing files; `user-override-only` for meta memories; `ttl` for external facts.
- **User forbids git reads**: record this in a memory (`type: feedback`, `volatility: none`, describing the policy), then use only `file-hash` + `ttl` + `user-override-only` for that project. Never try `git log` again.
- **Multi-repo project (e.g. Stealf backend + frontend)**: use `pin.type: git-multi` with per-repo commit pins; stale if any drifts.
- **Read-only mirror**: pin to upstream ref (URL + ref), not local HEAD. Use `user-override-only` or manual re-fetch.
- **Detached workspace / container**: use `file-hash` with optional `host` fingerprint. If host changes, flag suspect.
- **Fresh repo with no commits**: start with `file-hash`; upgrade to `git-commit` once there's a first commit.
- **Throwaway / squashed branches**: pin to `git merge-base origin/main HEAD`, not local HEAD.
- **Social / person facts**: use `ttl` with a short expiry (e.g. 90 days) and expect to reconfirm.

When no evidence signal is available for a claim, the memory should be written with `pin.type: user-override-only` AND a note in the body that the claim is unverifiable locally — Claude on read should flag it at first use, not assume it's still true.

### A.8 — MEMORY.md index

One line per memory file. Format: `- [Title](file.md) — one-line hook`. Keep under 150 lines (system truncates after 200). No frontmatter.

---

## Phase B — Persist active work state to HANDOFF.md

Single file at the project root (or the working dir the session started in): `<project-root>/HANDOFF.md`. Goal: a cold reader resumes in under 60 seconds.

**Template**:

```markdown
# Handoff — {project name}

_Last updated: {ISO date}_

> {one-paragraph project context, especially any privacy / threat / product constraint that shapes decisions}

## Status at a glance

- **Working directories:** ...
- **Active branch(es):** {list with short purpose}
- **Open PRs:** {count + one-liner each}
- **Current phase:** {what's in the middle, 1-2 lines}
- **Blocked on:** {external dependencies, decisions waiting}

## PRs in flight

| # | Branch | Repo | Commit | Purpose | Notes |
|---|--------|------|--------|---------|-------|
| 1 | ... | ... | `hash` | 1-line | e.g. "requires DB drop before deploy" |

## What was done this session

Bullet list, terse, not a narrative.

## Decisions made

Append-only. Each entry: date, decision, rationale, alternatives dismissed.

- **{date} — {decision}**: {rationale}. _Alternatives considered:_ ...

## Open questions / to validate with team

- {Question — who can answer — why it matters}

## TODOs (prioritized)

### P0 (next up)
### P1
### P2
### Won't do (with reason)

## Known risks / deploy notes

Migration steps, ordering constraints, feature flag states, DB drops.

## File map (only if useful)

Key files + purpose. Skip if `git log` covers it.

## Resume script for next session

1. Read this file top to bottom
2. `git log --oneline main..origin/main` on each repo
3. Check upstream PR statuses
4. Confirm next P0 with user
```

**Rules for Phase B**:

- **Merge, don't overwrite.** Preserve "Decisions made" entirely (append-only). Update "Status at a glance", "PRs in flight", TODOs — those are snapshots.
- **Be specific.** `src/controllers/YieldController.ts:57`, commit hashes, branch names, dates — not paraphrases.
- **No git operations.** Just write the file.
- **Keep under 500 lines.** Split into `/docs/` files only when genuinely needed.

---

## Phase C — Optional: update the project's CLAUDE.md

Only if BOTH are true:

- A `CLAUDE.md` already exists in the project
- This session produced a **durable, high-signal** insight worth adding

**What qualifies**: new architecture pattern validated by the team; threat model clarification; adopted convention that would help future sessions; updated "current state" / "last updated" fields if that pattern exists.

**What doesn't**: session-specific TODOs (go in HANDOFF.md); bug fix details (git log has them); things likely to change within a week.

**Rules for Phase C**:

- **Preserve everything else.** Merge, don't overwrite.
- **Match existing style.** French → French. Terse bullets → don't write paragraphs.
- **No diff for its own sake.** If nothing qualifies, skip the phase and say so in the report.
- **Never touch CLAUDE.md in a fork** if the upstream owns it — it leaks into PR diffs.

---

## Verification

Before reporting done:

1. Phase A: every new memory file has a matching `MEMORY.md` entry; required frontmatter (`name`, `description`, `type`, `volatility`, `state`, `created_at`) is present; `pin.type` is set correctly for the memory's scenario; no orphans.
2. Phase B: HANDOFF.md exists, is scannable, contains concrete references (file:line, commit hashes, branch names), not hand-wavy language.
3. Phase C: if CLAUDE.md was modified, the diff is <20 lines and existing content preserved verbatim.
4. No git state was changed. No `git add`, `commit`, `push`, `reset`, `checkout` by this skill.

## Report to the user

- **Phase A (memory):** list new memory files with `volatility` + `pin.type` per entry. If nothing qualified: "no durable learnings this session."
- **Phase B (HANDOFF.md):** path + word count + section headers. If augmented: what was preserved.
- **Phase C (CLAUDE.md):** changed lines with location, or "skipped — no durable insight."
- **Migration note** if legacy memories (pre-v2 schema) were touched: which ones, whether metadata was added.
- Confirm: safe to `/clear`. Next session resume path: memory auto-loads → read HANDOFF.md → `git status` → confirm P0 with user.

## Migrating legacy memories (v1 → v2)

Memories written with v1 of this skill lack the staleness schema (`volatility`, `state`, `pin`, `anchors`, `created_at`). When running `handoff` on a project with legacy memories:

1. **Don't force-migrate every memory.** Touch only those you're about to update or cite.
2. **When migrating**: add `volatility` based on content (user-preference → `none`; threat model → `low`; code-anchored → `code-anchored`; roadmap → `high`), set `state: fresh`, set `created_at` to the file mtime, set `pin.type: user-override-only` as a safe default (forces flag-on-first-use rather than assuming git-trackable).
3. **Flag the migration** in the handoff report so the user can later choose to upgrade `pin.type` for memories that would benefit from code anchoring.

## Rules (hard constraints)

- Never delete existing useful content from CLAUDE.md, HANDOFF.md, or memory. Merge.
- "Decisions made" entries are append-only.
- Never write memory entries that duplicate existing ones. Update in place.
- Never invoke git commands that change state (`add`, `commit`, `push`, `reset`, `checkout`).
- Specific > vague. File paths with line numbers. Commit hashes. Branch names.
- Write for a reader with zero context. No in-jokes, no unexplained acronyms.
- Idempotent: running `handoff` twice in a row should produce the same files (except `Last updated`).
- **Staleness is evidence-based, never time-based.** No auto-expiration on calendar age alone.

## Notes

- **Phase A** memory is global to the project directory (auto-loaded via the memory system).
- **Phase B** HANDOFF.md is local to the project — commit or gitignore per user preference.
- **Phase C** CLAUDE.md is team-facing — be conservative, only append durable insights.
- **Staleness check** is lazy — it happens when memories are recalled in a future session, not at write-time. This skill just makes sure the evidence is there for the future check to work.
- If the user ever says "purge memory for X" or "we don't do Y anymore", treat those as hard overrides: mark every memory with matching `domain_tags` as `contested` immediately, don't wait for lazy recall.
