---
name: jj-workflow
description: Guides Claude on using Jujutsu (jj) version control system. Use when working with jj repositories, making commits, syncing changes, or managing version control workflows.
---

# Jujutsu (jj) Workflow

Jujutsu is a modern version control system that provides a simpler mental model than Git while remaining Git-compatible. This skill covers the core concepts and workflow commands.

## Core Concepts

### Changes vs Commits
- **Change**: A mutable revision identified by a change ID (e.g., `kkmpptxz`). Changes can be modified.
- **Commit**: An immutable snapshot identified by a commit ID (SHA). Once created, commits are permanent.
- The working copy (`@`) is always a change that can be modified freely.

### Working Copy
- The working copy is denoted by `@` and represents your current state.
- `@-` refers to the parent of the working copy.
- Unlike Git, there's no staging area—all changes are automatically tracked.

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
jj squash                    # Squash current change into parent
jj squash --into @-          # Explicitly squash into parent
jj split <file> -m "msg"     # Split specific files into their own commit
jj edit <change-id>          # Edit an existing change
```

### Working with Branches

```bash
jj branch create <name>      # Create a branch pointing to @
jj branch set <name>         # Move branch to current change
jj branch list               # List all branches
```

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
jj log -r 'remote_branches()..@'    # Changes not yet on remote
```

## Key Differences from Git

1. **No staging area**: All changes are tracked automatically
2. **Mutable working copy**: The current change can always be modified
3. **Change IDs**: Stable identifiers that persist through rebases
4. **Anonymous branches**: You can work without named branches
5. **Automatic conflict handling**: Conflicts are recorded and can be resolved later
