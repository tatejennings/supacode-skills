# Worktree identity — the rules that govern deletion safety

Shared by `status`, `complete-feature`, `handoff-plan`, and `mission`. These
two rules decide whether a directory may be deleted and where a repo's plans
live; getting either wrong destroys work or writes files to the wrong place.

## Is this a disposable lane, or the primary checkout?

```bash
git -C <path> rev-parse --git-dir --git-common-dir   # one call, both values
```

- **Different** ⇒ a linked worktree (a lane). Deletable, once every other
  safety check passes.
- **Equal** ⇒ the primary checkout. **Never delete it**, under any
  circumstance, no matter what other checks say. Archive at most.

Inside Supacode, a lane also has `$SUPACODE_WORKTREE_ID` set — necessary but
not sufficient on its own, since the primary checkout has one too.

## What is this repo called?

The repo name keys the plan directory (`~/.claude/plans/<repo-name>/`), so it
must be derived from the **primary checkout**, never the current directory:

```bash
git rev-parse --git-common-dir     # → <primary>/.git
# repo-name = basename of <primary>
```

Inside a linked worktree the cwd basename is the *lane* name (e.g.
`vent-fix`), not the repo — using it sends plans to a directory nothing else
reads, and lane↔plan matching silently fails. Derive once per run and reuse;
it cannot change mid-run.

## Worktree and repo IDs

`supacode worktree list` and `repo list` print **URL-encoded absolute paths**
(`%2FUsers%2Fme%2FProjects%2Frepo%2F`). Always pass the exact string from the
list to `-w`/`-r`; never construct an ID by encoding a path yourself. Decode a
*copy* for display and matching only.

When matching a lane by folder name, match on the `%2F<name>%2F` boundary —
a bare suffix match means `foo` also matches `bar-foo`.
