---
name: create-jira-task
description: "Use when creating a Jira issue (subtask, task, production issue, bug, story). Supports natural language, smart defaults, and confirmation before creation."
disable-model-invocation: false
---

# Create Jira Task

Create a Jira issue of any type — subtasks, standalone tasks, production issues, bugs, stories, etc. Supports both structured flags and natural language input.

## Usage

```
/create-jira-task <args>
```

### Structured Format

```
/create-jira-task <ticket-id> <type> [summary]              # Subtask mode
/create-jira-task <issue-type> <summary> [--flags]           # Standalone mode
```

### Natural Language

The skill also accepts freeform input. Claude will parse the intent and confirm before creating. Examples:

```
/create-jira-task create a production issue about OOM error on scoring, assign to herfan
/create-jira-task BLCS-2370 needs a code review subtask
/create-jira-task bug: login fails on mobile, assign to selfhy, keep as todo
```

## Flags (all optional)

| Flag | Default | Description |
|------|---------|-------------|
| `--parent <ticket>` | none | Creates as subtask under this ticket |
| `--assignee <name>` | `roni` | Assignee name (resolved via team directory + API) |
| `--status <status>` | `todo` | Target status after creation |
| `--prefix <tags>` | `[BE]` | Prefix tags for subtask summaries (subtask mode only). Overrides default when parent has no `[bracketed]` tags. Use `""` to omit. |

## Constants

| Key | Value |
|-----|-------|
| Cloud ID | `74eaba08-c4fa-40de-a200-a44e70ca5dca` |
| Project Key | `BLCS` |
| Default Assignee Account ID | `638eb37f77acd224b3428329` (Roni Yusuf) |

### Issue Type IDs

| Issue Type | ID |
|------------|----|
| Epic | `10000` |
| Story | `10012` |
| Task | `10045` |
| Sub-task | `10046` |
| Bug | `10060` |
| Story Bug | `10354` |
| Production Issue | `10797` |

## Steps

### 1. Parse the input

Determine from the user's input (structured or natural language):

- **Issue type** — e.g., `Sub-task`, `Production Issue`, `Task`, `Story`, `Bug`
- **Summary** — the title/description text
- **Parent ticket** — if provided via `--parent` flag or if the first arg is a ticket ID (e.g., `BLCS-2370`)
- **Assignee** — from `--assignee` flag or natural language (default: `roni`)
- **Status** — from `--status` flag or natural language (default: `todo`)

**Mode detection:**
- If a Jira ticket ID is the first argument (pattern: `[A-Z]+-\d+`) → **Subtask mode** (issue type defaults to `Sub-task`, ticket is the parent context)
- If `--parent <ticket>` is provided (even without a ticket ID as first arg) → **Subtask mode** (issue type taken from input, parent from flag)
- Otherwise → **Standalone mode** (issue type taken from input, no parent, skip step 3)

If the issue type is ambiguous (e.g., user says "issue" without specifying Task vs Bug), ask the user to clarify before proceeding to the confirmation step.

### 2. Resolve the assignee

Read `.claude/skills/create-jira-task/team-directory.md` for known name→account ID mappings.

1. **Case-insensitive match** in the team directory table
2. If not found → call `lookupJiraAccountId` with cloud ID `74eaba08-c4fa-40de-a200-a44e70ca5dca` and the provided name
3. If API lookup succeeds with a single match → **auto-append** the new `| name | account-id |` row to `team-directory.md` using the Edit tool
4. If API lookup returns multiple matches → present the options to the user and let them pick
5. If API lookup fails → report the error and ask the user for the correct name

### 3. Fetch parent and check for duplicates (subtask mode only)

**REQUIRED SUB-SKILL:** Use `fetch-jira-context` with `--lightweight` to fetch the ticket. Extract from the lightweight output: issue type, parent key/summary, and subtasks list.

- If the ticket is not found → report error and stop
- If the ticket's type is `Sub-task` → use its parent key as the parent. If parent is also a Sub-task, report nested subtasks are not supported and stop. **Re-fetch the parent** with `fetch-jira-context --lightweight` to get its summary (for prefix tags in step 4) and subtasks (for duplicate check).
- If the ticket is a Story/Task → use the ticket itself as parent

**Duplicate check:** After resolving the parent, inspect the parent's subtasks from the lightweight output. If any existing subtask's summary matches the type being created (e.g., contains "Code Review" when creating a Code Review subtask), report it already exists (include key and link) and ask the user whether to proceed or skip.

### 4. Draft the summary

**Subtask mode:** Extract all `[bracketed]` prefix tags from the parent ticket's summary. Build: `<prefix tags> <type> - <remaining summary>`. If no `[bracketed]` tags exist, prepend the default prefix `[BE]` (or a user-provided `--prefix` value): `<prefix> <type> - <original summary>`.

The `--prefix` flag overrides the default `[BE]` prefix (e.g., `--prefix "[FE]"`, `--prefix "[BE][UW]"`). Pass `--prefix ""` to omit the prefix entirely.

Example: Parent = `[BE][UW 4W][PT][Review Aplikasi] Add Detail Asset API`, type = `Code Review`
Result = `[BE][UW 4W][PT][Review Aplikasi] Code Review - Add Detail Asset API`

Example (no tags): Parent = `Fix OOM in scoring service`, type = `Code Review`
Result = `[BE] Code Review - Fix OOM in scoring service`

Example (custom prefix): Parent = `Fix OOM in scoring service`, type = `Code Review`, `--prefix "[FE]"`
Result = `[FE] Code Review - Fix OOM in scoring service`

**Standalone mode:** Format as `[<domain>] <summary>`. The `<domain>` is a short tag representing the domain/scope (e.g., `UW` for underwriting, `Scoring`, `Operation`). Infer the domain from the user's description or ask if ambiguous.

Example: domain = underwriting, summary = `Negotiation APIs fail when summary not created`
Result = `[UW] Negotiation APIs fail when summary not created`

### 5. Confirm with user

Present a confirmation table:

| Field | Value |
|-------|-------|
| **Issue Type** | e.g., `Production Issue` |
| **Summary** | the drafted summary |
| **Parent** | parent ticket key (or "None — standalone") |
| **Assignee** | resolved name |
| **Target Status** | e.g., `TODO` |
| **Priority** | (Production Issue only) e.g., `High` |
| **Labels** | (Production Issue only) e.g., `26#1Quartal` |
| **Root Cause** | (Production Issue only) e.g., `Missed Error Handling on Development` |

Ask: "Does this look right? I'll create it once you confirm."

**Do NOT proceed until the user confirms.** If the user wants changes, adjust and re-confirm.

### 6. Create the issue

Use `createJiraIssue` with:
- `cloudId`: `74eaba08-c4fa-40de-a200-a44e70ca5dca`
- `projectKey`: `BLCS`
- `issueTypeName`: the resolved issue type (e.g., `Sub-task`, `Production Issue`, `Task`)
- `parent`: parent ticket key (only for subtasks)
- `summary`: confirmed summary
- `description`: contextual description (e.g., `<type> for <ticket-id> - <original summary>` for subtasks, or `Created via Claude Code` for standalone)
- `assignee_account_id`: resolved account ID

**Production Issue special field:** When `issueTypeName` is `Production Issue`, the required custom field `customfield_10225` (Root Cause Category) must be set via `additional_fields`. Ask the user which root cause applies, or infer from context. Common values:

| Value | ID |
|-------|----|
| Missed Error Handling on Development | `10603` |
| Missed on Development | `10602` |
| Missing/Incorrect Requirements Technical | `10988` |
| Missing/Incorrect Requirements Functional | `10987` |
| Data Issues | `10608` |
| Configuration Issue | `12182` |
| Application Issue | `12180` |
| External Systems Issues | `10605` |

Pass as: `additional_fields: {"customfield_10225": {"id": "<ID>"}}`

**Production Issue additional defaults:** After creation, set these fields via `editJiraIssue`:
- **Priority**: `High` (default for prod issues; use `Critical` if user specifies urgency)
- **Labels**: Include quartal label in format `YY#NQuartal` (e.g., `26#1Quartal` for 2026 Q1). Derive from current date: Q1=Jan-Mar, Q2=Apr-Jun, Q3=Jul-Sep, Q4=Oct-Dec.
- **Sprint**: Set to current sprint (`customfield_10020: <sprint_id>`). Find sprint ID by querying `customfield_10020` from any issue in `openSprints()`.

If creation fails, report the error and stop.

### 7. Transition status

Get available transitions via `getTransitionsForJiraIssue`, then:

- `todo` (default) → skip (already TODO on creation)
- `in-progress` → find transition with `name` containing "In Progress" and execute via `transitionJiraIssue`
- `done` → find transition with `name` containing "Done" and execute
- Any other value → try to match transition name (case-insensitive), report available transitions if not found

**Note:** If a transition fails or the target status can't be matched, clearly indicate that the **ticket was successfully created** at its default status (TODO) — only the status transition failed. Include the ticket link so the user can manually transition if needed.

### 8. Output result

Display a summary table:

| Field | Value |
|-------|-------|
| **Ticket** | `<key>` with link `https://bfifinance.atlassian.net/browse/<key>` |
| **Issue Type** | the issue type |
| **Summary** | the created summary |
| **Parent** | parent ticket key (or "N/A") |
| **Assignee** | assignee name |
| **Status** | final status after transition |

## Examples

```
/create-jira-task BLCS-2370 Code Review
# Subtask under BLCS-2370's parent, assigned to Roni, stays TODO

/create-jira-task Production Issue OOM error on scoring service
# Standalone Production Issue, assigned to Roni, stays TODO

/create-jira-task Task Follow-up refactoring --assignee herfan --status in-progress
# Standalone Task, assigned to Herfan, transitioned to In Progress

/create-jira-task BLCS-2370 Testing --status done
# Subtask under BLCS-2370, Testing, marked Done

/create-jira-task create a bug for payment gateway timeout, assign to selfhy
# Standalone Bug, assigned to Selfhy, stays TODO
```

## Notes

- **Sub-skill usage**: This skill is also invoked programmatically by `review-pr` (for Code Review subtask creation) and `cleanup-feature-flag` (for Feature Flag Cleanup subtasks). Keep `disable-model-invocation: false` to support both user-facing and programmatic invocation.
- **Team directory auto-grows**: The `team-directory.md` file caches resolved account IDs. First-time lookups hit the Jira API; subsequent lookups are instant. Each developer builds their own local directory since `.claude/skills/` is gitignored.
- **Jira description limitation**: The MCP plugin can't set rich-text descriptions (ADF JSON fails, markdown renders as plain text). Descriptions are kept minimal.
- **Cloud ID troubleshooting**: If API calls fail with a domain error using the UUID cloud ID, retry with the full domain `bfifinance.atlassian.net` as the `cloudId` parameter.
