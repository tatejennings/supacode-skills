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
3. Add an entry to `CHANGELOG.md` under the new version — what changed and
   why, per skill.
4. Check `README.md` for drift: the Skills table (new/renamed skills, changed
   one-line purposes) and the Workflows section (does the behavior described
   still match — flags, defaults, what asks vs. defers, what gets launched).
5. Publish it yourself — run both:
   `claude plugin marketplace update supacode-skills` (refreshes the catalog)
   and `claude plugin update supacode@supacode-skills` (refreshes the
   installed copy). Local marketplaces do not auto-refresh, and the first
   command alone does NOT update the installed plugin. Then tell the user the
   change applies to new sessions (existing ones need `/reload-plugins`).

## Naming

Skills are invoked through the plugin namespace (`/supacode:plan-feature`),
so skill names carry no `supa-` prefix. The workflow skills' descriptions end
with a "Formerly /supa-<name>" trigger alias from their pre-plugin life —
preserve those aliases when editing descriptions.
