# Changelog

Notable changes to the `supacode` plugin. Versions track
`plugins/supacode/.claude-plugin/plugin.json`.

## 0.7.2 — 2026-07-26

- Docs: the README skills table now has a "What you see in Supacode" column —
  worktrees and sessions created, the sidebar painted, worktrees deleted — so
  the benefit is visible without reading a skill. Workflow walkthroughs moved
  to `WORKFLOWS.md`, maintainer steps to `CONTRIBUTING.md`.
- Skill descriptions state their Supacode-visible outcome (new worktree +
  session, several lanes at once, worktree removal), which also broadens
  triggering on phrasings like "run this in a new worktree".

## 0.7.1 — 2026-07-26

- Docs: state the Supacode dependency up front — README requirement notice
  and Requirements section, and the plugin/marketplace descriptions now say
  the app is required. Corrected the Supacode link to https://supacode.sh
  and added it as the plugin `homepage`.

## 0.7.0 — 2026-07-26

- `plan-feature`: when a milestone **links to a plan markdown file** with
  implementation steps, adopt that plan instead of re-planning — skip
  research and drafting, carry the user's own words through verbatim (filling
  only missing sections like Verification), still run the adversarial review,
  and hand off. A review finding that would change an adopted plan's approach
  is a disqualifier, not a silent edit — the user decides. Links to design
  docs or specs without steps still plan normally.

## 0.6.0 — 2026-07-26

- New `auto` skill: `/supacode:auto <work>` — fire-and-forget shorthand,
  exactly equivalent to `/supacode:plan-feature <work> --auto`. Empty
  arguments ask instead of inventing a target; other flags pass through.

## 0.5.0 — 2026-07-26

- `mission`: lanes now **plan in parallel** — one subagent per approved lane
  runs `plan-feature --auto --plan-only` concurrently, then a single
  reviewer checks the finished plans against each other for real collisions,
  then survivors launch serially (fresh-base fetch per worktree). Collision
  losers are deferred, keeping the lane the user approved first.
- `plan-feature`: new `--plan-only` modifier (with `--auto`) — full research,
  draft, and adversarial review, but preps the handoff files instead of
  launching.
- `handoff-plan`: new `--prep` mode (save plan + prompt files, launch
  nothing) and `--launch <plan-file-path>` variant to launch a previously
  prepped lane.

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
