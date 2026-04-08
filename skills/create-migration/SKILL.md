---
name: create-migration
description: "Use when creating a new Flyway database migration — ensures correct V2_0_YYYYMMDDHHMI naming, timestamp conflict checks, and convention reminders."
disable-model-invocation: true
---

# Create Flyway Migration

Generate a new Flyway migration file with the correct naming convention.

## Naming Convention

`V2_0_YYYYMMDDHHMI__description.sql`

- Prefix: `V2_0_`
- Timestamp: `YYYYMMDDHHMI` — current date/time (24h format, minute precision)
- Separator: double underscore `__`
- Description: lowercase, hyphen-separated words
- Extension: `.sql`

## Steps

1. **Generate timestamp**: Use the current date/time to produce the `YYYYMMDDHHMI` portion. Run:
   ```bash
   date +%Y%m%d%H%M
   ```

2. **Build filename**: Combine into `V2_0_<timestamp>__<description>.sql`
   - Convert the user's description to lowercase, replace spaces with hyphens
   - Strip any characters that aren't alphanumeric or hyphens

3. **Check for conflicts**: Verify no existing migration shares the same timestamp prefix:
   ```bash
   ls src/main/resources/db/migration/V2_0_<timestamp>__* 2>/dev/null
   ```
   If a conflict exists, increment the minute by 1 and re-check. Repeat until no conflict.

4. **Create the file** at `src/main/resources/db/migration/<filename>`:
   - If `--sql` content was provided, write that as the file body
   - Otherwise, write an empty file with a comment header:
     ```sql
     -- Migration: <description>
     ```

5. **Show the result**: Print the created file path and remind about conventions:
   - `AND deleted = false` — add to any SELECT/UPDATE/DELETE on soft-delete tables
   - `CREATE INDEX CONCURRENTLY IF NOT EXISTS` — for new indexes
   - Index naming: `<table>_<col>_idx` (suffix convention)
   - `WHERE deleted = false` — add as a partial index filter on any table that extends `BaseEntity` (has `deleted` column). This keeps the index small and matches JPA's `@SQLRestriction("deleted=false")`.

## Arguments

| Arg | Required | Description |
|-----|----------|-------------|
| `<description>` | Yes | Short description (e.g., "add-column-rejection-source", "create-index-underwriting") |
| `--sql <content>` | No | SQL content to write into the file |

## Examples

- `/create-migration add-column-to-application` — creates `V2_0_202602261630__add-column-to-application.sql`
- `/create-migration create-index-collateral --sql "CREATE INDEX CONCURRENTLY IF NOT EXISTS collateral_application_id_idx ON collateral (application_id);"` — creates file with SQL content
