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

| Skill | Purpose |
|---|---|
| `/supacode:supacode-cli` | Reference for controlling the Supacode app via its CLI |
| `/supacode:plan-feature` | Plan a milestone or feature end-to-end with adversarial review; adopts a pre-written plan if the milestone links to one; `--auto` skips the approval gate and launches the executor lane |
| `/supacode:auto` | Fire-and-forget shorthand for `plan-feature <work> --auto` |
| `/supacode:handoff-plan` | Package a plan for a fresh worktree context; `--launch` automates it |
| `/supacode:mission` | Propose and launch a wave of parallel work lanes |
| `/supacode:status` | Dashboard of all lanes; `--reap` cleans up merged, dead ones; `--paint` color-codes the sidebar by verdict |
| `/supacode:complete-feature` | Close out a merged PR's worktree safely |

The workflow skills' old `/supa-*` names still work as trigger phrases.

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
