# Changelog

Notable changes to the `supacode` plugin. Versions track
`plugins/supacode/.claude-plugin/plugin.json`.

## 0.4.0 — 2026-07-26

- `status`: new `--paint` flag — tints each lane's Supacode sidebar entry by
  verdict (blue working, purple pr-open, teal/green merged, orange stalled,
  red needs-attention) and marks its title with a compact state glyph
  (`⇧#42`, `✓#42`, `⏸`, `⚠`), turning the sidebar into a live dashboard.
  Idempotent and loop-safe; composes with `--reap` (reap first, paint
  survivors); never touches worktrees outside the lane set; painting
  failures are cosmetic and never affect verdicts or reaping.

## 0.3.1 — 2026-07-26

- `plan-feature`: `--auto` mode now decides a fork alone only on an
  **overwhelming recommendation** (conventions, requirements, and standard
  practice all agree, and a wrong choice is cheap to reverse); anything less
  defers with the open question stated as the reason. Deferring on honest
  uncertainty is explicitly a correct outcome.
- `mission`: deferral framed as a designed outcome; deferred lanes are never
  re-run hoping for a different answer.

## 0.3.0 — 2026-07-26

- `mission`: after the wave report, offers (multi-select) to open an
  interactive planning session in a new tab for each deferred lane — launched
  without auto permission mode so the user can answer the trade-off questions
  that blocked auto-launch.

## 0.2.1 — 2026-07-26

- Sanitized for public release: generic install instructions, personal email
  removed from plugin author, executed migration runbook removed
  (machine-specific notes moved to the untracked `CLAUDE.local.md`).

## 0.2.0 — 2026-07-26

- Migrated the five workflow skills (`plan-feature`, `handoff-plan`,
  `complete-feature`, `status`, `mission`) into the plugin from user-level
  `~/.claude/skills/supa-*`, dropping the `supa-` prefix; old names remain as
  trigger aliases in each description.
- Review fixes rode in with the migration: `stalled`/`pr-open` verdict overlap
  resolved; `HEAD != headRefOid` scoped to merged PRs; mission preflight can
  no longer fast-forward the wrong branch; stale-base guard on `--launch`;
  plan↔lane mapping via `worktree:` frontmatter; boundary-safe worktree-ID
  matching; documented the `--location` visibility limitation.

## 0.1.0 — 2026-07-26

- Initial marketplace scaffold with the `supacode-cli` skill: detection,
  ID model, command map, and recipes for driving the Supacode app
  (worktrees, tabs, scripts, session launching) from a Claude context.
