---
name: jj-gh-stack
description: Manages GitHub stacked PRs from a Jujutsu (jj) repository with the gh stack CLI. Use when creating, growing, updating, unstacking, or merging a stack of pull requests in a jj repo, when a jj bookmark backs a stacked PR layer, or when gh-stack's git-oriented workflow (init/add/checkout/up/down/rebase/sync) would touch a jj repo. Replaces the local-tracking loop in the gh-stack skill with jj bookmarks plus jj git push plus gh stack link.
---

# GitHub stacked PRs from Jujutsu

`gh stack` is git-centric: `init`, `add`, `checkout`, `up`, `down`, `top`, `bottom`, `submit`,
`push`, `rebase`, and `sync` all move git branches, run `git rebase`, and keep stack state in
`.git/gh-stack`. In a jj repo those fight jj's model — branches move and commits get rewritten
behind jj's back while `@` stays put, and the resulting divergence has to be untangled by hand.

From a jj repo use only the GitHub-facing subcommands. `gh stack link --help` states the intent:
"designed for users who manage branches with external tools (e.g. jj, Sapling, ghstack, git-town,
etc…) and want to use GitHub stacked PRs without adopting local stack tracking."

Read `skill://gh-stack` for stack semantics, the `link`/`merge`/`unstack` flag reference, exit
codes, and layer-design guidance — all of it applies. This skill covers only what jj changes.

## Model

- One layer = one mutable jj commit on top of trunk plus **one bookmark per layer**.
- `gh stack link a b c` takes branches in **stack order, bottom to top**, positionally. `a` is
  based on trunk and merges first.
- `link` is the only way to write stack structure, and it keeps **no local state** (documented in
  gh-stack's own troubleshooting reference). GitHub holds the stack; the jj repo holds nothing, so
  the local commands have nothing to read. `gh stack view` fails outright here —
  `✗ failed to get current branch: failed to run git: not on any branch` (exit 2) — because jj
  leaves git HEAD detached. The same applies to `merge` with no argument. Get status from GraphQL
  or `gh pr view` instead.
- `jj git push` force-pushes rewritten bookmarks with lease; `link` pushes the branch arguments
  itself but never forces (`git.Push(remote, branches, force=false, atomic=true)`). So push
  rewrites with jj first, then relink — a reordered branch cannot be updated by `link`.

## Forbidden in a jj repo

| Never run | Why |
|---|---|
| `git add`, `git commit` | No staging area in jj; changes are already in `@`. gh-stack's git loop assumes one. |
| `gh stack init` / `add` / `checkout` | Create git branches and `.git/gh-stack` tracking that jj bookmarks don't mirror. |
| `gh stack up` / `down` / `top` / `bottom` | Read the local tracking file; absent or stale here. |
| `gh stack rebase` / `sync` / `rebase --continue` / `--abort` | `git rebase` behind jj's back, plus `.git/gh-stack-rebase-state`. Conflicts are jj's to resolve. |
| `gh stack push` / `submit` | Operate on locally tracked stack branches; use `jj git push`. |
| `git checkout`, `git rebase`, `git branch -f` | Same divergence problem without gh-stack involved. |

## Bookmark list in stack order

`link` needs bottom-to-top. `jj log` is newest-first by default, so order must be forced:

```bash
jj log -r 'roots(mutable() & ::@)::@' --no-graph --reversed \
  -T 'if(local_bookmarks, local_bookmarks.map(|b| b.name()).join(" ") ++ " ", "")'
```

Optionally alias it (edit `~/.config/jj/config.toml`):

```toml
[revset-aliases."substack()"]
definition = "roots(mutable() & ::@)::@"
doc = "Stack from its bottom through the working copy"

[aliases.sb]
definition = ["stack-bookmarks"]
doc = "Shorthand for stack-bookmarks"

[aliases.stack-bookmarks]
definition = [
  "--config", "revsets.log=substack()",
  "log", "--no-graph", "--reversed", "-T",
  'if(local_bookmarks, local_bookmarks.map(|b| b.name()).join(" ") ++ " ", "")'
]
doc = "Local bookmark names in substack(@), oldest first, space-separated"
```

Pipe the output into `link` with `xargs`; zsh does not word-split `$(...)`, so
`gh stack link $(jj sb)` passes one bogus argument there.

```bash
jj log -r 'roots(mutable() & ::@)::@' --no-graph --reversed -T 'if(local_bookmarks, local_bookmarks.map(|b| b.name()).join(" ") ++ " ", "")' | xargs gh stack link
```

Verify the printed order names your layers bottom-first before running it. Run `link` from the top
of the stack so the revset covers every layer, and check out the layer you intend to edit.

## Create a stack

1. Build the layers as separate commits, one concern each, bottom = foundational.
2. Create a bookmark per layer, bottom-up:

```bash
jj bookmark create auth-layer -r <change>
jj bookmark create api-routes -r <change>
```

   Alternative: `jj git push --change '<revset>'` auto-creates a bookmark per change from
   `templates.git_push_bookmark` (default `push-<changeid>`). It **ignores existing bookmarks**, so
   mixing the two produces duplicate branches and duplicate PRs. Pick one approach.
3. Push and set tracking in one step. A plain `jj git push -r <bookmarks>` refuses to create new
   remote bookmarks:

```bash
jj git push --bookmark auth-layer --bookmark api-routes
# "Refusing to create new remote bookmark api-routes@origin" means the tracking push was skipped
```

4. Link them bottom-to-top. New PRs are created as drafts with chained bases:

```bash
gh stack link auth-layer api-routes
```

5. `link` auto-generates PR titles and bodies. Replace them, then publish:

```bash
gh pr edit <number> --title "<conventional title>" --body-file <file>
gh stack link --open auth-layer api-routes    # drafts -> ready for review
```

## Update existing layers

`jj edit <layer>` (or rewrite while `@` is above it), then push the whole stack and relink:

```bash
jj git push -r 'roots(mutable() & ::@)::@'
gh stack link auth-layer api-routes
```

Relinking skips PRs that already exist and retargets bases. It cannot update a rewritten branch
itself, which is why the jj push comes first.

## Grow at the top

New commit and bookmark on top of the previous top layer, then:

```bash
jj git push --bookmark ui-layer
gh stack link auth-layer api-routes ui-layer
```

With a known stack number the first argument is a shortcut, appending only the delta:
`gh stack link 7 ui-layer`.

## Restructure: unstack, then relink

`gh stack` rejects PRs added to the middle of a stack, and never reorders or removes. Both require
unstacking on GitHub first. `gh stack unstack <stack-number>` works through the API with no local
state, so it is usable from a jj repo.

- **Insert in the middle or remove a layer:** `gh stack unstack <n>`, create or `jj abandon -r
  <change>` the layer, `jj git push` the stack, then `gh stack link` every layer again.
- **Reorder:** unstack first, then push and relink **from the bottom up, one layer at a time**.
  Reordering is destructive: GitHub marks a PR merged as soon as a push makes its head reachable
  from its current base, and a merged PR cannot be reopened — reviews and comments die with it.
  Never run a bulk `jj git push -r <stack>` against a still-stacked, reordered set.
- **After removal:** push deleted bookmarks with `jj git push --deleted` **after** relinking.
  Deleting the remote branch first closes the PR stacked above it as well.

## Merge

```bash
gh stack merge <pr-number> --yes --squash
```

Merges that PR and everything below it, all-or-nothing. Without an argument it resolves the stack
from the current git branch, which fails under jj's detached HEAD — always pass a PR or stack
number. Discovery of a stack number, or of your position in a stack, comes from the number shown in
the GitHub stack UI or the GraphQL query below. For a partial merge, `jj git fetch` afterwards and
rebase the surviving layers onto the new trunk before relinking. Once a stack has merged, prune the
per-layer bookmarks with `jj cleanup` (see the `jj-cleanup` skill).

## Status without local tracking

`gh pr view --json stack` does not exist. Use GraphQL, which exposes read-only `stack` and
`stackEntry` on `PullRequest` (`position` 1 is closest to trunk):

```bash
gh api graphql -f query='query($owner:String!,$name:String!,$number:Int!){repository(owner:$owner,name:$name){pullRequest(number:$number){number baseRefName stack{number size baseRefName} stackEntry{position}}}}' \
  -F owner=<owner> -F name=<repo> -F number=<pr-number>
```

`stack: null` means the PR is not stacked.

## Notes

- Rebase conflicts never reach gh-stack from a jj repo: `gh stack rebase --continue` / `--abort` and
  the exit-3 recovery in the gh-stack skill do not apply. Resolve in jj (`jj status`, edit the
  conflicted files, `jj` records conflicts as first-class).
- Stacks stay strictly linear, one parent and at most one child. Parallel work belongs in separate
  stacks — or separate jj workspaces.
- Command shapes here were checked against `gh stack` v0.1.1 and jj 0.45.1; `gh stack <command>
  --help` remains authoritative for flags.
