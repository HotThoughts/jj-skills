---
name: jj-cleanup
description: Removes the local jj bookmarks, revisions, and workspaces left behind by merged or closed GitHub pull requests, using the jj-cleanup binary through the `jj cleanup` / `jj cl` aliases. Use after a PR or stack merges or closes, when pruning stale bookmarks or workspaces, when `jj log` is cluttered with finished work, or when asked what is safe to delete locally. Also use to install jj-cleanup and its aliases when they are missing.
---

# jj-cleanup

`jj-cleanup` ([HotThoughts/jj-cleanup](https://github.com/HotThoughts/jj-cleanup)) deletes the local
bookmarks, revisions, and workspaces that outlive the GitHub pull requests they were opened for. It
plans first, prints exactly what it would remove, and asks before changing anything. It is not
`git clean`: no file is ever removed from a working copy.

## Prerequisite: binary plus aliases

If `jj cleanup` or `jj cl` report an unknown alias, or `jj-cleanup` is not on `PATH`, the tool is not
installed. Surface this to the user rather than installing silently — the shell installer pipes a
remote script into `sh`:

```bash
curl -LsSf https://github.com/HotThoughts/jj-cleanup/releases/latest/download/jj-cleanup-installer.sh | sh
jj-cleanup util install-aliases      # writes `jj cleanup` and `jj cl`
```

With Cargo: `cargo install --git https://github.com/HotThoughts/jj-cleanup` then
`jj-cleanup util install-aliases`. The installer targets user config; `--repo` scopes the aliases to
the current repository. By hand, the aliases are:

```toml
[aliases]
cleanup = ["util", "exec", "--", "jj-cleanup"]
cl = ["util", "exec", "--", "jj-cleanup"]
```

Requirements: jj 0.44 or newer (it refuses older versions — workspace cleanup needs the directory a
workspace reports) and an authenticated `gh`, which owns all GitHub access.

Check the install with `jj-cleanup --version` and `jj config get aliases.cleanup`.

## Commands

| Command | Effect |
|---|---|
| `jj cleanup list` | Print the plan. Always a dry run, never applies. |
| `jj cleanup` | Plan, prompt, apply. |
| `jj cleanup -y` | Plan and apply without the prompt. |
| `jj cleanup lock <bookmark>...` | Never treat these bookmarks as candidates. Repo-scoped. |
| `jj cleanup unlock <bookmark>...` | Stop protecting them. |
| `jj cleanup util dump` | Print the gathered jj and gh state as JSON — use to debug "nothing to clean". |
| `jj cleanup util install-aliases` | Write the `cleanup` and `cl` aliases. |

Options: `--state merged` (default) or `closed`; `--scan quick` (default, the repository `gh`
resolves) or `deep` (every GitHub remote); `--dry-run`; `--no-fetch`; `--no-workspaces`;
`--remove`; `-y`. Exit codes: 0 for success including a plan with nothing to do, 1 for an error or
an abort.

## Workflow

`jj cleanup` runs a fetch before planning unless `--no-fetch` is passed. Reviewing before applying is
the point of the tool, so drive it in two steps:

```bash
jj git fetch                 # optional; cleanup fetches too
jj cleanup list              # review the plan
jj cleanup                   # interactive, or `jj cleanup -y` after the user confirms
jj bookmark list && jj log   # verify
```

The plan names each action, e.g.:

```
  abandon 1 revision for bookmark "docs-jj-gh-stack"
```

Applying deletes the bookmark and abandons its revisions: they leave `jj log` and `all()`, while
squash-merged work is already in trunk. The abandoned revision stays addressable by change or commit
id and recoverable with `jj undo` or `jj op restore`.

Agents: never pass `-y` before showing the user the plan. `list` and `--dry-run` change nothing, but
they may record a snapshot operation for a workspace that has uncommitted changes, because reading
that state snapshots it (`--no-workspaces` reads no workspace and records nothing).

## Why a bookmark is kept

A bookmark is a candidate only when it has at least one associated pull request, none of those pull
requests is open, at least one is in the requested state set, and the bookmark is not locked, not
conflicted, not the default branch, and not sitting on trunk. Otherwise the run reports `Nothing to
clean` plus a `skipped:` and `warnings:` list. Real lines:

```
skipped:
  bookmark "docs-jj-gh-stack": checked out in a workspace
  bookmark "docs-jj-gh-stack": locked
  workspace "default": is the primary workspace

warnings:
  bookmark "docs-gh-stack-demo" carries a trailer for #8, which was opened from "docs-jj-gh-stack"
```

A pull request is associated either by a `PR: #N` trailer on the bookmark's tip commit, or by a pull
request whose head branch has the bookmark's name and whose head commit matches the tip. Since
cleanup abandons revisions and leaves the user's uncommitted work alone, it also protects a bookmark
whose revisions are still held by a kept bookmark or by a workspace this run leaves alone — a shared
stack base survives and is retried on a later run.

To make cleanup recognize a pull request, put the trailer in a trailing trailer block:

```bash
jj describe -r <change> -m "feat: add auth

PR: #42"
```

A prose mention (`PR: #7 was reverted`) is ignored, and a tip that disagrees with the pull request's
recorded head is kept with a mismatch warning so local work is not mistaken for the finished
pull request.

## Locks

`jj cleanup lock <bookmark>...` stores the name in repository-scoped jj config
(`jj-cleanup.locked-bookmarks`), so it travels with the repository and applies in every workspace.
Use it for a merged pull request whose branch the user still needs locally. `unlock` removes it.

## Workspaces

Workspaces participate by default. One is cleaned only when it is clean, is not the primary
workspace, is not the one being run from, holds a candidate's revisions, and its bookmark is not
locked; otherwise it is listed as skipped and its checked-out commits are protected. Forgetting a
workspace does not delete its directory — `--remove` does, and ignored files in that directory are
lost, which is why `--remove` is never the default.

## Notes

- `--state closed` also treats closed-unmerged pull requests as cleanable; the default skips them.
- A layer of a partially merged stack whose own pull request is still open is kept, per the
  candidate rules. Clean up after the whole stack merges, or expect the top layers to remain.
- gh-stack's own plumbing is unrelated: this removes local leftovers, and the stack on GitHub is
  untouched. Unstacking is `gh stack unstack` (see the `jj-gh-stack` skill).
- Behaviour above was checked against jj-cleanup 0.1.0 with jj 0.45.1; `jj cleanup <command> --help`
  and `jj-cleanup --help` are authoritative.
