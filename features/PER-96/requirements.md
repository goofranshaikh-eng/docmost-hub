# Requirements — PER-96

Search does not find XML tag names inside code blocks

## Context

A Docmost page contains a code block with XML content including tag names like `P_NUM` and `TEST`. Search for `TEST` returns the page, but search for `P_NUM` returns "No results found". Both terms appear in the page content and should be discoverable through whole-page search.

Upstream issue: https://github.com/docmost/docmost/issues/2264

## Acceptance Criteria

### Given a Docmost page with code block content
**When** a user searches for text that appears inside a code block  
**Then** the search returns that page in the results

**Verification:**
1. Create or navigate to a page containing a code block with the XML content:
   ```
   <ACT>
     <P_NUM>TEST</P_NUM>
   </ACT>
   ```
2. Search for "P_NUM"
3. Verify the page appears in search results
4. Search for "TEST"
5. Verify the page still appears in search results (regression check)

### Given the existing search behavior for non-code-block content
**When** the indexing fix is applied  
**Then** existing search functionality continues to work without degradation

**Verification:**
1. Search for text that appears outside code blocks on various pages
2. Verify all expected results are returned
3. Verify no performance regression in search response time

## Tech Notes

### Investigation Areas

1. **Postgres full-text indexing**: The app uses Postgres full-text search over page text. Investigate how code block content is currently processed during indexing:
   - Check if code blocks are stripped or escaped during text extraction
   - Review the `tsvector` generation for page content
   - Identify if special characters or formatting markers interfere with tokenization

2. **Code block parsing**: Examine how the page content is parsed before indexing:
   - Are code blocks wrapped in markers that get indexed?
   - Is there normalization that removes angle brackets or tag-like patterns?
   - Check if markdown code fence content is included in searchable text

3. **Search query processing**: Verify search term handling:
   - Are special characters in search terms (like underscores in `P_NUM`) properly escaped?
   - Does the search query use the same tokenization as the indexing?

### Scope

- **In scope**: Fix indexing of code block content so tag names are searchable
- **Out of scope**: Changes to search UI, search ranking, or search performance optimization beyond maintaining current behavior

## Open Questions

None — the behavior is clear from the reproduction case.
