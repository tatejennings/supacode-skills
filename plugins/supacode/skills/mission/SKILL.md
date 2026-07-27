---
name: mission
description: Work several milestones at once - each gets its own Supacode worktree and its own Claude session implementing it concurrently, while your session stays free. Discovers candidate milestones from the repo's roadmap (delegating to a project "what's next" skill when one exists), gets the user's approval of the wave, plans all approved lanes concurrently with parallel subagents, reviews the finished plans against each other for collisions, then launches each surviving lane in its own Supacode worktree. Monitoring and cleanup belong to /supacode:status. Usage - /supacode:mission [N]. Use when the user says "/supacode:mission", "launch a wave", "spin up parallel lanes", "what can we parallelize - go do it", or "orchestrate the next milestones". Formerly /supa-mission - the old name still refers to this skill.
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
from roadmap prose — that is guesswork. Real overlap detection happens after
planning: step 5's Phase B reviews the lanes' *actual finished plans* against
each other and defers on real overlap. Only skip pairing candidates that
obviously own the same subsystem.

## 4. Approval gate — always

Present the proposed wave with AskUserQuestion (multiSelect): one option per
candidate lane — label = lane name, description = one-line scope + any
collision/ordering notes. Never launch anything the user did not select, and
never skip this gate regardless of how the mission was invoked.

## 5. Plan in parallel, review collisions, launch survivors

**Phase A — plan all lanes concurrently.** Spawn one **general-purpose
subagent** per approved lane, ALL IN ONE MESSAGE so they run in parallel.
Each subagent's prompt tells it to invoke the `supacode:plan-feature` skill
with:

    <lane> --auto --plan-only

and to return ONLY a compact report: lane, proposed branch, plan file path,
and `ready` — or `disqualified: <reason>`. The subagent boundary is
load-bearing: each lane's research, plan, and review stay out of the mission
context, which is what lets a wave fit in one session.

- A lane that comes back `disqualified` is marked **deferred** with its
  reason; the wave continues with the rest. Deferral is a designed outcome,
  not a failure: a subagent that defers on a real open question did its job,
  and one that guesses to avoid deferring did not. Never re-run a deferred
  lane hoping for a different answer.

**Phase B — cross-lane collision review.** Once ALL subagents have returned
(and only if two or more lanes are `ready`), spawn ONE fresh reviewer agent
with the plan file paths of every ready lane. It applies plan-feature's
cross-lane axis over the *actual plans*: flag any file, subsystem, or test
baseline that two plans both modify (repo-mandated shared files like status
boards don't count — the executor contract handles those mechanically at
push time). The reviewer **reports collision groups; it does not pick
winners** — it has only the plan files, not the wave's ordering. You then
resolve each group yourself: keep the lane that came first in the order you
presented the options in §4 (a multiSelect answer is a set, not a ranking, so
presentation order is the only stable tiebreak) and defer the rest with
reason "overlaps <lane> on <what>".

**Phase C — launch the survivors.** For each remaining lane, in §4
presentation order, invoke the `supacode:handoff-plan` skill with:

    --launch <plan file path>

Launching stays serial on purpose: each `worktree-new --fetch` picks up the
freshest base, and launches are cheap — the parallelism that matters already
happened in Phase A.

If a launch does not produce a running session — handoff-plan falls back to
printing a copy-paste prompt when Supacode is unavailable, and inside a
subagent nobody would ever see that fence — mark the lane
`deferred: planned but not launched (<reason>)` and give its plan file path
in the report. A silently unlaunched lane is the one failure this wave must
never hide; §1's preflight makes it unlikely, not impossible.

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

    supacode tab new --input 'claude --remote-control "plan-<lane>" "Run /supacode:plan-feature <lane>. A mission deferred this lane because: <reason>. A draft plan already exists at <path or none> — if present, read it and UPDATE THAT FILE IN PLACE rather than drafting a new one."'

- The tab opens in the mission's own worktree (the primary checkout —
  `tab new` defaults to `$SUPACODE_WORKTREE_ID`), which is right: planning
  needs no worktree; /supacode:handoff-plan creates one at launch time.
- Keep the pointer prompt to that single line; strip quotes/newlines from the
  reason so it survives the nested shell quoting.
- The update-in-place instruction matters: a lane deferred *after* drafting
  already has a saved plan and `.prompt.md` pair. Drafting fresh would trip
  handoff-plan's collision guard into a `-2` slug, leaving two plans for one
  lane and a stale orphaned prompt file that /supacode:status would match.
- Do NOT pass `--permission-mode auto` here — unlike executor launches, this
  session exists precisely so the user can answer its questions; plan mode
  gates writes, and the user will be present.

The new sessions ask their trade-off questions there; the user answers in
those tabs. Lanes the user declines stay deferred in the report (unblock
later with `/supacode:plan-feature <lane>`). Either way, the mission then
stops — do not monitor the launched sessions and do not keep the mission
session busy; its job ended at launch.
