---
name: update-workflow
description: "Sub-skill for reading and writing workflow tracker YAML files. Called by other skills to update phase status after completing their work."
disable-model-invocation: true
---

# Update Workflow

Centralized sub-skill for reading and writing `docs/<ticket-id>/workflow.yml` tracker files. **Do not invoke directly** — called by other skills via:

```markdown
**REQUIRED SUB-SKILL:** Use `update-workflow` to mark phase <phase_key> as <status>.
```

## Interface

Callers specify:
- **phase_key**: One of: `analyze`, `start`, `plan_review`, `execute`, `simplify`, `test`, `create_pr`, `review_pr`, `address_review`, `revise_claude_md`
- **status**: One of: `pending`, `in_progress`, `done`, `skipped`
- **Optional extras**: `reason` (for `skipped`), `pr_number` (for `review_pr`), `part_key` (for multi-part phases)

## Logic

### Step 1: Derive Ticket ID

Parse current branch name:
```bash
git branch --show-current
```
Extract ticket ID: `feat/BLCS-2210` → `BLCS-2210`. Lowercase for file path: `blcs-2210`.

If on `master`, `main`, or a non-feature branch (no `/` separator) → **skip silently**. There is no tracker to update.

### Step 2: Read Tracker

Read `docs/<ticket-id>/workflow.yml`.

If the file does **not** exist:
- If the caller is `analyze-task`, `start-task`, or `workflow` → **create it** using the Initialization template below, then continue.
- Otherwise → **skip silently**. No tracker exists and this skill is not an initialization point.

### Step 3: Update Phase

1. Set `phases.<phase_key>.status` to the specified status.
2. If status is `done` → set `phases.<phase_key>.completed_at` to current ISO timestamp (`YYYY-MM-DDTHH:MM`).
3. If status is `skipped` → set `phases.<phase_key>.reason` to the provided reason string.
4. If phase_key is `review_pr` and `pr_number` is provided → set `phases.review_pr.pr_number` to the value.

### Step 4: Handle Multi-Part Phases

If the caller specifies a `part_key` and `phases.<phase_key>.parts` exists:
1. Update `phases.<phase_key>.parts.<part_key>.status` (and `completed_at` if done).
2. Derive the parent phase status from its parts:
   - Any part `in_progress` → parent `in_progress`
   - Any part `done` with remaining parts `pending` → parent `in_progress`
   - All parts `done` or `skipped` → parent `done`
   - All parts `pending` → parent `pending`

If no `part_key` is specified, update the phase directly (single-plan case).

### Step 5: Write Back

Write the updated YAML to `docs/<ticket-id>/workflow.yml`.

## Initialization Template

When creating a new tracker file, first ensure the directory exists (`docs/<ticket-id>/`), then write:

```yaml
ticket: <TICKET-ID>
branch: <full-branch-name>
created_at: "<YYYY-MM-DDTHH:MM>"

phases:
  analyze:
    name: "Analyze Task"
    status: pending
  start:
    name: "Start Task"
    status: pending
  plan_review:
    name: "Review & Approve Plan"
    status: pending
  execute:
    name: "Execute Plan"
    status: pending
  simplify:
    name: "Simplify"
    status: pending
  test:
    name: "Test Branch"
    status: pending
  create_pr:
    name: "Create PR"
    status: pending
  review_pr:
    name: "Review PR"
    status: pending
    pr_number: null
  address_review:
    name: "Address Review"
    status: pending
  revise_claude_md:
    name: "Revise CLAUDE.md"
    status: pending
```

Replace `<TICKET-ID>` with the uppercase ticket ID (e.g., `BLCS-2210`), `<full-branch-name>` with the git branch (e.g., `feat/BLCS-2210`), and `<YYYY-MM-DDTHH:MM>` with the current timestamp.

## Multi-Part Expansion

Called by `create-plan` when multiple `plan-part*.md` files are detected.

1. List files matching `docs/<ticket-id>/plan-part*.md`.
2. For each file, derive the part key and display name:
   - Filename: `plan-part1-migration.md` → key: `part1_migration`, name: `"Migration"`
   - Rule: take the suffix after `plan-part`, replace hyphens with underscores for the key, replace hyphens with spaces and title-case for the name.
   - Example: `plan-part2-list-endpoints.md` → key: `part2_list_endpoints`, name: `"List Endpoints"`
3. Add a `parts` map to the `execute`, `simplify`, and `test` phases. Each part starts as `status: pending`.
4. Write back the updated YAML.

Example result for `execute`:
```yaml
execute:
  name: "Execute Plan"
  status: pending
  parts:
    part1_migration:
      name: "Migration"
      status: pending
    part2_list_endpoints:
      name: "List Endpoints"
      status: pending
```

## Auto-Skip Rules

- When the caller is `review-pr` and reports **zero findings** → also mark `address_review` as `skipped` with reason `"No review findings"`.

## Caller Reference

| Calling Skill | Phase Key | Status | Special Behavior |
|---|---|---|---|
| `analyze-task` | `analyze` | `done` | Creates tracker if missing |
| `start-task` | `start` | `done` | Creates tracker if missing |
| `create-plan` | — | — | Expands `parts` if multi-part plan detected; does not mark any phase |
| `review-plan` | `plan_review` | `done` | Only on APPROVED verdict |
| `executing-plans` | `execute` (or part) | `done` | Also marks `plan_review: done` if still pending |
| `simplify` | `simplify` (or part) | `done` | External plugin — updated via `/workflow done simplify` |
| `test-branch` | `test` | `done` | After cleanup completes |
| `create-pr` | `create_pr` | `done` | Also sets `review_pr.pr_number` |
| `review-pr` | `review_pr` | `done` | Auto-skips `address_review` if zero findings |
| `address-review` | `address_review` | `done` | After responses posted |
| `auto-revise-claude-md` | `revise_claude_md` | `done` | External plugin — updated via `/workflow done revise_claude_md` |
