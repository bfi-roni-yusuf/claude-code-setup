# CLAUDE.md

## Project Overview

Camunda Platform 7 (7.23.0) BPM orchestration service (Java 17, Spring Boot 3.5.x) for BFI Finance's loan origination system ("Bravo"). Orchestrates loan workflows across product types (NDF 2W/4W, unsecured, sharia, multi-asset) by coordinating ~30+ external microservices.

### Prerequisites

- Java 17 (JDK)
- Maven 3.8+
- Docker & docker-compose (for local infrastructure)
- Make (GNU Make)

## Build & Run Commands

### Local Database Access

```bash
PGPASSWORD=postgres psql -h localhost -p 5432 -U postgres -d bpm   # Direct psql access (check application-local.yaml for actual port/creds)
```

### Make Targets

```bash
make run-local                # Run with default,local profiles (needs docker-compose infra)
make build                    # Clean + verify (compile + test)
make compile                  # Package without tests
make compile-local            # Package without tests, skip spotless
make unit-test-and-report     # Run tests with unit-test profile
make check-style              # Spotless check
make fix-style                # Spotless auto-fix
make integration-test          # Run integration tests
make stack-up                  # Build image + start all services via docker-compose
make stack-down                # Stop docker-compose services
make stack-logs                # Tail docker-compose logs
make reset                     # Clean + build + run local
docker-compose up -d           # Start local infra (Postgres:5430, RabbitMQ:5672, Redis:6379, Keycloak:8180)
```

### Running a single test

```bash
mvn test -Dtest=MyTestClassName -Dspring-boot.run.profiles=unit-test
mvn test -Dtest=MyTestClassName#methodName -Dspring-boot.run.profiles=unit-test
```

### Code formatting

Spotless (wrapping prettier-java 1.6.2) is enforced via Maven plugin (120 char line width, 2-space indent, no tabs). CI will fail on style violations. Run `make fix-style` before committing. Only files changed vs `origin/master` are checked (`ratchetFrom`). Additional steps: `trimTrailingWhitespace`, `endWithNewline`, `removeUnusedImports`, `formatAnnotations`. The formatter runs automatically during `validate` phase. **Inner class imports caveat**: `removeUnusedImports` strips static imports of inner classes (e.g., `import CommonConstants.OnlineGamblingIndicationStatus`). Always fully qualify inner enums/classes as `CommonConstants.InnerEnum` instead of static-importing them.

## Architecture

> **Detailed patterns**: See the `bpm-conventions` skill for full recipes, edge cases, and code examples for all sections below.

### Package Structure (`com.bfi.bravo`)

| Package | Role |
|---|---|
| `activity/` | Camunda `JavaDelegate` implementations — one class per BPMN service task |
| `activity/unified/` | Product-agnostic activities using factory/strategy pattern (~59 classes) |
| `activity/ndf2w/`, `activity/unsecured/`, `activity/sharia/`, `activity/ro/` | Product-specific activities |
| `activity/multiasset/`, `activity/longsurveyscoring/`, `activity/preapproval/` | Additional product/feature activities |
| `activity/operation/`, `activity/underwriting/` | Workflow-phase-specific activities |
| `connector/` | RabbitMQ publishers/listeners for async messaging |
| `adapter/http/` | ~50 Feign client packages (~112 client interfaces) for external service integration |
| `service/` | Business logic layer |
| `service/v2/`, `service/impl/v2/` | V2 API service interfaces and implementations — new V2 endpoints go in separate V2 services (e.g., `UnderwritingReviewV2Service`), not in the original service. Follows `UnderwritingDocumentCheckV2Service` precedent. V2 controllers inject both old and V2 services as needed. **Method naming**: Drop the `V2` suffix from method names — the `*V2Service` namespace already signals versioning (e.g., `saveAsDraftReviewRegularApplication`, not `...V2`). **Feature flags**: V2 endpoints behind newer flags (e.g., `featNDF4WCompanyCustomerProfile`) can omit older flag branches when the V2 DTO structure makes those branches structurally impossible (removed fields = can't execute old code path). |
| `controller/` | REST API endpoints (SpringDoc OpenAPI) |
| `repository/` | Spring Data JPA repositories (PostgreSQL, `CrudRepository`-based; use `JpaRepository` when pagination is needed) |
| `entity/` | JPA entities (~250 domain models) |
| `dto/` | Request/response DTOs organized by domain (`dto/{domain}/request/`, `dto/{domain}/response/`) |
| `config/` | Spring configuration (security, CORS, Redis, RabbitMQ, Feign, feature flags) |
| `constant/` | Process variable keys (`WorkflowConstants`), enums (~78 files) |
| `mapper/` | MapStruct mapper interfaces between entities and DTOs. Two patterns: `Mappers.getMapper()` singleton (e.g., `mapper/underwriting/`) and Spring `componentModel` (e.g., `mapper/surveyorassignment/`). Match the convention of the target package. |
| `annotation/` | Custom annotations (e.g., `@FeatureFlag`) |
| `auth/` | Authentication/authorization utilities |
| `exception/` | Custom exception classes (`BravoCommonException`, error constants) |
| `factory/` | Factory classes for creating domain-specific strategies/handlers |
| `helper/` | Shared helper/utility classes |
| `projection/` | JPA projections for optimized queries |
| `specification/` | JPA Specification builders for dynamic queries |
| `utils/` | General utility classes (`ApplicationUtil`, `PageRequestUtil`) |
| `validation/` | Custom validators and validation logic |

### Camunda Activity Patterns

Activities follow **two patterns**:

1. **`BaseActivity` subclass** — checks `ApplicationWorkflowConfig` to decide skip/run per application. Override `executeActivity(DelegateExecution)`.
2. **Direct `JavaDelegate`** — implements `JavaDelegate.execute()` directly. Used by most unified activities.

Service tasks reference Spring beans via `camunda:delegateExpression`, e.g., `#{setApplicationStatusUnifiedActivity}`. **Deleting activity classes:** Old process instances on previous BPMN versions still resolve `delegateExpression` beans at runtime. For service tasks (instant execution, near-zero risk), delete freely. For user tasks or message waits, keep beans until old instances drain or are migrated.


### BPMN Workflow Structure

~50 BPMN files in `src/main/resources/bpmn/`. Main workflow uses **modular Call Activity composition**:

- `unified-main-workflow.bpmn` → orchestrates loan lifecycle via Call Activities
- Sub-workflows: `unified-workflow-check.bpmn`, `unified-workflow-initial-scoring.bpmn`, `unified-workflow-survey.bpmn`, `unified-workflow-underwriting.bpmn`, `unified-operation-workflow.bpmn`
- Product-specific: `ndf2w.bpmn`, `ndf4w.bpmn`, `ndf4w-sharia.bpmn`, `unsecured.bpmn`

**Workflow lifecycle phases** (mapped to `WORKFLOW_GROUP` constants C, I, S, U, O):
Check → InitialScoring → Survey → Underwriting → Operation

**Link events are subprocess-scoped**: Link throw/catch events only connect within the same process level — they cannot cross subprocess boundaries. To route from inside a subprocess to a process-level link catch (e.g., "Salestrax Escalation"), use error-based propagation: error end event (inside subprocess) → boundary error event (on subprocess) → link throw event (at process level). See `ndf2w.bpmn` `Activity_04euzqy` for reference.

### Process Variables

Always use constants from `WorkflowConstants` (~355 lines). Key variables:
- `APPLICATION_ID_VARIABLE_KEY` — UUID identifying the application
- `WORKFLOW_STATUS_TO_BE_SET_KEY` — target application status (local variable)
- `WORKFLOW_RISK_TYPE_TO_BE_SET_KEY`, `WORKFLOW_CREDIT_MODEL_APPROVAL`, `WORKFLOW_SURVEY_TYPE`

## Integration Patterns

### External Services

**General LOV**: `general_lov` uses `group_name` as category identifier (not `code`). Detail items in `general_lov_detail` link via `general_lov_id`. Hierarchical LOVs use `parent` column pointing to parent `general_lov.id`. **Repository query trap**: `findByParentGroupName(name)` finds details under *child groups* whose parent has the given name (grandparent lookup). `findByGroupName(name)` finds details *directly* under the group. For UW negative info LOV (e.g., `UW_NEGATIVE_INFO_NEGATIVE_CASE`), details are stored directly — use `findByGroupName`.

**Feign Clients**: URL via `${setting.xxx.microservices.rootUrl}`, auth via `api-secret` header + `X-Idempotency-Key`. Per-client timeouts in `application-*.yaml`.

**RabbitMQ Messaging**:
- `CommonPublisher`: direct queue, fire-and-forget
- `CommonPublisherWithExchange`: exchange-based with **outbox pattern** — persists to `EventStore` before sending
- `CommonListenerWithExchange`: configurable retry with backoff via `X-Retries-Count` header
- Multi-broker: primary + Confins-specific broker (`spring.rabbitmq.multirabbitmq.confins.*`)

### Feature Flags

**Toggle mechanism**: `@Value("${setting.feature.config.featXxx}")` booleans. Check existing flags before adding new conditional behavior.

**Naming**: camelCase `featXxxYyy` in YAML, env var `ENABLE_FEATURE_CONFIG_XXX_YYY`. Product-scoped flags use prefix (e.g., `featNDF4W*`, `featNDF2W*`). New flags must be registered in both `application.yaml` (default `false`) and `application-local.yaml` (set `true` for local dev). After merging master, check for duplicate YAML keys — Spring Boot 3.x rejects duplicates at startup (`found duplicate key`). Check sub-keys too (e.g., two `review:` blocks under the same `underwriting:`).

**`@FeatureFlag` annotation**: Controllers use `@FeatureFlag(feature = "featXxx")` on endpoints — returns 404 when disabled. **Must be method-level** — `FeatureInterceptor` uses `getMethodAnnotation()`, so class-level `@FeatureFlag` is silently ignored. Service methods do NOT duplicate this check (controller is the single gate, keeping services flag-agnostic). In `@WebMvcTest` with `@ContextConfiguration(classes = { Controller.class })`, `FeatureFlagMethodAspect` is not loaded — flags are effectively ignored in tests.

### DTOs & Response Wrappers

**DTO base classes** (`dto/common/`):
- `BaseResponse<T>` — has `T id` + `int version`. Response DTOs with those fields must extend it. Requires constructor-level `@Builder` with `builderMethodName` (no class-level `@Builder`).
- `BaseSearch` — has `page`, `size`, `sort`, `sortDirection`. Paginated search request DTOs must extend it.

**`AssetPageResponse<T>`** (`dto/underwriting/response/`): Generic paginated response with summary status for asset list endpoints. Reuse instead of creating new page DTOs.

**Pagination** *(enforced by `enforce-page-request-util` hook)*: `PageRequestUtil.of(page, size, sort)` — caps page size via `setting.size.maxSizePerPage`. Never use `PageRequest.of()` or local `MAX_PAGE_SIZE`.

**Response wrappers**: `BravoBaseResponse { header, data }` (legacy) and `BravoCommonResponse<T> { code, data, details, message }` (newer TRD standard). Use `BravoCommonResponse.success(data)` for happy path. **Wrapping belongs in controllers** — services return raw DTOs, controllers wrap with `BravoCommonResponse.success(...)`.


### ApplicationUtil

**Reflected Entities**: Always use `ApplicationUtil.reflectedLoan(application)` instead of `application.getLoan()` — it returns `newLoan` if present (from amendment/restructuring), otherwise the original `loan`. Same pattern: `reflectedAsset()`, `reflectedCalculation()`, `reflectedSimulation()`.

**Status Filtering**: `ApplicationUtil.filterApplicationByStatus(application)` returns `true` for active, `false` for terminal (DONE, REJECTED, CANCELLED, DISBURSED, etc.). For paginated endpoints: apply as **pre-query guard** (not post-query stream filter — breaks page sizes). When rows link to different applications, add per-row JPQL filtering: `a.status NOT IN :excludedStatuses`. Reference: `findPageByUnderwritingId`, `getCollateralDetailByUnderwritingId`.

## Underwriting Domain Patterns

**UnderwritingSummary dual-signature pattern**: All methods have two overloads — `method(UUID applicationId, String nik)` (controller) and `method(UUID applicationId, Runnable validation)` (internal). Implement both when adding new summary endpoints.

**Audit fields on save**: Always call `entity.setUpdatedBy(caNik)` before saving underwriting-related entities — applies to ALL save paths including related entities.

**Status transitions on save**: Save/update operations transition status to WIP (`ASSIGNED` → `WIP_DOCUMENT`, `DOCUMENT_CHECKED` → `WIP_REVIEW`). Idempotent — skip if already WIP. Structure: `setUpdatedBy` + `save` unconditional, `setStatus` + `recordHistory` inside `if` guard.

**Review phase entity hierarchy**: `UnderwritingReviewSummary` → `UnderwritingReviewCollateral` (FK to `UnderwritingCollateralDetail`) + `UnderwritingReviewAdditionalNote` (type=`DOCUMENT`). Mirrors doc-check phase.

**CFCAT (review data for PT/Company)**: 4 types (General, Capacity = per-summary; Structure, Collateral = per-asset) under `featNDF4WCompanyCustomerProfile`. Validation required when `totalExposure < 3B IDR`.

**Review collateral ownership**: Always use `findReviewCollateralById(collateralId, uwId, excludedStatuses)` — `findById(collateralId)` alone is an IDOR vulnerability.

**Return reason service symmetry**: `UnderwritingDocumentCheckReturnReasonServiceImpl` and `UnderwritingReviewReturnReasonServiceImpl` are parallel — keep aligned when modifying either.

**Uniqueness constraint**: `findUnderwritingByApplicationIdAndIndex` expects exactly one active (non-deleted) underwriting per `applicationId`.

**"Daftar Info Negatif"**: LOV-based negative info list with exclusive filtering logic and known `isNotIndicated()` bug. See `bpm-conventions` skill for full details (LOV codes, FE contract, `filterExclusiveNegativeInfo`).

**Approval response `@JsonIncludeProperties` trap**: V1 approval DTOs (`UnderwritingNegotiationApplicationLoanResponse`, `...CalculationResponse`) extend base response classes but use `@JsonIncludeProperties` to filter exposed fields. `loan` only exposes `ltv` + `package_name` (NOT `amount`). `calculation` exposes `ntf_amount`, `capitalized_ntf_amount`, `tenor`, `rounded_up_installment`, `ltv`. When building new approval endpoints, "Loan Value" = `reflectedCalculation().getNtfAmount()`, not `reflectedLoan().getAmount()`.


## Database & Migrations

- **PostgreSQL** with **Flyway** (`spring.flyway.outOfOrder: true`)
- ~1217 migration files in `src/main/resources/db/migration/`
- **Naming convention**: `V2_0_YYYYMMDDHHMI__description.sql` (timestamp-based, double underscore separator)
- Example: `V2_0_202602031613__alter-table-operation-assignment.sql`
- **Soft-delete in migrations**: All entities extend `BaseEntity` (which has `deleted` column). In JPA, `@SQLRestriction("deleted=false")` handles filtering automatically, but Flyway migrations are raw SQL — always add `AND deleted = false` when selecting from any table that has the column.
- **JPQL explicit soft-delete consistency**: When writing custom `@Query` JPQL with explicit `AND x.deleted = false` on some joins, add it for ALL `BaseEntity` joins in that query. Don't mix explicit and implicit (`@SQLRestriction`) filtering in the same query.
- **Index migrations**: Use `CREATE INDEX CONCURRENTLY IF NOT EXISTS` for new indexes — Flyway auto-detects `CONCURRENTLY` and runs the migration as `[non-transactional]`.
- **Index naming**: Two patterns coexist — `idx_<table>_<col>` (older/prefix) and `<table>_<col>_idx` (newer/suffix). Prefer the **suffix** convention `<table>_<col>_idx` for new indexes — it's the established pattern for recent migrations.

## Testing

- **Unit tests**: JUnit 5 + Mockito. Use `@MockitoSettings(strictness = Strictness.LENIENT)`, `@Mock`, `@InjectMocks`. Do NOT use `@ExtendWith(MockitoExtension.class)` — `@MockitoSettings` already includes it.
- **Functional tests**: `functional/` package — BPMN process integration tests using Camunda BPM Assert + Camunda Mockito.
- **Test DB**: H2 in-memory with random UUID (`jdbc:h2:mem:testdb-${random.uuid}`).
- **WireMock stubs**: `src/test/resources/__files/` for HTTP client mocking.
- **Test data**: `src/test/resources/data/` for JSON fixtures.
- Tests run in parallel (JUnit Jupiter parallel execution enabled, dynamic factor 1).
- **Testing private methods**: Use `ReflectionTestUtils.invokeMethod(instance, "methodName", args...)` to test private validation methods directly. Feature flags set via `ReflectionTestUtils.setField(instance, "fieldName", value)`.
- **MockMvc null assertions**: Use `jsonPath("$.field", nullValue())` — `isEmpty()` throws on JSON `null`.
- **MockMvc Page property paths**: Use `$.total_elements` not `$.totalElements` (global snake_case Jackson config).
- **Exception types in test stubs**: Older `findById` methods throw `ResponseStatusException(NOT_FOUND)`, newer use `BravoCommonException` — check impl before writing stubs.
- **`BravoCommonException` assertion method**: Use `.getErrorMessage()` (returns enum) — NOT `.getMessage()` (returns `String`). Enum.equals(String) silently fails.
- **`@Version` in test builders**: Set `.version(0)` in entity builders to avoid null `Integer` auto-unbox NPE — but only on **constructor-level** `@Builder` entities (e.g., `underwritingReviewCollateralBuilder()`, `underwritingReviewSummaryBuilder()`). **Class-level** `@Builder` entities (`Underwriting`, `Application`, `Calculation`, `UnderwritingApprovalCollateral`, `UnderwritingApprovalApprover`, `SurveyorAssignment`, `UnderwritingCollateralDetail`) don't expose `BaseEntity` fields — omit `.version(0)` (follow existing test patterns).
- **`ErrorHandlerController` for error tests**: `@ContextConfiguration(classes = { Controller.class })` doesn't load `@ControllerAdvice`. To test error responses (e.g., `BravoCommonException` → 400), add `ErrorHandlerController.class` to the context: `@ContextConfiguration(classes = { ErrorHandlerController.class, Controller.class })`.
- **`BaseUnderwritingReviewData` subclass builders**: `.id(id)` silently fails — use `new Entity()` + `setId()` instead.
- **Test file naming**: Strict 1:1 — `FooImpl.java` → `FooImplTest.java`. Never split test files.


## Conventions

- **Jackson**: snake_case property naming globally, case-insensitive enums. Don't add redundant `@JsonProperty` on DTOs. **Exception**: Search DTOs via `ObjectMapper().convertValue()` DO need `@JsonProperty`.
- **Jackson getter mangling on S1 DTOs**: Fields like `aScore`/`bScore` cause duplicate JSON properties — fix with `@JsonAutoDetect(fieldVisibility = ANY, getterVisibility = NONE, isGetterVisibility = NONE)`. **Both** `getterVisibility` and `isGetterVisibility` are needed — `isGetterVisibility` suppresses `isXxx()` methods (boolean properties), which `getterVisibility` alone does not.
- **Lombok**: `@Slf4j`, `@Builder`, `@Data`, `@AllArgsConstructor` used extensively on entities and DTOs.
- **DTO annotation standard**: Always `@Data @Builder @NoArgsConstructor @AllArgsConstructor` (in that order), `private` fields. Extending `BaseResponse`/`BaseSearch`: constructor-level `@Builder(builderMethodName)`, no class-level `@Builder`. Order with `@EqualsAndHashCode`: `@Data` → `@EqualsAndHashCode` → `@Builder` → `@NoArgsConstructor` → `@AllArgsConstructor`.
- **Request DTO validation**: `@Valid @RequestBody`, boxed types (`Integer`, `Boolean`) with `@NotNull`, `@NotBlank` for strings. Primitives silently default — use boxed types for required fields.
- **Scheduling**: Uses ShedLock (`@SchedulerLock`) for distributed lock on scheduled tasks.
- **Soft deletes**: entities use `deleted` boolean + `deletedAt` timestamp, queries use `findBy...AndDeletedIsFalse`.
- **`BravoCommonException` constructor**: `BravoCommonException(BravoCommonErrorCode, ErrorMessageConstant)` — NOT `(ErrorMessageConstant, HttpStatus)`. There is no generic `DATA_NOT_FOUND` enum value; use domain-specific ones like `UNDERWRITING_REVIEW_SUMMARY_NOT_FOUND`.
- **Service finder naming**: `findEntity*` methods throw `BravoCommonException(NOT_FOUND)` when not found; `findOptionalEntity*` methods return `Optional` for cases where absence is a valid business scenario (e.g., returning `null` to the API).
- **Service layer boundaries**: Services must not inject repositories from other domain packages. Use the corresponding service interface instead (e.g., use `SurveyorAssignmentFinancialStatementService`, not `SurveyorAssignmentFinancialStatementRepository` from `UnderwritingSummaryServiceImpl`). When calling another service's `findById` (which already throws `NOT_FOUND`), don't wrap it in `orElseThrow` — call it directly. When adding a new repository method for a paginated query, also add the corresponding service method to the service interface and implementation — never call the repository directly from a different service.
- **`ApiPathConstants` in underwriting controllers**: Underwriting controllers extract sub-path segments (e.g., `/assets`, `/customers/{id}`) to `ApiPathConstants` constants. Other domains (surveyor, operation) use inline strings — but underwriting convention requires constants.
- **Correlation ID**: `CorrelationIdFilter` propagates tracing IDs across service calls.
- **Auth**: Keycloak OAuth2/JWT + internal service-secret auth via `JwtTokenFilter`.
- `api-secret` header grants only `ROLE_SYSTEM_SERVICE` — works for endpoints without role-specific `@Authorize`. Endpoints with `@Authorize(role = "UW_CREDIT_ANALYST,...")` require a Keycloak JWT with matching roles.
- `SecurityContextProvider.getCurrentUserId()` returns JWT `preferred_username` — throws `ClassCastException` with `api-secret` auth.
- **Constructor injection only** *(enforced by hook)*: Always use `@RequiredArgsConstructor` + `private final` fields for dependency injection. Never use field-level `@Autowired`. Setter injection only for genuinely optional dependencies. **Exception**: Controllers extending `BaseController` can't use `@RequiredArgsConstructor` — `BaseController` requires `SecurityContextProvider` in its constructor, so subclasses must write an explicit constructor with `super(securityContextProvider)` and manually assign other dependencies. **Legacy inconsistency**: Some classes have `@RequiredArgsConstructor` but also `@Autowired` non-final fields — this is an inconsistency, not a pattern. When adding new fields to such classes, always use `private final` (consistent with `@RequiredArgsConstructor`), never copy the `@Autowired` style.
- **Shared constants placement**: Domain-specific string constants that relate to existing constants in `AppConstants` (e.g., LOV codes alongside `NOT_INDICATED_SUFFIX`) belong in `AppConstants`, not as `private static final` in service impls. Only truly implementation-private constants (e.g., internal buffer sizes) stay in the class.
- **Paginated JPQL sorting**: Use `JpaSort.unsafe(direction, "alias.field")` — standard `Sort.by()` fails to resolve JPQL path aliases in custom `@Query` methods.
- **JPQL `LOWER()`/`UPPER()`**: Verify functional index exists before using in WHERE — forces sequential scan without one. Exception: `saci.licenseNumber`/`variantCode` stored uppercase — only `UPPER()` the parameter.
- **JPQL nullable `String` params**: Use `CAST(:param AS string)` in both `IS NULL` check and `CONCAT` — Hibernate 6 sends null strings as `bytea` to PostgreSQL. H2 won't catch this.
- **JPQL enum string comparison**: Use `CAST(entity.field AS string) = :stringParam` for `@Enumerated(EnumType.STRING)` columns against String parameters.
- **`ErrorMessageConstant` naming**: Generic names for non-domain errors (`INVALID_SORT_FIELD`), domain prefix only when semantically specific (`UNDERWRITING_COLLATERAL_NOT_OWNED`).
- **No eager `@Transactional(readOnly = true)`**: Only add when the method performs multiple related reads requiring a consistent snapshot.
- **Trim search input**: Always trim user-provided search/filter strings before passing to repository queries. Use `StringUtil.trimOrNull(value)` — returns trimmed string or `null` if blank. Do the trim once at the beginning of the service method and use the trimmed value everywhere.
- **Reuse existing DTOs**: Before creating new response/request DTOs, search for existing ones in the same domain package. Inner classes (e.g., `ReturnDetail`) or shared DTOs may already cover the fields needed. Only create new DTOs when the shape is genuinely different.
- **Local-only directories**: `docs/` is fully gitignored — task artifacts (analysis, plans, specs) are organized as `docs/<task-id>/` folders, local only, never committed. `CLAUDE.md` itself is also gitignored — edit directly on any branch, don't include in commits.
- **Git workflow**: Use normal `git checkout` for branch switching — no worktrees. When diffing a feature branch, use `git diff master...origin/<branch-name>`. **Stacked branches**: Check `git log --oneline HEAD --not master` first — if commits from another feature branch are present, diff against that parent branch instead (e.g., `git diff feat/BLCS-2654...HEAD`).


## Skill Priority (Java)

Three baseline skills define best-practice defaults for new code:
- `java-springboot` — Spring Boot patterns (DI, config, controllers, services, repos)
- `java-junit` — JUnit 5 testing (parameterized tests, @Nested, @DisplayName, AAA pattern)
- `java-docs` — Javadoc on non-obvious public/protected members only (skip boilerplate DTOs, getters, straightforward services)

**Resolution rule:**
1. `bpm-conventions` explicit rule → wins (deliberate project override)
2. `bpm-conventions` silent → baseline skill wins
3. Structural patterns (package layout, `@Slf4j`, existing class organization) → never change for consistency

**New code expectations:**
- Tests: use `@ParameterizedTest` for multi-case, `@Nested` for grouping, `@DisplayName` for readability, `methodName_should_X_when_Y` naming
- Javadoc: only on complex logic, non-trivial interfaces, and methods where behavior isn't obvious from the signature
- Spring patterns: follow java-springboot unless bpm-conventions has an explicit override

## CI/CD

- **PR pipeline** (`pr-pipeline.yaml`): Spotless check → compile → Snyk security scan → unit tests + SonarQube analysis.
- **CD pipeline**: `make build` → Docker image → GCP Artifact Registry → deploy SIT → UAT → PROD.
- Sharia variants have separate pipelines.
- Branch naming: `feat/TICKET-ID`, `fix/TICKET-ID`, `hotfix/TICKET-ID`.
- Commit style: `feat(scope): description`, `fix(scope): description`, `chore(release): version`.
- **PR base branch**: `master` (default for all features/fixes).
- **Hotfix strategy**: Create two branches — one targeting `master`, one targeting `release-*`. Both merged separately. Base each branch off its respective `origin/` remote (e.g., `git checkout -b hotfix/xxx origin/master`) to avoid carrying local-only commits.
- **PR format**: Always create as **draft**. Fill the repo's PR template (JIRA link, description, type of change, testing, checklist).
- **PR review comments**: Always use GitHub suggestion syntax (` ```suggestion `) for ANY actionable code change — never plain code blocks. This lets the author apply fixes with one click.
- **`gh` CLI SSH alias quirk**: Remote uses `githubwork` SSH alias, so prefix all `gh` commands with `GH_HOST=github.com` (e.g., `GH_HOST=github.com gh pr view 123`).
- **Pre-commit hook**: `.git/hooks/pre-commit` auto-runs `make fix-style` on staged Java files. Branch-aware — delegates to whatever formatter the Makefile configures (spotless on master, prettier on release). Personal/local only (not in git). Bypass with `--no-verify` if needed.

## Configuration Profiles

`src/main/resources/application-{profile}.yaml`: `local`, `local-standalone`, `dev`, `sit`, `uat`, `prod`, `test`, `unit-test`. Local dev uses `default,local`; docker stack uses `default,local-standalone`.

## Claude Code Automations

### Skills (invoke with `/skill-name`)

| Skill | Usage | Description |
|-------|-------|-------------|
| `/analyze-task <ticket>` | Clarify requirements | Surfaces requirement ambiguities interactively. Saves to `docs/<ticket-id>/analysis.md`. |
| `/bpm-conventions` | Writing/reviewing Java | Java/Spring conventions, recipes, edge cases, PR review checklist. Auto-triggers. |
| `/call-api <name>` | Test local endpoints | Calls local APIs with auto-generated data. No args = list APIs. |
| `/cleanup-feature-flag <flagName>` | Remove unused flag | Discovers flag usages across Java/YAML/BPMN/tests, generates cleanup plan. `--keyword`, `--jira`. |
| `/create-migration <desc>` | New Flyway migration | Generates migration with correct `V2_0_YYYYMMDDHHMI__description.sql` naming. |
| `/generate-seed-data <entity>` | Seed test data | SQL INSERTs from JPA entities. `--fk`, `--rows`, `--execute`. |
| `/create-plan <ticket>` | Plan implementation | Full planning methodology. Caller orchestrates review after. |
| `/review-plan [plan-file]` | Review a plan | Validates completeness, correctness, requirement traceability. |
| `/create-jira-task <args>` | Create any Jira issue | Any issue type. `--parent`, `--assignee`, `--status`. Confirms before creating. |
| `/fix-dto [target]` | Standardize DTOs | Fixes annotations, field visibility, `BaseResponse`/`BaseSearch` patterns. Accepts path/class/`--all`. |
| `/fetch-jira-context <ticket>` | Understand a ticket | Full mode: ticket + TRD. `--lightweight`: metadata only. Sub-skill used by other skills. |
| `/address-review [PR#]` | Address PR review | Evaluates reviewer comments 1-by-1, implements fixes, posts replies. Evidence: TRD > AC > plan > code. |
| `/review-pr <number> [aspects]` | Review a PR | Multi-aspect review. Aspects: `code`, `tests`, `errors`, `comments`, `types`, `simplify`, `all`. |
| `/create-pr [ticket]` | Create draft PR | Auto-generates title/body from plan, memory, git changes. |
| `/start-task <ticket>` | Start new task | Checks out master, creates branch, dispatches plan-creator agent. |
| `/test-branch` | Test all branch changes | REST via API calls, non-REST via temp endpoints, utils via unit tests. |
| `/workflow [TICKET-ID]` | Show workflow progress | Phase completion status. `skip <phase>`, `done <phase>`. Auto-detects ticket from branch. |

### Custom Agents

| Agent | Trigger | Description |
|-------|---------|-------------|
| `migration-reviewer` | After creating/modifying migrations | Reviews Flyway SQL for naming conventions, soft-delete safety (`AND deleted = false`), index best practices (`CONCURRENTLY`, partial index `WHERE deleted = false`), and SQL correctness. Auto-invoked by `pr-review-toolkit`. |
| `plan-creator` | When given a Jira ticket or asked for a plan | Orchestrates plan creation + review loop. Renamed from `implementation-planner`. |
| `plan-reviewer` | After creating a plan | Reviews plan for completeness, correctness, and requirement traceability. |
| `implementation-reviewer` | After executing an implementation plan | Reviews implementation against the plan and analysis docs (which already contain distilled TRD contracts and acceptance criteria). Verifies plan compliance, contract accuracy, and CLAUDE.md conventions. No direct Jira/TRD fetching — uses local docs only. Complements `pr-review-toolkit` (which covers code quality). |

### Hooks (automated in `settings.local.json`)

| Hook | Type | What it does |
|------|------|-------------|
| `git add` safety | PreToolUse | **Blocks** `git add -A`/`.`/`--all` — stage specific files. |
| `mvn` command safety | PreToolUse | **Blocks** `mvn compile/test/verify/clean/package/install`. |
| `wildcard-import-guard` | PreToolUse | **Blocks** wildcard imports (`import .*.*;`). |
| `constructor-injection-guard` | PreToolUse | **Warns** on field `@Autowired` — use `@RequiredArgsConstructor`. |
| `personal-config-edit-guard` | PreToolUse | **Warns** on `application-local.yaml`/`docker-compose.yaml` edits. |
| `skills-cli-guard` | PreToolUse | **Blocks** `skills add/install` — copy SKILL.md manually. |
| `migration-soft-delete` | PostToolUse | **Warns** on SQL without `deleted` clause. |
| `check-flyway-naming` | PostToolUse | **Warns** on non-standard migration filename. |
| `dto-noargs-constructor-guard` | PostToolUse | **Warns** on `@Builder` without `@NoArgsConstructor`. |
| `enforce-page-request-util` | PostToolUse | **Warns** on `PageRequest.of()` — use `PageRequestUtil.of()`. |
| `mockito-settings-guard` | PostToolUse | **Warns** on `@ExtendWith(MockitoExtension.class)` — use `@MockitoSettings`. |
| `enforce-reflected-entities` | PostToolUse | **Warns** on direct `getLoan()`/`getAsset()` etc. — use `ApplicationUtil.reflected*()`. |
| `feature-flag-class-level-guard` | PostToolUse | **Warns** on class-level `@FeatureFlag` — must be method-level. |
| `exception-assertion-guard` | PostToolUse | **Warns** on `.getMessage()` in tests — use `.getErrorMessage()`. |
| `feature-flag-dual-registration` | PostToolUse | **Reminds** to register `feat*` in both `application.yaml` and `-local.yaml`. |
| `service-response-wrapping-guard` | PostToolUse | **Warns** on `BravoCommonResponse` in service impl — controllers only. |
| `sql-restriction-guard` | PostToolUse | **Warns** on `BaseEntity` without `@SQLRestriction("deleted=false")`. |

### MCP Servers

Atlassian plugin enabled for Jira/Confluence access. Context7 and Serena available via global plugins. Use `psql` via Bash for local database access (see "Local Database Access" above).

## Interactive Workflows

When reviewing or creating implementation plans, NEVER rush ahead to the next question or assume conclusions. Wait for explicit user confirmation before moving on to the next item in any interactive Q&A flow.

## Planning & Plan Generation

Before generating implementation plans, verify: 1) correct constructor signatures from actual source code, 2) correct enum values from actual source code, 3) constants belong in the right class by checking existing patterns. Never assume — always grep/read first.

After generating any plan or spec document, self-proofread for: file map vs task mismatches, incorrect line numbers, truncated class names, stale draft text, and table formatting consistency before presenting to the user.

## General Behavior

When the user asks to 'resend' or 'repeat' a previous response, simply repeat the prior output. Do NOT re-read files or re-execute the workflow from scratch.

## Code Investigation

When a user describes an API endpoint or feature, do NOT assume it already exists or guess its structure. Ask clarifying questions first, or grep the codebase to confirm before proceeding.

## MCP / Tool Dependencies

Before any task requiring Jira access, verify the Atlassian MCP plugin is connected and authenticated. If unavailable, immediately inform the user rather than looping through ToolSearch attempts.
