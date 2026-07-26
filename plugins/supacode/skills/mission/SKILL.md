---
name: mission
description: Propose and launch a wave of parallel work lanes - discovers candidate milestones from the repo's roadmap (delegating to a project "what's next" skill when one exists), identifies collision risks, gets the user's approval of the wave, then launches each approved lane via supacode:plan-feature --auto in its own Supacode worktree. Monitoring and cleanup belong to /supacode:status. Usage - /supacode:mission [N]. Use when the user says "/supacode:mission", "launch a wave", "spin up parallel lanes", "what can we parallelize - go do it", or "orchestrate the next milestones". Formerly /supa-mission - the old name still refers to this skill.
---

# Mission: launch a wave of parallel lanes

Pick the next set of independently-workable milestones, get the user's
approval, and launch each one through the existing pipeline
(`supacode:plan-feature --auto` → `supacode:handoff-plan --launch`). You are the
dispatcher, not the monitor: once the wave is launched you report and stop —
`/supacode:status` (optionally wrapped in `/loop`) watches the lanes, and the user
merges every PR by hand.

Wave size: `$ARGUMENTS` may give N; default to proposing up to 3 lanes and
never launch more than ~3 in one mission — a bigger ask means two missions,
run one after the other.

## 1. Preflight — strictly read-only

- Require Supacode: `command -v supacode` and `$SUPACODE_REPO_ID`. Missing ⇒
  say so and stop (a mission IS launch machinery; there is no degraded mode —
  point the user at /supacode:plan-feature + /supacode:handoff-plan for manual lanes).
- Sync the base: from the **primary checkout**, `git fetch`, then fast-forward
  the default branch. `merge --ff-only @{u}` acts on whatever branch is
  checked out, so branch first: if `git -C <primary> branch --show-current`
  IS the default branch and the tree is clean, use
  `git -C <primary> merge --ff-only @{u}`; if the primary has a *different*
  branch checked out, use `git -C <primary> fetch origin <default>:<default>`
  (fast-forwards the ref without touching the working tree). Dirty tree or
  failed ff ⇒ skip with a warning. Never reset, never touch branches beyond
  the default.
- Do NOT invoke side-effecting project skills before the approval gate — a
  repo may have audit skills that commit/push (e.g. a roadmap-drift fixer).
  If one looks warranted, recommend it in the final report instead.

## 2. Discover candidates

- **Project skill first:** if the available-skills list has a project-level
  roadmap/"what's next" skill (names like `roadmap-status`, `whats-next`) that
  is read-only, invoke it and use its recommendations — including any
  parallelism guidance it emits.
- **Generic fallback:** parse the repo's planning surface — roadmap docs
  (`docs/*roadmap*`), status boards, TODO/issue lists. Candidates are chunks
  marked next/ready whose prerequisites are done.
- **Exclude in-flight work** from either path: branches of live lanes
  (`supacode worktree list` filtered to this repo — see /supacode:status §1 for
  the exact filter), open PRs (`gh pr list`), and anything the board marks
  in-flight.

## 3. Collision notes — shared files only

Identify repo-mandated files every PR must touch (status boards, changelogs,
checklist docs — the repo's CLAUDE.md usually names them). These WILL conflict
across parallel lanes; the countermeasure is mechanical and already in the
executor contract (rebase onto latest origin/main immediately before push,
re-resolve the shared file keeping only your lane's rows) — your job is just
to note the file(s) in each lane's handoff constraints.

Do NOT attempt fine-grained file-overlap prediction between candidate lanes
from roadmap prose — that is guesswork. Real overlap detection happens at
plan time: each lane's adversarial review runs a cross-lane axis against its
siblings' actual plans (`--siblings`) and disqualifies on real overlap.
Only skip pairing candidates that obviously own the same subsystem.

## 4. Approval gate — always

Present the proposed wave with AskUserQuestion (multiSelect): one option per
candidate lane — label = lane name, description = one-line scope + any
collision/ordering notes. Never launch anything the user did not select, and
never skip this gate regardless of how the mission was invoked.

## 5. Launch serially, one subagent per lane

For each approved lane, in sequence, spawn a **general-purpose subagent**
whose prompt tells it to invoke the `supacode:plan-feature` skill with:

    <lane> --auto --siblings <plan-file paths of the wave lanes already launched>

and to return ONLY a compact report: lane, branch, worktree ID, plan file
path, and `launched` — or `disqualified: <reason>` (the updated
supacode:plan-feature returns disqualifications instead of stopping when run in
mission context). The subagent boundary is load-bearing: each lane's research,
plan, and review stay out of the mission context, which is what lets a wave of
3 fit in one session.

- A lane that comes back `disqualified` is marked **deferred** with its
  reason; continue with the remaining lanes — never stall the wave. Deferral
  is a designed outcome, not a failure: a subagent that defers on a real open
  question did its job, and one that guesses to avoid deferring did not.
  Never re-run a deferred lane hoping for a different answer.
- Serial, not parallel: later lanes must receive earlier lanes' plan paths as
  siblings, and worktree creation fetches a fresh base each time.

## 6. Wave report

Report a table: lane · branch · worktree · plan file · status
(`launched` | `deferred: <reason>`). Close with:

- monitoring: `/supacode:status`, or `/loop 15m /supacode:status --reap` for
  hands-off watching + cleanup;
- the reminder that every PR is merged by the user on GitHub — nothing in
  this pipeline merges, ever.

## 7. Offer interactive planning for deferred lanes — then stop

If any lanes came back `deferred`, ask ONCE with AskUserQuestion
(multiSelect; one option per deferred lane, description = the deferral
reason) which of them to open an interactive planning session for. Skip the
question entirely when nothing was deferred. For each lane the user selects:

    supacode tab new --input 'claude --remote-control "plan-<lane>" "Run /supacode:plan-feature <lane>. A mission deferred this lane because: <reason>. Draft plan file, read it first if present: <path or none>."'

- The tab opens in the mission's own worktree (the primary checkout —
  `tab new` defaults to `$SUPACODE_WORKTREE_ID`), which is right: planning
  needs no worktree; /supacode:handoff-plan creates one at launch time.
- Keep the pointer prompt to that single line; strip quotes/newlines from the
  reason so it survives the nested shell quoting.
- Do NOT pass `--permission-mode auto` here — unlike executor launches, this
  session exists precisely so the user can answer its questions; plan mode
  gates writes, and the user will be present.

The new sessions ask their trade-off questions there; the user answers in
those tabs. Lanes the user declines stay deferred in the report (unblock
later with `/supacode:plan-feature <lane>`). Either way, the mission then
stops — do not monitor the launched sessions and do not keep the mission
session busy; its job ended at launch.
