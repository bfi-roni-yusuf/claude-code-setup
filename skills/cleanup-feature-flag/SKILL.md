---
name: cleanup-feature-flag
description: "Use when removing an unused feature flag that was never rolled out — discovers all usages across Java, YAML, BPMN, and tests, then delegates cleanup plan creation to an agent."
disable-model-invocation: true
---

# Cleanup Feature Flag

Discover all usages of an unused feature flag (one that was always `false` / never rolled out), gather full codebase context, then hand off to an agent to produce and execute a cleanup plan.

**Announce at start:** "Using cleanup-feature-flag skill to discover and plan removal of `<flagName>`."

## Usage

```
/cleanup-feature-flag <flagName> [--keyword KEYWORD] [--jira TICKET-ID]
```

Or natural language: "clean up the feature flag featXxx", "remove featXxx, jira ticket BLCS-1234"

## Arguments

| Arg | Required | Default | Description |
|-----|----------|---------|-------------|
| `<flagName>` | Yes | -- | Flag name in camelCase (`featXxx`) or ENV format (`ENABLE_FEATURE_XXX`) |
| `--keyword` | No | Flag name minus `feat` prefix (uppercased) | Override keyword for related flag search |
| `--jira` | No | -- | Jira ticket ID — passed to cleanup plan for subtask creation |

## Scope

- **In scope:** Java code, YAML config, BPMN files (with user approval), tests
- **Out of scope:** Database migrations (Flyway is append-only history)

---

## Step 1: Resolve Flag Name

Accept either camelCase or ENV format as input. **Look up** `application.yaml` to resolve both forms — do NOT derive the ENV var algorithmically (naming is inconsistent across the codebase).

### If input is camelCase (e.g., `featCheckBranchAllocation`):

1. Grep `src/main/resources/application.yaml` for the camelCase name
2. From the matching YAML line, extract the ENV var inside `${...:-...}` or `${...}`
   - Example line: `featCheckBranchAllocation: ${ENABLE_FEATURE_CHECK_BRANCH_ALLOCATION:false}`
   - Extracted ENV var: `ENABLE_FEATURE_CHECK_BRANCH_ALLOCATION`

### If input is ENV format (e.g., `ENABLE_FEATURE_CHECK_BRANCH_ALLOCATION`):

1. Grep `src/main/resources/application.yaml` for the ENV var
2. Extract the camelCase YAML key from the matching line

### Also extract:

- **Full property path**: Determine the YAML nesting path. Most flags live under `setting.feature.config.flagName`. Check the YAML structure to confirm.
- **Product-scoped paths**: Grep all `application*.yaml` for the camelCase name to find any product-scoped variants (e.g., `setting.ndf2w.featXxx`, `setting.ndf4w.featXxx`). Record all paths found. **Note:** Product-scoped flags may have slightly different camelCase names than the parent — the keyword search in Step 2 helps catch these.

### Abort condition

If the flag is not found in any YAML file, report: "Flag `<flagName>` not found in application YAML configuration. Aborting." and stop.

### Output

Report to the user:

```
Resolved flag:
  camelCase:      featXxx
  ENV var:        ENABLE_FEATURE_XXX
  Property path:  setting.feature.config.featXxx
  Default value:  false
  [Product paths: setting.ndf2w.featXxx (if any)]
```

---

## Step 2: Discover Related/Child Flags

A parent flag may have child flags that only exist because the parent is enabled. Since the parent was never enabled, all children are also dead.

Run **both methods in parallel**:

### Method A: Keyword search

1. Derive keyword: strip `feat` prefix from camelCase name, uppercase the remainder (e.g., `featAsliri` → `ASLIRI`). If `--keyword` provided, use that instead.
2. Case-insensitive grep all `src/main/resources/application*.yaml` for lines containing the keyword
3. Filter to lines that look like property definitions (contain `${` or a colon with a value)
4. Exclude the parent flag itself from results
5. **If >10 matches**: warn the user and suggest using `--keyword` to narrow the search

### Method B: Code crawling

1. Find the class(es) where the parent flag is `@Value`-injected: grep `src/main/java` for `@Value("${<full-property-path>}")`
2. Read those classes fully
3. Look for references to **other** feature flags in the same class — other `@Value("${setting.feature.config....")` annotations or `feat*` field names
4. For each candidate, resolve its camelCase name and ENV var (grep YAML)

### Combine and confirm

Merge results from both methods. Deduplicate by camelCase name. For each candidate, note discovery source.

**If candidates found**, present to the user:

```
Found parent flag: featXxx (ENABLE_FEATURE_XXX)

Potentially related flags:
  1. featXxxChild (ENABLE_FEATURE_XXX_CHILD) — keyword + code crawl
  2. cnvXxxRequestUri (CNV_XXX_REQUEST_URI) — keyword only
  3. featXxxRetry (ENABLE_FEATURE_XXX_RETRY) — code crawl only

Include all in cleanup? (y/n, or specify numbers to exclude)
```

Wait for the user's response. The confirmed list (parent + selected children) becomes the **target flag set** for Step 3.

**If no candidates found**, proceed with just the parent flag. Inform the user: "No related flags found. Proceeding with single flag cleanup."

---

## Step 3: Full Discovery

Run full discovery for **every flag in the target flag set**. Discovery runs in two passes.

### Pass 1: Direct references

For each flag in the target set, run these searches. **Parallelize independent searches** (items 1-5, 6a, 7a, 8-10 can all run in parallel).

| # | Target | Search pattern | How |
|---|--------|---------------|-----|
| 1 | **YAML config** | camelCase name AND ENV var | Grep all `src/main/resources/application*.yaml` |
| 2 | **`@Value` injection** | `@Value("${<property-path>}")` | Grep `src/main/java/**/*.java` for the full property path |
| 3 | **`@FeatureFlag`** | `@FeatureFlag(feature = "<camelCase>")` | Grep `src/main/java/**/*.java` |
| 4 | **`@FeatureFlagMethod`** | `@FeatureFlagMethod(feature = "<camelCase>")` | Grep `src/main/java/**/*.java` |
| 5 | **`@ConditionalOnProperty`** | `@ConditionalOnProperty(name = "<property-path>"...)` | Grep `src/main/java/**/*.java`. Note both `havingValue = "true"` (dead bean) and `havingValue = "false"` (always-active bean) variants. Paired classes (e.g., `FooServiceImpl` with `false` + `FooEnhancementServiceImpl` with `true`) are common. |
| 6 | **Java conditionals** | `if (flagField)` / `if (!flagField)` / ternary | Read files from #2, find conditional usage of the injected field |
| 7a | **FeatUtil field** | camelCase name | Grep `src/main/java/**/FeatUtil.java` |
| 7b | **FeatUtil methods** | (depends on 7a) | If 7a found a field: read `FeatUtil.java`, identify which public methods reference that field |
| 7c | **FeatUtil callers** | (depends on 7b) | If 7b found methods: grep all `src/main/java/**/*.java` for callers of those method names |
| 8 | **FeatureFlagConfigProperties** | camelCase name | Grep `src/main/java/**/FeatureFlagConfigProperties.java` |
| 9 | **BPMN conditions** | full property path (e.g., `setting.feature.config.flagName`) | Grep `src/main/resources/bpmn/**/*.bpmn` |
| 10 | **Product-scoped refs** | alternate property paths found in Step 1 | Grep YAML and BPMN for each product-scoped path |
| 11 | **Tests — flag setup** | `"camelCase"` (in quotes) | Grep `src/test/java/**/*.java` — catches `ReflectionTestUtils.setField(..., "camelCase", ...)` and other string refs |

### Pass 2: Derived references

These depend on Pass 1 results:

| # | Target | How |
|---|--------|-----|
| 12 | **BPMN delegate expressions** | For each activity class from #2, #4, or #5 that will be **fully deleted** (class exists only for this flag): extract its Spring bean name (lowercase first letter of class name), grep `src/main/resources/bpmn/**/*.bpmn` for `#{beanName}` |
| 13 | **Test classes for deleted classes** | For each Java class that will be fully deleted: check for a corresponding test file at the mirror path under `src/test/java` with `Test` suffix |

### Output format

Produce a structured summary. Group by flag, then by category. Include file paths and line numbers.

```
=== Discovery Summary ===

Target flags: featXxx, featXxxChild

--- featXxx (ENABLE_FEATURE_XXX) ---

  YAML config:
    - src/main/resources/application.yaml:712
    - src/main/resources/application-local.yaml:345

  @Value injection:
    - src/main/java/com/bfi/bravo/service/impl/MyServiceImpl.java:23
      Field: `private boolean featXxx`

  Java conditionals:
    - src/main/java/com/bfi/bravo/service/impl/MyServiceImpl.java:45-52
      Pattern: `if (featXxx) { ... }` — delete entire block
    - src/main/java/com/bfi/bravo/service/impl/MyServiceImpl.java:67-70
      Pattern: `if (!featXxx) { ... }` — keep body, remove wrapper

  @ConditionalOnProperty: none
  @FeatureFlag: none
  @FeatureFlagMethod: none
  FeatUtil: none
  FeatureFlagConfigProperties: none

  BPMN conditions:
    - src/main/resources/bpmn/ndf2w.bpmn:234
      Expression: `environment.getProperty('setting.feature.config.featXxx') == 'true'`

  Tests:
    - src/test/java/com/bfi/bravo/service/impl/MyServiceImplTest.java:89
      `ReflectionTestUtils.setField(myService, "featXxx", true)`

--- featXxxChild (ENABLE_FEATURE_XXX_CHILD) ---
  [... same format ...]

--- Derived references ---

  BPMN delegate expressions: none
  Test classes to delete: none
```

Display this summary to the user before proceeding to Step 4.

---

## Step 4: Hand off to Agent

Spawn a **general-purpose agent** (Agent tool) to produce the cleanup implementation plan.

### Agent tool parameters

- `description`: "Cleanup plan for <flagName>"
- `subagent_type`: "general-purpose"
- `model`: "opus"
- `prompt`: Build from the template below, filling in the discovered context

### Agent prompt template

Fill `[PLACEHOLDERS]` with actual values from Steps 1-3:

````
You are producing an implementation plan to remove unused feature flags from the bravo-bpm-service codebase. These flags were never rolled out (always `false`), so all "enabled" code paths are dead code.

## Target Flag Set

[INSERT: list each flag with camelCase name, ENV var, property path]

## Discovery Results

[INSERT: the full structured summary from Step 3 — every file path, line number, and usage pattern]

## Cleanup Rules

Apply these rules when producing the plan steps:

1. **`if (flagName)` blocks** → delete the entire `if` block (body is dead code)
2. **`if (!flagName)` blocks** → keep body, remove conditional wrapper (negated path was always taken)
3. **`@FeatureFlag` annotated endpoints** → delete entirely (returns 404 when disabled — never reachable)
4. **`@FeatureFlagMethod` annotated methods** → delete entirely (returns 501 when disabled — never reachable)
5. **`@ConditionalOnProperty` with `havingValue = "true"`** → delete the entire class (dead bean — never instantiated). Check for a paired class with `havingValue = "false"` (always-active counterpart) and remove its `@ConditionalOnProperty` annotation (no longer conditional).
6. **`@ConditionalOnProperty` with `havingValue = "false"`** → keep the class (always active), but remove the `@ConditionalOnProperty` annotation since the flag no longer exists. If the class implements an interface shared with the `havingValue = "true"` counterpart, verify no Spring ambiguity after the other class is deleted.
7. **`@Value` flag field** → delete the field declaration and clean up all usages per rules 1-2. Note: `@Value` fields are typically NOT `final` / NOT constructor-injected. Verify the field modifier — if `final`, a constructor parameter also needs removal.
8. **YAML entries** → remove from ALL profile files (`application.yaml`, `application-local.yaml`, `application-dev.yaml`, `application-sit.yaml`, `application-uat.yaml`, `application-prod.yaml`, `application-test.yaml`, `application-unit-test.yaml`, etc.)
9. **FeatUtil methods** → delete if exclusively for this flag. If shared with other live flags, add a plan step that asks the user interactively. Also clean up callers of deleted FeatUtil methods.
10. **FeatureFlagConfigProperties field** → delete the boolean field
11. **BPMN gateway conditions** → add a plan step that asks the user interactively — include the condition expression and surrounding flow context in the step description
12. **BPMN delegate expressions** → add a plan step that asks the user interactively
13. **Tests** → delete `ReflectionTestUtils.setField` lines for the flag AND delete test methods that exclusively test the flagged path. Keep methods testing the always-active path (just remove the flag setup line).
14. **Empty classes** → if a class becomes empty after removal, delete it and its test file
15. **Shared code** (used by both dead and live paths) → add a plan step that asks the user interactively
16. **Unused imports** → clean up in ALL modified files
17. **Don't touch Flyway migrations**
18. **Edge case**: if this is the last flag using `@FeatureFlagMethod` infrastructure, note it in the summary

**Codebase conventions** (follow these when producing the plan):
- Test file naming: strict 1:1 mapping — `FooImpl.java` → `FooImplTest.java`
- Feature flags set in tests via `ReflectionTestUtils.setField(instance, "fieldName", value)`
- `@SQLRestriction("deleted=false")` on entities — don't touch
- Constructor injection via `@RequiredArgsConstructor` — no field `@Autowired`

## Plan Format

Produce a plan document with this structure:

### Header
```
# Cleanup: [flagName] Feature Flag Removal

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Remove unused feature flag `[flagName]` and all dead code gated behind it.
**Flags removed:** [list all flags in target set]
```

### Steps
Use `- [ ]` checkbox items. Each step must include:
- **Step title** with clear action
- **File path** — exact path to create/modify/delete
- **What to do** — precise instructions (exact code to remove/modify, with line numbers)
- **Why** — which cleanup rule applies

Order: YAML removal → Java field removal → conditional cleanup → `@ConditionalOnProperty` class cleanup → annotation cleanup → FeatUtil → FeatureFlagConfigProperties → BPMN (interactive) → tests → empty class deletion → import cleanup

### Verification Checklist
- [ ] All YAML entries removed from all profiles
- [ ] No remaining references to the flag's camelCase name in `src/main/java`
- [ ] No remaining references to the flag's ENV var in any YAML
- [ ] No remaining references in BPMN files (or user confirmed to keep)
- [ ] All modified files have clean imports
- [ ] No Flyway migrations were touched

### Execution Instructions

Include this section verbatim:

> **For the executing agent:** This plan uses checkbox tracking for cross-session continuity.
> 1. Read this file and identify the first unchecked (`- [ ]`) step — that's where you start.
> 2. Before starting a step, confirm all prior steps are marked `- [x]`.
> 3. After completing each step, **immediately update this file** to mark it `- [x]`.
> 4. After completing a batch (default: 3 steps), stop and report progress to the user.
> 5. When all implementation steps are `- [x]`, proceed to the **Verification Checklist** below.
> 6. Check off each verification item as you confirm it. If any item fails, fix before proceeding.

## Branch & Commit

After producing the plan, create the branch immediately:

```bash
git fetch origin master
git checkout -b chore/cleanup-[parentFlagName] origin/master
```

Commit message after execution: `chore: remove unused feature flag [parentFlagName]`
If children included, list them in the commit body.

Print a summary of all planned changes after generating the plan.

Create the directory `docs/cleanup-[parentFlagName]/` if it doesn't exist. Save the plan to: `docs/cleanup-[parentFlagName]/plan.md`

[IF_JIRA — omit this entire section if `--jira` was not provided]
## Jira

As the final plan step, add:
- [ ] **Step N: Create Jira subtask**
  Invoke `/create-jira-task [TICKET-ID] Feature Flag Cleanup --status in-progress`
[END_IF_JIRA]
````

### Fill the template

Replace all `[PLACEHOLDERS]`:
- `[INSERT: list each flag...]` → the target flag set from Step 2
- `[INSERT: the full structured summary...]` → the complete discovery output from Step 3
- `[flagName]` / `[parentFlagName]` → the parent flag's camelCase name
- `[IF_JIRA]...[END_IF_JIRA]` → include the Jira section only if `--jira` was provided; replace `[TICKET-ID]` with the actual ticket ID

### After agent completes

The agent will return the plan. Offer the user execution options:

```
Cleanup plan saved to `docs/cleanup-<flagName>/plan.md`.

Two execution options:

1. **Subagent-Driven (this session)** — interactive review between each step (safer, more control)
2. **Parallel Session (separate)** — batched autonomous execution in a new session (faster)

Which approach?
```
