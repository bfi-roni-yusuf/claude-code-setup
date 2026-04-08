---
name: test-branch
description: "Use when all implementation is complete on a feature branch and code needs to be tested before creating a PR."
disable-model-invocation: true
---

# Test Branch

Discover all code changes on the current feature branch and test them comprehensively. REST endpoints are tested via API calls (Phase 1). Non-REST code — activities, listeners, schedulers — is tested via temporary endpoints (Phase 2). Utils, mappers, and validators are routed to existing unit tests. Discovery uses `git diff` for direct changes and Serena `find_referencing_symbols` for upward tracing.

**Prerequisites:** Requires the Serena MCP plugin (for `find_referencing_symbols`). If unavailable, fall back to `Grep`-based class name search (less precise).

## 0. Base Branch Detection

Detect the correct base branch dynamically to support stacked branches:

```bash
CURRENT=$(git branch --show-current)
PARENTS=$(git branch --merged HEAD --no-merged master --format='%(refname:short)' | grep -v "^${CURRENT}$")
# If PARENTS is empty → base = master
# If PARENTS has entries → base = closest parent (fewest commits in `$parent..HEAD`)
```

**Rationale:** `git branch --merged HEAD --no-merged master` lists branches that are ancestors of HEAD but not yet in master. Excluding the current branch leaves only parent branches in the stack. The closest parent (fewest unique commits) is the correct diff base.

Use `git diff <base>...HEAD` for all subsequent diffs.

## 1. Discover Changed Files

```bash
git diff <base>...HEAD --name-only
```

Categorize each changed file:

| Category | Pattern | Routing |
|---|---|---|
| Controller | `controller/**/*.java` | Phase 1 (direct) |
| Service/Impl | `service/**/*.java` | Trace upward |
| Repository | `repository/**/*.java` | Trace upward |
| DTO | `dto/**/*.java` | Trace upward |
| Mapper | `mapper/**/*.java` | Trace upward; also Phase 2 (unit test) |
| Utility | `utils/**/*.java`, `helper/**/*.java` | Trace upward; also Phase 2 (unit test) |
| Constant | `constant/**/*.java` | Trace upward |
| Activity | `activity/**/*.java` | Phase 2 (temp endpoint) |
| Connector (RabbitMQ) | `connector/**/*.java` | Phase 2 (temp endpoint) |
| Scheduled Task | Contains `@Scheduled` or `@SchedulerLock` | Phase 2 (temp endpoint) |
| Entity | `entity/**/*.java` | Phase 2 (skip/report) |
| Config | `config/**/*.java` | Phase 2 (skip/report) |
| BPMN | `*.bpmn` | Phase 2 (skip/report) |
| Migration | `db/migration/**/*.sql` | Phase 2 (skip/report) |
| Non-Java | YAML, properties, etc. | Phase 2 (skip/report) |

## 2. Trace Files Upward

Use Serena `find_referencing_symbols` to walk each non-controller Java file upward through the dependency chain:

1. Get the class name from the changed file
2. Call `find_referencing_symbols` with `name_path` = class name and `relative_path` = file path relative to project root
3. If any referencing symbol is in `controller/` → record it for Phase 1
4. If not, recurse on each referencing class (up to **3 hops** max per branch). If a symbol has more than **10** referencing classes at any hop, list the count and ask the user which to trace (fan-out limit). Each result includes the file path — use that as `relative_path` for the next hop.
5. Record the full change chain for reporting (e.g., `ReturnUtil -> ServiceImpl -> Controller`)

**Routing:**
- Files that reach a controller → **Phase 1**
- Files that don't reach a controller → **Phase 2**
- Files that reach both → tested in **both phases** (no deduplication — they test different things)

---

## Phase 1: REST Endpoint Testing

### 3. Classify Endpoints

For each controller method reached:
- **`NEW`**: method exists in HEAD but not in base (`git diff <base>...HEAD -- <controller>`)
- **`MODIFIED`**: method exists in both but has changes
- **`AFFECTED`**: method unchanged, reached via transitive trace

Output a classification table:

```
| Endpoint | Category | Change Chain |
|---|---|---|
| GET /v2/underwriting/... | NEW | (direct) |
| PUT /v2/underwriting/... | MODIFIED | (direct) |
| GET /v1/underwriting/... | AFFECTED | ReturnUtil -> ReviewServiceImpl -> ReviewController |
```

### 4. Understand Parameters

**`NEW` endpoints** — full deep read:
- **Request DTO**: field names, types, `@JsonProperty`, superclass (`BaseSearch` for paginated)
- **Response DTO**: field names, `@JsonNaming` strategy
- **Service impl**: sort field mapping, filter logic, validation, status preconditions
- **Repository**: JPQL query (to understand what data is needed)

Pay attention to:
- `BaseSearch` subclasses: query params use **camelCase** (not snake_case) — controllers use `new ObjectMapper()` without global snake_case config
- Sort field values: check `resolveSortField` or equivalent in the service
- Enum filters: check enum values in the relevant constants class

**`MODIFIED` / `AFFECTED` endpoints** — targeted read:
- Only read the changed parts of the chain
- Focus on contract changes (new params, changed sort fields, new validations)
- Skip full deep-dive if only internal logic changed with no contract impact

### 5. Test Each Endpoint

**REQUIRED SUB-SKILL:** Read `call-api/SKILL.md` **before making any curl calls** — it references `setup.md` for auth, base URL, and common setup.

Read `call-api/test-data.md` for existing test data IDs. Test depth by category:

| Category | Depth |
|---|---|
| **NEW** | Full matrix: default params, each sort field, sort direction, each filter, combined filters, invalid sort, edge cases |
| **MODIFIED** | Focused: happy path + changed behavior |
| **AFFECTED** | Minimal: happy path only, verify 200 and response shape |

For paginated endpoints, verify: `total_elements` count, `content` sorting, filter reduction.

### 6. Seed Data if Needed

If any test returns 404/500 due to missing data:
1. Read `call-api/test-data.md` → find the **Seed Dependency Group**
2. Run the **check query** to identify missing entities
3. Invoke `/generate-seed-data` with `--fk` and `--execute` flags per entity
4. Retry the API call

### 7. Register in call-api

- **`NEW`** → always register in `call-api/SKILL.md` index + `apis/*.md` definition
- **`MODIFIED` / `AFFECTED` already in call-api** → leave as-is (update only if contract changed)
- **`MODIFIED` / `AFFECTED` NOT in call-api** → ask user whether to register

---

## Phase 2: Non-REST Code Testing

Files that didn't reach a controller in the trace.

### 8. Classify Non-REST Files

| File Category | Strategy | How |
|---|---|---|
| Activity (`activity/**`) | Temp endpoint | Expose activity logic via throwaway controller |
| RabbitMQ Listener (`connector/**`) | Temp endpoint | Expose listener's `handle()` via throwaway controller |
| Scheduled Task | Temp endpoint | Expose scheduler's core method via throwaway controller |
| Service/impl consumed only by above | Covered transitively | Tested when the consuming activity/listener/scheduler gets a temp endpoint |
| Util, mapper, validator, DTO | Unit test | Find and run existing `*Test.java`, or flag if none exists |
| Entity | Skip (report only) | Note "entity changed — verify migration exists" |
| Config | Skip (report only) | Note "global config — manual verification needed" |
| BPMN | Skip (report only) | Note "BPMN changed — manual flow review needed" |
| Migration | Skip (report only) | Note "migration added" (migration-reviewer hook covers this) |

### 9. Create Temp Endpoints

All temp endpoints go in a single file: `controller/temp/TempTestController.java`. Each endpoint gets its own `@PostMapping("/temp/test-{n}")` path. No `@Authorize` or `@FeatureFlag` (so `api-secret` auth works). Single file = easy revert. The existing `git add` safety hook (blocks `git add -A` / `git add .`) reduces the risk of accidentally staging it.

#### Activity Temp Endpoints

Activities have two patterns:
1. **`JavaDelegate`** — `execute(DelegateExecution)` reads variables like `APPLICATION_ID_VARIABLE_KEY`
2. **`BaseActivity`** — `executeActivity(DelegateExecution)` with skip/run gating via `ApplicationWorkflowConfig`

Both take `DelegateExecution` which can't be easily faked via REST. The skill:
1. Reads the activity class, identifies core logic (DB reads, Feign calls, DB writes)
2. Identifies the key input — usually `applicationId` (UUID)
3. Creates a temp controller that:
   - Accepts `applicationId` as a path param
   - Injects the same dependencies the activity uses
   - Inlines the relevant business logic (skipping `DelegateExecution` variable access)
   - No `@Authorize` or `@FeatureFlag`
4. Curls it with a test application ID from `call-api/test-data.md`

#### RabbitMQ Listener Temp Endpoints

Accept the message payload as `@RequestBody` and call the listener's processing logic directly.

#### Scheduled Task Temp Endpoints

Expose the scheduler's core method, accept any config params as query params.

#### Seed Data

If a temp endpoint returns 404/500 due to missing test data, invoke `/generate-seed-data` with `--fk` and `--execute` flags. All seeded rows use `created_by = 'test-seed'` for safe cleanup. If no seeding was needed, the cleanup step skips seed row deletion.

### 10. Unit Test Strategy

For files classified as "unit test" (utils, mappers, validators, DTOs):
1. Find the corresponding test file (e.g., `MyUtil.java` → `MyUtilTest.java`)
2. If test exists → report the command: `mvn test -Dtest=MyUtilTest -Dspring-boot.run.profiles=unit-test`
3. If no test exists → report: "No test found for `MyUtil.java` — consider adding one"

The skill reports which tests are relevant — it does **not** run `mvn` commands (blocked by hook) or write/fix tests. The user runs the tests manually.

### What the Skill Does NOT Do

- Does NOT mock `DelegateExecution` or spin up Camunda engine
- Does NOT start BPMN processes (functional test territory)
- Does NOT create new unit tests (only finds existing ones)
- Does NOT set up WireMock or tunnels for Feign clients

### Edge Cases

**Activities calling external Feign clients:** Many activities call external APIs (Pefindo, AliCloud, StrategyOne). On local, these fail unless the service is reachable (SIT tunnel, WireMock). Report Feign failures as "External dependency unreachable" — not code bugs.

**BaseActivity skip/run gating:** Temp endpoint inlines business logic directly, bypassing the skip check. Intentional — testing the logic, not the gating.

**Multiple activities changed:** All temp endpoints in `TempTestController.java` with sequential `/temp/test-1`, `/temp/test-2` paths mapped in the report.

**Files tracing to both controller and activity:** Tested in both phases, both results reported. No conflict.

**No changed Java files:** Both phases produce empty results. Report shows only skip/report-only items. Skill finishes quickly.

---

## 11. Combined Report

### Change Impact Summary

```text
Base branch: <detected-base>
Changed files: N
  -> REST endpoints: X (Phase 1)
  -> Non-REST paths: Y (Phase 2)
  -> Skip/report-only: Z
```

### Phase 1 Results

| Endpoint | Category | Change Chain | Tests | Pass | Fail |
|---|---|---|---|---|---|
| `GET /v2/underwriting/...` | NEW | (direct) | 9 | 8 | 1 |

### Phase 2 Results

| Changed File | Consumer | Strategy | Result | Notes |
|---|---|---|---|---|
| `MyActivity` | Camunda activity | Temp endpoint | 200 OK | Tested with app `abc-123` |
| `SomeListener` | RabbitMQ listener | Temp endpoint | 200 OK | |
| `ReturnUtil` | Used by `MyActivity` | Covered transitively | Covered | Via MyActivity temp endpoint |
| `DateUtil` | util | Unit test | 3/3 passed | |
| `MyMapper` | mapper | Unit test | No test file | |
| `V2_0_202603...sql` | migration | Skip | — | Migration added |
| `unified-main-workflow.bpmn` | BPMN | Skip | — | Manual flow review needed |

### Cleanup Status

```text
Pending cleanup:
  - controller/temp/TempTestController.java
  - Seed rows: DELETE FROM ... WHERE created_by = 'test-seed' (if any were seeded)

Say "clean up" when done testing.
```

## 12. Cleanup

Temp endpoints and seed data remain after tests complete so the user can manually re-test. Once the user says "done" or "clean up":

1. `git checkout -- controller/temp/TempTestController.java` (or delete if new)
2. Execute `DELETE FROM ... WHERE created_by = 'test-seed'` for any seeded rows (skip if none were seeded)
3. Report cleanup complete

## Workflow Tracking

**REQUIRED SUB-SKILL:** After section 12 cleanup completes (user said "clean up" and temp endpoints/seed data are removed), use `update-workflow` to mark phase `test` as `done`.

If the skill is testing a multi-part plan (parts exist in the tracker), mark the relevant part of `test` as done instead of the parent phase.
