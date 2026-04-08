---
name: fix-dto
description: "Use when DTOs need annotation standardization — missing @NoArgsConstructor, redundant @JsonProperty, wrong BaseResponse/BaseSearch inheritance. Accepts file path, class name, package, or current branch diff."
disable-model-invocation: true
---

# Fix DTO

Standardize request/response DTOs to follow project conventions. Fixes annotation sets, field visibility, redundant `@JsonProperty`, and `BaseResponse`/`BaseSearch` inheritance patterns.

## Target Resolution

Parse the argument to determine which files to fix:

| Input | Scope |
|---|---|
| No argument | DTOs changed on the current branch (`git diff master --name-only \| grep '/dto/'`) |
| File path (contains `/` or ends with `.java`) | That single file |
| Class name (e.g., `ReturnReasonAssetSaveRequest`) | Find with: `find src/main/java -path "*/dto/*" -name "<ClassName>.java"` |
| Package fragment (e.g., `underwriting/request`) | All `.java` files under `src/main/java/com/bfi/bravo/dto/<fragment>/` |
| `--all` | All files under `src/main/java/com/bfi/bravo/dto/` (**use with caution — 2000+ files**) |

## Fix Rules

Apply these rules **in order** to each target file. Read the file first, apply all applicable fixes, then write once.

### Rule 1: Field visibility — package-private to `private`

**Detect**: Field declarations inside the DTO class body that lack an access modifier. A field declaration is a line matching the pattern: optional annotation lines followed by a type and identifier without `private`, `protected`, or `public`.

**Fix**: Add `private` keyword.

```java
// Before
String name;
Long id;
List<Item> items;

// After
private String name;
private Long id;
private List<Item> items;
```

**Exceptions**: Do NOT modify fields inside inner `static class` definitions (e.g., `Entity` inner classes in constants), enum constants, or static final fields.

### Rule 2: `@Getter/@Setter` to `@Data`

**Detect**: Class has `@Getter` and/or `@Setter` but no `@Data`.

**Fix**: Remove `@Getter` and `@Setter`, add `@Data`.

```java
// Before
@Getter
@Setter
@Builder
public class MyDto { ... }

// After
@Data
@Builder
public class MyDto { ... }
```

**Exception**: If only `@Getter` exists (no `@Setter`) — this is an intentionally immutable DTO. Skip this rule.

### Rule 3: Missing `@NoArgsConstructor`

**Detect**: Class has `@Builder` but no `@NoArgsConstructor`.

**Fix**: Add `@NoArgsConstructor`. Also add `@AllArgsConstructor` if missing.

### Rule 4: Missing `@AllArgsConstructor`

**Detect**: Class has `@Builder` and `@NoArgsConstructor` but no `@AllArgsConstructor`, and does NOT extend `BaseResponse` or `BaseSearch`.

**Fix**: Add `@AllArgsConstructor`.

### Rule 5: Redundant `@JsonProperty`

**Detect**: A field annotated with `@JsonProperty("snake_case_name")` where the value is the exact snake_case conversion of the Java field name.

The snake_case conversion rule: insert `_` before each uppercase letter and lowercase everything. Examples:
- `checkStatus` → `check_status`
- `holdNotes` → `hold_notes`
- `applicationId` → `application_id`
- `id` → `id`

**Fix**: Remove the `@JsonProperty` annotation line. If `@JsonProperty` was the only annotation on that field, just remove the line. If the import `com.fasterxml.jackson.annotation.JsonProperty` becomes unused after all removals, remove the import too.

```java
// Before
@JsonProperty("check_status")
private UnderwritingDocumentCheckStatus checkStatus;

// After
private UnderwritingDocumentCheckStatus checkStatus;
```

**Exception**: Keep `@JsonProperty` if the value does NOT match the auto-derived snake_case (e.g., `@JsonProperty("id")` on a field named `underwritingId` — the names differ intentionally).

### Rule 6: Inherited DTO — class-level `@Builder` to constructor `@Builder`

**Detect**: Class extends `BaseResponse` or `BaseSearch` AND has class-level `@Builder` annotation (not on a constructor).

**Fix**:
1. Remove class-level `@Builder`
2. Generate an explicit constructor with ALL fields (parent + child)
3. Add `@Builder(builderMethodName = "xxxBuilder")` on the constructor, where `xxx` is the class name with first letter lowercased
4. Call `super(...)` with parent fields

For `BaseResponse<T>` parent fields: `T id, int version`
For `BaseSearch` parent fields: `Integer page, Integer size, String sort, Sort.Direction sortDirection`

```java
// Before
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class MyDto extends BaseResponse<Long> {
  private String name;
  private String code;
}

// After
@Data
@EqualsAndHashCode(callSuper = false)
@NoArgsConstructor
@AllArgsConstructor
public class MyDto extends BaseResponse<Long> {
  private String name;
  private String code;

  @Builder(builderMethodName = "myDtoBuilder")
  public MyDto(Long id, int version, String name, String code) {
    super(id, version);
    this.name = name;
    this.code = code;
  }
}
```

### Rule 7: Missing `@EqualsAndHashCode` on inherited DTOs

**Detect**: Class extends `BaseResponse` or `BaseSearch` but has no `@EqualsAndHashCode` annotation.

**Fix**:
- `BaseResponse` → add `@EqualsAndHashCode(callSuper = false)`
- `BaseSearch` → add `@EqualsAndHashCode(callSuper = true)`

Place immediately after `@Data` (before `@Builder`, `@NoArgsConstructor`, `@AllArgsConstructor`). Add the import `lombok.EqualsAndHashCode` if not present.

## Execution Steps

1. **Resolve targets**: Determine the list of files based on the input argument.
2. **Show count**: Print `Found N DTO file(s) to check.`
3. **Process each file**: Read, analyze, apply all applicable rules, edit.
4. **Skip clean files**: If a file has no violations, skip silently.
5. **Report per file**: For each fixed file, print a one-liner: `Fixed <ClassName>: <rules applied>` (e.g., `Fixed ReturnReasonAssetSaveRequest: fields→private, removed 3 @JsonProperty`)
6. **Summary**: Print total files checked, files fixed, files skipped.
7. **Rule 6 impact check**: For each class where Rule 6 was applied, search for broken builder references:
   - Use the Grep tool: pattern `ClassName\.builder\(\)`, path `src/`, type `java`
   - Report each hit: `⚠ ClassName.builder() → ClassName.xxxBuilder() at <file>:<line>`
   - If hits found, ask: "Found N broken builder references. Fix them?"
   - If yes, replace `ClassName.builder()` with `ClassName.xxxBuilder()` in each affected file using the Edit tool.

## Important Notes

- **Do not change field types, names, or add/remove fields** — this skill only fixes annotations and visibility.
- **Do not change the class name or package**.
- **Preserve existing `@JsonProperty`** annotations that have non-standard mappings (value differs from auto-derived snake_case).
- **Preserve field ordering** — do not reorder fields.
- **Run `make fix-style` after** if fixing many files — Prettier may need to reformat.
- When fixing inherited DTOs (Rule 6), verify the constructor parameter order matches: parent fields first, then child fields in declaration order.

## Examples

```bash
# Fix DTOs changed on current branch
/fix-dto

# Fix a single class by name
/fix-dto ReturnReasonAssetSaveRequest

# Fix all DTOs in underwriting request package
/fix-dto underwriting/request

# Fix a specific file
/fix-dto src/main/java/com/bfi/bravo/dto/underwriting/request/ReturnReasonAssetSaveRequest.java

# Fix ALL DTOs (caution: 2000+ files)
/fix-dto --all
```
