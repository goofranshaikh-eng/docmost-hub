# Requirements — PER-96

**Type:** bug  
**Title:** Search does not find XML tag names inside code blocks  
**Epic:** (none)

---

## Context

A Docmost page contains a code block with XML tags. Searching for text inside those tags produces inconsistent results: some tag names are found, others are not.

**Reproduction:**
1. Create a page with a code block containing:
   ```
   <ACT>
     <P_NUM>TEST</P_NUM>
   </ACT>
   ```
2. Search for `TEST` → page is returned
3. Search for `P_NUM` → "No results found"

Both strings appear in the same code block on the same page. Both must be findable.

---

## Acceptance Criteria

### AC1: Search finds previously-missing tag name
**Given** a page exists with a code block containing `<P_NUM>TEST</P_NUM>`  
**When** a user searches for `P_NUM`  
**Then** the page appears in search results

### AC2: Search still finds previously-working tag content
**Given** a page exists with a code block containing `<P_NUM>TEST</P_NUM>`  
**When** a user searches for `TEST`  
**Then** the page appears in search results

### AC3: Search behavior is consistent across code block content
**Given** a page exists with a code block containing multiple XML tags  
**When** a user searches for any text string visible in that code block  
**Then** the page appears in search results if the string matches

---

## Notes

- Confirmed locally on the General space page "Testing Docmost" (http://localhost:5173/)
- Title-only search is off; whole-page search is the target behavior
- The bug is in how page text is extracted or indexed for search, not in the search query itself
- Both `P_NUM` and `TEST` are in the same code block; one works, one does not — the extraction or indexing step is dropping or mangling tag names

---

## Verification

1. Create a test page with the reproduction code block
2. Run search for `P_NUM` and confirm the page is returned
3. Run search for `TEST` and confirm the page is still returned
4. Verify no regression: existing searchable content (outside code blocks, in other formats) still returns expected results
