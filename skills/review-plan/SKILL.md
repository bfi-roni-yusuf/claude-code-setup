---
name: review-plan
description: "Use when an implementation plan has been produced and needs validation before any code is written."
disable-model-invocation: false
---

Verify that implementation plans are complete, correct, and traceable to requirements — before any code is written. Evaluate against this codebase's Java 17 / Spring Boot / Camunda BPM conventions.

## Usage

```
/review-plan <plan-file-path> [ticket-id]
```

- **plan-file-path** (required): Path to the plan file, e.g., `docs/blcs-2650/plan.md`
- **ticket-id** (optional): Jira ticket ID. If omitted, extracted from the plan's `**Ticket**: BLCS-XXXX` header.

## Gather Context (parallel)

Launch all three sub-steps simultaneously — they are independent.

### a. Read CLAUDE.md

Read `./CLAUDE.md` in the project root. Extract the **Conventions** section — this is your dynamic convention checklist for Pass 2. Do not rely on memorized conventions; CLAUDE.md is the source of truth and may have been updated.

### b. Read the plan file

Parse every implementation step. For each step, extract:
- The expected file path
- The reference pattern (e.g., "Follow `FooServiceImpl`")
- All "Note" / "Gotcha" / "Remember" lines

### c. Fetch Jira/TRD (if ticket ID available)

**REQUIRED SUB-SKILL:** Use `fetch-jira-context` to fetch the ticket, related tickets, and TRD. The sub-skill produces a structured requirements summary with acceptance criteria, API contracts, DB changes, and business rules.

If no ticket ID is available (not in plan header and not provided), skip this sub-step. Pass 2 semantic checks that depend on requirements (traceability, scope creep, TRD alignment) will be skipped.

## Pass 1: Mechanical (fast)

Objective checks that don't require interpreting requirements. For each check, use `Glob`/`Grep`/`Read` to verify against the actual codebase.

### Checks

| Check | How |
|-------|-----|
| **References resolve** | For each "Reference: `FooServiceImpl`" — `Glob` for the class file. Report exact match or closest alternative if not found. |
| **Package placement valid** | Target packages exist, or parent package exists for new sub-packages. `Glob` for the directory. |
| **Layer completeness** | For each *new* entity/service/controller in the plan, verify its expected counterparts exist or are planned. Skip `@Embeddable` entities, internal-only services (no controller needed), and DTOs explicitly noted as reused from other domains. |
| **Migration naming + timestamp** | Matches `V2_0_YYYYMMDDHHMI__description.sql` format. `Glob` for `src/main/resources/db/migration/V2_0_YYYYMMDDHHMM*` to check for timestamp collision. |
| **Dependency ordering** | Steps that produce artifacts (migration, entity) come before steps that consume them (repository, service, controller). |
| **Plan format** | Has `- [ ]` checkboxes, file paths, reference patterns, gotchas — per the `create-plan` skill's format spec. |
| **Plan hygiene** | See hygiene checks below. |

### Plan Hygiene

These checks ensure the plan is clean and unambiguous for the executing agent.

| Check | How |
|-------|-----|
| **No completed/skipped steps** | Steps marked `- [x]` or containing `SKIP` should be removed entirely — they add noise. The plan should only contain work to be done. |
| **Flat step numbering** | Steps must use sequential integers (`Step 1`, `Step 2`, `Step 3`). No sub-steps like `Step 4a`, `Step 4b` — flatten them. Sub-steps obscure the true step count and make progress tracking harder. |
| **Notes inline** | Gotchas, notes, and warnings must be inside the step they apply to — not in standalone paragraphs between steps or in separate "notes" sections. The executing agent processes steps sequentially; context that lives outside a step may be missed. |
| **No stale cross-references** | Step references in notes, summary tables, and verification checklist must use current step numbers. After renumbering, grep for old step references (e.g., "Step 4a", "Step 6") in the full plan. |
| **No contradictory statements** | Two statements in the plan must not say opposite things (e.g., checklist says `UNDERWRITING_PLATFORM` while code says `UW_CREDIT_ANALYST + UW_APPROVER`). Often caused by partial updates after design changes. |
| **No stale design text** | Notes referencing decisions, methods, or approaches that were revised during planning should reflect the final design, not the original. Look for vestiges like "Revised approach:" blocks that describe what was already incorporated. |

**If any check fails** -> return findings immediately. Do NOT proceed to Pass 2. Fast feedback on structural issues lets the planner fix them before deeper analysis.

## Pass 2: Semantic (deep)

Only runs if Pass 1 passes entirely.

### Checks

| Check | How |
|-------|-----|
| **Requirement traceability** | For every plan step, identify the specific Jira AC or TRD section it addresses. Steps with no trace -> flag as SCOPE CREEP. |
| **TRD contract alignment** | API paths, request/response field names and types, HTTP methods must match TRD spec exactly. Watch for snake_case in JSON vs camelCase in Java (Jackson handles globally). |
| **Scope creep detection** | Flag steps that add functionality beyond what the ticket requires: extra indexes not in TRD, bonus validation not in AC, unnecessary feature flags, "nice to have" additions. |
| **Actionability** | Are instructions precise enough for autonomous execution? Flag vague phrases: "add validation logic", "handle edge cases", "implement similar to..." without specifics. Every step must specify *exactly what to do*. |
| **Convention compliance** | Verify gotchas include relevant CLAUDE.md conventions for the touched domains. Examples: `updatedBy` before save (underwriting), `ApplicationUtil.reflected*()` (Application entity), `PageRequestUtil.of()` (pagination), `CAST(:param AS string)` (nullable JPQL params). |
| **Edge case coverage** | Does the plan address: nullability, soft-delete in queries (`AND deleted = false`), pagination with `PageRequestUtil`, `@Version` in test builders, `@SQLRestriction("deleted=false")` on entities? |

**Baseline skill compliance check:**
Verify the plan incorporates baseline best practices:
- Test steps specify JUnit 5 patterns (parameterized tests, @Nested, @DisplayName, descriptive naming) — not just generic "add unit tests"
- New service/controller steps reference java-springboot patterns where bpm-conventions is silent
- Complex new public methods are flagged for Javadoc
- Flag if plan defaults to copying existing test style without considering java-junit patterns

## Output Format

Structure your review as:

```markdown
## Plan Review: <ticket-id>

**Plan**: <file path>
**Pass 1 (Mechanical)**: PASS / FAIL
**Pass 2 (Semantic)**: PASS / FAIL / SKIPPED
**Verdict**: APPROVED / CHANGES_REQUESTED
**Confidence**: HIGH / MEDIUM (set by the caller, not this skill — determined by invocation context)
  HIGH = reviewed in separate context (agent dispatch) or standalone invocation
  MEDIUM = self-review (autonomous mode, same context as planner)

### Mechanical Findings
- [M1] severity: description — fix instruction
- [M2] ...

### Semantic Findings
- [S1] severity: description — fix instruction
- [S2] ...

### Fix Instructions
[Ordered list of specific changes needed to the plan file]
```

## Severity Levels

- **❌ Critical** — Missing reference, wrong pattern, scope creep. Blocks approval.
- **⚠️ Issue** — Ambiguous instruction, missing gotcha, ordering problem. Blocks approval.
- **💡 Suggestion** — Non-blocking improvement.

## Verdict Criteria

- **APPROVED**: No ❌ or ⚠️ findings. Both passes PASS.
- **APPROVED with suggestions**: No blocking findings, but 💡 suggestions exist.
- **CHANGES_REQUESTED**: Any ❌ or ⚠️ finding exists. List all blocking findings with fix instructions.

## Edge Cases

| Situation | Behavior |
|-----------|----------|
| No Jira ticket in plan header | Skip TRD alignment and scope creep checks (no requirements to trace against). Mechanical checks only. |
| Split plan (multiple parts) | Review the specific part file. Verify prior parts' outputs exist (e.g., if Part 2 references an entity from Part 1). |
| Plan has unchecked (`- [ ]`) steps | Review ALL steps regardless of checkbox state — this is a plan review, not an implementation review. |

## Critical Rules

- **Read the actual codebase** — Never review based on assumptions. `Glob`/`Grep` to verify every reference and package claim.
- **Be specific** — "Reference not found" is NOT acceptable. Say: "Reference `FooServiceImpl` not found. Closest match: `FooServiceV2Impl` in `com.bfi.bravo.service.impl.v2`."
- **Cite the source** — For every finding, cite where the requirement comes from: "TRD section 3.2 specifies...", "CLAUDE.md requires...", "Plan step 5 says...".
- **Don't nitpick style** — Prettier handles formatting. Focus on logic, contracts, conventions, and completeness.

## Dependencies

- **Sub-skill:** `fetch-jira-context` (Jira + TRD fetching)
- **Tools:** `Grep`, `Glob`, `Read` (codebase validation)

## Notes

- This skill is invoked by the `plan-reviewer` agent (thin wrapper) or by the `plan-creator` agent directly (autonomous self-review mode)
- Can also be invoked standalone via `/review-plan docs/blcs-2650/plan.md`
- The Confidence field is set by the caller, not this skill — it depends on invocation context

## Workflow Tracking

**REQUIRED SUB-SKILL:** After producing the review verdict:
- If verdict is `APPROVED` or `APPROVED with suggestions` → use `update-workflow` to mark phase `plan_review` as `done`.
- If verdict is `CHANGES_REQUESTED` → do not update the tracker (plan needs fixes first).
