---
name: pull-latest
description: "Pull latest default branch (master/main), merge into current feature branch, prune remotes, and clean up [gone] branches. Usage: /pull-latest"
---

# Pull Latest

Sync local workspace with the remote default branch, merge into your feature branch if applicable, and clean up stale branches.

## Steps

### 1. Detect default branch

Determine whether the remote uses `master` or `main`. Check for the existence of well-known branch names first (fast, no network):

```bash
git show-ref --verify --quiet refs/remotes/origin/master && echo master || \
  (git show-ref --verify --quiet refs/remotes/origin/main && echo main)
```

If neither exists, fall back to `origin/HEAD` (which can be misconfigured — e.g., pointing to a release branch):

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'
```

Store the result (e.g., `master` or `main`) — use it everywhere below as `<default>`.

### 2. Record current branch

```bash
git branch --show-current
```

Store as `<original>`. This determines whether to merge later.

### 3. Stash if dirty

Check for uncommitted changes:

```bash
git status --porcelain
```

- If there is output → run `git stash push -m "pull-latest auto-stash"` and remember that a stash was made
- If clean → skip, no stash needed

### 4. Checkout default branch and pull

```bash
git checkout <default> && git pull origin <default>
```

If pull fails (e.g., merge conflict on default branch), report the error and stop.

### 5. Fetch with prune

```bash
git fetch --prune
```

This removes remote tracking refs for branches deleted on the remote.

### 6. If was on a feature branch — return and merge

**Skip this step if `<original>` equals `<default>`.**

```bash
git checkout <original>
git merge <default>
```

- If merge conflict → **stop and notify the user**. Do not auto-resolve. The user needs to handle merge conflicts manually.
- If merge succeeds → continue

### 7. Pop stash if stashed

**Skip if no stash was made in step 3.**

```bash
git stash pop
```

- If stash pop conflicts → warn the user that their local changes conflict with the merged changes. They need to resolve manually.

### 8. Clean up stale branches

Find and delete local branches whose remote tracking branch is gone, plus orphaned PR review branches:

```bash
# [gone] branches — remote was deleted
git branch -vv | grep ': gone]' | awk '{print $1}'

# PR review branches — temporary, no remote tracking
git branch --format='%(refname:short) %(upstream)' | awk '$2 == "" && $1 ~ /^pr-.*-review$/ {print $1}'
```

- Run both commands and combine the results
- If no matches → report "No stale branches to clean up"
- For each match, run `git branch -D <branch>`. **Never delete the current branch.**
- Report which branches were deleted and why (gone vs PR review)

## Output

After completing all steps, provide a brief summary:

- Default branch pulled (how many commits behind it was, if visible from pull output)
- Whether merge into feature branch was performed
- Number of pruned remote refs (if any from fetch --prune output)
- Branches deleted (list names) or "no stale branches"
