# Example — Stealf: after shipping 5 PRs + threat model audit

_Real session, 2026-04-20, anonymized where necessary._

After a session of 8h+ on a dual-wallet neobank codebase (Solana, privacy
features via Arcium MPC), invoking `handoff` produced:

- **Phase A** — 8 memory files written to
  `~/.claude/projects/-home-user-Documents-Stealf/memory/` (+ `MEMORY.md`
  index). Covering: user role, repo topology, threat model, commit
  conventions, CodeRabbit rate limit behaviour, PR style, stack
  references, and a parallel-subagent git gotcha.
- **Phase B** — a 138-line `HANDOFF.md` at the project root documenting 6
  PRs in flight, 6 architectural decisions (append-only), 4 open
  questions for the team, and a prioritized TODO list.
- **Phase C** — **skipped**. The team's `CLAUDE.md` was already
  well-curated and nothing this session qualified as durable enough to
  merit a diff. This was the right call: respecting the team's
  file-ownership boundary is more important than adding one more "AI
  wrote this" comment.

## Sample memory file (Phase A output)

```yaml
---
name: Stealf dual-wallet threat model
description: The hard privacy rule of Stealf — backend must not be able to correlate a user identity with their stealth address.
type: project
volatility: low
state: fresh
created_at: 2026-04-20
last_verified_at: 2026-04-21
pin:
  type: git-commit
  repo: /home/user/Documents/Stealf/backend
  commit: 065353ad69b1621942abdc9a9f7272f96ecfda8d
  paths:
    - src/models/User.ts
    - src/models/FaucetClaim.ts
    - src/models/FaucetCooldown.ts
    - src/services/yield/constant.ts
anchors:
  - path: src/models/User.ts
    sha256: 185b042c4be8b7abb0ce132903b1c8468bde195bce58a30b76960452b7b429c1
    claim_excerpt: "User schema has no stealf_wallet field (privacy: deliberately not persisted)"
domain_tags: [threat-model, privacy, stealth-wallet, dual-wallet]
---

Stealf is a dual-wallet neobank. Cash wallet (KYC, Turnkey) is stored on
User. The `stealf_wallet` field is intentionally NOT persisted in the
User model — adding it would let a backend compromise correlate every
user with their stealth address, defeating the product's privacy pitch.

**Why:** ...
**How to apply:** ...
```

## Key property demonstrated

One day later, returning to the project, this memory was reverified:

```bash
git log 9d247494..065353a --oneline -- src/models/User.ts
# (one commit touching the file: the ownership PR that was merged)
```

Claude read the updated `User.ts` content, confirmed the `no stealf_wallet`
claim still held after the new PR, bumped `last_verified_at: 2026-04-21`
and set `pin.commit: 065353a`. The memory went `fresh → suspect → fresh`
via evidence, not via a calendar.
