---
name: jj-update-pr
description: Updates existing GitHub pull request descriptions with AI-generated content based on the current diff. Use when the user wants to update a PR description, refresh PR content, or sync PR with latest changes.
---

# Update GitHub PR Description

This skill enables updating existing pull request descriptions with fresh AI-generated content based on the current state of the change.

## Permission Requirements

**CRITICAL**: This workflow requires `jj` and `gh` CLI access with authentication. Always use:

```
required_permissions: ["all"]
```

## Workflow

When the user asks to update a PR description (e.g., "update the PR", "refresh PR description", "sync PR with changes"):

### Step 1: Identify the Change

Default to `@-` (parent of working copy) unless the user specifies a different change.

### Step 2: Get the Branch Name

```bash
jj log -r <change> -T 'bookmarks' --no-graph
```

### Step 3: Verify PR Exists

```bash
gh pr view <branch>
```

If no PR exists, suggest using the `jj-create-pr` workflow instead.

### Step 4: Get Updated Title

Extract the first line of the change's description:

```bash
jj log -r <change> -T description --no-graph | head -1
```

### Step 5: Analyze the Current Diff

```bash
jj diff -r <change>
```

### Step 6: Generate Updated PR Description

Based on the diff, write a fresh PR description:

- **Summary**: One sentence describing the overall change
- **Changes**: Bullet points of key modifications
- Keep it brief and focused on "what" and "why"

Example format:
```markdown
## Summary
Brief description of what this PR accomplishes.

## Changes
- Added X to handle Y
- Refactored Z for better performance
- Fixed bug where A caused B
```

### Step 7: Update the PR

Update both title and body:

```bash
gh pr edit <branch> --title "<new_title>" --body "<new_description>"
```

Or update body only:

```bash
gh pr edit <branch> --body "<new_description>"
```

## Complete Example

User: "Update the PR for @-"

```bash
# 1. Get branch name
jj log -r @- -T 'bookmarks' --no-graph
# Output: "push-abc123"

# 2. Verify PR exists
gh pr view push-abc123
# (shows PR details)

# 3. Get updated title
jj log -r @- -T description --no-graph | head -1
# Output: "feat: add user authentication"

# 4. Get current diff
jj diff -r @-
# (analyze the output)

# 5. Update PR with new description
gh pr edit push-abc123 \
  --title "feat: add user authentication" \
  --body "## Summary
Adds JWT-based user authentication to the API.

## Changes
- Added auth middleware for token validation
- Created login and register endpoints
- Added user model with password hashing
- Updated API documentation"
```

## When to Use

- After making additional changes to a commit that already has a PR
- When the original PR description is outdated or incomplete
- After rebasing or squashing changes
- When the user asks to "refresh" or "update" the PR

## Error Handling

- If no bookmark/branch exists on the change, the PR may not be findable—ask user for PR number
- If PR doesn't exist, suggest creating one with `jj-create-pr` workflow
- If update fails, show the error and suggest checking PR status with `gh pr view`
