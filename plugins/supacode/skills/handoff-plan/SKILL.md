---
name: handoff-plan
description: Hand the plan developed in this conversation off to a fresh Claude session in its own Supacode worktree, so implementation happens in a separate lane instead of this session. Saves the plan to a file and produces an executor contract - implement on a new branch, commit as it goes, self-review, open a PR (never merging it), report back. With --launch it creates the worktree and starts the session automatically with Remote Control enabled; --prep saves the files and launches nothing. Use when the user says "/supacode:handoff-plan", "hand this off", "run this in a new worktree", "spin up a session to build this", "write a handoff prompt", or "package this plan for another context". Use when the user says "/supacode:handoff-plan", "hand this off", "write a handoff prompt", or "package this plan for another context". Formerly /supa-handoff-plan - the old name still refers to this skill.
---

# Plan Handoff

You are the **planning** context. Your job is to package the plan — not execute it.
Do not start implementing anything here.

## Steps

### 1. Collect the plan

The plan is whatever was agreed in this conversation (including an approved
plan-mode plan). If no concrete plan exists yet, say so and stop — never invent one.

If `$ARGUMENTS` is given, treat it as the slug and/or extra instructions to fold
into the handoff. Mode flags:

- `--launch` — follow the **Launch mode** section at the end instead of
  stopping after printing the prompt. If `--launch` is followed by an
  absolute path to an *existing* plan file (a lane prepped earlier), skip
  steps 1–2 and launch from that file.
- `--prep` — follow the **Prep mode** section: save the files, launch
  nothing.

### 2. Save the plan to a file

Write the full plan to
`~/.claude/plans/<repo-name>/<YYYY-MM-DD>-<slug>.md`, with the frontmatter,
collision guard, and naming rules in
**`references/plan-file-format.md`** — read it before writing. `<slug>` is a
short kebab-case name for the work; the reference covers everything else
(including why the path lives outside the repo, and why the worktree half of
the collision guard is conditional on Supacode being installed).

Plan file sections (skip any with no real content):

- **Goal** — one paragraph; what "done" looks like.
- **Context** — repo, relevant background, why now.
- **Decisions already made** — choices locked during planning, with one-line
  rationale each, so the executor doesn't re-litigate them.
- **Steps** — ordered and concrete; name exact files/symbols where known.
- **Verification** — commands, tests, manual checks that prove it works.
- **Out of scope** — explicit non-goals.

### 3. Print the handoff prompt

Output the prompt in ONE fenced code block (nothing else inside the fence) so it
can be copied in a single gesture. Fill in the template:

```
Execute the plan at <absolute plan file path>. Read the whole file before doing anything.

**Summary:** <2–4 sentences: the goal and the chosen approach>

**Key constraints:** <bullets of locked decisions the executor must not re-open>

**How to work:**
1. You should be in a fresh worktree/checkout. Create a branch <feat|fix|chore|docs>/<slug> off main before touching anything (if the worktree was created with that branch already checked out, just confirm you're on it).
2. Follow the repo's CLAUDE.md rules (branch naming, required doc updates, test commands).
3. Execute the plan step by step. Commit in logical increments with clear messages.
4. If reality contradicts the plan on details, adapt and record the deviation for your final summary. If the plan's core approach turns out to be wrong, stop and report back instead of improvising a new design.
5. Verify per the plan's Verification section before considering any step done.
6. When the work is complete, run a LOW-EFFORT code review of the full branch diff (use the /code-review skill at low effort if available; otherwise review the diff yourself for bugs and regressions). Fix anything real it finds, commit the fixes, and re-run the tests.
7. Save durable session learnings to memory NOW, BEFORE opening the PR (per the memory system's own conventions) — new conventions or traps discovered while implementing that the next session would otherwise rediscover the hard way. Save early because once this tab closes, the lane becomes eligible for external cleanup (/supacode:status --reap deletes merged lanes that have no open tabs), and anything unsaved goes with it.
8. Only once the review is clean AND tests pass: immediately before pushing, fetch and rebase onto latest origin/main; if other lanes are running in parallel (the Key constraints will say so), re-resolve any repo-mandated shared files every PR must touch (status boards etc.), keeping your edit to the minimal rows for your lane. Then push the branch and open a PR (follow the repo's CLAUDE.md rules for pushing and PRs — account checks, any required PR flow like /ship). If review findings can't be resolved or tests won't pass, stop and report back instead of opening a PR.
9. NEVER merge the PR. Merging requires the user's explicit approval, always — leave the PR open for them, and keep this session alive: after they merge, they may run /supacode:complete-feature here to verify the merge, save learnings to memory, and delete this worktree.

**Finish with a final summary:** branch name, PR link, what was implemented, deviations from the plan and why, test results, review findings and how they were resolved, and anything left for follow-up.
```

### 4. After the fence

Below the code block, tell the user:

- The plan file path.
- A ready-to-run worktree command they can use, e.g.
  `git worktree add ../<repo-name>-<slug> -b <branch> main`
  (adjust the base branch to the repo's default).

## Prep mode (`--prep`)

For callers (like /supacode:mission) that plan now and launch later. Do
steps 1–2 (collect and save the plan), then also write the filled-in handoff
prompt (step 3's template) to the sibling file
`~/.claude/plans/<repo-name>/<YYYY-MM-DD>-<slug>.prompt.md` — but print no
copy-paste fence, create no worktree, launch nothing. Report just the two
file paths. The lane launches later via `--launch <plan file path>`.

## Launch mode (`--launch`)

Fully automates the handoff using the Supacode CLI (`supacode`) — no copy-paste.
Only works when running inside Supacode; check `command -v supacode` and
`$SUPACODE_REPO_ID`. If either is missing, say so and fall back to the normal
print-the-prompt flow.

When `--launch` was given an existing plan file's path, steps 1–2 are already
done: use that file and its sibling `.prompt.md` (regenerate the prompt file
from the plan via step 3's template only if the sibling is missing), and start
at step 2 below. Take the **branch from the plan's `branch:` frontmatter** and
the slug from its filename — the filename is `<date>-<slug>.md` and carries no
`feat|fix|chore|docs` prefix, so deriving the branch from it would create an
unprefixed branch that violates the naming rule the executor is then told to
follow. Missing `branch:` frontmatter ⇒ stop and say so rather than guessing.

1. **Save the prompt too.** After saving the plan file, write the filled-in
   handoff prompt (the same template as step 3) to a sibling file:
   `~/.claude/plans/<repo-name>/<YYYY-MM-DD>-<slug>.prompt.md`. (Already
   done for lanes that came through `--prep`.)
2. **Create the worktree:**

       supacode repo worktree-new --branch <type>/<slug> --base <default-branch> --name <slug> --fetch

   (`--repo` defaults to `$SUPACODE_REPO_ID`. `--fetch` is mandatory when the
   repo has a remote — lanes launched minutes apart must not fork from a stale
   local base; omit it only for remote-less repos. But `--fetch` only fetches,
   and it is unverified whether `--base <default-branch>` resolves to the
   local or the remote ref — so ALSO fast-forward the local default branch
   first: from the primary checkout, `git fetch origin <default>:<default>`
   when the default branch is not checked out there, or
   `git merge --ff-only @{u}` when it is checked out and clean. If neither is
   possible, warn that the lane may fork from a stale base.) This checks the
   new branch out in the worktree, so the executor's "create a branch" step
   becomes a confirmation.
3. **Find the new worktree ID:** run `supacode worktree list` and pick the
   URL-encoded path ending in `%2F<name>%2F` — the encoded-slash boundary
   matters, or a lane named `foo` also matches one named `bar-foo` (worktrees
   default to `~/.supacode/repos/<repo-name>/<name>/`). Take the exact string
   from the list — never construct an ID by URL-encoding a path yourself. Append it to
   the plan file's frontmatter as `worktree: <exact-id>` (this is how
   /supacode:status maps lanes back to plans).
4. **Launch the session** in auto permission mode (so it doesn't stall on
   routine edit/test prompts) with Remote Control enabled, named after the slug
   so it's identifiable from claude.ai/mobile:

       supacode tab new --worktree <worktree-id> --input 'claude --permission-mode auto --remote-control "<slug>" "Read the file <absolute prompt path> and execute its instructions exactly."'

   Keep the prompt to that single short pointer sentence — the full contract
   lives in the prompt file, so nothing multi-line has to survive shell
   quoting. Use `auto`, not `bypassPermissions` — genuinely risky actions
   should still reach the user (remotely, via Remote Control). (If the
   installed `claude` lacks either flag, launch without it and mention that in
   the report.)
5. **Report** (instead of the copy-paste fence): worktree path, branch, the
   plan and prompt file paths, and that the session is running. Do not monitor
   or manage the launched session — it reports to the user directly.
