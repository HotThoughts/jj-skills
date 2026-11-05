---
name: jj-workflow
description: Guides Claude on using Jujutsu (jj) version control system. Use when working with jj repositories, making commits, syncing changes, or managing version control workflows.
---

# Jujutsu (jj) Workflow

Jujutsu is a modern version control system that provides a simpler mental model than Git while remaining Git-compatible. This skill covers the core concepts and workflow commands.

## CRITICAL: Avoid Interactive Mode

Always use `-m` to prevent jj from opening an editor:

```bash
# WRONG — opens editor, blocks AI
jj new
jj describe
jj squash

# CORRECT — non-interactive
jj new -m "message"
jj describe -m "message"
jj squash -m "message"
```

`jj split` is non-interactive only when given both `-m` and paths: `jj split -r <change> -m "type: first part" <path>...` moves those paths into a new first commit and leaves the rest, with the original description, in the second. Without paths it opens a diff editor; without `-m` it blocks on the description editor. Never pass `-i`/`--interactive` (`jj split`, `jj squash`, `jj absorb`, `jj commit`), and avoid `jj resolve` (merge tool), `jj file edit`, and `jj arrange` (TUI).

## Minimal Output

Every read command MUST be bounded and summary-level. Full diffs and unbounded logs are the #1 token cost in this skill.

| Instead of | Use |
| --- | --- |
| `jj diff` / `jj diff -r <change>` | `jj diff --stat` (per-file histogram) or `jj diff --summary` (file names) |
| `jj log` | `jj log -n 10` (bounded) |
| `jj op log` | `jj op log --limit 20` |
| `jj status` to see *which files* | `jj diff --stat -r @` |

Drill into one file's actual diff only when the stat line isn't enough: `jj diff -r <change> -- <path>`. `--stat`/`--summary`/`--name-only` are valid on both `jj diff` and `jj op log`.

## Core Concepts

### Changes vs Commits
- **Change**: A mutable revision identified by a change ID (e.g., `kkmpptxz`). Changes can be modified.
- **Commit**: An immutable snapshot identified by a commit ID (SHA). Once created, commits are permanent.
- The working copy (`@`) is always a change that can be modified freely.

### Working Copy
- The working copy is denoted by `@` and represents your current state.
- `@-` refers to the parent of the working copy.
- Unlike Git, there's no staging area—all changes are automatically tracked.

## When to Use What

| Situation | Do This |
| --- | --- |
| Starting new work | `jj new -m "what I'm trying"` |
| Forgot to start with jj new | `jj describe -r @ -m "type: what I'm doing"` (do this immediately) |
| Change has no description | Run the **Description Check Protocol** (see below) |
| `jj git push` rejected with "no description set" | `jj describe -r <change> -m "type: message"` |
| Work is done, move on | `jj new -m "next task"` |
| Annotate what you did | `jj describe -m "feat: auth"` |
| Broke something | `jj op log` → `jj op restore <id>` |
| Undo one file | `jj restore --from @- <path>` |
| Exclude file from current change | `jj restore --from @- <path>` |
| Stop tracking an ignored file | Add ignore rule first, then `jj file untrack <path>` |
| Stop tracking ignored directory contents | Add ignore rule first, then `jj file untrack 'glob:dir/**'` |
| Combine messy commits | `jj squash -m "combined message"` |
| Split some files into their own commit | `jj split -r <change> -m "type: message" <path>...` |
| Fold fixups into the commits that last touched those lines | `jj absorb` |
| Delete a file in another revision without checking it out | `jj file delete -r <change> <path>` |
| Bring the stack up to date with trunk | `jj git fetch`, then `jj rebase -o 'trunk()'` |
| Move a bookmark forward to new work | `jj bookmark advance --to <change>` |
| Same change ID shows as divergent (`??`) | `jj converge -r <change-id> --no-interactive` (`-r` avoids the "pick a change" prompt; `--no-interactive` exits instead of prompting when heuristics are inconclusive) |
| Try something risky | `jj new -m "experiment"`, then `jj abandon @` if it fails |

## Sandbox

Jujutsu can be configured to sign commits (can be checked with `jj config list --include-overridden | grep -i sign`). In certain cases, this makes `jj` commands fail within the sandbox. If this is the case, re-run the command outside the sandbox. When signing is not configured, every `jj` command is supposed to work within the sandbox, there is no need to bypass it.

Signing on push is configured with `revsets.git-sign-on-push` (jj 0.47.1+, a revset such as `~signed() & mine()`), which replaces the deprecated `git.sign-on-push = true`. Since 0.47, `jj git push` signs in a separate operation before pushing, so if the push itself fails, the commits stay signed and a retry does not sign again.

## Essential Commands

### Viewing State

```bash
jj status          # High-level repo status (conflicts, bookmarks)
jj log -n 10       # Recent history, bounded
jj diff --stat     # Uncommitted changes: per-file histogram
jj show @          # Current change details
```

### Creating and Describing Changes

```bash
jj new                                # Create a new empty change on top of @
jj describe -r @ -m "feat: message"   # Set commit message for current change (always use -r to target explicitly)
jj new -m "feat: message"             # Create new change with message
```

### Syncing with Remote

```bash
jj git fetch                   # Fetch bookmarks and tags from remote (no rebase)
jj rebase -o 'trunk()'         # Rebase the branch containing @ onto the latest trunk
jj bookmark advance            # Move the nearest bookmark(s) on @'s ancestry forward to @ (--to <rev> for another target)
jj git push                    # Push current changes to remote (requires description on the change)
jj git push --bookmark <name>  # Push AND set up remote tracking (like git push -u origin <branch>)
```

`jj tug` is not a built-in command. It is a community alias that moves bookmarks forward and does not fetch or rebase. `jj bookmark advance` is the built-in replacement, so prefer it over `jj tug`. Rebase with `-o`/`--onto`. `-d`/`--destination` still works but is deprecated.

`jj git push` will **reject changes without descriptions** ("no description set"). Always run the Description Check Protocol before pushing.
`jj git push --bookmark <name>` is the unified push+track command — it creates the bookmark on the remote and sets up tracking in one step.

### Modifying History

```bash
jj squash                        # Squash current change into parent
jj squash --into @-              # Explicitly squash into parent
jj restore --from @- <path>      # Remove a file from current change
jj split -m "type: msg" <path>   # Move <path> into its own commit (needs -m and paths to stay non-interactive)
jj absorb                        # Move each hunk of @ into the mutable ancestor that last touched those lines
jj file delete -r <change> <path> # Delete a file in any revision; descendants are rebased
jj file untrack <path>           # Stop tracking a path that is already ignored
jj edit <change-id>              # Edit an existing change
```

### Untracking Ignored Files

Use `jj file untrack` when a file should stay on disk locally but stop being tracked by jj. This is an ignore-driven workflow: add the path to `.gitignore` first, then untrack it.

`jj file untrack` only works for paths that are already ignored, usually via `.gitignore` or `.git/info/exclude` in colocated workspaces.

```bash
# 1. Add an ignore rule first
echo "local-config.json" >> .gitignore

# 2. Stop tracking the ignored file
jj file untrack local-config.json

# Untrack ignored directory contents with a fileset glob
jj file untrack 'glob:.agents/**'
```

Use `jj restore --from @- <path>` when you only want to remove a path from the current change. Use `jj file untrack <path>` when you want jj to stop tracking an ignored path going forward.

### Working with Bookmarks

```bash
jj bookmark create <name> -r @  # Create a bookmark at @
jj bookmark set <name> -r @     # Move bookmark to @
jj bookmark list                 # List all bookmarks
```

### Working with Workspaces

Workspaces are jj's equivalent of git worktrees — multiple working directories backed by the same repository. Each workspace has its own `@` (working copy), enabling true parallel development without branch locking.

```bash
jj workspace add <path>          # Create new workspace at path (e.g., ../myproject-fix)
jj workspace list                # List all workspaces with their roots and @ revisions
jj workspace remove <name>       # Snapshot its @, forget it, delete its directory (and Git worktree)
jj workspace forget <name>       # Stop tracking a workspace; leaves the directory on disk
jj workspace root                # Print the root path of current workspace
jj workspace update-stale        # Fix a stale working copy after concurrent changes
```

**Key differences from git worktrees:**
- No branch locking — multiple workspaces can check out the same revision simultaneously
- Each workspace gets its own `@` change automatically on creation
- Changes made in one workspace don't affect another's `@`
- In a colocated repo (jj 0.46+), `jj workspace add` also creates a Git worktree (detached HEAD), so `git` and Git-aware tools work there too. Control this with `--colocate`/`--no-colocate` or `git.colocate`.
- By default, `jj undo`/`jj redo` refuse an operation made in another workspace. To revert that specific operation, use `jj op revert <op-id>`. Use `--allow-cross-workspace` only when the user asks for it.

`jj workspace remove` keeps the working-copy commit, but it deletes the directory, including ignored files such as `.env` and build output. Check before removing a workspace someone else created.

## Commit Message Format

Use **Conventional Commits** format:

- `feat:` — New feature
- `fix:` — Bug fix
- `refactor:` — Code change that neither fixes a bug nor adds a feature
- `perf:` — Performance improvement
- `docs:` — Documentation changes
- `chore:` — Maintenance tasks
- `test:` — Adding or updating tests

Examples:
```bash
jj describe -r @ -m "feat: add user authentication"
jj describe -r @ -m "fix: resolve null pointer in parser"
jj describe -r @ -m "refactor: extract validation logic"
```

## Description Check Protocol

**Always run this protocol before pushing or creating a PR.** `jj git push` rejects changes without descriptions. This is a hard requirement, not a quality preference.

### Step 1: Check if a description exists

```bash
jj log -r <change> -T description --no-graph
```

Read the full output once: first line = title, remainder = body. One call — never the head/tail split.

**Gate logic:**

- **First line is non-empty**: The change has a meaningful description — stop here. Do not overwrite.
- **First line is blank AND no body**: Proceed to Step 2 to generate a full description.
- **First line is blank BUT body exists**: Generate a title only, then prepend it to the existing body using `jj describe --stdin` (see Step 5). Never use `-m` in this case — it would replace the body.

### Step 2: Pick the right change

Default to `@` (working copy), but verify with `jj diff -r @ --stat` first:

- If `@` has modified files, use `@`.
- If `@` has no modified files (fresh "next task" placeholder), check `@-` before using it:

```bash
# Does @ hold the work? (empty output = placeholder)
jj diff -r @ --stat

# Does @- hold the work?
jj diff -r @- --stat

# Check @- bookmarks for trunk markers
jj log -r @- -T 'bookmarks' --no-graph
```

**Safety gate — stop and ask the user if ANY of these are true:**

- `@-` has an empty diff (nothing to describe)
- `@-` carries a trunk bookmark (`main`, `master`, or any bookmark with `@origin` suffix like `main@origin`)
- `@-` has no bookmark AND no diff (ambiguous — could be a bare trunk ancestor)

Only proceed with `@-` when it has a non-empty diff AND no trunk/remote bookmarks. If `@-` carries `main` because work was rebased onto it during conflict resolution, it will have a non-empty diff — but still ask the user to confirm before mutating a revision that holds a trunk bookmark.

### Step 3: Analyze the diff

```bash
jj diff -r <change> --stat
```

Focus on the high-level nature of changes: file paths, new vs modified files, and the overall purpose. If the diff is empty, warn that there are no changes to describe — do not generate a description. If the stat isn't enough to pick a type, drill into one file: `jj diff -r <change> -- <path>`.

### Step 4: Determine the conventional commit type

Map the diff content to a type:

| Type | Signals in the diff |
|---|---|
| `feat:` | New files, new functions/exports, new API routes, new components, new public methods |
| `fix:` | Bug fixes, error handling additions, null/type guards, boundary condition checks |
| `refactor:` | Code moves/renames, extractions, restructuring with no new behavior |
| `perf:` | Caching, async/parallel, lazy loading, memoization, reduced allocations |
| `docs:` | Only documentation files changed (README, markdown, comments, JSDoc) |
| `chore:` | Dependencies, config files, CI/CD, build tooling, formatting, lint rules |
| `test:` | Only test files changed (`*_test.*`, `*.spec.*`, `__tests__/`, test fixtures) |

**Priority when multiple types apply**: if only docs changed → `docs:`; if only tests changed → `test:`; otherwise use the first matching type from: `feat > fix > perf > refactor > chore`.

### Step 5: Generate and apply the description

**Case A: No existing description (most common)**

```bash
jj describe -r <change> -m "type: concise imperative description"
```

**Case B: Blank first line but existing body (from Step 1 gate)**

Use `--stdin` to prepend the title while preserving the body:

```bash
printf 'type: concise title\n\n' | cat - <(jj log -r <change> -T description --no-graph | tail -n +2) | jj describe -r <change> --stdin
```

Never use `-m` for this case — it would replace the entire description and lose the body.

Rules for the description:
- Use present tense imperative mood ("add" not "adds" or "added")
- Keep it under 72 characters
- No trailing period
- Capitalize after the colon only for proper nouns

### Edge Cases

- **Empty diff**: Warn the user — there's nothing to describe. Do not generate a placeholder.
- **Existing description (non-empty first line)**: Never overwrite. The check in Step 1 gates on the first line being empty.
- **Blank first line with body**: Use Case B in Step 5 (`--stdin`) to prepend a title while preserving the existing body. Never use `-m`.
- **Large diff**: Focus on the most significant files and changes. Don't try to enumerate every line.
- **`@-` with trunk bookmark**: If `@-` carries `main`, `master`, or a remote-tracking bookmark (e.g., `main@origin`), stop and ask the user which change to target. See Step 2 safety gate.

## Common Patterns

### Starting New Work
```bash
jj git fetch                        # Sync with remote
jj new 'trunk()' -m "feat: new feature"  # Start new change on latest trunk
# ... make changes ...
# Before pushing: run the Description Check Protocol if @- has no description
jj describe -r @- -m "feat: new feature"  # Ensure description is set
jj new                              # Finalize and start next
```

### Amending Current Change
Simply make changes—they're automatically included in `@`. Use `jj describe -r @ -m "type: message"` to update the message if needed. If the change has no description yet, run the **Description Check Protocol**.

### Rebasing onto Latest
```bash
jj git fetch
jj rebase -o 'trunk()'    # Rebases the whole branch containing @ (default is -b @)
```

### Viewing What Will Be Pushed
```bash
# Run the Description Check Protocol first — jj git push requires descriptions
jj log -r 'remote_bookmarks()..@'    # Changes not yet on remote
```

## Recovery

The operation log records every operation. Nothing is lost.

```bash
jj op log --limit 20   # Recent operations, bounded
jj undo                # Undo last operation (refuses one from another workspace by default)
jj op revert <id>      # Revert one specific operation, keeping later ones
jj op restore <id>     # Jump to any past state
```

### Working with Multiple Workspaces

```bash
# Create a new workspace for parallel work
jj workspace add ../myproject-hotfix
cd ../myproject-hotfix
jj new -m "fix: critical hotfix"     # New @ in the new workspace
# ... make changes, push, etc. ...

# Back in original workspace — unaffected
cd ../myproject
jj log -n 10                         # Other workspace's changes appear in shared history

# Clean up when done (the hotfix commit stays in history)
jj workspace remove myproject-hotfix # Workspace name defaults to the directory name
```

**Use cases:**
- Run long test suites in one workspace while coding in another
- Work on a hotfix without disturbing your in-progress feature
- Compare file states across different revisions simultaneously

## Key Differences from Git

1. **No staging area**: All changes are tracked automatically
2. **Mutable working copy**: The current change can always be modified
3. **Change IDs**: Stable identifiers that persist through rebases
4. **Anonymous branches**: You can work without named branches
5. **Automatic conflict handling**: Conflicts are recorded and can be resolved later
6. **Workspaces**: Multiple working copies without branch locking (vs git worktrees which require a unique branch per worktree)

Behaviour here was checked against jj 0.46.0. The 0.47 items (push signing) come from the jj changelog. `jj <command> --help` is authoritative.
