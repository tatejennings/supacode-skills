# supacode-skills

Local Claude Code plugin marketplace holding all Supacode-related skills in a
single plugin named `supacode`. This repo contains only markdown and JSON —
there is no build step.

## Where things go

- New skills: `plugins/supacode/skills/<skill-name>/SKILL.md` (folder + one file).
- Marketplace catalog: `.claude-plugin/marketplace.json` (repo root).
- Plugin identity/version: `plugins/supacode/.claude-plugin/plugin.json`.
- Skills are invoked namespaced: `/supacode:<skill-name>`.

## Skill-authoring rules

- The frontmatter `description` is the only thing Claude sees before deciding
  to load a skill — put all trigger phrases and "use when" contexts there, and
  make it pushy (skills undertrigger by default). Keep the body imperative.
- Any fact about the `supacode` CLI must be verified against live
  `supacode help <subcommand>` output before it goes in a skill — never from
  memory. Worktree/repo IDs are URL-encoded absolute paths; tab/surface IDs
  are UUIDs.
- Keep SKILL.md under ~500 lines; overflow goes in a `references/` subfolder
  with clear pointers from the body.

## After any skill edit

1. `claude plugin validate .` from the repo root.
2. Bump `version` in `plugins/supacode/.claude-plugin/plugin.json`.
3. Remind the user to run `/plugin marketplace update supacode-skills`
   (or `/reload-plugins` mid-session) — local marketplaces do not auto-refresh.

## Migration status

The `supa-*` workflow skills in `~/.claude/skills/` (plan-feature,
handoff-plan, complete-feature, status, mission, …) are still canonical there;
another context is actively adding more. Do NOT copy them into this repo
unprompted — the move happens once, deliberately, via `docs/MIGRATION.md`,
when the user says go.
