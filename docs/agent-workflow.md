# Agent Workflow

## Overview

A development workflow using Claude Code skills and agents:

```
Analyze (grooming) → Start task → Approve plan → Execute → Simplify → Test → Create PR → Review → Address findings → Session end
```

---

## The Workflow

### Phase 0: Analyze Task (Sprint Grooming)

**You type:**

```
/analyze-task BLCS-2210
```

**What happens:**

Fetches the Jira ticket and TRD, cross-references the codebase, surfaces ambiguities, and walks through questions interactively. Unanswered questions are saved in `docs/blcs-2210/analysis.md` for follow-up in Jira.

> Run this during sprint grooming or planning — before the sprint starts. Helps clarify requirements early so the plan is solid.

---

### Phase 1: Start the Task

**You type:**

```
/start-task BLCS-2210
```

**What happens:**

The `/start-task` skill:

1. Checks out master and pulls latest
2. Creates a branch with the correct prefix based on issue type (e.g., `feat/BLCS-2210`, `fix/BLCS-2210`)
3. Dispatches the `plan-creator` agent in the background

The `plan-creator` agent:

1. Fetches Jira ticket, related tickets, and TRD from Confluence
2. Analyzes the codebase for existing patterns
3. Writes a plan to `docs/blcs-2210/plan.md`
4. Auto-reviews the plan (max 3 iterations) — fixes issues found during mechanical and semantic validation

> If the plan exceeds ~15 steps, the planner auto-splits into part files (e.g., `plan-part1-migration-and-entities.md`, `plan-part2-endpoints.md`).

---

### Phase 2: Review & Approve the Plan

Open the plan file and review it.

- **If it looks good:** proceed to Phase 3
- **If changes needed:** tell Claude what to adjust, or edit the plan file yourself
- **If you want a fresh-eyes review:** run `/review-plan docs/blcs-2210/plan.md` — dispatches a separate reviewer agent with its own context

---

### Phase 3: Execute the Plan

**You type:**

```
Execute the plan at docs/blcs-2210/plan.md
```

**What happens:**

The `executing-plans` skill activates. It:

1. Loads and critically reviews the plan
2. Implements steps in batches
3. Pauses at checkpoints for you to review progress
4. Continues after your approval

Implementation order follows the plan (typically: migration -> entity -> repository -> service -> DTO -> controller -> tests).

> **Split plans:** Execute each part in a **separate session** to keep the context window clean. Finish part 1 entirely (including review and fixes) before starting part 2.

> **Independent steps:** Use `dispatching-parallel-agents` skill to implement independent steps simultaneously (e.g., two unrelated endpoints).

---

### Phase 4: Simplify

**You type:**

```
/simplify
```

**What happens:**

Reviews the changed code for reuse opportunities, quality issues, and efficiency improvements. Fixes issues found.

---

### Phase 5: Test Locally

**You type:**

```
/test-branch
```

**What happens:**

Discovers all code changes on the current branch and tests them:

- **Phase 1 (REST):** Tests REST endpoints via curl against the local server
- **Phase 2 (Non-REST):** Tests activities, listeners, schedulers via temp endpoints; utils/mappers via unit tests

Reports combined results.

> You also run `mvn test` manually — Claude never runs Maven commands.

---

### Phase 6: Create PR

**You type:**

```
/create-pr
```

**What happens:**

Auto-generates a draft PR with title and body from git changes, implementation plan, and project memory. Uses the repo's PR template.

---

### Phase 7: Review PR

**You type:**

```
/review-pr <pr-number>
```

**What happens:**

The `/review-pr` skill:

1. Fetches Jira ticket for requirements context (local docs first, Jira fallback)
2. Launches specialized review agents in parallel:
   - **code-reviewer** — style, patterns, logic issues
   - **silent-failure-hunter** — empty catch blocks, swallowed errors
   - **code-simplifier** — unnecessary complexity
   - **test-analyzer** — test coverage gaps
   - **type-design-analyzer** — type quality (if new types added)
   - **comment-analyzer** — comment accuracy
   - **migration-reviewer** — Flyway SQL safety (auto-invoked if migrations exist)
   - **implementation-reviewer** — plan compliance (auto-invoked if plan file exists)
3. For other developers' PRs: auto-creates a "Code Review" subtask in Jira
4. Cross-references requirements alignment (acceptance criteria, TRD contracts)
5. Presents findings one-by-one — you decide "fix or skip" (own PR) or "include or skip" (others' PR)
6. For others' PRs: posts inline review comments with `suggestion` blocks

> You can specify aspects: `/review-pr 9999 code tests errors`

---

### Phase 8: Address Review Findings

**You type:**

```
/address-review <pr-number>
```

**What happens:**

For **self-review** (own PR): applies fixes from Phase 7 findings you chose to fix.

For **reviewer comments** (from teammates or Copilot): evaluates each comment technically before implementing — prevents blind application of incorrect suggestions. Walks through comments 1-by-1.

> After fixing, push and the cycle continues until the PR is approved.

---

### Phase 9: Session End

**You type:**

```
/auto-revise-claude-md
```

**What happens:**

Reviews the conversation and updates CLAUDE.md with learnings — new patterns, conventions, or corrections discovered during the session.

---

## Alternative Paths

| Scenario | What to do |
|----------|------------|
| Follow-up fix on merged task | `/create-plan BLCS-2210` directly (skip `/start-task` — branch already exists) |
| Quick hotfix (no plan) | Skip Phases 0-4, go straight to `/create-pr` then `/review-pr` |
| Small migration + entity change | `/simplify` and `/review-pr` may be sufficient — skip `/test-branch` |
| Debugging test failures | Paste the error — `systematic-debugging` skill activates automatically |

---

## Quick Reference

| Phase | You Type | What Runs |
|-------|----------|-----------|
| 0 | `/analyze-task BLCS-XXXX` | Surfaces ambiguities, saves analysis |
| 1 | `/start-task BLCS-XXXX` | Creates branch + dispatches `plan-creator` agent |
| 2 | *(read plan, approve)* + optional `/review-plan` | Manual + optional `plan-reviewer` agent |
| 3 | `Execute the plan at docs/blcs-xxxx/plan.md` | `executing-plans` skill |
| 4 | `/simplify` | Code quality cleanup |
| 5 | `/test-branch` | Local endpoint + unit testing |
| 6 | `/create-pr` | Draft PR from changes + plan |
| 7 | `/review-pr <number>` | Multi-agent review (code + implementation + migration) |
| 8 | `/address-review <number>` | Fix findings or handle reviewer comments |
| 9 | `/auto-revise-claude-md` | Capture session learnings |

---

## Utility Skills

Not part of the main flow, but available anytime:

| Skill | When to Use |
|-------|-------------|
| `/create-plan <ticket>` | Follow-up work on already-merged tasks |
| `/create-migration <desc>` | Ad-hoc Flyway migration with correct naming |
| `/fix-dto [target]` | Batch DTO annotation/visibility standardization |
| `/call-api <name>` | One-off local endpoint testing |
| `/generate-seed-data <entity>` | SQL INSERTs from JPA entity definitions |
| `/create-jira-task <args>` | Create subtasks, bugs, or follow-up tickets |
| `/cleanup-feature-flag <flag>` | Discover and clean up unused feature flags |
| `/fetch-jira-context <ticket>` | Understand a ticket's requirements without planning |

---

## Tips

- **Context preservation**: Phases 3-5 should happen in the **same session** so Claude retains context of what it coded.
- **New session for big plans**: If a plan has multiple parts, start a fresh session per part. Finish each part (execute + simplify + test) before starting the next.
- **Agent memory**: `plan-creator` and `implementation-reviewer` have persistent memory — they learn recurring patterns and common mistakes across sessions.
- **Plan auto-review**: The `plan-creator` auto-reviews its own plan (`Confidence: MEDIUM`). For critical tickets, also run `/review-plan` for a fresh-context review (`Confidence: HIGH`).
- **You run Maven manually**: Claude never runs `mvn test` or `mvn compile`. Run tests yourself and paste failures if any.
- **Copilot comment resolution**: Copilot threads block merge even though they're automated. Resolve via GraphQL `resolveReviewThread` mutation before merging.

---

## Files & Configuration

### Custom Agents (4)

| File | Purpose | Triggered By |
|------|---------|--------------|
| `.claude/agents/plan-creator.md` | Creates plans from Jira/TRD/codebase + auto-review loop | `/start-task`, `/create-plan` |
| `.claude/agents/plan-reviewer.md` | Two-pass plan validation (mechanical + semantic) | `/review-plan` |
| `.claude/agents/implementation-reviewer.md` | Verifies plan/TRD compliance after implementation | `/review-pr` (auto, when plan exists) |
| `.claude/agents/migration-reviewer.md` | Flyway SQL safety review | `/review-pr` (auto, when migrations exist) |

### Agent Persistent Memory

| Directory | Contents |
|-----------|----------|
| `.claude/agent-memory-local/plan-creator/` | MEMORY.md + topic files (error-codes, TRD notes, architecture) |
| `.claude/agent-memory-local/implementation-reviewer/` | MEMORY.md (common findings, recurring violations) |

### Project Skills (15)

| Skill | Type |
|-------|------|
| `/analyze-task` | Main workflow |
| `/start-task` | Main workflow |
| `/create-pr` | Main workflow |
| `/review-pr` | Main workflow |
| `/review-plan` | Main workflow |
| `/address-review` | Main workflow |
| `/test-branch` | Main workflow |
| `/create-plan` | Utility |
| `/create-migration` | Utility |
| `/fix-dto` | Utility |
| `/call-api` | Utility |
| `/generate-seed-data` | Utility |
| `/create-jira-task` | Utility |
| `/cleanup-feature-flag` | Utility |
| `/fetch-jira-context` | Sub-skill (used by other skills) |

### Superpowers Skills (used in main session)

| Skill | When |
|-------|------|
| `executing-plans` | Phase 3 — step-by-step plan implementation |
| `systematic-debugging` | Activates when errors are pasted |
| `dispatching-parallel-agents` | Independent plan steps |
| `brainstorming` | Creative/design work |
| `verification-before-completion` | Before claiming work is done |
| `receiving-code-review` | Evaluating reviewer comments |

### Plugin Skills

| Skill | When |
|-------|------|
| `/simplify` | Phase 4 — code quality cleanup |
| `/auto-revise-claude-md` | Phase 9 — session-end learnings |
| `/pr-review-toolkit:review-pr` | Dispatched by `/review-pr` |
| `/commit` | Alternative to `/create-pr` |

### Hooks (16 in `settings.local.json`)

**PreToolUse (5) — block or warn before action:**

| Hook | Matcher | Action |
|------|---------|--------|
| `git add` safety | Bash | **Blocks** `git add -A`, `.`, `--all` |
| `mvn` command safety | Bash | **Blocks** `mvn compile/test/verify/clean/package/install` |
| `wildcard-import-guard` | Edit\|Write | **Blocks** wildcard imports in Java files |
| `constructor-injection-guard` | Edit\|Write | **Warns** on `@Autowired` — use `@RequiredArgsConstructor` |
| `personal-config-edit-guard` | Edit\|Write | **Warns** on `application-local.yaml`/`docker-compose.yaml` edits |

**PostToolUse (11) — warn after action:**

| Hook | Matcher | Action |
|------|---------|--------|
| `migration-soft-delete` | Write\|Edit | **Warns** if migration SQL lacks `deleted` clause |
| `check-flyway-naming` | Write | **Warns** if migration filename doesn't match convention |
| `dto-noargs-constructor-guard` | Write\|Edit | **Warns** `@Builder` without `@NoArgsConstructor` |
| `enforce-page-request-util` | Write\|Edit | **Warns** `PageRequest.of()` in service impl |
| `mockito-settings-guard` | Write\|Edit | **Warns** `@ExtendWith(MockitoExtension.class)` |
| `enforce-reflected-entities` | Write\|Edit | **Warns** direct `application.getLoan()/getAsset()` |
| `feature-flag-class-level-guard` | Write\|Edit | **Warns** `@FeatureFlag` at class level on controllers |
| `exception-assertion-guard` | Write\|Edit | **Warns** `.getMessage()` with `ErrorMessageConstant` in tests |
| `feature-flag-dual-registration` | Write\|Edit | **Reminds** to register flags in both YAML files |
| `service-response-wrapping-guard` | Write\|Edit | **Warns** `BravoCommonResponse` in service impl |
| `sql-restriction-guard` | Write\|Edit | **Warns** entity missing `@SQLRestriction("deleted=false")` |

### Convention Guards (hookify rules — conversation-level)

| Guard | What it enforces |
|-------|-----------------|
| `protect-prod-config` | Don't edit `application-prod.yaml` or `application-uat.yaml` |
| `transactional-readonly-guard` | Don't add `@Transactional(readOnly = true)` eagerly |
| `jpasort-unsafe-guard` | Use `JpaSort.unsafe()` not `Sort.by()` for JPQL |
| `dto-inherited-builder-guard` | Constructor-level `@Builder` for `BaseResponse`/`BaseSearch` subclasses |
| `base-response-builder-guard` | `builderMethodName` required on `BaseResponse` subclass builders |

### Other Files

| File | Purpose |
|------|---------|
| `.claude/settings.local.json` | Project-level hooks, plugin overrides |
| `docs/<ticket-id>/` | Task artifacts — analysis, plans (gitignored, local only) |
| `docs/agent-workflow.md` | This file (gitignored, local only) |
