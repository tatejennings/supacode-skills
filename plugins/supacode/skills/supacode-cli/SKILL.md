---
name: supacode-cli
description: Control the Supacode macOS app from a Claude context via the `supacode` CLI - create and manage worktrees, launch Claude sessions in new terminal tabs, run/stop a worktree's configured scripts, and manage tabs/surfaces/repos. Use this skill whenever the task involves the Supacode app or CLI in any way - creating a worktree for parallel work, launching an executor session in another worktree, running a worktree's build/test/dev scripts, archiving or deleting a finished worktree, checking which worktrees/tabs exist, or interpreting SUPACODE_* environment variables - even if the user just says "spin up a worktree", "open a lane", or "start a session over there" without naming Supacode.
---

# Supacode CLI

Supacode is a macOS app that manages git repositories as sets of **worktrees**,
each with its own terminal **tabs** (which contain split **surfaces**). The
`supacode` CLI (shipped inside the app at
`/Applications/supacode.app/Contents/Resources/bin/supacode`) lets a Claude
session drive the app: create worktrees, open tabs running arbitrary commands
(including new `claude` sessions), run per-worktree scripts, and clean up.

Everything here is generated from the CLI's own `supacode help` output. If a
command's flags matter to what you're doing, confirm with
`supacode help <subcommand>` — the app updates and this file can lag.

## Am I inside Supacode?

Check before relying on any of this:

- `command -v supacode` — CLI on PATH.
- `$TERM_PROGRAM` = `supacode` — this terminal is a Supacode surface.
- `SUPACODE_*` env vars identify where this session lives:

| Variable | Meaning | Format |
|---|---|---|
| `SUPACODE_REPO_ID` | The repo this worktree belongs to | URL-encoded absolute path |
| `SUPACODE_WORKTREE_ID` | This worktree | URL-encoded absolute path |
| `SUPACODE_WORKTREE_PATH` | This worktree | plain absolute path |
| `SUPACODE_ROOT_PATH` | Repo root checkout | plain absolute path |
| `SUPACODE_TAB_ID` | This terminal tab | UUID |
| `SUPACODE_SURFACE_ID` | This pane within the tab | UUID |
| `SUPACODE_SOCKET_PATH` | IPC socket the CLI talks over | filesystem path |

If the CLI is missing or the env vars are unset, you are not in Supacode —
fall back to plain `git worktree` and say so rather than failing silently.

## The ID model (main gotcha)

Worktree and repo IDs are **URL-encoded absolute paths** — e.g.
`%2FUsers%2Fme%2FProjects%2Fmy-repo%2F` is `/Users/me/Projects/my-repo/`.
`worktree list` and `repo list` print these encoded IDs, one per line. To find
a worktree you just created, list and match on the encoded folder name. Tab
and surface IDs are plain UUIDs.

Most commands default their `--worktree`/`--repo`/`--tab`/`--surface` flags to
the current session's own `SUPACODE_*` values, so from inside a worktree you
can usually omit them — pass them only to act on a *different* worktree.

All commands accept `--timeout <seconds>` (default 180; `0` = wait forever).

## Command map

```
supacode open                       # bring the app to the front
supacode settings                   # open app settings
supacode socket                     # list active IPC sockets

supacode repo list                  # repo IDs (URL-encoded paths)
supacode repo open <abs-path>       # open/register a repository
supacode repo worktree-new --branch <b> [--base <ref>] [--name <folder>]
                           [--fetch] [--location <parent-dir>]

supacode worktree list [--focused]  # worktree IDs; --focused = current only
supacode worktree focus|pin|unpin|archive|unarchive|delete [-w <id>]
supacode worktree script list [-w <id>]        # user-defined scripts + UUIDs
supacode worktree run  [-c <script-uuid>]      # default: primary run script
supacode worktree stop [-c <script-uuid>]      # default: all run-kind scripts

supacode tab list [-w <id>] [--focused]        # tab UUIDs in a worktree
supacode tab new [-w <id>] [--input '<cmd>']   # new tab, optionally running cmd
supacode tab focus|close [-t <uuid>]

supacode surface list|focus|close [-s <uuid>]
supacode surface split [--direction h|v] [--input '<cmd>']
```

## Recipes

### Create a worktree on a new branch

```bash
supacode repo worktree-new --branch feat/my-slug --base main --name my-slug --fetch
```

- `--repo` defaults to `$SUPACODE_REPO_ID`; the new branch is checked out in
  the new worktree.
- `--name` sets the folder name (defaults to the branch name — slashes in
  branch names make awkward folders, so pass `--name`).
- Worktrees default to `~/.supacode/repos/<repo-name>/<name>/` unless
  `--location` overrides the parent directory.
- Add `--fetch` when the repo has a remote so `--base` is current.

### Launch a Claude session in a worktree

```bash
supacode tab new --worktree <worktree-id> \
  --input 'claude --permission-mode auto --remote-control "my-slug" "Read the file /abs/path/to/prompt.md and execute its instructions exactly."'
```

- The whole command must survive one level of shell quoting — keep the inline
  prompt to a single short pointer sentence and put the real instructions in a
  file the new session reads. Nothing multi-line inside `--input`.
- Prefer `--permission-mode auto` over `bypassPermissions`: routine edits
  proceed, genuinely risky actions still reach the user (via Remote Control
  if they're away). If the installed `claude` lacks a flag, launch without it
  and report that.

### Run the worktree's scripts (build / test / dev server)

Supacode worktrees can have user-configured scripts — use them as the
verification hook instead of guessing at build commands:

```bash
supacode worktree script list      # empty output = none configured (not an error)
supacode worktree run              # primary run-kind script
supacode worktree run -c <uuid>    # a specific script
supacode worktree stop             # stop all run-kind scripts
```

Long-lived scripts (dev servers) keep running after `run` returns; pair every
`run` of a server-style script with an eventual `stop`.

### Worktree lifecycle

`focus` to bring one to the front for the user; `pin`/`unpin` and
`archive`/`unarchive` to organize the sidebar (archive is the safe
"out of sight, recoverable" option); `delete` to remove it permanently.

## Safety rails

- **Never delete the primary checkout.** A disposable worktree has
  `git rev-parse --git-dir` ≠ `git rev-parse --git-common-dir`; equal means
  primary — archive it at most, never `worktree delete`.
- **Deleting the current worktree kills your own tab** (and the session in
  it). If that is intended (self-cleanup after a merge), make `delete` the
  literal last command after all reporting is done. The tab dying right after
  is success, not failure.
- Before any `delete`: confirm `git status --porcelain` is empty and any
  PR/branch state is fully pushed or merged — deletion silently discards
  local-only work. Prefer `archive` whenever unsure.
- Don't `tab close` or `surface close` terminals you didn't open; other
  sessions (including other Claude contexts) may be running in them.
