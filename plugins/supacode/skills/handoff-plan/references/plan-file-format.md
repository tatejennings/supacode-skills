# Plan file format — written by handoff-plan, read by status and complete-feature

One spec, three consumers. If these drift apart, lanes stop matching their
plans and tombstones land on the wrong file — so change this file, not a copy.

## Location and naming

```
~/.claude/plans/<repo-name>/<YYYY-MM-DD>-<slug>.md          # the plan
~/.claude/plans/<repo-name>/<YYYY-MM-DD>-<slug>.prompt.md   # executor contract
```

- `<repo-name>` comes from the **primary checkout** — see
  `supacode-cli/references/worktree-identity.md`.
- The path is deliberately **outside the repo**: a git worktree cannot see
  another checkout's untracked files, but any absolute path is readable.
- Scans must **exclude `*.prompt.md`** — it is a sibling artifact, not a plan.

## Frontmatter

Written at save time:

```yaml
---
branch: <type>/<slug>        # feat|fix|chore|docs — the ONLY source of the type prefix
milestone: <name, or free-form>
created: <YYYY-MM-DD>
---
```

Added at launch, once the worktree exists:

```yaml
worktree: <exact URL-encoded ID from `supacode worktree list`>
```

Added at close-out (the tombstone — by `complete-feature` or `status --reap`):

```yaml
status: merged
pr: <number>
merged: <YYYY-MM-DD>
```

Frontmatter is **advisory**. Live truth always comes from git/gh/supacode; the
tombstone exists only so a completed lane can be told from a crashed one after
its worktree is gone.

## Matching a lane to its plan

In this order:

1. **`worktree:` against the lane's exact ID** — the strong key. Immune to
   branch-name reuse.
2. **`branch:` against the lane's branch** — fallback.
3. **Legacy** (no frontmatter): slug-in-filename ↔ branch suffix. Note as
   "legacy" when reporting.

**Only a match from key 1 may be written to.** Branch names get reused over
time, so tombstoning a branch-matched file can mark an unrelated old plan as
merged. Weaker matches are fine to *display*, never to modify.

## Collision guard

Before saving, if the target plan file already exists — or, when
`command -v supacode` succeeds, a live worktree of that name appears in
`supacode worktree list` — suffix the slug (`-2`, `-3`, …). Never overwrite an
existing plan or reuse a live worktree name.

Exception: a caller deliberately revising a prepped plan (a mission's deferred
lane) should **update that file in place** instead of creating a suffixed
twin, which would orphan the original's `.prompt.md`.
