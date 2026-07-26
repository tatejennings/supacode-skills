# Migration runbook: `~/.claude/skills/supa-*` → this plugin

> **Executed 2026-07-26.** All five workflow skills were moved in with the
> `supa-` prefix dropped (`/supacode:plan-feature` etc.); old names remain as
> trigger aliases in each description. Kept for reference if more user-level
> skills ever migrate.

## 0. Inventory

List what exists at migration time (the set is still growing):

```bash
ls ~/.claude/skills/ | grep '^supa-'
```

Known at time of writing: `supa-plan-feature`, `supa-handoff-plan`,
`supa-complete-feature`, `supa-status`, `supa-mission` — plus whatever the
other context has added since.

## 1. Decide naming (once, for all skills)

Plugin skills invoke as `/supacode:<skill-name>`, so the `supa-` prefix
becomes redundant:

- **Keep prefix** (`/supacode:supa-plan-feature`): zero edits to trigger
  phrases and cross-references; preserves user muscle memory.
- **Drop prefix** (`/supacode:plan-feature`): cleaner, but every cross-skill
  reference and every "Use when the user says /supa-…" trigger phrase in every
  description must be rewritten consistently.

Cross-references that must stay consistent whichever way you go:
`supa-plan-feature` invokes `supa-handoff-plan` (`--auto` mode);
`supa-handoff-plan`'s prompt tells executors about `supa-complete-feature`;
`supa-mission` launches lanes via `supa-plan-feature --auto`; `supa-status`
is referenced by `supa-mission` for monitoring/cleanup.

## 2. Move

For each skill folder:

```bash
mv ~/.claude/skills/<skill> plugins/supacode/skills/<new-name>/
```

Only Supacode-related skills move — leave unrelated user-level skills
(e.g. `apple-bootstrap`, `swiftui-pro`) where they are.

## 3. Validate, version, publish

```bash
claude plugin validate .
# bump version in plugins/supacode/.claude-plugin/plugin.json
git add -A && git commit
```

Then `/plugin marketplace update supacode-skills` and reinstall/`/reload-plugins`.

## 4. Verify BEFORE deleting originals

In a fresh session with the plugin installed, confirm each migrated skill
appears (namespaced) and loads. Only then remove any user-level leftovers so
skills never trigger twice. (If step 2 used `mv`, there is nothing to delete —
this check is the safety net for any copies made instead.)

## 5. Smoke test the chain

Run `/supacode:supa-plan-feature` (or renamed equivalent) on a trivial item
and confirm it can still find and invoke its sibling skills end-to-end.
