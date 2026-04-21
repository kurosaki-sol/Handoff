# Example — returning to a dormant project 6 months later

You left a project to go work on something else. Six months pass. You
come back and start a new Claude Code session. What happens to your
memories?

## The memory files are still there

All 8 memories from your last handoff are in
`~/.claude/projects/<slug>/memory/`. `MEMORY.md` auto-loads into the new
session. Claude sees the index on boot, exactly as it did the day you
left.

## Nothing is marked stale automatically

The project hasn't moved. `git log <pin.commit>..HEAD -- <paths>` is
empty for every `git-commit`-pinned memory:

```bash
git log 065353a..HEAD -- src/models/User.ts src/models/FaucetClaim.ts
# (empty — no commits touched those paths)
```

The memories stay `fresh`. No calendar-driven invalidation because none
is appropriate: nothing in the code contradicts any claim.

## A naive calendar TTL would have been wrong

Contrast with the "expire memories after 90 days" strategy. Under that
rule, half the memories would now be suspect even though they are
verifiably still true. You'd spend the first 20 minutes of your session
re-confirming things that are literally unchanged — a waste.

## TTL-pinned memories are the exception

The one memory that **will** be flagged is this one:

```yaml
---
pin:
  type: ttl
  expires: 2026-10-21
  reason: "Reconfirm every 6 months in case Helius adds HMAC."
---
```

You're past the expiry. Claude marks it `suspect`, re-reads the current
Helius docs on first recall, and either refreshes
`last_verified_at`/`expires` or writes a replacement memory. This is
correct: the original pin said explicitly "external vendor behavior —
recheck".

## What you see in practice

You type `what were we in the middle of`. Claude reads `HANDOFF.md` (60s)
plus the memory index (free, already in context), tells you:

- PRs X and Y are open upstream
- These two decisions were made the day you left
- Three questions were still open for the team
- Next P0 was task Z

Zero catch-up. No hallucinated state. Any fact Claude relies on is
either evidence-fresh or explicitly flagged as needing re-verification.
