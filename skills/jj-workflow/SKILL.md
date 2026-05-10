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

`jj split` is inherently interactive — no non-interactive mode exists. Use `jj restore --from @- <path>` to remove a file from the current change instead.

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
| Try something risky | `jj new -m "experiment"`, then `jj abandon @` if it fails |

## Permission Requirements

**CRITICAL**: Jujutsu commands require GPG signing and SSH/GitHub authentication. Always request elevated permissions when running `jj` or `gh` commands:

```
required_permissions: ["all"]
```

Never run `jj` commands in the default sandbox—they will fail due to authentication requirements.

## Essential Commands

### Viewing State

```bash
jj status          # Show working copy status
jj log             # View commit history
jj diff            # Show uncommitted changes
jj show @          # Show current change details
```

### Creating and Describing Changes

```bash
jj new                                # Create a new empty change on top of @
jj describe -r @ -m "feat: message"   # Set commit message for current change (always use -r to target explicitly)
jj new -m "feat: message"             # Create new change with message
```

### Syncing with Remote

```bash
jj tug                         # Fetch updates and rebase current change onto latest remote
jj git fetch                   # Fetch from remote without rebasing
jj git push                    # Push current changes to remote (requires description on the change)
jj git push --bookmark <name>  # Push AND set up remote tracking (like git push -u origin <branch>)
```

`jj git push` will **reject changes without descriptions** ("no description set"). Always run the Description Check Protocol before pushing.
`jj git push --bookmark <name>` is the unified push+track command — it creates the bookmark on the remote and sets up tracking in one step.

### Modifying History

```bash
jj squash                        # Squash current change into parent
jj squash --into @-              # Explicitly squash into parent
jj restore --from @- <path>      # Remove a file from current change (non-interactive alternative to split)
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
jj workspace list                # List all workspaces with their @ revisions
jj workspace forget <name>       # Stop tracking a workspace (run from another workspace)
jj workspace root                # Print the root path of current workspace
jj workspace update-stale        # Fix a stale working copy after concurrent changes
```

**Key differences from git worktrees:**
- No branch locking — multiple workspaces can check out the same revision simultaneously
- Each workspace gets its own `@` change automatically on creation
- Changes made in one workspace don't affect another's `@`

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
# Check the first line (used as PR title)
jj log -r <change> -T description --no-graph | head -1

# If first line is blank, check for an existing body
jj log -r <change> -T description --no-graph | tail -n +2
```

**Gate logic:**

- **First line is non-empty**: The change has a meaningful description — stop here. Do not overwrite.
- **First line is blank AND no body**: Proceed to Step 2 to generate a full description.
- **First line is blank BUT body exists**: Generate a title only, then prepend it to the existing body using `jj describe --stdin` (see Step 5). Never use `-m` in this case — it would replace the body.

### Step 2: Pick the right change

Default to `@` (working copy), but verify with `jj status` first:

- If `@` has modified files, use `@`.
- If `@` has no modified files (fresh "next task" placeholder), check `@-` before using it:

```bash
# Check if @- has actual work
jj diff -r @-

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
jj diff -r <change>
```

Focus on the high-level nature of changes: file paths, new vs modified files, and the overall purpose. If the diff is empty, warn that there are no changes to describe — do not generate a description.

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

### Common Pattern: Ensuring a Change Has a Description

```bash
# 1. Check current description
jj log -r @ -T description --no-graph

# 2. If empty, check which change has the work
jj status

# 3. Analyze the diff (use @- if @ is a placeholder)
jj diff -r @

# 4. Determine type from the diff, then set the description
jj describe -r @ -m "type: concise description"
```

## Common Patterns

### Starting New Work
```bash
jj tug                              # Sync with remote
jj new -m "feat: new feature"       # Start new change
# ... make changes ...
# Before pushing: run the Description Check Protocol if @- has no description
jj describe -r @- -m "feat: new feature"  # Ensure description is set
jj new                              # Finalize and start next
```

### Amending Current Change
Simply make changes—they're automatically included in `@`. Use `jj describe -r @ -m "type: message"` to update the message if needed. If the change has no description yet, run the **Description Check Protocol**.

### Rebasing onto Latest
```bash
jj tug    # Fetches and rebases in one command
```

### Viewing What Will Be Pushed
```bash
# Run the Description Check Protocol first — jj git push requires descriptions
jj log -r 'remote_bookmarks()..@'    # Changes not yet on remote
```

## Recovery

The operation log records every operation. Nothing is lost.

```bash
jj op log              # See all operations
jj undo                # Undo last operation
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
jj log                               # Other workspace's changes appear in shared history

# Clean up when done
jj workspace forget hotfix           # Run from any other workspace
rm -rf ../myproject-hotfix           # Remove the directory
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
