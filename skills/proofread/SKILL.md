---
name: proofread
description: Use when finishing a plan, design, analysis, or spec document — before sharing, committing, or handing off for review. Also use when reviewing someone else's draft document for mechanical quality issues.
---

# Proofread Document

Mechanical quality pass over documents. Catches issues that content review misses: numbering gaps, stale markers, broken cross-references, and structural defects.

**Not for:** Code review, architectural feedback, or content correctness — those are separate concerns.

## Usage

`/proofread <file-path>` — proofread a specific file.
`/proofread` (no args) — proofread the most recently created or edited doc in `docs/`.

When no file is specified, find the target by checking recent git status or the last document written in the current conversation.

## Checklist

Read the document end-to-end. Report findings grouped by category with location, issue, and fix. If the document is clean, say so — don't invent issues.

### 1. Numbering & Ordering

- Step/section numbers sequential — no gaps, no duplicates
- Sub-item numbering resets correctly under each parent (1.1, 1.2, 2.1 — not continuing as 1.3 under section 2)
- Checkbox items (`- [ ]`) match their declared step numbers
- Table row numbering (e.g., `| # |` columns) is sequential

### 2. Spelling & Names

- Misspelled words, especially technical terms and proper nouns
- Class/method/file names consistent throughout (don't alternate `CfcatAssetListResponse` / `CFCATAssetListResponse`)
- If names reference real code, verify they match codebase conventions (check with Grep/Glob if unsure)

### 3. Stale Markers

- No unresolved `TODO`, `TBD`, `FIXME`, `HACK`, `XXX`, `[fill in]`, `???`, `...` (as placeholder)
- No placeholder text ("lorem ipsum", "example here", "describe this")
- No commented-out draft paragraphs or strikethrough remnants meant to be removed

### 4. Trailing & Orphaned Content

- Nothing after the logical end of the document (scratch notes, old drafts, debug output)
- No "Notes to self" or "Open questions" sections that should have been resolved or moved to a ticket
- No leftover template sections that were never filled in (empty headers, skeleton tables)

### 5. Cross-References

- All "see step N", "at line X", "as described in section Y" point to real targets in the document
- File paths in the document are plausible (correct package structure, extension)
- Links (URLs, Confluence page IDs, Jira ticket IDs) look well-formed
- "Reference:" pointers in implementation steps cite the correct file/line

### 6. Consistency

- Same concept uses same term throughout (don't alternate "collateral" and "asset" for the same thing unless explicitly defined)
- Tense is consistent within sections (don't mix "will create" and "creates" for planned steps)
- Code style in inline examples is uniform (same indent, same casing convention)

### 7. Tables

- All rows have the same number of columns as the header
- Header separator row (`|---|---|`) matches column count
- No empty cells that should have content (vs. intentionally blank)
- Column alignment is consistent (left-align text, right-align numbers)

### 8. Code Blocks

- All code blocks are fenced and language-tagged (` ```java `, not bare ` ``` `)
- Opening and closing fences match (no unclosed blocks)
- Indentation within code blocks is consistent
- No syntax errors obvious from visual inspection (unclosed braces, missing semicolons in Java)

### 9. Logical Gotchas

- Steps that reference something defined in a later step (forward dependency without explanation)
- Steps that reference utilities/methods assumed to exist but not created in any step — verify they exist in the codebase
- Duplicate entries in lists or tables (same item listed twice)
- Contradictions between sections (e.g., "no new feature flags needed" in one section but a flag mentioned in another)
- Summary table counts must match actual listed items (e.g., "8 tests" in a coverage table vs the 8 scenarios listed in the step)
- Line-number references to source code are point-in-time snapshots — flag if they're used as authoritative anchors rather than hints

## Output Format

```
## Proofread Results

**Document**: <file path>
**Verdict**: <N issues found | Clean>

### Issues

1. **[Category]** Line/Section X — <what's wrong>
   Fix: <suggested correction>

2. ...

### Notes
<Optional: non-blocking observations, style suggestions, or ambiguities worth a second look>
```
