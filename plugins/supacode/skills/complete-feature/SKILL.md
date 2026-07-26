---
name: complete-feature
description: Close out a finished piece of work after its PR has been merged, removing its Supacode worktree and tab - verifies the PR is actually merged, that nothing uncommitted or unpushed would be lost, saves any durable session learnings to memory, prints a close-out summary, then deletes the current worktree if (and only if) every safety check passes. Use when the user says "/supacode:complete-feature", "the PR is merged, clean up", "close out this feature", or "wrap up and delete this worktree". Formerly /supa-complete-feature - the old name still refers to this skill.
---

# Complete Feature

Run from inside the executor session, in the worktree whose PR the user has
merged. Verify everything is truly done, preserve what matters, then remove the
worktree. The order below is load-bearing: deleting the worktree kills this
session's own tab, so it is always the LAST action, after the summary.

## 1. Safety checks — ALL must pass

Run these first; if ANY fails, report exactly what failed and STOP. Never
delete on a failed check — tell the user what to resolve instead (they can
`supacode worktree archive` manually if they want it out of the sidebar).

1. **This is a disposable linked worktree, not the primary checkout:**
   - `$SUPACODE_WORKTREE_ID` is set, and
   - `git rev-parse --git-dir` differs from `git rev-parse --git-common-dir`
     (equal means primary checkout — NEVER delete that).
2. **The PR is merged:** `gh pr view --json state,url,mergedAt,headRefOid` for
   the current branch reports state `MERGED`. No PR found, or state
   OPEN/CLOSED-unmerged → stop.
3. **Nothing exists beyond the merged PR:**
   - `git status --porcelain` is empty (no uncommitted or untracked files), and
   - `git rev-parse HEAD` equals the PR's `headRefOid` — commits made after the
     PR merged would be silently lost by deletion. (Comparing to the PR head,
     not to main, keeps this correct under squash merges.)

## 2. Save learnings to memory

Before deleting anything, harvest the session for durable knowledge, following
the memory system's own conventions (one fact per file, frontmatter, indexed in
MEMORY.md). (The handoff contract now has executors save learnings before
opening the PR, so this pass is a double-check — still do it: sessions learn
things after the PR opens too.)

- **New conventions or traps** discovered while implementing — the kind of
  thing the next session in this codebase would otherwise rediscover the hard
  way. Skip anything the repo itself now records (code, docs, git history).
- **Update stale memories:** if an existing memory tracks this work (e.g. an
  "in flight" note or a chunk-conventions file), update it to reflect the merge
  (PR number, merged status, date). Delete memories the merge made wrong.
- If the session produced nothing worth keeping, say so — don't invent a
  memory.

## 3. Tombstone the plan file

If a plan file for this lane exists, mark it merged. Find it by matching the
current branch against `branch:` frontmatter in `~/.claude/plans/<repo-name>/`
(repo name = basename of the primary checkout via
`git rev-parse --git-common-dir`, never this worktree's basename; exclude
`*.prompt.md`). Add/update frontmatter: `status: merged`, `pr: <number>`,
`merged: <YYYY-MM-DD>`. No matching file ⇒ skip silently. The tombstone is
advisory — git/gh remain the truth; /supacode:status uses it to tell a completed
lane from a crashed one after the worktree is gone.

## 4. Close-out summary — BEFORE deletion

Print the final summary now (this tab is about to disappear): branch, PR link,
what merged, memories written/updated, confirmation that all safety checks
passed, and that the worktree is about to be deleted.

## 5. Delete the worktree — last action

    supacode worktree delete

(Defaults to the current worktree via `$SUPACODE_WORKTREE_ID`.) Run it as the
very last command. Expect the session/tab to end with it; that is normal and
means it worked. The merged branch's local ref lives in the shared git dir and
can be cleaned up later from the primary checkout (e.g. a gone-branch sweep).
