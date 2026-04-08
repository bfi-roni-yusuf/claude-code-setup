---
name: workflow
description: "Use when checking workflow progress, manually marking phases as done/skipped, or viewing next steps for the current ticket."
disable-model-invocation: true
---

# Workflow Tracker

Display and manage the workflow tracker for a Jira ticket. Provides cross-session visibility into which phases have been completed, skipped, or are still pending.

## Usage

```
/workflow [TICKET-ID]        # Show status (auto-detect from branch if no ID)
/workflow skip <phase>       # Mark phase as skipped (prompts for reason)
/workflow done <phase>       # Manually mark phase as done
```

**Phase keys:** `analyze`, `start`, `plan_review`, `execute`, `simplify`, `test`, `create_pr`, `review_pr`, `address_review`, `revise_claude_md`.

## Step 1: Resolve Ticket ID

If a ticket ID argument was provided (e.g., `/workflow BLCS-2210`), use it directly.

Otherwise, parse from current branch:
```bash
git branch --show-current
```
Extract ticket ID: `feat/BLCS-2210` → `BLCS-2210`. Lowercase for file path: `blcs-2210`.

If on `master`, `main`, or a non-feature branch **and** no ticket argument was given, show an error and stop:
```text
No ticket ID found. Usage: /workflow BLCS-XXXX
```

## Step 2: Load or Initialize Tracker

Read `docs/<ticket-id>/workflow.yml`.

If the file does not exist → **create it** using the `update-workflow` sub-skill's initialization logic:
1. Ensure directory `docs/<ticket-id>/` exists.
2. Write the initialization template with all 10 phases set to `pending`.
3. Re-read the file.

## Step 3: Handle Commands

### `skip <phase>`

1. Prompt the user: `"Why is <phase name> being skipped?"`
2. Wait for the user's reason.
3. **REQUIRED SUB-SKILL:** Use `update-workflow` to mark phase `<phase>` as `skipped` with the provided reason.
4. Display updated status (continue to Step 4).

### `done <phase>`

1. **REQUIRED SUB-SKILL:** Use `update-workflow` to mark phase `<phase>` as `done`.
2. Display updated status (continue to Step 4).

### No command (display only)

Continue directly to Step 4.

## Step 4: Display Status

Render the tracker in this format:

```text
Workflow: BLCS-2210                            branch: feat/BLCS-2210
---------------------------------------------------------------------
  x  Phase 0: Analyze Task              SKIPPED  (Small task)
  v  Phase 1: Start Task                DONE     2026-03-26 10:30
  v  Phase 2: Review & Approve Plan     DONE     2026-03-26 10:45
  >  Phase 3: Execute Plan              IN PROGRESS
       v Part 1: Migration & Entities   DONE     2026-03-26 11:00
       > Part 2: Endpoints              IN PROGRESS
  .  Phase 4: Simplify                  PENDING
  .  Phase 5: Test Branch               PENDING
  .  Phase 6: Create PR                 PENDING
  .  Phase 7: Review PR                 PENDING
  .  Phase 8: Address Review            PENDING
  .  Phase 9: Revise CLAUDE.md          PENDING
---------------------------------------------------------------------
  > Current: Execute Plan (Part 2: Endpoints)
  > Next: /simplify
```

**Status icons:**
| Icon | Meaning |
|------|---------|
| `v` | done |
| `x` | skipped (reason shown in parentheses) |
| `>` | in_progress |
| `.` | pending |

**Phase numbering** (fixed mapping):
| Phase Key | Number | Display Name |
|-----------|--------|-------------|
| `analyze` | 0 | Analyze Task |
| `start` | 1 | Start Task |
| `plan_review` | 2 | Review & Approve Plan |
| `execute` | 3 | Execute Plan |
| `simplify` | 4 | Simplify |
| `test` | 5 | Test Branch |
| `create_pr` | 6 | Create PR |
| `review_pr` | 7 | Review PR |
| `address_review` | 8 | Address Review |
| `revise_claude_md` | 9 | Revise CLAUDE.md |

**Multi-part phases:** If a phase has `parts`, render each part indented under the parent with its own icon, name, status, and timestamp.

**Current & Next logic:**
- **Current**: The first phase (or part) with `in_progress` status. If none are `in_progress`, use the first `pending` phase that immediately follows the last `done` phase.
- **Next**: The skill command for the next `pending` phase after the current one.

**Next command mapping:**

| Phase Key | Suggested Command |
|-----------|-------------------|
| `analyze` | `/analyze-task <TICKET-ID>` |
| `start` | `/start-task <TICKET-ID>` |
| `plan_review` | `/review-plan` |
| `execute` | Execute the implementation plan (then `/workflow done execute`) |
| `simplify` | `/simplify` (then `/workflow done simplify`) |
| `test` | `/test-branch` |
| `create_pr` | `/create-pr` |
| `review_pr` | `/review-pr <PR#>` |
| `address_review` | `/address-review <PR#>` |
| `revise_claude_md` | `/auto-revise-claude-md` (then `/workflow done revise_claude_md`) |

**External plugin reminder:** For `simplify`, `execute`, and `revise_claude_md`, append `(then /workflow done <phase>)` to the next command because their plugin skills can't auto-update the tracker.

## Step 5: Skip Warnings

After rendering the status display, check for **gaps** — pending phases that appear before a `done` or `in_progress` phase.

### Critical vs Optional Phases

| Phase | Critical? | Rationale |
|-------|-----------|-----------|
| `analyze` | No | Pre-sprint, often skipped for small tasks |
| `start` | No | Manual branch creation is valid |
| `plan_review` | **Yes** | Executing an unreviewed plan is risky |
| `execute` | No | Can't be skipped in practice |
| `simplify` | No | Nice-to-have, not always needed |
| `test` | **Yes** | Easy to forget, high impact |
| `create_pr` | No | Can't skip if you want to merge |
| `review_pr` | **Yes** | Self-review catches real issues |
| `address_review` | No | Only relevant if review had findings |
| `revise_claude_md` | **Yes** | Easiest to forget |

### Warning Logic

For each phase in order:
- If `status == "pending"` AND any later phase is `done` or `in_progress`:
  - If the phase is **critical** → show a warning with suggested action.
  - If the phase is **optional** → no warning (visible as `PENDING` in the display is sufficient).

### Warning Format

```text
! Phase 5 (Test Branch) was not completed before creating PR
  > Run /test-branch, or /workflow skip test "migration-only change"
```

Show all warnings together after the status display, before the `Current` / `Next` lines.
