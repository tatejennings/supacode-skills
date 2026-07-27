# Contributing

## Adding or editing a skill

Each skill is a folder with one file:
`plugins/supacode/skills/<name>/SKILL.md`, with `name` + `description`
frontmatter. The description is the triggering mechanism — every "when to use"
phrase belongs there, not in the body.

Facts about the `supacode` CLI must come from live `supacode help <subcommand>`
output, never from memory. Worktree and repo IDs are URL-encoded absolute
paths; tab and surface IDs are UUIDs.

**Shared specs live in `references/`, not in copies.** Two rules are read by
several skills and are dangerous when they drift:

- `supacode-cli/references/worktree-identity.md` — primary-vs-linked worktree
  proof (deletion safety), repo-name derivation, ID conventions.
- `handoff-plan/references/plan-file-format.md` — plan file path, frontmatter,
  match order, collision guard.

Change the reference, not a skill's restatement of it. Repetition elsewhere
(the launch incantation, "nothing ever merges") is deliberate — an instruction
present where it's needed can't be missed.

Test without installing:

```bash
claude --plugin-dir ./plugins/supacode   # overrides the installed copy for that session
```

## After any skill edit

1. `claude plugin validate .` from the repo root.
2. Bump `version` in `plugins/supacode/.claude-plugin/plugin.json`.
3. Add a `CHANGELOG.md` entry under that version.
4. Check `README.md` (skills table) and `WORKFLOWS.md` for drift.
5. Publish locally — local marketplaces do not auto-refresh, and the first
   command alone does **not** update the installed copy:

   ```bash
   claude plugin marketplace update supacode-skills
   claude plugin update supacode@supacode-skills
   ```

   Changes apply to new sessions; existing ones need `/reload-plugins`.

## Releasing

`/ship` — a project-level skill in `.claude/skills/`. It verifies the version,
changelog, README, and git state agree, then pushes, tags `v<version>`, and
creates a GitHub release with notes from the changelog entry.
