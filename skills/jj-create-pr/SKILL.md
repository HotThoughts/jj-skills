---
name: jj-create-pr
description: Creates GitHub pull requests from Jujutsu changes with AI-generated descriptions. Use when the user wants to create a PR, push changes for review, or open a pull request.
---

# Create GitHub PR from Jujutsu Change

This skill enables creating GitHub pull requests from jj changes with automatically generated PR descriptions based on the diff.

## Permission Requirements

**CRITICAL**: This workflow requires `jj` and `gh` CLI access with authentication. Always use:

```
required_permissions: ["all"]
```

## Workflow

When the user asks to create a PR (e.g., "create a PR", "push for review", "open PR for @-"):

### Step 1: Identify the Change

Default to `@-` (parent of working copy) unless the user specifies a different change.

```bash
# Get the change ID being pushed
jj log -r <change> --no-graph -T 'change_id ++ "\n"' | head -1
```

### Step 2: Get PR Title

Extract the first line of the change's description:

```bash
jj log -r <change> -T description --no-graph | head -1
```

If the description is empty, run the **Description Check Protocol** (see `jj-workflow` skill) to generate a conventional commit description from the diff, then re-extract the title:

1. Check `jj status` — if `@` is a placeholder (no modified files), evaluate `@-` carefully: verify it has a non-empty diff AND no trunk bookmarks (`main`, `master`, `main@origin`) before targeting it. If `@-` carries a trunk bookmark, ask the user which change to use.
2. Analyze `jj diff -r <change>` to understand what changed
3. Determine the conventional commit type (feat/fix/refactor/perf/docs/chore/test) from the diff
4. Set the description: `jj describe -r <change> -m "type: concise description"` (or use `--stdin` if preserving an existing body — see jj-workflow protocol)
5. Re-run the title extraction command to get the exact first line

### Step 3: Analyze the Diff

Get the diff to understand what changed:

```bash
jj diff -r <change>
```

### Step 4: Generate PR Description

Based on the diff, write a concise PR description:

- **Summary**: One sentence describing the overall change
- **Changes**: Bullet points of key modifications
- Keep it brief and focused on "what" and "why"
- Do not list every line change—summarize meaningfully

Example format:
```markdown
## Summary
Brief description of what this PR accomplishes.

## Changes
- Added X to handle Y
- Refactored Z for better performance
- Fixed bug where A caused B
```

### Step 5: Push the Change

First check if the change already has a named bookmark:

```bash
jj log -r <change> -T 'bookmarks' --no-graph
```

If no bookmark exists, create one by slugifying the change description (e.g. `"feat: add auth"` → `feat-add-auth`):

```bash
jj bookmark create <slug> -r <change>
jj git push --bookmark <slug>
```

Fallback — let jj auto-create a `push-<hash>` bookmark:

```bash
jj git push -c <change>
```

Parse the branch name from output. Look for patterns:
- `Creating bookmark <branch>` or `Add bookmark <branch>`
- `Move sideways bookmark <branch> from`

### Step 6: Get Default Branch

```bash
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'
```

Fallback to `main` if unavailable.

### Step 7: Create the PR

```bash
gh pr create \
  --base <default_branch> \
  --head <branch_name> \
  --title "<pr_title>" \
  --body "<generated_description>" \
  --assignee @me
```

## Complete Example

User: "Create a PR for @-"

```bash
# 1. Get title (generates one from diff if description is empty — see Step 2)
jj log -r @- -T description --no-graph | head -1
# Output: "feat: add user authentication"
# If empty: run the Description Check Protocol to generate one, then re-extract

# 2. Get diff
jj diff -r @-
# (analyze the output)

# 3. Check for existing bookmark, create one if missing
jj log -r @- -T 'bookmarks' --no-graph
# Output: (empty — no bookmark yet)
jj bookmark create feat-add-user-authentication -r @-
jj git push --bookmark feat-add-user-authentication
# Output: "Add bookmark feat-add-user-authentication..."

# 4. Get default branch
gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'
# Output: "main"

# 5. Create PR with generated description
gh pr create \
  --base main \
  --head feat-add-user-authentication \
  --title "feat: add user authentication" \
  --body "## Summary
Adds JWT-based user authentication to the API.

## Changes
- Added auth middleware for token validation
- Created login and register endpoints
- Added user model with password hashing" \
  --assignee @me
```

## PR Description Guidelines

When generating the description:

1. **Be concise**: 3-5 bullet points max for most PRs
2. **Focus on impact**: What does this change enable or fix?
3. **Skip obvious details**: Don't mention formatting or trivial changes
4. **Use present tense**: "Adds X" not "Added X"
5. **Group related changes**: Combine related file changes into one point

## Error Handling

- If push fails, report the error and stop
- If branch name cannot be determined, show the push output for debugging
- If PR creation fails (e.g., PR already exists), suggest `gh pr view` to see existing PR
