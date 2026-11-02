# Story — Search does not find XML tag names inside code blocks

**Type:** bug  
**Epic:** (standalone)  
**Covers:** (standalone bug)  
**ID:** PER-96  
**Title:** Search does not find XML tag names inside code blocks  
**Status:** backlog  
**Assignee:**  
**Blocked:**  
**Blocked reason:**  

---

## Description

Search does not return pages when querying for XML tag names that appear inside code blocks. Specifically, searching for `P_NUM` (a tag name inside a code block) returns no results, while searching for `TEST` (the tag content) correctly returns the page.

**Reproduction:**
1. Navigate to a Docmost page containing this code block:
   ```
   <ACT>
     <P_NUM>TEST</P_NUM>
   </ACT>
   ```
2. Search for "TEST" — page is found ✓
3. Search for "P_NUM" — "No results found" ✗

**Expected:** Both `P_NUM` and `TEST` should be findable via whole-page search.

**Upstream:** https://github.com/docmost/docmost/issues/2264

---

## Acceptance Criteria

### AC1: Code block tag names are searchable

**Given** a Docmost page with a code block containing XML tag names  
**When** a user searches for a tag name that appears inside the code block  
**Then** the page appears in the search results  

**Verify:**
1. Create or navigate to page "Testing Docmost" in General space
2. Ensure page contains code block:
   ```
   <ACT>
     <P_NUM>TEST</P_NUM>
   </ACT>
   ```
3. Search for "P_NUM"
4. Assert page "Testing Docmost" appears in results
5. Record search response time as baseline

### AC2: Existing search behavior is preserved

**Given** the indexing fix has been applied  
**When** a user searches for content inside code blocks (tag content) or outside code blocks  
**Then** all previously-working searches continue to return expected results with no performance regression  

**Verify:**
1. Search for "TEST" (tag content inside code block)
2. Assert page "Testing Docmost" still appears in results
3. Search for text outside code blocks on various pages (e.g., page titles, body paragraphs)
4. Assert expected pages appear in results
5. Compare search response times to baseline from AC1
6. Assert no response time increase >10%

---

## Technical Specification

### Investigation

The issue lies in how Postgres full-text indexing processes code block content. Root cause is likely one of:

1. **Code block stripping during extraction**: Code blocks may be excluded from searchable text when page content is prepared for indexing
2. **Special character handling**: Angle brackets (`<`, `>`) or underscores in tag names may be stripped or cause tokenization issues in `tsvector` generation
3. **Markdown fence markers**: Code fence delimiters (` ``` `) may interfere with content extraction

**Postgres full-text indexing primer:**
- Docmost uses Postgres `tsvector` for search indexing over page text
- `tsvector` tokenizes text and normalizes tokens for search
- Default tokenization splits on whitespace and punctuation
- Custom configurations can preserve or strip specific characters

### Files to Investigate

Based on typical Node/React wiki architecture and Postgres full-text patterns:

```
/home/goos/projects/docmost/
├── apps/server/src/
│   ├── database/
│   │   └── migrations/          # Check for search index definitions
│   ├── services/
│   │   └── search/              # Search query and indexing logic
│   └── pages/
│       └── page.service.ts      # Page content processing before indexing
└── packages/editor-ext/         # Markdown/code block parsing (if applicable)
```

**Priority paths:**
1. `apps/server/src/services/search/` — search indexing and query logic
2. `apps/server/src/pages/page.service.ts` — page content extraction for indexing
3. Database migration files defining the `tsvector` column and triggers

### Hypotheses and Fix Approaches

**Hypothesis 1: Code blocks excluded from indexed text**
- **Check:** Locate where page content is converted to plain text for indexing
- **Fix:** Ensure code block content is included in the text passed to `to_tsvector()`
- **Files:** `page.service.ts` or search service's content extraction function

**Hypothesis 2: Angle brackets stripped during tokenization**
- **Check:** Review `tsvector` generation — default config may treat `<>` as delimiters and discard them
- **Fix:** Use a custom text search configuration preserving `<>` and `_`, or normalize tags before indexing (e.g., ` P_NUM ` instead of `<P_NUM>`)
- **Files:** Migration defining the `tsvector` column; search service query logic

**Hypothesis 3: Markdown fence markers interfere**
- **Check:** Examine if ` ``` ` delimiters or language hints (e.g., ` ```xml `) are included in extracted text
- **Fix:** Strip fence markers cleanly without removing code block content
- **Files:** Content extraction in `page.service.ts`

### Recommended Fix Path

1. **Trace the indexing flow:**
   - Find where page markdown is converted to searchable text (likely `page.service.ts` or a helper in `services/search/`)
   - Log the extracted text before `to_tsvector()` is called
   - Confirm whether code block content is present in the logged text

2. **Verify tokenization:**
   - Check the `tsvector` output for a test page with the code block
   - Run a direct Postgres query: `SELECT to_tsvector('english', '<P_NUM>TEST</P_NUM>');`
   - If `P_NUM` is missing from the output, angle brackets are the culprit

3. **Apply the fix:**
   - **If code blocks are excluded:** Add code block content to the extracted text
   - **If angle brackets are stripped:** Normalize tag names in the extracted text (replace `<TAG>` with ` TAG `) or use a custom text search config
   - **If both:** Combine fixes

4. **Test:**
   - Rebuild search index for affected pages (or trigger re-indexing on save)
   - Verify `P_NUM` and `TEST` are both searchable
   - Run regression tests on non-code-block searches

### Constraints

- **No schema changes** unless unavoidable; prefer logic changes in the application layer
- **No breaking changes** to existing search API or query format
- **Preserve performance:** Indexing and search response times must not degrade

---

## References

- Upstream issue: https://github.com/docmost/docmost/issues/2264
- Repo: https://github.com/goofranshaikh-eng/docmost.git
- Base branch: `main`
- Test page: "Testing Docmost" in General space (local: http://localhost:5173/)

---

## Notes

- Postgres full-text search docs: https://www.postgresql.org/docs/current/textsearch.html
- If fix requires re-indexing, provide a migration or script to rebuild the search index for existing pages
- Consider adding a test case to prevent regression: page with code block, assert both tag name and tag content are searchable

---

## Open Questions

None — reproduction is confirmed and scope is clear.
