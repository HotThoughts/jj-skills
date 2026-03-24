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
| Forgot to start with jj new | `jj describe -m "what I'm doing"` (do this immediately) |
| Work is done, move on | `jj new -m "next task"` |
| Annotate what you did | `jj describe -m "feat: auth"` |
| Broke something | `jj op log` → `jj op restore <id>` |
| Undo one file | `jj restore --from @- <path>` |
| Exclude file from current change | `jj restore --from @- <path>` |
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
jj new                           # Create a new empty change on top of @
jj describe -m "feat: message"   # Set commit message for current change
jj new -m "feat: message"        # Create new change with message
```

### Syncing with Remote

```bash
jj tug             # Fetch updates and rebase current change onto latest remote
jj git fetch       # Fetch from remote without rebasing
jj git push        # Push current changes to remote
```

### Modifying History

```bash
jj squash                        # Squash current change into parent
jj squash --into @-              # Explicitly squash into parent
jj restore --from @- <path>      # Remove a file from current change (non-interactive alternative to split)
jj edit <change-id>              # Edit an existing change
```

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
jj describe -m "feat: add user authentication"
jj describe -m "fix: resolve null pointer in parser"
jj describe -m "refactor: extract validation logic"
```

## Common Patterns

### Starting New Work
```bash
jj tug                              # Sync with remote
jj new -m "feat: new feature"       # Start new change
# ... make changes ...
jj new                              # Finalize and start next
```

### Amending Current Change
Simply make changes—they're automatically included in `@`. Use `jj describe` to update the message if needed.

### Rebasing onto Latest
```bash
jj tug    # Fetches and rebases in one command
```

### Viewing What Will Be Pushed
```bash
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
