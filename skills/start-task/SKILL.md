---
name: start-task
description: "Use when starting a new task from a Jira ticket — switching branches, setting up workspace, and beginning plan creation."
disable-model-invocation: true
---

# Start Task

Set up workspace for a new Jira ticket and kick off plan creation.

## Usage

```
/start-task <ticket-id>
```

Or natural language: "lets move on to new task BLCS-1234", "start working on BLCS-1234", "new task BLCS-1234"

## Steps

### 1. Check for dirty working tree

Run `git status --porcelain`. If there are uncommitted changes, **warn the user and stop**. Do not auto-stash — let the user decide how to handle it.

### 2. Pull latest master and checkout

```bash
git fetch origin master:master && git checkout master
```

This updates the local `master` ref to match `origin/master` without needing to be on master first, then checks it out. If the fast-forward fetch fails (e.g., local master has diverged), fall back to:

```bash
git checkout master && git pull origin master
```

### 3. Fetch Jira ticket and determine branch prefix

**REQUIRED SUB-SKILL:** Use `fetch-jira-context` with `--lightweight` to fetch the ticket. Extract the **issue type** from the lightweight output and map to branch prefix:

| Issue Type | Prefix |
|-----------|--------|
| Bug | `fix/` |
| Story, Sub-task, Task | `feat/` |

Exception: if the user explicitly said "hotfix", use `hotfix/` regardless of issue type.

### 4. Create branch

```bash
git checkout -b {prefix}{TICKET-ID}
```

Example: `git checkout -b feat/BLCS-1234`

**If branch already exists**: ask the user whether to switch to it (`git checkout`) or delete and recreate.

### 5. Dispatch plan creator

Launch the **plan-creator agent** as a subagent:

- `subagent_type`: "plan-creator"
- `model`: "opus"
- Pass the ticket ID and instruct it to follow the `create-plan` skill
- The agent will produce the plan in `docs/<ticket-id-lowercase>/`

While the subagent runs, inform the user that planning is in progress and they'll see the result when it completes.

## Notes

- This skill composes `fetch-jira-context --lightweight` (for branch prefix) and `fetch-jira-context` full mode (via the plan-creator) with git operations
- Branch names use ticket ID only — no description suffix (e.g., `feat/BLCS-1234`, not `feat/BLCS-1234/some-description`)
- The plan creator subagent runs autonomously — keeps the main conversation context clean

## Workflow Tracking

**REQUIRED SUB-SKILL:** After Step 4 completes (branch created), use `update-workflow` to:
1. Create the tracker if it doesn't exist (this skill is an initialization point)
2. Mark phase `start` as `done`

Note: The plan-creator subagent dispatched in Step 5 runs independently — `create-plan` will handle its own tracker update for multi-part expansion when it finishes.
