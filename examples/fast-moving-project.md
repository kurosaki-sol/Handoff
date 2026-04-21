# Example — fast-moving project, 200 commits in 2 days

The opposite scenario of `dormant-project-return.md`. You stepped away
for 48 hours. In that window the team shipped a significant refactor:
200+ commits, including renaming some model fields and restructuring the
auth flow.

## What happens when you come back

### On session start

The memory system auto-loads `MEMORY.md` as usual. No verification yet —
verification is lazy.

### On first recall of an affected memory

You ask "what's the User model look like again?" Claude pulls the memory
`stealf-threat-model.md`. Before using it, Claude runs:

```bash
git log 065353a..HEAD -- src/models/User.ts
```

This time the output is non-empty — 12 commits touched `User.ts` in the
last 48h. The memory transitions `fresh → suspect`.

Claude re-reads `User.ts` on the current HEAD and compares to the
memory's claim. Two possible outcomes:

#### Outcome A — claim still holds

The refactor renamed fields but the key claim ("no `stealf_wallet` on
User") is preserved. Claude updates the memory:

```yaml
state: fresh
last_verified_at: 2026-04-23
pin:
  commit: <new HEAD>
```

Tells you "heads up, User.ts changed but the privacy property is
intact."

#### Outcome B — claim is now broken

The refactor accidentally added `stealf_wallet` as a User field for a
feature that's now pending review. Claude sees the contradiction,
transitions the memory to `contested`, and **refuses to act on the old
value**. You get:

```
The memory says `stealf_wallet` is deliberately NOT on the User model
(privacy threat model, 2026-04-20). But upstream/main now has a
`stealf_wallet` field added in commit 7f9a1b2. Should I archive the
memory or does this addition violate the threat model?
```

You answer. Memory either gets replaced (new decision documented in the
append-only log) or the team reverts the offending commit.

## The key property

In a fast-moving window, **lazy verification scales to exactly the
volume of claims you actually use**. Ten affected memories don't all get
re-verified eagerly; only the ones you recall pay the check cost.

Even better: Claude doesn't quietly use stale info. The contradiction
surfaces **before** it shapes your next action.

## Counter-example: what the pre-staleness design would have done

Under the old `brain-dump` design (no staleness schema), the memory
would still say "no `stealf_wallet`" on day 3. You'd ask "should I add
this privacy protection?", Claude would say "already done, it's a
design decision from last sprint, don't touch it", and you'd waste
hours confused about why the feature behaves differently from what the
memory claims.

Calendar TTL would not have saved you — 48 hours is well inside any
reasonable TTL window. Only **evidence-based** verification catches
this.
