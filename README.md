# supacode-skills

A Claude Code **plugin marketplace** for skills that drive
[Supacode](https://supacode.sh) — letting a Claude session create worktrees,
launch other Claude sessions, run scripts, and manage the plan → implement →
verify → PR workflow across parallel lanes of work.

> **Requires [Supacode](https://supacode.sh)** — a macOS app that manages git
> repositories as sets of worktrees, each with its own terminal tabs. These
> skills drive it through its `supacode` CLI, so they only work when Claude
> Code is running inside a Supacode terminal. Without the app installed there
> is nothing for them to control: the skills detect its absence and stop
> rather than half-working, and you'd get more out of plain `git worktree`
> and Claude Code on their own.

## Requirements

- **[Supacode](https://supacode.sh)** (macOS), with Claude Code running in one
  of its terminal tabs. The `supacode` CLI ships inside the app.
- **[Claude Code](https://claude.com/claude-code)** with plugin support.
- **[`gh`](https://cli.github.com)**, authenticated — the workflow skills read
  PR state and open pull requests with it.

## Layout

```
.claude-plugin/marketplace.json      # marketplace catalog
plugins/supacode/                    # the single "supacode" plugin
  .claude-plugin/plugin.json         # plugin identity + version
  skills/<skill-name>/SKILL.md       # one folder per skill
```

## Install

```bash
# from a local clone:
git clone <this-repo-url> supacode-skills
claude plugin marketplace add ./supacode-skills
claude plugin install supacode@supacode-skills
```

(`claude plugin marketplace add` also accepts a GitHub `owner/repo` directly.
Inside a session the slash-command equivalents are `/plugin marketplace add …`
and `/plugin install supacode@supacode-skills`.)

## Skills

The point of all of this: work that would otherwise occupy your session —
research, planning, implementing, reviewing — gets pushed into **separate
worktrees running their own Claude sessions**, in parallel, while your main
session stays free. These skills create those lanes, keep them honest, show
you their state, and clean them up.

| Skill | What it does | What you see in Supacode |
|---|---|---|
| `/supacode:plan-feature` | Researches the codebase and docs with parallel agents, drafts an execution-ready plan, then has a cold-reader agent attack it for gaps, feasibility, and blast radius. Adopts your own plan file if the milestone links to one. | Nothing yet — planning only. With `--auto`, continues straight into the handoff below. |
| `/supacode:auto` | Shorthand for `plan-feature <work> --auto`: plan, review, and launch in one shot, no approval gate. | A new worktree and a running session, as below. |
| `/supacode:handoff-plan` | Writes the plan and an executor contract to disk, then starts a Claude session that implements it, self-reviews, and opens a PR — never merging. | **A new worktree** on its own branch, with **a new tab** running Claude in it, named after the work and reachable from your phone via Remote Control. |
| `/supacode:mission` | Finds the next ready milestones, plans every approved one **concurrently**, checks the finished plans against each other for file collisions, then launches the survivors. | **Several worktrees and sessions at once** — one lane per milestone, each implementing independently. |
| `/supacode:status` | Rebuilds the truth from `git`/`gh`/`supacode` every run — branch, PR state, whether the session is alive, and a verdict per lane. `--reap` deletes provably-finished lanes; `--paint` writes verdicts into the UI. | A dashboard table, plus (with `--paint`) **your sidebar tinted by state** — green merged, purple PR-open, red needs-attention — and finished worktrees **disappearing** as they're reaped. |
| `/supacode:complete-feature` | Run inside a lane after you merge its PR: verifies the merge, checks nothing unpushed would be lost, saves what it learned to memory. | **The worktree and its tab vanish** — the last thing it does is delete itself. |
| `/supacode:supacode-cli` | The CLI reference the others build on; also handles one-off requests directly. | Whatever you asked for — a worktree, a split pane, a script started or stopped. |

Every lane ends at an open PR: **nothing in this pipeline ever merges.** The
workflow skills' old `/supa-*` names still work as trigger phrases.

## Workflows

The skills compose into a pipeline: **plan → hand off → implement → PR →
merge (always you, by hand) → clean up**. Each lane of work gets its own
Supacode worktree and Claude session, so your main session stays free.

```
plan-feature ──► handoff-plan ──► [executor session] ──► PR ──► you merge
     ▲                                                            │
   mission (launches several lanes at once)          complete-feature / status --reap
```

**→ [WORKFLOWS.md](WORKFLOWS.md)** walks through each one with examples: a
single feature reviewed at every step, fire-and-forget, parallel waves,
checking in on lanes, and driving the app directly.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for adding skills, the after-edit
checklist, and releasing.
