---
name: analyze-task
description: "Use when given a Jira ticket ID to analyze before implementation — surfaces ambiguities and requirement gaps interactively."
disable-model-invocation: true
---

# Analyze Task

Fetch full Jira/TRD context for a ticket, perform lightweight codebase cross-referencing to surface ambiguities, then walk through each question interactively with the user. Unanswered questions are saved as pending in the analysis doc for the user to follow up manually. Resolved answers and analysis are saved for downstream use by `/create-plan`.

## Usage

```
/analyze-task <ticket-id>
```

**Workflow position:**
```
/analyze-task BLCS-1234     -> understand & clarify requirements
/start-task BLCS-1234       -> branch + implementation plan (reads analysis file)
```

## Phase 1: Fetch Context & Cross-Reference

### Step 1 — Fetch Jira context

**REQUIRED SUB-SKILL:** Use `fetch-jira-context` to fetch the ticket, related tickets, and TRD. The sub-skill produces a structured requirements summary with acceptance criteria, API contracts, DB changes, and business rules.

After the sub-skill completes, also extract from the raw ticket response (not included in `fetch-jira-context` output):
- **Reporter** display name — useful context for identifying who to ask about requirements
- **Comments** — scan for clarifications, decisions, or answers to previously posted questions

**Subtask handling:** If the ticket is a subtask (`issuetype.subtask: true`), also fetch the **parent story** (`fields.parent.key`) with full description, comments, and issue links. The parent typically has the real requirements, TRD links, and stakeholder context. Use the parent's reporter and comments as the primary context source — subtask descriptions are often just "Backend implementation".

### Step 2 — Check for existing analysis

Before scanning the codebase, check for `docs/<ticket-id-lowercase>/analysis.md`. If found, this is a **re-run**:
- Load existing resolved, pending, and skipped questions
- Re-fetch Jira context captures new comments/replies since last run
- Continue to Step 3 — the scan may find new things based on codebase changes

### Step 3 — Shallow codebase scan

Based on entities mentioned in the ticket/TRD, perform existence checks:

| What to check | How | Example question it surfaces |
|---|---|---|
| Referenced tables exist | `Grep` for table name in migrations | "TRD says `underwriting_negotiation` but only `underwriting_negotiation_summary` exists" |
| Referenced services/clients exist | `Glob` for class name | "TRD calls `PefindoApiClient` — no such client, is this new?" |
| Referenced endpoints exist | `Grep` in controllers | "TRD says PATCH `/api/v1/foo` but only GET exists" |
| Feature flags already exist | `Grep` in YAML | "Flag `featNDF4WJudol` already exists — reuse or new flag?" |
| Column name conflicts | `Grep` in migrations for the target table | "TRD adds `type` column but `collateral_type` already exists" |
| Referenced enum values exist | `Grep` for enum class + value | "TRD says `APPROVE` but `UnderwritingReviewSummaryValidationProcess` only has PROCEED/RETURN/CANCEL" |

Intentionally shallow — just existence checks, not architectural analysis. Leaves "how to build it" to `/create-plan`.

### Step 4 — Generate categorized question list

Group questions into categories:
- **Business Rules** — unclear logic, missing edge cases
- **API Contract** — missing fields, ambiguous types, status codes
- **Database** — column conflicts, missing specs
- **Integration** — unknown services, unclear data ownership
- **Testing** — missing error response specs, unclear test data requirements, unspecified validation rules
- **Scope** — ticket boundary unclear, potential overlap with other tickets
- **Conflict** — TRD vs ticket vs comments disagree

For re-runs: merge with existing questions. Previously resolved questions stay resolved. Previously pending questions are re-presented for the user to confirm, update, or re-categorize.

## Phase 2: Interactive Discussion

### Step 1 — Present overview

Show the full numbered list grouped by category so the user sees the big picture:

```
Found 7 questions across 3 categories:

## Business Rules
1. TRD says "reject if score < 500" but doesn't specify which score (Pefindo? Internal?)
2. What happens when collateral valuation is null — skip or block?

## API Contract
3. Response DTO includes `approvalDate` but TRD doesn't specify the format (ISO? epoch?)
4. PATCH endpoint — is it partial update or full replace?

## Conflict
5. Ticket says "NDF 4W only" but TRD shows NDF 2W in the product list
```

For re-runs, also show previously resolved/skipped questions with their status so the user can review or update them.

### Step 2 — Walk through 1-by-1

**One question per message.** Present the question with context, then STOP. Do NOT present the next question in the same message as the resolution acknowledgment — wait for the user to respond or signal readiness. The user may have follow-up concerns that change the answer.

For each question, present it with context (relevant TRD excerpt or codebase finding). When possible, include your **recommended answer** based on codebase evidence — this gives the user something concrete to confirm or push back on, rather than an open-ended question.

#### Challenge layer

When the user answers a question, **evaluate the answer** before recording it:

- **Codebase contradiction** — the answer conflicts with what the code actually does. Example: user says "use PATCH for partial update" but every similar endpoint in the domain uses PUT with full replace. Push back with evidence.
- **Cross-answer inconsistency** — the answer contradicts a previous answer in this session. Example: Q2 answer assumes field is nullable, Q5 answer assumes it's required. Flag the conflict.
- **Unstated downstream impact** — the answer has implications the user may not have considered. Example: user says "add a new column" but the table has 50M+ rows and no migration strategy for backfill.

If none of these apply, accept the answer and move on — don't grill for the sake of grilling. The goal is catching real blind spots, not being adversarial on every answer.

**Verify before challenging.** If you can confirm or refute the user's answer by exploring the codebase, do that first. Only challenge with evidence, not hunches.

**Challenge format** (one per message, after the user's answer):

> Your answer: [paraphrase]
> However, [evidence or concern].
> Still want to go with [their answer], or reconsider?

**Example:**
```
Q3: PATCH endpoint — is it partial update or full replace?
Recommended: Full replace (all 4 similar endpoints in underwriting use PUT)

User: "Partial update — FE only sends changed fields"

> Your answer: partial update (FE sends changed fields only).
> However, all 4 existing underwriting endpoints use PUT with full replace
> (UnderwritingReviewSummaryController:87, :104, :121, :138).
> Partial update would be the first PATCH in this domain.
> Still want to go with partial update, or align with existing pattern?
```

If the user confirms after a challenge, lock it in and move on. **Maximum one challenge per question** — don't re-challenge after confirmation.

#### User responses

| Response | Action |
|----------|--------|
| Answer the question | Evaluate answer (challenge layer above), then record |
| "skip" | Record as skipped (with optional reason), move to next |
| "pending" | Record as pending (non-blocking) — prompt for who should be asked |
| "pending, blocking" | Record as **blocking** pending question — prompt for who should be asked |
| "pending @name" or "ask @name" | Record as pending (non-blocking), note **that specific person** as the one to ask |
| "pending @name, blocking" or "ask @name, blocking" | Record as **blocking** pending, note that person |
| "pending all @name" | Record all remaining unanswered questions as pending for that person (non-blocking). Add `, blocking` to make all blocking. |
| "skip rest" | Record all remaining as skipped, end walkthrough |
| "cancel" or "abort" | Stop the walkthrough. Save whatever was resolved so far to the analysis file. |

**Blocking vs non-blocking:** Blocking means this question must be answered before `/create-plan` can produce a correct plan. Non-blocking means the plan can work around it (assume a default, add a TODO).

**"Ask who?" prompt.** When the user says bare `pending` without a name, prompt "who should be asked?" to record the contact. This helps the user remember who to follow up with when they read the analysis doc later.

**Lenient parsing.** Don't require exact punctuation — any response containing both "pending" and "blocking" is a blocking pending, regardless of comma placement. Same for `@name` detection. Also accept "post" or "jira" as synonyms for "pending" (backwards compatibility).

## Phase 3: Save Analysis

### Step 1 — Save analysis file

Create the directory `docs/<ticket-id-lowercase>/` if it doesn't exist. Save to `docs/<ticket-id-lowercase>/analysis.md`:

```markdown
# Analysis — <TICKET-ID>

**Ticket**: <ID> — <Summary>
**Analyzed**: <date>
**Last Updated**: <date> *(only shown on re-runs)*
**Status**: <N> resolved, <N> pending, <N> skipped

## Requirements Summary
<from fetch-jira-context output>

## Resolved Questions
| # | Category | Question | Answer |
|---|----------|----------|--------|
| 1 | Business Rules | Which score type? | Pefindo score (confirmed by user) |

## Pending Questions
| # | Category | Question | Ask | Blocking? |
|---|----------|----------|-----|-----------|
| 3 | API Contract | approvalDate format? | @caliandra | no |
| 5 | Business Rules | Which score threshold? | @caliandra | **yes** |

## Skipped Questions
| # | Category | Question | Reason |
|---|----------|----------|--------|
| 6 | Scope | Is retry logic part of this ticket? | Not relevant — handled by existing infra |

## Codebase Findings
- `underwriting_negotiation_summary` table exists (V2_0_202601...)
- No `PefindoApiClient` found — likely new integration
```

This file is in `docs/` (gitignored — local-only, never committed).

## Edge Cases

- **No TRD linked:** Skip codebase cross-referencing that depends on TRD specifics. Still analyze ticket description and comments for ambiguities.
- **No questions found:** Report clean analysis, save the analysis file with empty question tables.
- **All questions answered by user:** Save analysis file with all resolved.
- **Ticket not found:** Error and stop (inherited from `fetch-jira-context`).
- **Cancel/abort mid-walkthrough:** Save whatever was resolved so far to the analysis file. Report how many questions were resolved vs remaining.
- **Re-running on same ticket:** Re-fetch Jira context (captures new comments). Load existing analysis file. Re-present ALL questions with their previous status. User confirms, updates, or re-categorizes each. Overwrite the file with updated results.
- **Skipped questions:** Recorded in the analysis file with optional reason — preserves the skill's analytical work for downstream consumers.
- **Cross-ticket pending questions:** When a question was already posted/recorded from a sibling ticket's analysis, reference it (e.g., "same as BLCS-2653 Q4, posted on BLCS-2592") instead of duplicating.

## Notes

- This skill is a **pre-planning clarification step** — designed to run before `/start-task` or `/create-plan`
- Can also be invoked standalone via `/analyze-task BLCS-2343` when you just need to understand a ticket deeply
- The analysis file is consumed by `/create-plan` — resolved questions become requirements, blocking pending questions trigger warnings

## Workflow Tracking

**REQUIRED SUB-SKILL:** After Phase 3 completes (analysis file saved to `docs/<ticket-id>/analysis.md`), use `update-workflow` to:
1. Create the tracker if it doesn't exist (this skill is an initialization point)
2. Mark phase `analyze` as `done`
