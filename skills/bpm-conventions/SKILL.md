---
name: bpm-conventions
description: "Java/Spring conventions for bravo-bpm-service. Use when writing, reviewing, testing, or modifying Java code — covers DTOs, services, controllers, repositories, tests, migrations, JPQL, and PR review standards."
---

# BPM Conventions

Java/Spring convention reference for bravo-bpm-service. Loaded automatically during implementation and review.

## When to Use
- Writing or modifying Java code in this project
- Reviewing PRs that touch DTOs, services, controllers, JPA queries, or migrations
- Not needed for: BPMN-only changes, YAML-only config, documentation, skill files

## 0. Context Frame

This is a large, mature Spring Boot service:
> Java 17, Spring Boot 3.5.x, Camunda Platform 7 (7.23.0)
> ~250 JPA entities, ~1200 Flyway migrations, ~50 BPMN workflows
> ~50 Feign client packages (~112 interfaces) for external service integration

**Priority rule:** Baseline skills (java-springboot, java-junit, java-docs) define best-practice defaults. Project conventions below override only where explicitly stated. When diverging from baseline, note inline: `"Using X instead of project pattern Y because ..."`

## 1. Best Practices Baseline

> Reference Context7 MCP for current Spring Boot/JPA API signatures before relying on training data. Especially important for setup/config, unfamiliar APIs, and version-sensitive behavior.

> Priority details: best practices take precedence over project patterns, but note divergence so the user stays informed. When unsure whether a project pattern is still current, check the actual codebase (grep for recent usage) before following the pattern blindly.

## 2. DTOs & Serialization

### Standard Annotation Set

Every DTO must have these annotations in this order:

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class MyDto {
    private String field; // all fields must be private
}
```

When `@EqualsAndHashCode` is present (BaseResponse/BaseSearch subclasses), the order becomes:

```java
@Data
@EqualsAndHashCode(callSuper = false) // or callSuper = true for BaseSearch
@NoArgsConstructor
@AllArgsConstructor
public class MyDto { ... }
```

Note: no class-level `@Builder` when extending BaseResponse/BaseSearch -- see inheritance recipes below.

### BaseResponse Inheritance Recipe

`BaseResponse<T>` has `T id` + `int version`. Response DTOs with those fields must extend it.

Lombok `@Builder` on a subclass doesn't call `super()` -- so `id`/`version` are never set. Fix: use constructor-level `@Builder` with a custom `builderMethodName` that calls `super(id, version)`.

```java
@Data
@EqualsAndHashCode(callSuper = false)
@NoArgsConstructor
@AllArgsConstructor
public class MyResponse extends BaseResponse<UUID> {

    @Builder(builderMethodName = "myResponseBuilder")
    public MyResponse(UUID id, int version, String customField) {
        super(id, version);
        this.customField = customField;
    }

    private String customField;
}
```

Key rules:
> `@EqualsAndHashCode(callSuper = false)` -- required, Lombok warns without it
> No class-level `@Builder` -- only constructor-level with `builderMethodName`
> Constructor must call `super(id, version)`
> Reference: `UnderwritingCollateralDetailResponse`

### BaseSearch Inheritance Recipe

`BaseSearch` has `page`, `size`, `sort`, `sortDirection`. Paginated search request DTOs must extend it.

```java
@Data
@EqualsAndHashCode(callSuper = true)
@NoArgsConstructor
@AllArgsConstructor
public class MySearchRequest extends BaseSearch {
    private String filterField;
}
```

> `@EqualsAndHashCode(callSuper = true)` -- includes parent pagination fields in equals/hashCode
> Controllers bind via `@RequestParam Map<String, Object>` + `new ObjectMapper().convertValue(request, MySearchRequest.class)`. Reference: `UnderwritingApprovalController`

### AssetPageResponse<T> Constructor

Generic paginated response with summary status for asset list endpoints. Reuse instead of creating new page DTOs.

```java
new AssetPageResponse<>(page, Summary.builder().status(statusString).build())
```

Already used by doc-check and CFCAT features.

### Jackson S1 Getter Mangling

Fields starting with a single lowercase letter + uppercase (like `aScore`, `bScore`) cause Jackson to mangle getter `getAScore()` to property `ascore` (all lowercase). This does NOT merge with `@JsonProperty("AScore")`, producing duplicate JSON properties (`"ascore"` + `"AScore"`) -- breaks HMAC signatures for StrategyOne API calls.

**Fix:** Add class-level annotation:
```java
@JsonAutoDetect(fieldVisibility = JsonAutoDetect.Visibility.ANY, getterVisibility = JsonAutoDetect.Visibility.NONE)
```

**Not affected:** Fields like `roRiskLevel` (multiple lowercase before uppercase) -- getter `getRoRiskLevel()` to `roRiskLevel` matches the field name, so Jackson merges correctly.

Note: `JsonUtils.convertToJson()` uses a plain ObjectMapper (no SNAKE_CASE) -- both ObjectMappers produce the same duplicate output for affected fields.

### @JsonProperty -- When Needed vs Redundant

> **Redundant**: Standard DTOs serialized by Spring's global snake_case ObjectMapper. Don't add `@JsonProperty("snake_case")` -- the global config handles it.
> **Needed**: Search request DTOs bound via `new ObjectMapper().convertValue()` in controllers. That plain `ObjectMapper` lacks the global snake_case config, so `@JsonProperty` is required on those DTOs.
> Don't add `@JsonNaming(SnakeCaseStrategy.class)` on DTOs -- the global config already does this.

### Request DTO Validation

> Use `@Valid @RequestBody` on controller parameters
> Use boxed types (`Integer`, `Boolean`) with `@NotNull` for required fields -- primitives (`int`, `boolean`) silently default to `0`/`false` when absent from JSON, making missing-field detection impossible
> Use `@NotBlank` for required string fields
> Reference: `ReturnReasonAssetSaveRequest`

## 3. Controllers

### Response Wrapping

> Controllers wrap with `BravoCommonResponse.success(data)` -- services return raw DTOs
> `BravoBaseResponse { header, data }` (legacy) vs `BravoCommonResponse<T> { code, data, details, message }` (newer TRD standard, `@JsonInclude(NON_NULL)`)
> No global `ResponseBodyAdvice` auto-wrapping -- each controller method wraps explicitly
> Check TRD spec for which format to use per endpoint
> Never return `BravoCommonResponse` from a service method (hook enforces)

### Search Endpoint Binding

Paginated search endpoints use `@RequestParam Map` + manual ObjectMapper conversion:

```java
@GetMapping("/search")
public BravoCommonResponse<Page<MyDto>> search(
        @RequestParam Map<String, Object> request) {
    MySearchRequest searchRequest = new ObjectMapper()
            .convertValue(request, MySearchRequest.class);
    return BravoCommonResponse.success(myService.search(searchRequest));
}
```

> The plain `ObjectMapper` here lacks global snake_case config -- search DTOs need `@JsonProperty`
> Reference: `UnderwritingApprovalController`

### @FeatureFlag -- Method Level Only

`FeatureInterceptor` uses `getMethodAnnotation()` -- class-level `@FeatureFlag` is **silently ignored**.

```java
// WRONG -- silently ignored
@FeatureFlag(feature = "featXxx")
@RestController
public class MyController { ... }

// CORRECT -- method-level
@GetMapping("/endpoint")
@FeatureFlag(feature = "featXxx")
public ResponseEntity<?> myEndpoint() { ... }
```

> Service methods do NOT duplicate feature flag checks -- controller is the single gate
> In `@WebMvcTest` with `@ContextConfiguration`, `FeatureFlagMethodAspect` is not loaded -- flags ignored in tests

### ApiPathConstants Convention

Underwriting controllers extract sub-path segments to `ApiPathConstants` constants:

```java
@GetMapping(ApiPathConstants.ASSETS + "/{assetId}" + ApiPathConstants.CUSTOMERS + "/{customerId}")
```

> Other domains (surveyor, operation) use inline strings -- only underwriting requires constants

### BaseController Exception

Controllers extending `BaseController` can't use `@RequiredArgsConstructor` -- `BaseController` requires `SecurityContextProvider` in its constructor.

```java
@RestController
public class MyController extends BaseController {

    private final MyService myService;

    public MyController(SecurityContextProvider securityContextProvider,
                        MyService myService) {
        super(securityContextProvider);
        this.myService = myService;
    }
}
```

## 4. Services

### Finder Naming Convention

> `find*` methods -- throw `BravoCommonException(NOT_FOUND)` when entity not found
> `findOptional*` methods -- return `Optional` for cases where absence is a valid business scenario (e.g., returning `null` to the API)

### Cross-Domain Service Boundaries

> Services must not inject repositories from other domain packages -- use the corresponding service interface instead
> Example: use `SurveyorAssignmentFinancialStatementService`, not `SurveyorAssignmentFinancialStatementRepository` from `UnderwritingSummaryServiceImpl`
> When calling another service's `findById` (which already throws `NOT_FOUND`), don't wrap it in `orElseThrow` -- call it directly
> When adding a new repository method for a paginated query, also add the corresponding service method to the service interface and implementation

### Trim Search Input

Always trim user-provided search/filter strings before passing to repository queries:

```java
public Page<MyDto> search(MySearchRequest request) {
    String name = StringUtil.trimOrNull(request.getName());
    String code = StringUtil.trimOrNull(request.getCode());
    // use trimmed values everywhere below
    return repository.search(name, code, ...);
}
```

> `StringUtil.trimOrNull(value)` returns trimmed string or `null` if blank
> Trim once at the beginning of the service method, use trimmed value everywhere

### Audit Fields on Save

> Always call `entity.setUpdatedBy(caNik)` before saving underwriting-related entities
> The `updatedBy` field tracks who last modified the record
> Applies to ALL save paths -- including saves on related entities like `UnderwritingCollateralDetail` or `UnderwritingReturnReasonAdditionalNote`

### Status Transitions -- WIP Idempotency Pattern

Save/update (draft) operations must transition the underwriting status to WIP:
- Document check phase: `ASSIGNED` -> `WIP_DOCUMENT`
- Review phase: `DOCUMENT_CHECKED` -> `WIP_REVIEW`

Structure the code so `setUpdatedBy` + `save` are **unconditional**, and only `setStatus` + `recordHistory` are inside the idempotent `if` guard:

```java
// Always update the modifier
entity.setUpdatedBy(caNik);

// Only transition status if not already WIP
if (!WIP_DOCUMENT.equals(entity.getStatus())) {
    entity.setStatus(WIP_DOCUMENT);
    checkPointHistoryService.recordHistory(entity, WIP_DOCUMENT);
}

repository.save(entity);
```

> `updatedBy` must always reflect the latest modifier -- never put it inside the `if` guard
> `recordHistory` itself has built-in WIP idempotency (updates `updatedAt` on existing row if latest category matches) -- the `if` guard is a clean-code practice, not a duplicate-prevention requirement

### Underwriting Dual-Signature Pattern

All methods in `UnderwritingSummaryService` have two overloads:

```java
// Controller overload -- validates via validateUser
void method(UUID applicationId, String nik);

// Internal overload -- custom validation
void method(UUID applicationId, Runnable validation);
```

> Always implement both when adding new summary endpoints

### No BravoCommonResponse in Service Layer

> Services return raw DTOs -- wrapping with `BravoCommonResponse.success()` belongs in controllers only
> Shell hook `service-response-wrapping-guard` auto-enforces this

## 5. JPA & JPQL

### Nullable String Params in Functions

Hibernate 6 sends null `String` params as untyped `bytea` to PostgreSQL. Functions like `UPPER(CONCAT('%', :param, '%'))` fail with `function upper(bytea) does not exist` when `param` is null.

**Fix:** Use `CAST(:param AS string)` in both the `IS NULL` check and the `CONCAT`:

```sql
-- In @Query JPQL
WHERE (CAST(:name AS string) IS NULL
       OR UPPER(e.name) LIKE UPPER(CONCAT('%', CAST(:name AS string), '%')))
```

> H2 tests won't catch this -- only surfaces against real PostgreSQL
> Apply to every nullable String parameter used in JPQL functions

### Enum String Comparison

For `@Enumerated(EnumType.STRING)` fields, use `CAST` to compare enum columns against String parameters:

```sql
WHERE CAST(e.status AS string) = :statusParam
```

> Useful for merging boolean+enum param pairs into a single String param (reducing repository method parameter count)

### LOWER()/UPPER() Index Requirement

Before using case-insensitive functions in JPQL `WHERE` clauses, verify a functional index exists in migrations:

```sql
-- Migration: CREATE INDEX CONCURRENTLY IF NOT EXISTS table_col_lower_idx ON table (LOWER(column));
-- JPQL: WHERE LOWER(e.column) LIKE LOWER(CONCAT('%', :param, '%'))
```

> Without a functional index, the function forces a sequential scan
> Some tables have `upper()` indexes (e.g., `asset`, `lead_reserve`), many don't -- always check

**Exception:** `saci.licenseNumber` and `saci.variantCode` are stored uppercase -- only `UPPER()` the search parameter, not the column. A plain btree index suffices:

```sql
WHERE saci.licenseNumber = UPPER(:licenseNumber)
```

### Explicit Soft-Delete Consistency

When writing custom `@Query` JPQL with explicit `AND x.deleted = false` on some joins, add it for ALL `BaseEntity` joins in that query:

```sql
-- WRONG: mixed explicit and implicit (@SQLRestriction) filtering
SELECT e FROM Entity e JOIN e.related r WHERE r.deleted = false

-- CORRECT: explicit on all BaseEntity joins
SELECT e FROM Entity e JOIN e.related r WHERE e.deleted = false AND r.deleted = false
```

> Don't mix explicit and implicit (`@SQLRestriction`) filtering in the same query

### Pagination

> Use `JpaSort.unsafe(direction, "alias.field")` for JPQL -- standard `Sort.by()` fails to resolve JPQL path aliases in custom `@Query` methods
> Use `PageRequestUtil.of(page, size, sort)` for page creation -- caps page size via `setting.size.maxSizePerPage` (hook enforces)
> Exception: internal batch jobs with config-injected sizes may use `PageRequest.of()` directly

### New Paginated Repo Methods

When adding a new repository method for a paginated query:
> Add corresponding method to the service **interface**
> Add implementation in the service **impl**
> Never call the repository directly from a different service

## 6. Entities

### @SQLRestriction on BaseEntity Subclasses

Every entity extending `BaseEntity` must have:

```java
@Entity
@SQLRestriction("deleted=false")
public class MyEntity extends BaseEntity { ... }
```

> Without it, queries return soft-deleted rows
> Shell hook `sql-restriction-guard` auto-enforces this

### Reflected Entities

Always use `ApplicationUtil` reflected methods instead of direct getters:

```java
// WRONG -- returns original loan even during amendment/restructuring
application.getLoan();

// CORRECT -- returns newLoan if present, otherwise original
ApplicationUtil.reflectedLoan(application);
```

Same pattern: `reflectedAsset()`, `reflectedCalculation()`, `reflectedSimulation()`.

> Shell hook `enforce-reflected-entities` auto-enforces this (excludes test files and `ApplicationUtil.java` itself)

### Soft-Delete Pattern

> Entities use `deleted` boolean + `deletedAt` timestamp (inherited from `BaseEntity`)
> Repository queries: `findBy...AndDeletedIsFalse`
> `@SQLRestriction("deleted=false")` handles JPA-level filtering automatically
> Raw SQL (migrations, native queries) must add `AND deleted = false` manually

### Setter Overrides for *Enc Fields

Entities with encrypted columns (e.g., `nameEnc`, `addressEnc`) override Lombok setters to auto-populate the encrypted field when the plaintext field is set.

> Before flagging a "missing `*Enc` assignment" in PR review, **always read the entity source** -- the setter override may handle it transparently
> Example: calling `entity.setName("value")` may internally also set `entity.nameEnc` via the overridden setter

## 7. Testing

### Setup Convention

```java
@MockitoSettings(strictness = Strictness.LENIENT)
class MyServiceImplTest {

    @Mock
    private MyRepository myRepository;

    @InjectMocks
    private MyServiceImpl myService;
}
```

> Use `@MockitoSettings(strictness = Strictness.LENIENT)`, `@Mock`, `@InjectMocks`
> Do NOT use `@ExtendWith(MockitoExtension.class)` -- `@MockitoSettings` already includes it (hook enforces)

### File Naming -- Strict 1:1

> `FooImpl.java` -> `FooImplTest.java` -- never split test files
> Never create `FooImplSpecificTest.java` or `FooImplV2Test.java`
> Add new tests to the existing test class, using helper methods or `setUpXxxContext()` patterns to isolate setup when new tests need different state from `@BeforeEach`

### MockMvc Null Assertions

`jsonPath("$.field").isEmpty()` does NOT correctly match JSON `null` -- it throws in Hamcrest.

```java
// WRONG -- throws on JSON null
.andExpect(jsonPath("$.data.field").isEmpty())

// CORRECT
.andExpect(jsonPath("$.data.field", nullValue()))
// with: import static org.hamcrest.Matchers.nullValue;
```

### MockMvc Page Property Paths

Spring `Page` properties serialize as snake_case due to global Jackson config:

```java
// WRONG
.andExpect(jsonPath("$.totalElements").value(5))

// CORRECT
.andExpect(jsonPath("$.total_elements").value(5))
```

> Same for `total_pages`, `number_of_elements`, etc.

### Entity Builders -- @Version NPE Trap

JPA `@Version` is `Integer` (null by default), but `BaseResponse` constructor expects `int`. Auto-unboxing null -> NPE.

```java
// WRONG -- version is null, NPE on BaseResponse construction
MyEntity.builder().id(uuid).build();

// CORRECT -- always set version in test builders
MyEntity.builder().id(uuid).version(0).build();
```

> DB-loaded entities always have version set by Hibernate -- only test-built objects have this problem

### BaseUnderwritingReviewData Subclass Builders

These entities have dual `@Builder` annotations (class-level + constructor-level with `builderMethodName`). The builder's `.id(id)` silently fails to set the parent's `id` field.

```java
// WRONG -- id is not set despite appearing to be
MyReviewData entity = MyReviewData.builder().id(uuid).field("value").build();

// CORRECT -- use constructor + setter
MyReviewData entity = new MyReviewData();
entity.setId(uuid);
entity.setField("value");
```

### Exception Assertions

`BravoCommonException.getErrorMessage()` returns `ErrorMessageConstant` enum. `getMessage()` returns `String`.

```java
// WRONG -- Enum.equals(String) is always false, test silently passes
assertEquals(ErrorMessageConstant.NOT_FOUND, exception.getMessage());

// CORRECT
assertEquals(ErrorMessageConstant.NOT_FOUND, exception.getErrorMessage());
```

> Shell hook `exception-assertion-guard` auto-enforces this

### Exception Types in Test Stubs

> Older service `findById` methods (e.g., `UnderwritingServiceImpl`) throw `ResponseStatusException(HttpStatus.NOT_FOUND)`
> Newer finder methods use `BravoCommonException`
> Always check the actual implementation before writing `doThrow`/`thenThrow` stubs

### Private Methods

```java
// Invoking private methods
ReflectionTestUtils.invokeMethod(myService, "privateMethodName", arg1, arg2);
```

### Feature Flags in Tests

```java
// Setting feature flag values
ReflectionTestUtils.setField(myService, "featMyFlag", true);
```

### PageRequestUtil.maxSizePerPage in Tests

Static field set via constructor injection -- defaults to 0 in pure Mockito tests (no Spring context):

```java
@BeforeEach
void setUp() {
    ReflectionTestUtils.setField(PageRequestUtil.class, "maxSizePerPage", 50);
}
```

### @WebMvcTest with @ContextConfiguration

When using `@WebMvcTest` with `@ContextConfiguration(classes = { Controller.class })`, `FeatureFlagMethodAspect` is not loaded -- feature flags are effectively ignored in tests. No workaround needed; just be aware that `@FeatureFlag`-annotated endpoints won't return 404 in these tests.

## 8. Migrations

### Naming Convention

```
V2_0_YYYYMMDDHHMI__description.sql
```

> Timestamp-based, double underscore separator
> Example: `V2_0_202602031613__alter-table-operation-assignment.sql`
> Shell hook `check-flyway-naming` auto-enforces this

### Index Migrations

```sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS table_col_idx ON table_name (column_name);
```

> Flyway auto-detects `CONCURRENTLY` and runs the migration as `[non-transactional]`
> Index naming: prefer suffix convention `table_col_idx` for new indexes (newer pattern)
> Partial index for soft-delete: `WHERE deleted = false` when the index only needs active rows

### Soft-Delete in Raw SQL

JPA's `@SQLRestriction("deleted=false")` handles entity queries, but Flyway migrations are raw SQL:

> Always add `AND deleted = false` when selecting from any table that has the column
> Shell hook `migration-soft-delete` warns when migration has SELECT/UPDATE/DELETE without `deleted` clause

### Never Modify Existing Migrations

> Flyway checksums prevent re-running modified migrations
> Create a new migration to alter previous changes

## 9. Underwriting Domain Reference

### Review Phase Entity Hierarchy

```
UnderwritingReviewSummary
+-- UnderwritingReviewCollateral (per asset, FK -> UnderwritingCollateralDetail)
+-- UnderwritingReviewAdditionalNote (type = DOCUMENT)
```

Mirrors doc-check phase:
```
DocumentCheckSummary
+-- CollateralDetail
+-- DocumentCheckAdditionalNote
```

> Save APIs require `validation_process = RETURN` on the review summary (equivalent to `need_return = true` in doc-check)

### Review Collateral Ownership -- IDOR Prevention

When fetching `UnderwritingReviewCollateral` in a context where the underwriting ID is known:

```java
// WRONG -- IDOR vulnerability (any valid collateral ID from any underwriting succeeds)
reviewCollateralRepository.findById(collateralId);

// CORRECT -- scoped to the underwriting
reviewCollateralRepository.findReviewCollateralById(collateralId, uwId, excludedStatuses);
```

### Return Reason Service Symmetry

`UnderwritingDocumentCheckReturnReasonServiceImpl` and `UnderwritingReviewReturnReasonServiceImpl` are parallel implementations (same for their controllers).

> When modifying one, check the other for consistency
> Private helpers (`validateApplicationStatus`, `safeMapReturnReason`, `buildAssetName`, sort field mapping) are duplicated -- keep them aligned until extracted to shared utility

### Negotiation Summary FK

> Links via `underwriting_review_summary_id` (NOT `underwriting_id`)
> To query negotiation summaries for a UW, join through `underwriting_review_summary` first

### Uniqueness Constraint

> `findUnderwritingByApplicationIdAndIndex` expects exactly one active (non-deleted) underwriting per `applicationId`
> Seed/test data must not have multiple active underwritings for the same application -- causes `NonUniqueResultException` at runtime

### Daftar Info Negatif (Negative Info List)

`UnderwritingReviewNegativeInfoService.getNegativeInfoResponsesBySummaryId()` returns `List<UnderwritingReviewNegativeInfoResponse>` — each item has `generalLovDetail` (from `general_lov_detail` table).

**FE contract**: `generalLovDetail.id` → FE calls LOV API for group name (e.g., "Negative Case", "Dokumen"). `generalLovDetail.value` → Description column (e.g., "Tidak Terindikasi"). Synthetic entries appended at response time must use real `general_lov_detail` IDs (FE can't resolve group name without them).

**LOV detail codes**: `NOT_INDICATED` (exact code, `AppConstants.NOT_INDICATED_CODE`) after S1 migration — **not** `*_NOT_INDICATED` suffix. `NOT_INDICATED_SUFFIX` is legacy/broken for production data.

**`filterExclusiveNegativeInfo`**: Applies exclusive logic per LOV group — if a group has "not indicated" entries, only those are kept.

> **Known bug**: `isNotIndicated()` (used by `filterExclusiveNegativeInfo`) still uses broken `endsWith(NOT_INDICATED_SUFFIX)` — needs separate fix

### UnderwritingReviewSummaryValidationProcess Enum

> Values: `PROCEED`, `RETURN`, `CANCEL`
> There is no `APPROVE` value -- use `PROCEED` when testing the "not RETURN" path

### UnderwritingReviewSummaryAnalysisProcess Enum

> `getLabel()` returns the enum name (e.g., `"RECOMMENDATION_NOT_RECOMMENDATION"`)
> `getDescription()` returns the human-readable form (e.g., `"RECOMMENDATION/NOT RECOMMENDATION"`)
> These are different -- verify against the `RECOMMENDATION_NOT_RECOMMENDATION` entry in `ApplicationConstants.java` when in doubt

## 10. PR Review Checklist

When reviewing PRs (own or others'), check for these items. Frame findings as **practical correctness concerns** -- explain the actual risk, don't cite "per CLAUDE.md."

> Discuss findings **one at a time** with user -- never dump all findings in a single message
> Verify agent findings against actual source files before posting

### Checklist

1. **DTO annotations** -- correct set (`@Data @Builder @NoArgsConstructor @AllArgsConstructor`), correct order, `private` fields
2. **@FeatureFlag placement** -- method level, not class level (class-level is silently ignored)
3. **ApplicationUtil.reflected*()** -- not direct `application.getLoan()` etc. (stale data in amendment scenarios)
4. **@SQLRestriction** -- present on new entities extending `BaseEntity` (queries return soft-deleted rows without it)
5. **setUpdatedBy before save** -- in underwriting domain, `updatedBy` must always reflect the latest modifier
6. **Soft-delete filtering** -- custom queries with explicit `AND deleted = false` on ALL `BaseEntity` joins (not just some)
7. **PageRequestUtil.of()** -- in service impls, not `PageRequest.of()` (bypasses page size cap)
8. **Constructor injection** -- `@RequiredArgsConstructor` + `private final`, not field `@Autowired`
9. **BravoCommonResponse** -- only in controllers, never in service layer
10. **Test assertions** -- `.getErrorMessage()` not `.getMessage()` for `BravoCommonException`, `nullValue()` not `isEmpty()` for JSON null
11. **Entity *Enc setter overrides** -- check entity source before flagging missing encrypted field assignments
12. **BaseResponse/BaseSearch inheritance** -- constructor-level `@Builder` with `builderMethodName`, not class-level `@Builder`

## 11. Hook Reference (Read-Only)

These hooks auto-enforce conventions -- no manual checking needed for items they cover. Listed here so skill users know what's already automated.

### Shell Hooks (settings.local.json)

| Hook | Type | Action | What it checks |
|------|------|--------|----------------|
| git add safety | PreToolUse | **Block** | `git add -A`, `git add .`, `git add --all` |
| mvn command safety | PreToolUse | **Block** | `mvn compile/test/verify/clean/package/install` |
| wildcard-import-guard | PreToolUse | **Block** | `import .*.*;` in Java files |
| constructor-injection-guard | PreToolUse | **Warn** | `@Autowired` in Java files |
| personal-config-edit-guard | PreToolUse | **Warn** | Edits to `application-local.yaml` or `docker-compose.yaml` |
| migration-soft-delete | PostToolUse | **Warn** | Migration with SELECT/UPDATE/DELETE but no `deleted` clause |
| check-flyway-naming | PostToolUse | **Warn** | Migration filename not matching `V2_0_YYYYMMDDHHMI__description.sql` |
| dto-noargs-constructor-guard | PostToolUse | **Warn** | DTO with `@Builder` but no `@NoArgsConstructor` |
| enforce-page-request-util | PostToolUse | **Warn** | `PageRequest.of()`/`ofSize()` or `Pageable.ofSize()` in service impl |
| mockito-settings-guard | PostToolUse | **Warn** | `@ExtendWith(MockitoExtension.class)` in test files |
| enforce-reflected-entities | PostToolUse | **Warn** | `application.getLoan()/getAsset()/getCalculation()/getSimulation()` |
| feature-flag-class-level-guard | PostToolUse | **Warn** | `@FeatureFlag` at class level on controllers |
| exception-assertion-guard | PostToolUse | **Warn** | `.getMessage()` with `ErrorMessageConstant` in tests |
| feature-flag-dual-registration | PostToolUse | **Remind** | New `feat*` flag in `application.yaml` or `application-local.yaml` |
| service-response-wrapping-guard | PostToolUse | **Warn** | `BravoCommonResponse` in service impl |
| sql-restriction-guard | PostToolUse | **Warn** | Entity extends `BaseEntity` without `@SQLRestriction` |

### Hookify Rules (Conversation-Level)

| Guard | What it enforces |
|-------|-----------------|
| protect-prod-config | Don't edit `application-prod.yaml` or `application-uat.yaml` |
| transactional-readonly-guard | Don't add `@Transactional(readOnly = true)` eagerly to service impls |
| jpasort-unsafe-guard | Use `JpaSort.unsafe()` not `Sort.by()` in service impl for JPQL queries |
| dto-inherited-builder-guard | DTOs extending `BaseResponse`/`BaseSearch` use constructor-level `@Builder` |
| base-response-builder-guard | DTOs extending `BaseResponse` use `builderMethodName` on constructor-level `@Builder` |
