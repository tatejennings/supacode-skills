# supacode-skills

A local Claude Code **plugin marketplace** for skills that drive the
[Supacode](https://supacode.app) macOS app — letting a Claude context create
worktrees, launch sessions, run scripts, and manage the plan → implement →
verify → PR workflow.

## Layout

```
.claude-plugin/marketplace.json      # marketplace catalog
plugins/supacode/                    # the single "supacode" plugin
  .claude-plugin/plugin.json         # plugin identity + version
  skills/<skill-name>/SKILL.md       # one folder per skill
docs/MIGRATION.md                    # runbook for moving ~/.claude/skills/supa-* in here
```

## Install

```bash
# one-time: register this repo as a marketplace, then install the plugin
claude plugin marketplace add /Users/tate/Documents/Projects/supacode-skills
claude plugin install supacode@supacode-skills
```

(Inside a session the slash-command equivalents are `/plugin marketplace add …`
and `/plugin install supacode@supacode-skills`.)

## Skills

| Skill | Purpose |
|---|---|
| `/supacode:supacode-cli` | Reference for controlling the Supacode app via its CLI |
| `/supacode:plan-feature` | Plan a milestone or feature end-to-end with adversarial review |
| `/supacode:handoff-plan` | Package a plan for a fresh worktree context; `--launch` automates it |
| `/supacode:mission` | Propose and launch a wave of parallel work lanes |
| `/supacode:status` | Dashboard of all lanes; `--reap` cleans up merged, dead ones |
| `/supacode:complete-feature` | Close out a merged PR's worktree safely |

The workflow skills' old `/supa-*` names still work as trigger phrases.

## After editing skills

Local marketplaces do **not** auto-refresh. After changing anything here:

1. Bump `version` in `plugins/supacode/.claude-plugin/plugin.json`.
2. `claude plugin validate .` from the repo root.
3. `/plugin marketplace update supacode-skills` (or `/reload-plugins` to pick
   up changes mid-session).

## Developing a skill

- Add a folder: `plugins/supacode/skills/<name>/SKILL.md` with `name` +
  `description` frontmatter. The description is the triggering mechanism —
  put every "when to use" phrase there, not in the body.
- Test without installing: `claude --plugin-dir ./plugins/supacode` (overrides
  the installed copy for that session).
- Validate before committing: `claude plugin validate .`
