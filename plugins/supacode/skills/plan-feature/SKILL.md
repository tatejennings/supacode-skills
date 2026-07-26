---
name: plan-feature
description: Plan a piece of work end-to-end - a roadmap milestone/chunk OR a free-form feature/bug description. Enters plan mode, researches docs and codebase with parallel agents, drafts an execution-ready plan, then adversarially reviews it for completeness, holes, single-context feasibility, and blast radius. Usage - /supacode:plan-feature <milestone-or-description> [--auto] [--siblings <plan-file-paths>]. With --auto it skips the plan-approval gate and chains straight into /supacode:handoff-plan --launch; --siblings adds a cross-lane collision axis to the review for waves launched by /supacode:mission. Use when the user says "/supacode:plan-feature A4", "/supacode:plan-feature fix the vent double-tap bug", "plan milestone X", "plan this feature/fix", or "help me plan <chunk>". Formerly /supa-plan-feature - the old name still refers to this skill.
---

# Milestone / Feature Planning

Produce an execution-ready plan for the work in `$ARGUMENTS` — either a named
roadmap milestone/chunk (e.g. "A4") or a free-form description of a feature or
bug (e.g. "the vent doesn't re-fire on a repeat tap"). The plan will *likely*
(but not certainly) be handed to a fresh worktree context via
`/supacode:handoff-plan` afterwards — so it must be self-contained and use
handoff-compatible sections. You plan; you do not implement anything.

Scale everything to the work: a roadmap chunk gets the full treatment below; a
small bug or feature gets the same *structure* with lighter research and a
quicker review — never skip the review entirely.

## 0. Preconditions

- If `$ARGUMENTS` is empty, ask what to plan before doing anything else.
- Check `$ARGUMENTS` for `--auto`; the rest is the milestone name or the work
  description.
- Check `$ARGUMENTS` for `--siblings <paths>` — plan-file paths of lanes
  launching concurrently with this one (passed by /supacode:mission). Read those
  plans during research; they feed the cross-lane axis of the review (step 4)
  and its disqualifier (step 5).
- **Normal mode:** if not already in plan mode, call `EnterPlanMode` now (load
  its schema via ToolSearch if needed) and stay in plan mode for the entire
  skill.
- **`--auto` mode:** do NOT enter plan mode — its approval gate cannot be
  auto-approved, and the user has explicitly waived approval. Instead, hold the
  same discipline manually: research and plan only; the ONLY writes allowed are
  plan/prompt files under `~/.claude/plans/` and the supacode launch commands
  at the end. Never touch repo files.

## 1. Pin down the requirements

- **Named milestone:** find where it's defined — roadmap docs (`docs/*roadmap*`,
  status boards), design docs, in-repo TODO/issue lists. Quote the actual
  requirements in the plan's Context section; don't paraphrase from memory.
- **Free-form description:** the user's words ARE the requirement — restate them
  as concrete acceptance criteria (what behavior changes, how you'd observe it),
  and check the roadmap/design docs for anything that overlaps, conflicts, or
  already covers it. If a roadmap chunk already covers this work, say so and
  plan against that chunk's definition instead.
- Either way: check whether the work is already marked in-flight or partially
  done (status board, recent commits/PRs) before planning it from scratch.

## 2. Research with parallel agents

Launch research agents **in a single message** so they run concurrently. Give each
a specific question and tell it what to return (facts, file paths, constraints —
not file dumps):

- **Code agent** (Explore): the code the work touches — current architecture,
  the seams/types/tests it will build on, existing patterns to match. For a bug,
  also: the reproduction path and the likely defect site(s).
- **Docs agent** (Explore): everything the design docs say about this work —
  requirements, constraints, naming conventions, prior decisions, related
  milestones it depends on or feeds. Skip for small fixes with no design-doc
  surface.
- **Architecture agent** (Plan) — only when the approach is genuinely non-obvious:
  have it propose and compare implementation strategies.

For a small bug/feature, the code agent alone (or direct searching yourself) is
usually enough. Read the reports; follow up inline on anything load-bearing they
left vague.

## 3. Draft the plan

Use exactly these sections (they match what `/supacode:handoff-plan` packages):

- **Goal** — one paragraph; what "done" looks like.
- **Context** — milestone source (file + quoted requirements), relevant background.
- **Decisions already made** — every choice you or the user locked, with one-line
  rationale, so an executor doesn't re-litigate.
- **Steps** — ordered and concrete; name exact files/symbols; include any
  repo-mandated bookkeeping (e.g. roadmap/status-board updates required by the
  repo's CLAUDE.md). If the repo has a convention for marking work in-flight
  (a status board row, a tracker label), include that marking as an early
  step so parallel contexts see the lane is taken.
- **Verification** — commands, tests, sims, manual checks per step and overall.
- **Out of scope** — explicit non-goals, especially adjacent milestones.

Where a real trade-off needs the user's call, use AskUserQuestion **before**
finalizing; where a conventional default exists, decide it yourself and record it
under Decisions. In `--auto` mode, don't ask — and hold a high bar for deciding
alone: proceed on a fork only when one option is **overwhelmingly recommended**
— the codebase's own conventions, the requirements, and standard practice all
point the same way, and choosing wrong would be cheap to reverse. Record it
under Decisions with the rationale. Anything less — competing options with real
trade-offs, information you looked for and could not find, or a choice that
would be expensive to undo — is a fork you don't own: defer rather than guess
(see step 5). Deferring on honest uncertainty is a correct outcome of this
skill, not a failure; a plausible-but-wrong guess costs an entire executor run,
while a deferral costs one interactive question.

## 4. Adversarial review

Spawn a **fresh** general-purpose agent (not a fork — a cold reader, so it doesn't
inherit your drafting bias). Give it the full plan text plus pointers to the
requirements source (milestone definition or the stated acceptance criteria), and
have it attack on four axes:

1. **Completeness** — is every requirement covered by a step? Does every step
   have a verification?
2. **Holes & gaps** — unstated assumptions, missing edge cases, migrations or
   data-format changes glossed over, test debt, doc-update obligations skipped.
3. **Single-context feasibility** — can ONE context execute this end-to-end?
   Limits (one broken badly, or any two broken at once → split):
   - ≤ ~10 ordered steps in the plan
   - ≤ ~15 modified files (reads unlimited)
   - ≤ ~1,500 lines of new/changed code
   - at most ONE new subsystem or major architectural decision
   - repetitive same-shape edits: > ~25 by hand → script it or split it
   - expensive verify loops (> ~5 min per iterate-verify cycle): plan must
     need ≤ ~3 cycles; open-ended tuning always gets its own context
   - migrations of persisted formats count DOUBLE their file count
   The principle: limits measure decisions the executor must hold
   simultaneously, not raw file count. If it fails, the finding is "split
   it" — propose the seam(s) for sequential handoffs.
4. **Blast radius** — everything else the change touches: shared types, save/data
   formats, serialized content, test baselines, in-flight parallel work that
   could collide. Flag anything that makes the change hard to revert.
5. **Cross-lane collision** — only when `--siblings` was given: read the
   sibling plan files and flag any file, subsystem, or test baseline both this
   plan and a sibling will modify (repo-mandated shared files like status
   boards don't count — those are expected and handled mechanically at push
   time). Real overlap here is an auto-launch disqualifier, not a note.

Fold real findings back into the plan. Note findings you dismissed and why (these
become ammunition against re-litigation later).

## 5. Present — or auto-hand-off

**Normal mode:**

- Present the final plan via `ExitPlanMode` for approval.
- If the review recommended splitting, present the split as the plan: milestone →
  ordered sub-plans, each independently handoff-able.
- Close by noting the user can run `/supacode:handoff-plan <slug>` to package it for
  a fresh worktree context — but don't run it unasked; they may execute inline.

**`--auto` mode:** after folding in the review findings, invoke the
`supacode:handoff-plan` skill (via the Skill tool) with args `--launch <slug>` so it
saves the plan, creates the Supacode worktree, and starts the executor session.
Then report: the work planned, plan summary, review verdict, worktree/branch,
plan file path.

Auto-launch is DISQUALIFIED — stop and present to the user instead — when any of
these hold:

- the adversarial review says split (never auto-launch a multi-context chain);
- any fork lacked an overwhelming recommendation (step 3's bar) — never launch
  on a guess; state the open question(s) as the disqualification reason so an
  interactive session can pick up exactly there;
- the work appears already in flight elsewhere;
- the review's cross-lane axis found real overlap with a sibling plan
  (`--siblings`);
- the work overlaps a currently live lane — inside Supacode, check the
  branches of existing worktrees (`supacode worktree list`) before launching;
- not running inside Supacode (no `supacode` CLI / `$SUPACODE_REPO_ID`).

**Mission context:** when this skill runs inside a subagent spawned by
/supacode:mission, "stop and present to the user" means: RETURN the
disqualification reason (and the plan file path, if one was drafted) as the
subagent's result — deciding what to do with a deferred lane is the mission's
job, not yours. Never AskUserQuestion from inside a subagent.
