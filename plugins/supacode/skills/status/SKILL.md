---
name: status
description: Dashboard of every Supacode worktree "lane" for the current repo - branch, PR state, session liveness, clean/dirty, and a verdict per lane saying what needs your attention. With --reap it deletes provably-finished lanes after strict safety checks; with --paint it tints each lane's sidebar entry by verdict, turning the Supacode sidebar itself into a live dashboard. Non-interactive and conservative by construction, so looping it is safe. Use when the user says "/supacode:status", "/supacode:status --reap", "/supacode:status --paint", "how are the lanes doing", "status of my worktrees", "any lanes to clean up", "reap merged worktrees", "color-code my worktrees", "paint the lanes", or sets up "/loop 15m /supacode:status --reap --paint". Formerly /supa-status - the old name still refers to this skill.
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

With `--paint` in `$ARGUMENTS`, additionally write each lane's verdict into
the Supacode sidebar (§6) so lane health is visible without running the skill.
Both flags compose: `--reap --paint` reaps first, then paints the survivors.

## 0. Repo identity

Derive `<repo-name>` from the **primary checkout**, never the cwd — see
`supacode-cli/references/worktree-identity.md` for the rule and why the cwd
basename is wrong inside a lane. Derive it once; reuse it throughout.

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

**Short-circuit first:** if the lane set is empty, scan for plan files
(local, free). No lanes *and* no plan files ⇒ report "no lanes" and stop
without touching the network — the common case for a looped
`/supacode:status` on an idle repo. If plan files do exist, continue: the
orphan scan below needs the PR list to tell a completed lane from a crashed
one, and reporting every orphan as "needs attention" would make a looped run
cry wolf.

Fetch PR state ONCE for the whole repo from the primary checkout, then join
locally by branch:

    gh pr list --state all --limit 100 \
      --json headRefName,number,state,mergedAt,headRefOid,url

If a branch has several PRs, prefer the open one; else the most recently
updated. Then gather per-lane facts — **issue every lane's commands in a
single message so they run concurrently**, and use these batched forms rather
than one call per fact (git via `-C <decoded-path>`; if you ever need `gh` in
a lane, `gh` has no `-C` — use a subshell `(cd <path> && gh …)`):

    git -C <wt> rev-parse --git-dir --git-common-dir HEAD   # 3 facts, 1 call
    git -C <wt> status --porcelain -b                       # branch + dirty
    git -C <wt> log -1 --format=%cr                         # commit age
    supacode tab list -w <id>                               # session

That is 4 calls per lane instead of 6, and the first two carry four facts
between them:

- **Linked-worktree proof:** `--git-dir` differs from `--git-common-dir`.
  Equal ⇒ it's a primary checkout that slipped through — drop it from the
  lane set entirely.
- **HEAD:** the third line, for the `headRefOid` comparison.
- **Branch and dirty:** `status --porcelain -b` prints `## <branch>...` as its
  first line, then one line per dirty path — no dirty lines ⇒ clean.
- **Detached HEAD:** the `##` line names no real branch (git renders this as
  `## HEAD (no branch)`, but treat any first line you cannot parse into a
  branch name as detached rather than trusting that exact string) ⇒ verdict
  `needs-attention`, never reapable. When it matters, confirm with
  `git -C <wt> branch --show-current` returning empty.
- **Session:** empty tab list ⇒ session dead. Non-empty ⇒ "possibly alive" —
  tabs are bare UUIDs and a tab can outlive its process, so tab-present is
  never proof of life, only grounds for caution.
- **Plan file:** match per `handoff-plan/references/plan-file-format.md`
  (`worktree:` first, then `branch:`, then legacy slug; `*.prompt.md`
  excluded). Note which key matched — §5 only tombstones exact `worktree:`
  matches. No match ⇒ show "(no plan file)".

Isolate errors per lane: if any command fails for one lane, give it verdict
`unknown` (under `needs-attention`) and keep rendering the table — one broken
lane must not abort the report.

**Orphan plan files:** a plan file whose `branch:`/slug has no live worktree.
Look its branch up in the repo-wide PR list: MERGED ⇒ report "completed";
otherwise report "orphaned — needs attention" (crashed launch or manually
removed worktree). Orphans are never reap candidates.

Backfill a tombstone **only** when the plan carries a `worktree:` key whose
lane is provably gone — i.e. the same exact-key rule as §5. An orphan matched
by branch/slug alone is reported, never written: a reused branch name would
stamp a new PR's number onto an unrelated old plan, which is the failure this
release exists to prevent. Report the backfill as skipped and why.

## 3. Verdicts

Keep the set small; each lane gets exactly one:

| Verdict | Condition |
|---|---|
| `working` | tabs present; PR absent or open |
| `pr-open` | PR open (tabs irrelevant) |
| `merged-reapable` | PR MERGED + clean + `HEAD == headRefOid` + **no tabs** |
| `merged-live` | PR MERGED + clean + `HEAD == headRefOid` + tabs present |
| `stalled` | **no tabs** + no open PR (PR absent or closed-unmerged) |
| `needs-attention` | anything contradictory: dirty + merged, detached HEAD, merged PR with `HEAD != headRefOid`, derivation errors (`unknown`) |

Resolving overlaps, in order:

1. **`needs-attention` wins over everything** — a contradictory lane is never
   reported as healthy, and never reaped.
2. `working` vs `pr-open` on "PR open + tabs present" ⇒ `pr-open`; the PR
   supersedes, the session is just waiting.

Notes on the conditions: `pr-open` deliberately has no finer
awaiting-review/awaiting-merge split — either way the next action is yours on
GitHub. `stalled` means abandoned mid-flight; commit age alone never triggers
it. Under `needs-attention`, `HEAD != headRefOid` only counts on a **merged**
PR (post-merge commits deletion would lose) — on an open PR, HEAD ahead of the
PR head is just unpushed work, which is normal.

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

Only lanes with verdict `merged-reapable` are candidates. For each, confirm
all five below. Checks **1, 3 and 5 must be re-run immediately before the
delete**: a dirty file or a reopened tab can appear in the seconds since the
scan, and check 1 is the guard whose failure destroys the primary repo — its
cost is one command, so never trade it for a cached value (a `git worktree
move`/`repair`, or the user reorganizing a checkout mid-run, is rare but
unrecoverable). Checks 2 and 4 may reuse §2's results: a merged PR's state
and head OID are settled.

1. Linked worktree: `git -C <wt> rev-parse --git-dir` ≠ `--git-common-dir`
   (defense-critical; never delete a primary checkout).
2. PR state is `MERGED`.
3. `git -C <wt> status --porcelain` is empty — re-run this right before the
   delete as a race guard.
4. `git -C <wt> rev-parse HEAD` equals the PR's `headRefOid` (squash-safe;
   post-merge commits would be silently lost).
5. `supacode tab list -w <id>` is empty — a lane with ANY tab is never deleted
   unattended, even fully merged (that's `merged-live`: report it instead).
   Treat this as necessary but **not sufficient**: a tab can outlive its
   process, so an empty list means "nobody is watching", not "no work is in
   flight". Checks 2–4 are what actually make deletion safe.

If all five hold, in this order:

1. **Tombstone the plan file** — per the write rule in
   `handoff-plan/references/plan-file-format.md`: an exact `worktree:` match,
   or a `branch:` match on a file that carries no `worktree:` key at all (the
   manual-worktree flow). Add/update `status: merged`, `pr: <number>`,
   `merged: <YYYY-MM-DD>`. Advisory only — git/gh remain the truth. A file
   whose `worktree:` names a different lane is **reported, never written**:
   branch names get reused, so a stale plan for an old `fix/auth` would
   otherwise be tombstoned by a new one.
2. **Delete:** `supacode worktree delete -w <id>` using the exact ID string
   from `worktree list`.
3. **Update stale memories:** if the memory system tracks this lane (an
   "in flight" note, a chunk-conventions file), update it to merged (PR #,
   date) per the memory system's own conventions. This stands in for the
   memory-maintenance half of /supacode:complete-feature, which external reaping
   bypasses.

Report per candidate: reaped, or skipped with the exact failed check. Any
failed check ⇒ skip, no prompt, no retry — the next run will see it again.

## 6. `--paint` — the sidebar as dashboard

For every lane in the final lane set (after any reaping), write its verdict
into the Supacode sidebar:

    supacode worktree appearance -w <id> --color <color> --title "<title>"

using the exact ID string from `worktree list`. Verdict → appearance:

| Verdict | `--color` | `--title` |
|---|---|---|
| `working` | `blue` | `<lane>` |
| `pr-open` | `purple` | `<lane> ⇧#<pr>` |
| `merged-live` | `teal` | `<lane> ✓#<pr>` |
| `merged-reapable` | `green` | `<lane> ✓#<pr>` |
| `stalled` | `orange` | `<lane> ⏸` |
| `needs-attention` (incl. `unknown`) | `red` | `<lane> ⚠` |

`<lane>` is the worktree folder name. Rules:

- **Repaint everything, every run** — appearance is a pure function of the
  current verdict, so stale colors self-heal and the flag is loop-safe
  (`/loop 15m /supacode:status --reap --paint`). Setting `--title` each time
  also overwrites any marker from a previous verdict.
- **Never paint anything outside the lane set** — not the primary checkout,
  not other repos' worktrees, not orphans (their worktree is gone anyway).
  The user's own titles/colors on non-lane worktrees are theirs.
- **Painting is cosmetic and non-fatal**: an `appearance` failure downgrades
  that lane to a report line ("paint failed: <error>") and never affects
  verdicts, reaping, or the exit report. Never let painting abort the scan.
- Reaped lanes need no cleanup — deletion removes the sidebar entry with the
  worktree.
- The sidebar shows last-run state, not live truth — the report remains the
  authoritative output; git/gh remain the source of verdicts.
