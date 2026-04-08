---
name: create-pr
description: "Use when creating a pull request for the current feature branch — auto-generates title and body from available context."
disable-model-invocation: true
---

# Create PR

Create a draft PR with an auto-generated title and body by gathering context from multiple sources.

## Usage

```
/create-pr [TICKET-ID]
```

## Arguments

| Arg | Required | Default | Description |
|-----|----------|---------|-------------|
| `[TICKET-ID]` | No | Parsed from branch name | Override ticket ID (e.g., `BLCS-1234`) |

## Constants

| Key | Value |
|-----|-------|
| GitHub Repo | `bfi-finance/bravo-bpm-service` |
| GitHub Account | `bfi-roni-yusuf` (switch before `gh` commands) |
| GH_HOST prefix | `GH_HOST=github.com` (required for all `gh` commands) |
| PR Base Branch | `master` |
| PR Template | `.github/pull_request_template.md` |

## Information Priority

When generating the PR body, resolve conflicts using this priority (highest first):

1. **Changes (git diff)** — ground truth of what was implemented
2. **Plan (`docs/<ticket-id lowercase>/`)** — structured intent and rationale
3. **Memory (project memory entries)** — post-plan decisions and pivots

Generation sequence is plan-first (backbone) → memory (amendments) → changes (validation). Priority determines which source wins when they disagree.

## Workflow

### Phase 1: Validate State

1. Get current branch name:
   ```bash
   git branch --show-current
   ```
2. **Abort** if branch is `master`, `main`, or starts with `release-`
3. Check for uncommitted changes:
   ```bash
   git status --porcelain
   ```
   If there are staged or unstaged changes, **warn** the user: "You have uncommitted changes. The PR will only include committed code. Continue?" If they decline, abort.
4. Check commits ahead of master:
   ```bash
   git log origin/master..HEAD --oneline
   ```
   **Abort** if no commits found.
5. Check if branch is pushed. If not, or if local is ahead of remote, **auto-push**:
   ```bash
   git push -u origin HEAD
   ```
6. Check for an existing PR on this branch:
   ```bash
   GH_HOST=github.com gh pr list --head <branch-name> --repo bfi-finance/bravo-bpm-service --state open
   ```
   If a PR already exists, display its URL and **abort** — no duplicate PR needed.

### Phase 2: Extract Ticket ID & PR Type

1. Parse the branch name (e.g., `feat/BLCS-1234`):
   - **Ticket ID**: everything after the prefix slash (e.g., `BLCS-1234`)
   - **Type**: the prefix before the slash

2. Branch prefix → commit type mapping:

   | Prefix | Type |
   |--------|------|
   | `feat/` | `feat` |
   | `fix/` | `fix` |
   | `hotfix/` | `fix` |
   | `chore/` | `chore` |

3. If argument was provided, use it as the ticket ID (overrides branch parsing)
4. If no ticket ID found from either source → **abort** with error
5. **Infer scope** from changed files:
   ```bash
   git diff origin/master...HEAD --name-only
   ```
   Extract the domain from each changed Java file:
   - Files in domain-specific subdirectories (`dto/underwriting/`, `entity/underwriting/`, `adapter/http/underwriting/`, `activity/underwriting/`) → use the subdirectory name (e.g., `underwriting`)
   - Files in flat packages (`controller/`, `service/`, `service/impl/`) → use the class name prefix (e.g., `UnderwritingReviewController` → `underwriting`)
   - Files in `connector/` → scope `connector`

   Use the most common domain across all changed files. If domains are evenly split, use the Jira ticket's component or the one that appears in the most non-test files.

### Phase 3: Gather Context

Read all of these in parallel:
- `git diff origin/master...HEAD --stat` — file change summary
- `git log origin/master..HEAD --oneline` — commit list
- Plan file at `docs/<TICKET-ID lowercase>/plan.md` (or `plan-part*.md`) — read if exists, skip if not
- Analysis file at `docs/<TICKET-ID lowercase>/analysis.md` — read if exists, skip if not
- Workflow tracker at `docs/<TICKET-ID lowercase>/workflow.yml` — read if exists, used for "Manual tested" checkbox
- Search project memory for the ticket ID: grep `MEMORY.md` for the ticket ID (e.g., `BLCS-1234`), then read any linked memory files. Memory files are at the path shown in `MEMORY.md` links (relative to the memory directory). If no files match, skip — memory is optional context.

**Jira fallback (only if no plan or analysis found):** If neither plan nor analysis doc exists, invoke `/fetch-jira-context <TICKET-ID> --lightweight` to get the ticket summary and parent for the PR title. Local docs already contain the ticket summary and parent info from planning — re-fetching from Jira is redundant when they exist.

### Phase 4: Generate PR Title & Body

**Title:** `type(scope): [TICKET-ID] description`
- `type` and `scope` from Phase 2
- `description`: use Jira ticket summary. If the ticket is a subtask with a generic name, use the **parent story summary** instead. Generic names: summaries that are only role labels like "BE Implementation", "FE Implementation", "QA Testing", "BE", "FE", or contain no domain-specific words. Summaries with bracketed prefixes like `[BE] [UW 4W] Save CFCAT` are NOT generic — strip the brackets and use the core description. Lowercase the first word to match conventional commit style.

**Body:** Fill the repo's PR template structure. Replace the template's default `BLOS-` prefix with the actual ticket ID.

Generate the "Description & Technical Solution" section using this sequence:
1. Start with the **plan** as the backbone — summarize key implementation steps and technical approach
2. Amend with **memory** — note any post-plan decisions, resolved blockers, or scope changes
3. Validate against **changes** — ensure the description matches what was actually implemented. Drop any plan steps that weren't implemented; add any changes not in the plan.

**Checkbox auto-check rules:**

"Type of change" — check exactly one based on branch prefix:

| Prefix | Checkbox |
|--------|----------|
| `feat/` | New feature |
| `fix/`, `hotfix/` | Bug fix |
| `chore/` | Enhancement |

"How Has This Been Tested?" — check "New unit tests added" only if `*Test.java` files appear in the diff. Check "Manual tested" only if the workflow tracker (`docs/<TICKET-ID lowercase>/workflow.yml`) has phase `test` with `status: done` or `status: skipped`; leave unchecked if `pending` or tracker doesn't exist.

"Checklist" — check both items.

### Phase 5: Preview & Confirm

Display the complete PR as it will appear:

```
Title: feat(underwriting): [BLCS-1234] add CFCAT list asset endpoint

Body:
─────────────────────────────────────────
JIRA link: https://bfifinance.atlassian.net/browse/BLCS-1234

## Description & Technical Solution
...

## Type of change
...
─────────────────────────────────────────

Create this draft PR? (yes/no)
```

If **yes** → proceed to Phase 6.
If **no** → abort. User can edit and retry, or create manually in GitHub.

### Phase 6: Create Draft PR

```bash
gh auth switch --user bfi-roni-yusuf 2>/dev/null
GH_HOST=github.com gh pr create --draft \
  --title "<generated-title>" \
  --body-file /tmp/pr-body.md \
  --base master \
  --repo bfi-finance/bravo-bpm-service
```

Write the body to a temp file (`/tmp/pr-body.md`) to avoid shell escaping issues with the `--body` flag.

Display the PR URL after creation. Clean up the temp file.

## Examples

```
/create-pr
# Parses ticket from branch name, gathers context, generates PR

/create-pr BLCS-1234
# Overrides ticket ID (useful for unusual branch names)
```

## Workflow Tracking

**REQUIRED SUB-SKILL:** After Phase 6 completes (draft PR created successfully), use `update-workflow` to:
1. Mark phase `create_pr` as `done`.
2. Extract the PR number from the `gh pr create` output and set `review_pr.pr_number` in the tracker.
