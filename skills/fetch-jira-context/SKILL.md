---
name: fetch-jira-context
description: "Use when a Jira ticket ID is available and you need to understand the requirements, acceptance criteria, and TRD before proceeding with a task. Supports --lightweight for minimal ticket info."
---

# Fetch Jira Context

Fetch and summarize a Jira ticket's context. Two modes:

- **Full** (default) — ticket + related tickets + TRD from Confluence. For planning, review, and implementation workflows.
- **Lightweight** (`--lightweight`) — ticket only, no related tickets, no TRD. For quick lookups where only basic ticket fields are needed.

Designed as a reusable sub-skill for other workflows.

## Usage

```
/fetch-jira-context <ticket-id>                # full mode (default)
/fetch-jira-context <ticket-id> --lightweight   # lightweight mode
```

## Constants

| Key | Value |
|-----|-------|
| Jira Cloud ID | `74eaba08-c4fa-40de-a200-a44e70ca5dca` |
| Jira Domain | `bfifinance.atlassian.net` |
| Common Project Keys | `BLCS`, `BLOS` |

---

## Lightweight Mode

When `--lightweight` is passed, execute **only** step 1 below, then produce the lightweight output. Skip steps 2–4.

### Step 1. Fetch the Jira ticket

Use `getJiraIssue` with `cloudId: "74eaba08-c4fa-40de-a200-a44e70ca5dca"` and `fields: ["summary", "issuetype", "priority", "status", "labels", "parent", "subtasks"]`. Extract:
- Summary
- Issue type (Story, Bug, Sub-task, etc.)
- Priority, status, labels
- Parent key and summary (if Sub-task or has parent)
- Subtasks list (if Story/Epic/Task with subtasks)

If the ticket is not found, report the error and stop.

### Lightweight output

```markdown
## Ticket — <TICKET-ID>

| Field | Value |
|-------|-------|
| **Summary** | <summary> |
| **Type** | <issue type> |
| **Priority** | <priority> |
| **Status** | <status> |
| **Parent** | <parent key> — <parent summary> (or "—" if none) |
| **Labels** | <labels comma-separated> (or "—" if none) |

### Subtasks (<count>)
| Key | Summary | Status |
|-----|---------|--------|
| <key> | <summary> | <status> |
```

Omit the Subtasks section if the ticket has no subtasks.

---

## Full Mode (default)

When no `--lightweight` flag is passed, execute all steps below.

### 1. Fetch the Jira ticket

Use `getJiraIssue` with `cloudId: "74eaba08-c4fa-40de-a200-a44e70ca5dca"` and `fields: ["summary", "description", "issuetype", "priority", "status", "labels", "parent", "subtasks", "issuelinks", "comment"]`. Extract:
- Summary, description, acceptance criteria
- Issue type (Story, Bug, Sub-task, etc.)
- Priority, labels, components
- Linked issues (blocks, is-blocked-by, relates-to, parent epic)
- Comments (often contain clarifications from the team)

If the ticket is not found, report the error and stop.

### 2. Fetch related tickets

- If the ticket is a **Sub-task** → fetch the parent ticket for full context
- If the ticket has a **parent epic** → fetch it for broader scope
- Fetch **linked issues** (blocks, is-blocked-by, relates-to) — business context is often spread across multiple tickets
- Fetch **sub-tasks** if the ticket is a Story/Epic — they show the breakdown

### 3. Fetch TRD from Confluence (if referenced)

Scan the ticket description and comments for Confluence links (`https://bfifinance.atlassian.net/wiki/...` or page IDs).

If a TRD link is found:
1. Use `getConfluencePage` with `cloudId: "74eaba08-c4fa-40de-a200-a44e70ca5dca"` and `contentFormat: "markdown"`
2. If the page is too large or markdown conversion is lossy, save ADF JSON to `/tmp/<ticket-id>-trd.json` and use `Grep` to search for specific sections by keyword
3. Extract key sections: API contracts (request/response schemas), database changes, business rules, workflow changes, integration points

### 4. Produce requirements summary

Output a structured summary:

```markdown
## Requirements Summary — <TICKET-ID>

**Ticket**: <ID> — <Summary>
**Type**: <Issue Type> | **Priority**: <Priority>
**Parent/Epic**: <Parent key — summary> (if any)

### What is required
<1-3 sentences describing the functional requirement>

### Acceptance Criteria
- <Extracted from ticket description or TRD>

### API Contracts (from TRD)
| Method | Path | Request | Response |
|--------|------|---------|----------|
| ... | ... | ... | ... |

### Database Changes (from TRD)
- <Table changes, new columns, migrations needed>

### Business Rules & Edge Cases
- <Extracted from ticket, comments, or TRD>

### Related Tickets
| Ticket | Relation | Summary |
|--------|----------|---------|
| ... | parent / blocks / related | ... |
```

Omit sections that don't apply (e.g., no API Contracts section if there's no TRD).

---

## Notes

- This skill is a **sub-skill** — designed to be referenced by other skills (`review-pr`, `create-plan`, `review-plan`, etc.)
- Can also be invoked directly via `/fetch-jira-context BLCS-2343` when you just need to understand a ticket
- The requirements summary (full mode) is displayed to the user for confirmation before the calling skill proceeds
- **Lightweight mode** is designed for callers that only need basic ticket metadata (e.g., `start-task` for branch prefix, `create-jira-task` for parent context and duplicate check)
