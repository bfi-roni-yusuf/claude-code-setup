---
name: create-plan
description: "Creates detailed implementation plans from Jira tickets and TRDs. Use when given a Jira ticket ID or asked to plan an implementation."
disable-model-invocation: false
---

Translate business requirements into precise, actionable implementation plans that an AI coding agent can execute autonomously with minimal ambiguity. Plans must be specific to this codebase's Java 17 / Spring Boot / Camunda BPM stack and follow existing patterns.

## Goal

Given a Jira ticket, perform deep analysis and produce a comprehensive implementation plan. Read every related ticket and TRD, examine the existing codebase for patterns, and produce a plan so detailed that implementation becomes mechanical.

## Phase 1: Deep Research & Understanding

When given a Jira ticket ID:

**REQUIRED SUB-SKILL:** Use `fetch-jira-context` to fetch the ticket, related tickets, and TRD. The sub-skill produces a structured requirements summary with acceptance criteria, API contracts, DB changes, and business rules.

After getting the requirements summary, check for a prior analysis file:

**Check for `/analyze-task` output:** Look for `docs/<ticket-id-lowercase>/analysis.md`. If found, load it as prior context with these priorities:
- **Resolved Questions** — high value. These are confirmed answers from the user. Treat as additional requirements.
- **Codebase Findings** — supplementary hints only. Your own deeper scan takes precedence if findings conflict (the analysis file may be stale).
- **Pending Questions** — check the `Blocking?` column. If any pending question is marked **yes**, warn the user before producing a plan — these represent unknowns that prevent correct implementation. Non-blocking pending questions can be planned around (assume a default, add a TODO).

Then continue with codebase analysis:

1. **Examine the codebase** to understand existing patterns:
   - Search for similar features already implemented (use `Grep`, `Glob`, `Read`)
   - Identify the exact packages, classes, and patterns that the new feature should follow
   - Check for existing entities, DTOs, services, controllers, repositories in the relevant domain
   - Look at BPMN files if workflow changes are involved
   - Check `WorkflowConstants`, enums, and configuration files for relevant constants
   - Review existing Flyway migrations for the affected tables
   - Check feature flag conventions if the ticket mentions feature gating

2. **Identify integration points**:
   - Which Feign clients are involved?
   - Which RabbitMQ exchanges/queues?
   - Which external services?
   - What authentication/authorization is needed?

## Phase 2: Analysis & Design Decisions

Before writing the plan, resolve these:

1. **Pattern selection**: Which existing pattern does this feature align with? (e.g., BaseActivity vs JavaDelegate, BravoBaseResponse vs BravoCommonResponse, BaseResponse inheritance, BaseSearch pagination)
2. **Package placement**: Exactly which packages will new classes go in?
3. **Naming conventions**: Follow existing Java class naming patterns per domain (check existing classes in the target package)
4. **Database design**: Table names, column types, foreign keys, indexes, soft-delete support
5. **Migration strategy**: Flyway naming with correct timestamp format
6. **Test strategy**: What needs unit tests? What needs functional/integration tests? What mocking approach?
7. **Feature flags**: Does this need a feature flag? What's the naming convention?
8. **Edge cases**: What can go wrong? Null handling, concurrent access, idempotency

**Baseline skill integration:**
When planning Java implementation steps, consult and incorporate:
- `java-springboot` — for service/controller/repository patterns in new code
- `java-junit` — for test approach: specify @ParameterizedTest for multi-case scenarios, @Nested for grouping, @DisplayName for readability, methodName_should_X_when_Y naming
- `java-docs` — for Javadoc expectations: flag complex public methods that need documentation in the plan
- `bpm-conventions` overrides baseline skills where it has explicit rules

Test steps in the plan should specify which JUnit 5 patterns to use, not just "add unit tests."

## Phase 3: Implementation Plan Output

Produce a plan with this structure:

### Plan Header
- **Ticket**: [ID] — [Summary]
- **Epic/Parent**: [if applicable]
- **TRD Reference**: [Confluence link if any]
- **Estimated Complexity**: Low / Medium / High / Very High
- **Affected Domains**: [list of domain packages affected]

### Prerequisites
- Any infrastructure changes needed first
- Any dependent tickets that must be completed first
- Any configuration/environment setup

### Implementation Steps

Organize as checkbox items in dependency order. Use `- [ ]` markdown checkboxes so the executing agent can mark steps as `- [x]` when completed. This enables **cross-session progress tracking** — a new session can read the plan file, see which steps are done, and continue from the next unchecked step.

Each step MUST include:

1. **Step title** — clear action prefixed with step number (e.g., "Step 1: Create Flyway migration for `underwriting_return_reason` table")
2. **File path** — exact path where the file should be created/modified (e.g., `src/main/resources/db/migration/V2_0_202602271430__create-underwriting-return-reason.sql`)
3. **What to do** — precise instructions, not vague. Include:
   - For new classes: full class signature, annotations, which class to extend/implement, key methods to implement, dependencies to inject
   - For modifications: which method to change, what logic to add/modify, before/after behavior
   - For migrations: exact SQL with column types, constraints, indexes
   - For DTOs: all fields with types, validation annotations, inheritance (BaseResponse, BaseSearch)
   - For tests: which scenarios to cover, what to mock, expected assertions
4. **Reference pattern** — point to an existing class in the codebase that follows the same pattern (e.g., "Follow the pattern in `UnderwritingCollateralDetailServiceImpl`")
5. **Gotchas/Notes** — anything non-obvious (e.g., "Remember to use `ApplicationUtil.reflectedLoan()` not `application.getLoan()`", "Use `CAST(:param AS string)` for nullable JPQL params")

**Format example:**
```markdown
- [ ] **Step 1: Create Flyway migration for `foo` table**
  - **File**: `src/main/resources/db/migration/V2_0_202602271430__create-foo.sql`
  - **What**: CREATE TABLE with columns...
  - **Reference**: `V2_0_202601151200__create-bar.sql`
  - **Note**: Add `deleted` column for soft-delete

- [ ] **Step 2: Create `Foo` entity**
  - **File**: `src/main/java/com/bfi/bravo/entity/foo/Foo.java`
  - **What**: JPA entity with @SQLRestriction("deleted=false")...
  - **Reference**: `Bar.java`
```

### Summary Tables

Include these **only when applicable**. Keep them brief — reference the step number for full details. Do NOT duplicate content from the steps.

**Database Changes** (if any migration steps exist):
| Table | Change | Step |
|-------|--------|------|
| `foo` | New table | Step 1 |

**API Contract** (if any controller steps exist):
| Method | Path | Auth | Feature Flag | Step |
|--------|------|------|--------------|------|
| POST | `/api/v1/foo` | JWT | `featFoo` | Step 8 |

**Configuration Changes** (if any new properties/flags):
Brief list of new YAML properties and feature flags with step references.

**Test Coverage** (always include):
| Test Class | Scenarios | Step |
|------------|-----------|------|
| `FooServiceImplTest` | 5 scenarios (happy + 4 error paths) | Step 12 |

### Execution Instructions

**Include this section verbatim in every generated plan.** The executing agent reads the plan file, not this skill prompt.

> **For the executing agent:** This plan uses checkbox tracking for cross-session continuity.
> 1. Read this file and identify the first unchecked (`- [ ]`) step — that's where you start.
> 2. Before starting a step, confirm all prior steps are marked `- [x]`.
> 3. After completing each step, **immediately update this file** to mark it `- [x]`.
> 4. After completing a batch (default: 3 steps), stop and report progress to the user.
> 5. When all implementation steps are `- [x]`, proceed to the **Verification Checklist** below.
> 6. Check off each verification item as you confirm it. If any item fails, fix before proceeding.

### Verification Checklist

**Generate a contextual checklist** based on what the ticket actually touches. Pick from the pool below — only include items relevant to this plan. Do NOT include items for domains/features not involved.

**Always include:**
- [ ] No wildcard imports
- [ ] Error paths handled with appropriate `BravoCommonException`
- [ ] Prettier formatting will pass (120 char width, 2-space indent)

**Include if plan has migrations:**
- [ ] Soft-delete handled in migrations (`deleted` column) and queries (`AND deleted = false`)
- [ ] Indexes use `CREATE INDEX CONCURRENTLY IF NOT EXISTS`

**Include if plan has pagination:**
- [ ] `PageRequestUtil.of()` used (not `PageRequest.of()` or local `MAX_PAGE_SIZE`)

**Include if plan touches Application entity:**
- [ ] `ApplicationUtil.reflected*()` used instead of direct getters

**Include if plan is in underwriting domain:**
- [ ] `updatedBy` set before saving underwriting entities
- [ ] Status transitions trigger `recordHistory()`

**Include if plan has feature flags:**
- [ ] Feature flags registered in both `application.yaml` and `application-local.yaml`
- [ ] Controller uses `@FeatureFlag` — service does NOT duplicate the check

## Plan Splitting Rule

If the plan exceeds ~15 implementation steps, you **must** split it into separate chunks. Each chunk:
- Is a standalone plan file (e.g., `blcs-2204-part1-migration-and-entities.md`, `blcs-2204-part2-list-endpoints.md`)
- Has its own prerequisites section (referencing what the previous chunk produced)
- Can be executed in a fresh agent session without needing the other plans in context
- Stays under ~15 steps

Split along natural domain boundaries (e.g., per endpoint group, per feature slice). Tightly coupled steps (migration -> entity -> repository for the same table) stay together. The goal is execution fitness — each chunk must fit comfortably in an executing agent's context window.

Each split plan file must include its own **Execution Instructions** and **Verification Checklist** sections, and its own set of checkboxes. This way each part is independently trackable across sessions.

## Output Location

Create the directory `docs/<ticket-id-lowercase>/` if it doesn't exist. Save implementation plans to `docs/<ticket-id-lowercase>/plan.md` (or `plan-part<N>-<description>.md` in the same folder when splitting). This directory is gitignored — plans are local only, never committed.

## Critical Rules

- **Never be vague**. "Add validation logic" is NOT acceptable. Instead: "Add null check for `applicationId` parameter — throw `BravoCommonException(BravoCommonErrorCode.NOT_FOUND, ErrorMessageConstant.APPLICATION_NOT_FOUND)` if null. Then call `applicationService.findById(applicationId)` which already throws NOT_FOUND if missing — do NOT wrap in orElseThrow." **Note:** Constructor is `BravoCommonException(BravoCommonErrorCode, ErrorMessageConstant)` — always verify actual enum values exist before using them in plans.
- **Always reference existing patterns**. Every new class should have a "follow the pattern of X" reference.
- **Think about the full chain**: Controller -> Service -> Repository -> Entity -> DTO -> Migration. Don't miss any layer.
- **Consider backward compatibility**: Will this break existing APIs? Existing BPMN processes? Existing data?
- **Validate against TRD**: If a TRD exists, every API contract, field name, and business rule in the TRD must be reflected in the plan. Call out any discrepancies.

## Communication Style

- Be direct and precise. No filler.
- If the Jira ticket is ambiguous or missing critical information, list specific questions that need answers before the plan can be finalized. Don't guess at business requirements.
- If the TRD conflicts with the Jira ticket, flag it explicitly.
- If you find the task is actually multiple independent work items, suggest splitting and explain why.

## Phase 4: Proofread

**REQUIRED SUB-SKILL:** After saving the plan file(s), invoke `proofread` on each plan file. Fix all reported issues (numbering gaps, stale markers, broken cross-references, table mismatches, contradictions) before proceeding to plan review. This is a mechanical pass — do not revisit design decisions here.

## Plan Review (Caller Responsibility)

After this skill produces the plan, the **caller** should invoke plan review. This skill does NOT embed a review loop — it produces the plan and returns.

**Interactive mode (main conversation):**
1. This skill produces the plan (Phases 1-3)
2. Dispatch `plan-reviewer` agent with the plan file path (separate context, fresh eyes)
3. Read agent findings
4. If CHANGES_REQUESTED: fix plan, re-dispatch (max 3 iterations)
5. If APPROVED: present plan to user

**Autonomous mode (`plan-creator` agent):**
1. Agent invokes this skill (Phases 1-3)
2. Agent invokes `review-plan` skill directly (same context, role-switch)
3. If CHANGES_REQUESTED: fix plan, re-invoke (max 3 iterations)
4. If APPROVED: return final plan

> After the plan is written, the caller should invoke plan review. In interactive mode, dispatch the `plan-reviewer` agent. In autonomous mode, invoke the `review-plan` skill directly.

## Workflow Tracking

**REQUIRED SUB-SKILL:** After Phase 4 completes (plan file(s) proofread and fixed), use `update-workflow` to:
1. Check if multiple `plan-part*.md` files were written in `docs/<ticket-id>/`.
2. If multiple parts exist → call `update-workflow` to expand multi-part structure in the tracker for `execute`, `simplify`, and `test` phases.
3. If single plan (no parts) → no tracker update needed from this skill.

Do NOT mark any phase as `done` — this skill produces the plan. The `review-plan` skill marks `plan_review` as done after approval.
