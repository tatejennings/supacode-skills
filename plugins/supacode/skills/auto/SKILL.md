---
name: auto
description: Fire-and-forget shorthand - plan a milestone/feature/bug and launch it in one shot, no approval gates, ending with a new Supacode worktree and a Claude session already implementing it. Exactly equivalent to /supacode:plan-feature <work> --auto. Use when the user says "/supacode:auto A4", "/supacode:auto fix the vent bug", "just do X", "auto-plan X", "plan and launch X", "fire and forget X", or names work they want handled end-to-end without reviewing the plan first.
---

# Auto

Shorthand for the fire-and-forget pipeline. Invoke the `supacode:plan-feature`
skill (via the Skill tool) with args:

    $ARGUMENTS --auto

That is the entire job — plan-feature owns everything from here: research,
adversarial review, the overwhelming-recommendation bar for decisions, the
disqualifiers that stop an auto-launch, and the handoff/launch itself.

- If `$ARGUMENTS` is empty, ask what to work on first — never invent a target.
- If `$ARGUMENTS` already contains `--auto`, don't duplicate it.
- Pass any other flags through untouched (e.g. `--plan-only`,
  `--siblings …`).

Remember what `--auto` means for the user: no plan approval, but every safety
property still holds — deferral on real open questions, nothing ever merges a
PR, and disqualified work comes back as a report instead of a launch.
