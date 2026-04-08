---
name: review-pr
description: "Use when given a PR number to review, or when reviewing someone else's pull request for code quality and requirements alignment."
disable-model-invocation: true
---

# Review PR

Review a pull request by first understanding the requirements (Jira ticket, TRD), then running a comprehensive multi-aspect code review.

## Usage

```
/review-pr <pr-number> [review-aspects]
```

## Arguments

| Arg | Required | Default | Description |
|-----|----------|---------|-------------|
| `<pr-number>` | Yes | -- | GitHub PR number (e.g., `9535`) |
| `[review-aspects]` | No | `all` | Space-separated review aspects: `comments`, `tests`, `errors`, `types`, `code`, `simplify`, `all` |

## Constants

| Key | Value |
|-----|-------|
| GitHub Repo | `bfi-finance/bravo-bpm-service` |
| GitHub Account | `bfi-roni-yusuf` (switch before `gh` commands) |
| GH_HOST prefix | `GH_HOST=github.com` (required for all `gh` commands) |

## Workflow

### Phase 1: Fetch PR Context

1. **Switch GitHub account** and fetch PR details:
   ```bash
   gh auth switch --user bfi-roni-yusuf 2>/dev/null
   GH_HOST=github.com gh pr view <pr-number> --repo bfi-finance/bravo-bpm-service
   ```

2. **Extract from PR output:**
   - **PR author** — the `author` field (e.g., `herfanh`, `selfhygtg`, `bfi-roni-yusuf`)
   - **Jira ticket ID** — look for patterns:
     - `https://bfifinance.atlassian.net/browse/BLCS-XXXX` (PR template JIRA link)
     - `https://bfifinance.atlassian.net/browse/BLOS-XXXX` (alternative project)
     - `BLCS-XXXX` or `BLOS-XXXX` mentioned in title, body, or branch name
     - Branch name pattern: `feat/BLCS-XXXX`, `fix/BLCS-XXXX`, `hotfix/BLCS-XXXX`

3. **Get the diff** for review scope:
   ```bash
   GH_HOST=github.com gh pr diff <pr-number> --repo bfi-finance/bravo-bpm-service --name-only
   ```

### Phase 2: Access PR Branch

**If PR author is `bfi-roni-yusuf` (own PR):**

The branch is already local. Simply checkout:
```bash
git checkout <pr-branch-name>
```
Use the main repo working directory for all subsequent file reads and agent work. Remember the **original branch** name so you can switch back in Phase 7.

**If PR author is someone else:**

Use a git worktree to isolate the PR branch — avoids disrupting the current working branch.
```bash
# Fetch the PR head into a local branch
git fetch origin pull/<pr-number>/head:pr-<pr-number>-review

# Create a worktree at /tmp for the review
git worktree add /tmp/pr-<pr-number>-review pr-<pr-number>-review
```
All subsequent file reads and agent work must use `/tmp/pr-<pr-number>-review/` as the working directory.

**Do NOT clean up here** — the worktree/branch is needed through Phase 6. Cleanup happens in Phase 7.

### Phase 3: Fetch Requirements, Then Launch Reviews

**Step 1 — Requirements context** (blocking, if ticket found). **Step 2 — Code review + subtask** (parallel, with requirements passed to review agents).

**Step 1 — Requirements context** (if ticket found):

The strategy depends on the PR author:

**Own PR** (author is `bfi-roni-yusuf`):
Read local docs first — they already contain distilled TRD contracts, acceptance criteria, and business rules from planning:
- Plan file at `docs/<ticket-id-lowercase>/plan.md` (or `plan-part*.md`)
- Analysis file at `docs/<ticket-id-lowercase>/analysis.md`
- Search project memory for the ticket ID in `MEMORY.md`

If neither plan nor analysis exists, fall back to full `/fetch-jira-context <ticket-id>` via a subagent.

**Other developer's PR:**
Launch a subagent (Agent tool, `model: "opus"`) with this prompt:
> Invoke `/fetch-jira-context <ticket-id>`. Return the full structured requirements summary as your final output.

Local docs are unavailable for others' PRs (gitignored, on the author's machine only), so full Jira/TRD fetch is required.

**Wait for Step 1 to complete** before proceeding. Store the requirements summary (acceptance criteria, TRD contracts, business rules) — it will be passed to review agents in Step 2.

**Step 2a — Code review** (launch in parallel with Step 2b):
Invoke all relevant review skills/agents based on the PR contents. Use the working directory determined in Phase 2 (main repo for own PR, worktree for others). Review agents use git diff against the base branch to identify changes.

**Pass requirements context to all review agents** — include the requirements summary from Step 1 in each agent's prompt. This enables agents to flag requirements mismatches (missing acceptance criteria, TRD contract violations, scope creep) alongside code quality issues.

**Always invoke:**
- `/pr-review-toolkit:review-pr [aspects]` — dispatches the requested review aspects (code quality, tests, errors, comments, types, simplify)

**Conditionally invoke** (in parallel with the above, as additional agents):
- **`migration-reviewer` agent** — if the PR changes any files in `src/main/resources/db/migration/`. Reviews Flyway SQL naming, soft-delete safety, index practices.
- **`implementation-reviewer` agent** — if an implementation plan exists in `docs/<ticket-id-lowercase>/` for the ticket (check for `plan.md` or `plan-part*.md`). Reviews implementation against the plan and TRD.

**Baseline skill checks (new code only):**
- Tests: new test classes use @ParameterizedTest where multiple cases exist, @Nested for logical grouping, @DisplayName for readability, descriptive methodName_should_X_when_Y naming
- Spring patterns: new services/controllers follow java-springboot patterns where bpm-conventions doesn't have an explicit override
- Javadoc: complex public methods have Javadoc, but boilerplate DTOs/getters/straightforward services don't have unnecessary documentation
- Don't flag existing code that doesn't follow baseline skills — only new/modified code in the PR diff

**Step 2b — Code Review subtask** (launch in parallel with Step 2a, if ticket found AND PR author ≠ `bfi-roni-yusuf`):
Launch a subagent (Agent tool, `model: "opus"`) with this prompt:
> Invoke `/fetch-jira-context <ticket-id> --lightweight`. From the lightweight output, determine the parent (if Sub-task, use its parent key; otherwise use the ticket itself) and check the subtasks list for an existing "Code Review" subtask assigned to Roni.
>
> **Constants:**
> - Assignee Account ID: `638eb37f77acd224b3428329` (Roni Yusuf)
>
> **Steps:**
> 1. Invoke `/fetch-jira-context <ticket-id> --lightweight` to get the ticket's type, parent, and subtasks.
> 2. Determine the parent: if type is `Sub-task`, use its parent key; otherwise use the ticket itself. If the ticket is a Sub-task, re-invoke `/fetch-jira-context <parent-key> --lightweight` to get the parent's subtasks.
> 3. Check the subtasks list — if any subtask summary contains "Code Review" **AND** is assigned to Roni (account ID `638eb37f77acd224b3428329`), report it already exists (include key and link) and stop. Other developers' Code Review subtasks do not count — Roni needs their own.
> 4. If no matching Code Review subtask exists, create one using MCP `createJiraIssue` directly with:
>    - cloudId: `74eaba08-c4fa-40de-a200-a44e70ca5dca`
>    - projectKey: `BLCS`
>    - issueTypeName: `Sub-task`
>    - summary: `Code Review`
>    - description: `Backend implementation`
>    - parentKey: (the parent determined in step 2)
>    - assigneeAccountId: `638eb37f77acd224b3428329`
> 5. **Transition the subtask to "In Progress"** (transition ID `21`) — review agents are already running.
> 6. Return the subtask key, link, and whether it was newly created or already existed.

**Skip conditions:**
- **No Jira ticket found** → skip Step 1 and Step 2b, run only code review (Step 2a without requirements context)
- **PR author is `bfi-roni-yusuf`** → skip Step 2b (reviewing own PR, no subtask needed)

### Phase 4: Requirements Alignment Check

Wait for all Step 2 agents to complete. Display the Jira requirements summary to the user, then compile a **requirements alignment** section. Review agents already had requirements context, so their findings may include requirements-related issues — incorporate those alongside your own cross-reference:

- **Completeness**: Are all acceptance criteria addressed by the PR?
- **Contract compliance**: Do APIs match TRD specs (paths, request/response shapes, status codes)?
- **Missing items**: Anything in the ticket/TRD not covered by the PR?
- **Scope creep**: Any changes that go beyond what the ticket asks for?

Format:
```markdown
## Requirements Alignment

### Acceptance Criteria Coverage
| Criteria | Status | Notes |
|----------|--------|-------|
| ... | Covered / Missing / Partial | ... |

### TRD Contract Compliance (if applicable)
| API | TRD Spec | PR Implementation | Match? |
|-----|----------|-------------------|--------|
| ... | ... | ... | Yes/No/Partial |

### Gaps & Concerns
- [List any missing items or mismatches]
```

### Phase 5: Verify and Discuss Findings 1-by-1

After presenting the summary, **deduplicate findings across all agents** (multiple agents often flag the same issue), then **verify each finding by reading the actual source files** — agents see truncated diffs and may produce false positives. Dismiss any findings that don't hold up against the real code.

**Behavioral claims require deeper verification**: When a finding claims "method X causes Y behavior" (e.g., "creates duplicates", "bypasses validation", "skips null check"), **read method X's implementation** — not just the changed lines. Agents infer behavior from method names and surrounding context but never read the called method's body. This is the most common source of false positives in correctness findings.

**IMPORTANT: Present findings ONE AT A TIME.** Do NOT list all findings in a single message. Present one finding, wait for the user's response, then present the next.

The framing depends on the PR author:

**Own PR** (author is `bfi-roni-yusuf`):
1. Present the finding briefly — severity, file:line, what's wrong, suggested fix.
2. Ask **fix or skip?**
3. **Wait for user response** before presenting the next finding.
4. After all findings discussed, compile the "fix" items as an action list.

**Other developer's PR:**
1. Present the finding briefly — severity, file:line, what's wrong, suggested fix.
2. Ask **include or skip?**
3. **Wait for user response** before presenting the next finding.
4. After all findings discussed, compile included ones for posting in Phase 6. If all skipped, go to Phase 6 with no inline comments.

### Phase 6: Submit GitHub PR Review

**Skip this phase if PR author is `bfi-roni-yusuf`** — reviewing own PR, no formal review submission needed.

For other developers' PRs, submit a formal PR review via the GitHub API. The review verdict depends on the discussion outcome:

| Outcome | Event | Description |
|---------|-------|-------------|
| All findings skipped (no comments) | `APPROVE` | PR looks good, no actionable issues |
| Some findings included, none critical | `COMMENT` | Non-blocking feedback |
| Critical findings included | `REQUEST_CHANGES` | Must fix before merge |

**Posting inline comments** (if any included findings):

- **Always use ` ```suggestion ` blocks** — never plain code blocks. This lets the author apply fixes with one click.
- Use `POST /repos/bfi-finance/bravo-bpm-service/pulls/<pr>/comments` endpoint for inline comments (NOT the `/reviews` body — suggestions in the review body don't render as clickable).
- For multi-line suggestions, use `start_line` + `line` + `start_side: "RIGHT"` + `side: "RIGHT"`.
- For new files, include `commit_id` and `side: "RIGHT"` — GitHub can't resolve paths without `commit_id`.
- **Verify before posting**: Read the actual source files from the working directory (main repo for own PR, worktree for others) to confirm findings — review agents see truncated diffs and may produce false positives.

**Submitting the review:**

```bash
GH_HOST=github.com gh api repos/bfi-finance/bravo-bpm-service/pulls/<pr>/reviews \
  -f event="APPROVE|COMMENT|REQUEST_CHANGES" \
  -f body="<optional summary>"
```

For `APPROVE` with no comments, a brief body like "LGTM" is sufficient. For `COMMENT` or `REQUEST_CHANGES`, summarize the key findings in the body.

### Phase 7: Finalize and Cleanup

After the review is fully complete (findings discussed 1-by-1, comments posted if any):

1. **Transition Code Review subtask to "Done"** (transition ID `31`) — the review cycle is complete. Skip if no subtask was created (own PR).
2. **Restore branch state:**
   - **Own PR:** Switch back to the original branch noted in Phase 2:
     ```bash
     git checkout <original-branch>
     ```
   - **Other's PR:** Remove the worktree and temp branch:
     ```bash
     git worktree remove /tmp/pr-<pr-number>-review && git branch -D pr-<pr-number>-review
     ```

**Always run cleanup before ending the review** — stale worktrees accumulate and open subtasks create noise.

## Output Format

The final output combines:
1. **PR Summary** — title, author, branch, files changed
2. **Code Review Subtask** — link to the Jira subtask (created or existing), or skipped if own PR
3. **Requirements Context** — Jira ticket summary and key requirements
4. **Code Review Results** — from pr-review-toolkit agents (Critical / Important / Suggestions / Strengths)
5. **Requirements Alignment** — completeness and contract compliance check

## Examples

```
/review-pr 9535
# Full review of PR #9535 — fetches Jira context, runs all review aspects

/review-pr 9535 code tests
# Reviews only code quality and test coverage

/review-pr 9535 simplify
# Only simplification review
```

## Tips

- Run this on PRs from other developers to review their work with full context
- The Jira/TRD fetch runs in parallel with code review — no added wait time
- If the PR has no Jira link, the skill still works — it just skips requirements alignment
- For your own PRs, run before requesting human review to catch issues early

## Workflow Tracking

**REQUIRED SUB-SKILL:** After Phase 7 completes (review finalized and branch state restored), use `update-workflow` to:
1. Mark phase `review_pr` as `done`.
2. If **zero findings were included** in the review (all findings skipped, or no findings at all) → also mark `address_review` as `skipped` with reason `"No review findings"`.

This only applies when reviewing your **own PR** (author is `bfi-roni-yusuf`). When reviewing others' PRs, don't update the workflow tracker — it's their workflow, not yours.
