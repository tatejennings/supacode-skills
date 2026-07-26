---
name: status
description: Dashboard of every Supacode worktree "lane" for the current repo - branch, PR state, session liveness, clean/dirty, and a per-lane verdict (working, pr-open, merged-reapable, merged-live, stalled, needs-attention). With --reap it also deletes merged, clean, session-dead lanes after strict safety checks - non-interactive and conservative by construction, so wrapping it in a loop is safe. Use when the user says "/supacode:status", "/supacode:status --reap", "how are the lanes doing", "status of my worktrees", "any lanes to clean up", "reap merged worktrees", or sets up "/loop 15m /supacode:status --reap". Formerly /supa-status - the old name still refers to this skill.
---

# Lane Status (+ Reap)

Report the state of every parallel lane (Supacode worktree) for the current
repo, derived entirely from observable sources — `supacode worktree list`,
`git`, `gh`, and the plan files in `~/.claude/plans/`. Never trust stored
state over what git/gh say right now.

With `--reap` in `$ARGUMENTS`, additionally delete lanes that are provably
finished. Reap is **strictly non-interactive**: every ambiguous case is
reported, never deleted, and never prompts. That is what makes
`/loop 15m /supacode:status --reap` safe — deletion requires ALL checks to pass;
anything else degrades to a report line.

## 0. Repo identity

Derive `<repo-name>` from the **primary checkout**, never the cwd:

    git rev-parse --git-common-dir   # → <primary>/.git ; repo-name = basename of <primary>

Running from inside a linked worktree (e.g. `~/.supacode/repos/<repo>/<lane>/`)
must still yield the primary repo's name — the cwd basename there is the lane
name, which breaks every lookup below.

## 1. Enumerate lanes

`supacode worktree list` prints URL-encoded absolute paths — **these exact
strings are the worktree IDs.** URL-decode a copy of each line for matching,
but always pass the original, unmodified string to `-w`; never construct an ID
by URL-encoding a path yourself.

A line is a lane of this repo only if its decoded path starts with
`~/.supacode/repos/<repo-name>/` — **including the trailing slash after the
repo name**, otherwise `<repo-name>` prefix-matches sibling repos whose names
are superstrings (e.g. `myrepo` vs `myrepo-website`). Note the list also
contains primary checkouts of every repo; the filter plus the linked-worktree
check below excludes them.

Known limitation: this filter only sees worktrees in the default location — a
lane created with `worktree-new --location` elsewhere is invisible to this
dashboard (and to /supacode:mission's in-flight exclusion). Don't use
`--location` for lanes this pipeline should track.

## 2. Derive per-lane facts

Fetch PR state ONCE for the whole repo from the primary checkout, then join
locally by branch:

    gh pr list --state all --limit 100 \
      --json headRefName,number,state,mergedAt,headRefOid,url

If a branch has several PRs, prefer the open one; else the most recently
updated. Then per lane (git via `-C <decoded-path>`; if you ever need `gh` in
a lane, `gh` has no `-C` — use a subshell `(cd <path> && gh …)`):

- **Linked-worktree proof:** `git -C <wt> rev-parse --git-dir` differs from
  `git -C <wt> rev-parse --git-common-dir`. Equal ⇒ it's a primary checkout
  that slipped through — drop it from the lane set entirely.
- **Branch:** `git -C <wt> branch --show-current`. Empty ⇒ detached HEAD ⇒
  verdict `needs-attention`, never reapable.
- **Dirty:** `git -C <wt> status --porcelain` non-empty.
- **Last commit age:** `git -C <wt> log -1 --format=%cr`.
- **Session:** `supacode tab list -w <id>`. Empty ⇒ session dead. Non-empty ⇒
  "possibly alive" — tabs are bare UUIDs and a tab can outlive its process, so
  tab-present is never proof of life, only grounds for caution.
- **Plan file:** scan `~/.claude/plans/<repo-name>/*.md`, **excluding
  `*.prompt.md`**. Match YAML frontmatter `worktree:` against the lane's exact
  ID first (written at launch by /supacode:handoff-plan; immune to branch-name
  reuse across time), falling back to `branch:` against the lane branch;
  legacy files without frontmatter match by slug-in-filename ↔ branch suffix
  (note them as "legacy"). No match ⇒ show "(no plan file)".

Isolate errors per lane: if any command fails for one lane, give it verdict
`unknown` (under `needs-attention`) and keep rendering the table — one broken
lane must not abort the report.

**Orphan plan files:** a plan file whose `branch:`/slug has no live worktree.
Look its branch up in the repo-wide PR list: MERGED ⇒ report "completed" and
backfill the tombstone (see §5 frontmatter write) if absent; otherwise report
"orphaned — needs attention" (crashed launch or manually removed worktree).
Orphans are never reap candidates.

## 3. Verdicts

Keep the set small; each lane gets exactly one:

| Verdict | Condition |
|---|---|
| `working` | tabs present; PR absent or open |
| `pr-open` | PR open, session dead-or-alive irrelevant to the user's next action (no finer awaiting-review/awaiting-merge split) — the next action is theirs: review/merge on GitHub |
| `merged-reapable` | PR MERGED + clean + `HEAD == headRefOid` + **no tabs** |
| `merged-live` | PR MERGED + checks pass but tabs still present |
| `stalled` | **no tabs** + no open PR (PR absent or closed-unmerged — abandoned mid-flight; an open PR makes it `pr-open` instead, and commit age alone never triggers this) |
| `needs-attention` | anything contradictory: dirty + merged, detached HEAD, merged PR with `HEAD != headRefOid` (post-merge commits that deletion would lose; on an open PR, HEAD ahead of the PR head is just unpushed work — normal, not a flag), derivation errors (`unknown`) |

`working` and `pr-open` overlap on "PR open + tabs present" — that is
`pr-open` (the PR supersedes; the session is just waiting).

## 4. Output

One compact table: lane · milestone (from plan file) · branch · PR (#/state) ·
session · last commit · verdict. Then one line per **non-working** lane saying
the next action, e.g.:

- `merged-live` → "merged — close its tab or run /supacode:complete-feature there"
- `merged-reapable` → "run /supacode:status --reap to clean it up" (or reap now if
  `--reap` was passed)
- `stalled` → "session gone with unmerged work — reopen a session there or
  archive it"
- orphaned plan file → what it was and that its worktree is gone

## 5. `--reap` — delete provably-finished lanes

Only lanes with verdict `merged-reapable` are candidates. For each, re-verify
ALL of these immediately before deletion (yes, again — the world moves between
scan and delete):

1. Linked worktree: `git -C <wt> rev-parse --git-dir` ≠ `--git-common-dir`
   (defense-critical; never delete a primary checkout).
2. PR state is `MERGED`.
3. `git -C <wt> status --porcelain` is empty — re-run this right before the
   delete as a race guard.
4. `git -C <wt> rev-parse HEAD` equals the PR's `headRefOid` (squash-safe;
   post-merge commits would be silently lost).
5. `supacode tab list -w <id>` is empty — a lane with ANY tab is never deleted
   unattended, even fully merged (that's `merged-live`: report it instead).

If all five hold, in this order:

1. **Tombstone the plan file** (if matched): add/update frontmatter
   `status: merged`, `pr: <number>`, `merged: <YYYY-MM-DD>`. Advisory only —
   git/gh remain the truth.
2. **Delete:** `supacode worktree delete -w <id>` using the exact ID string
   from `worktree list`.
3. **Update stale memories:** if the memory system tracks this lane (an
   "in flight" note, a chunk-conventions file), update it to merged (PR #,
   date) per the memory system's own conventions. This stands in for the
   memory-maintenance half of /supacode:complete-feature, which external reaping
   bypasses.

Report per candidate: reaped, or skipped with the exact failed check. Any
failed check ⇒ skip, no prompt, no retry — the next run will see it again.
