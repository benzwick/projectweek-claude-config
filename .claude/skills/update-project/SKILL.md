---
name: update-project
description: Update an existing project page - creates branch, helps with edits, creates PR when ready
argument-hint: "[project-name] [action]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - Bash
---

# Update Project

This skill helps update an existing NA-MIC Project Week project page.

## Usage

```
/update-project ProjectName start    # Start editing (creates branch)
/update-project ProjectName          # Continue editing (shows current state)
/update-project ProjectName finish   # Commit, push, and create PR
```

## Actions

### `start` - Begin editing session

1. Find the project in the current Project Week directory
2. Create a feature branch:
   ```bash
   git checkout master
   git pull upstream master  # sync first
   git checkout -b PW##_YEAR_Location/ProjectName-updates
   ```
3. Show current project README for reference
4. Ready for user to request changes

### (no action) - Continue editing

1. Check we're on the correct branch
2. Show current state of project README
3. Show uncommitted changes if any
4. Ready for user to request more changes

### `finish` - Complete and create PR

1. Show diff of all changes made
2. Ask user to confirm
3. Commit changes:
   ```bash
   git add PW##_YEAR_Location/Projects/ProjectName/
   git commit -m "PW## ProjectName: Update <summary of changes>"
   ```
4. Push branch:
   ```bash
   git push -u origin PW##_YEAR_Location/ProjectName-updates
   ```
5. Check if PR already exists for this branch (open OR closed):
   ```bash
   FORK_OWNER=$(git remote get-url origin | sed 's/.*[:/]\([^/]*\)\/ProjectWeek.*/\1/')
   BRANCH=$(git branch --show-current)
   # Check for open PR
   gh pr list --repo NA-MIC/ProjectWeek --head "$FORK_OWNER:$BRANCH" --json url,state
   # Also check for closed PRs
   gh pr list --repo NA-MIC/ProjectWeek --head "$FORK_OWNER:$BRANCH" --state closed --json url,state,mergedAt
   ```
6. Handle existing PRs:
   - **Open PR exists**: Just show the URL, no need to create a new one
   - **Merged PR exists**: Inform user the changes are already in master
   - **Closed (not merged) PR exists**: Create a NEW branch with `-v2` suffix (or increment existing version),
     push it, and create a fresh PR. This avoids issues with stale branch references:
     ```bash
     NEW_BRANCH="${BRANCH}-v2"  # or -v3, etc. if -v2 exists
     git checkout -b "$NEW_BRANCH"
     git push -u origin "$NEW_BRANCH"
     # Then create PR with new branch
     ```
8. Create PR if none exists (IMPORTANT: must use --head flag for fork-based PRs):
   ```bash
   gh pr create \
     --repo NA-MIC/ProjectWeek \
     --base master \
     --head "$FORK_OWNER:$BRANCH" \
     --title "PW## ProjectName: <brief summary of updates>" \
     --body "## Summary
   - <bullet points describing changes made>"
   ```

   **IMPORTANT:** This is updating an EXISTING project, not adding a new one. The PR title should describe
   what was updated (e.g., "PW44 SlicerCBM: Update progress and add screenshots"), NOT "Add project".

9. Return PR URL to user

## Finding the project

Search in order:
1. Current PW directory: `PW##_YEAR_Location/Projects/ProjectName/`
2. Case-insensitive glob if exact match not found
3. Show options if multiple matches

## Tips

- User can make multiple rounds of changes before `finish`
- If user wants to abandon changes: `git checkout master` discards uncommitted work
- Branch naming includes `-updates` suffix to distinguish from new project branches
- Commit message should summarize what was actually changed (objectives, progress, etc.)

## Handling Closed PRs

If a PR was closed without merging (e.g., due to merge conflicts or reviewer requested changes):

1. **Sync master with upstream first:**
   ```bash
   git fetch upstream
   git checkout master
   git reset --hard upstream/master
   ```

2. **Create a fresh branch from updated master:**
   ```bash
   git checkout -b PW##_YEAR_Location/ProjectName-updates-v2
   ```

3. **Cherry-pick or reapply the changes** from the old branch, or if the old branch has the changes:
   ```bash
   git cherry-pick <commit-hash>  # for each commit
   # OR rebase if preferred
   ```

4. **Push and create new PR** with the fresh branch

This ensures the new PR is based on the latest master and avoids stale reference issues.
