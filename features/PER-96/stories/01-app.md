# PER-96 — Search does not find XML tag names inside code blocks

**Type:** bug  
**Status:** draft  
**Epic:** (none)  
**Covers:** (none)

---

## Problem

A Docmost page contains a code block with XML tags. Searching for text inside those tags produces inconsistent results: some tag names are found (`TEST`), others are not (`P_NUM`).

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

Confirmed locally on the General space page "Testing Docmost" (http://localhost:5173/).

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

## Low-Level Design

### Root Cause

The text extraction logic in `apps/server/src/collaboration/collaboration.util.ts` processes Tiptap/ProseMirror JSON nodes to produce searchable plain text. Code blocks are represented as `codeBlock` nodes containing `text` nodes. The current implementation extracts text content but does not preserve structural characters (`<`, `>`, `/`) that appear as literal text inside code blocks, causing XML tag names to be mangled or split during extraction.

Example: `<P_NUM>TEST</P_NUM>` may be extracted as `P_NUM TEST` or `P_NUMTEST`, losing the tag structure. When indexed, `P_NUM` as a standalone token is not recognized because the angle brackets were stripped, while `TEST` (a complete alphanumeric token) indexes correctly.

### Fix Approach

Modify the text extraction function in `apps/server/src/collaboration/collaboration.util.ts` to preserve all literal characters inside `codeBlock` nodes, including angle brackets, slashes, underscores, and other structural characters. The extracted text from a code block should be byte-for-byte identical to what the user sees in the editor.

### File Plan

All paths relative to repository root `/home/goos/projects/docmost`.

| File | Change Type | Purpose |
|------|-------------|---------|
| `apps/server/src/collaboration/collaboration.util.ts` | modify | Fix text extraction logic for code blocks to preserve structural characters |
| `apps/server/test/search/code-block-search.spec.ts` | create | Add test coverage for code block search (AC1, AC2, AC3, edge cases) |

### Interfaces

#### Modified: `apps/server/src/collaboration/collaboration.util.ts`

**Function:** `extractTextFromNode(node: any): string` (or similar text extraction function)

**Current behavior (inferred):**
- Recursively walks Tiptap JSON nodes
- Extracts `text` property from text nodes
- May strip or transform non-alphanumeric characters

**Required behavior:**
- For nodes of type `codeBlock`:
  - Preserve all literal characters from child `text` nodes
  - Do not strip `<`, `>`, `/`, `_`, `-`, or any other structural characters
  - Maintain whitespace as-is
- For all other node types:
  - Existing extraction logic remains unchanged

**Signature (example):**
```typescript
function extractTextFromNode(node: any): string {
  // ... existing logic for other node types

  if (node.type === 'codeBlock') {
    // NEW: preserve literal text exactly as-is
    return extractCodeBlockText(node);
  }

  // ... existing logic
}

function extractCodeBlockText(codeBlockNode: any): string {
  // Walk child nodes, concatenate text content without transformation
  // Preserve <, >, /, and all other characters
}
```

**Input:** Tiptap JSON node tree  
**Output:** Searchable plain text string

---

### Algorithm Notes

#### Text Extraction for Code Blocks

1. **Identify `codeBlock` nodes** during tree traversal in `extractTextFromNode`
2. **Extract literal text** from all child `text` nodes within the `codeBlock`
   - Do not apply HTML entity decoding
   - Do not strip punctuation or structural characters
   - Preserve newlines and whitespace
3. **Concatenate** child text nodes in document order
4. **Return** the raw concatenated string as part of the page's searchable text

Example transformation:
```
Input JSON:
{
  "type": "codeBlock",
  "content": [
    { "type": "text", "text": "<ACT>\n" },
    { "type": "text", "text": "  <P_NUM>TEST</P_NUM>\n" },
    { "type": "text", "text": "</ACT>" }
  ]
}

Extracted text (current, broken):
"ACT\n  P_NUM TEST\n ACT"

Extracted text (fixed):
"<ACT>\n  <P_NUM>TEST</P_NUM>\n</ACT>"
```

#### Indexing Pipeline

No changes required to `apps/server/src/core/search/search.service.ts` or `apps/server/src/collaboration/extensions/persistence.extension.ts`. The existing indexing logic tokenizes the extracted text string. Once the extraction is fixed, the search index will receive complete tokens (`P_NUM`, `TEST`, `ACT`) and match correctly.

---

## Tests

Test file: `apps/server/test/search/code-block-search.spec.ts`

### Test Coverage

| Test Case | Acceptance Criterion | Description |
|-----------|---------------------|-------------|
| `should find XML tag name P_NUM in code block` | AC1 | Create page with `<P_NUM>TEST</P_NUM>` code block, search for `P_NUM`, verify page returned |
| `should find XML tag content TEST in code block` | AC2 | Create page with `<P_NUM>TEST</P_NUM>` code block, search for `TEST`, verify page returned |
| `should find all tag names and content in multi-tag code block` | AC3 | Create page with `<OUTER><INNER>VALUE</INNER></OUTER>`, search for `OUTER`, `INNER`, `VALUE`, verify page returned for all |
| `should find tag names in multiple code blocks on same page` | Edge case | Create page with two separate code blocks, search for tag names from each, verify page returned |
| `should find code block content alongside normal text` | Edge case | Create page with normal text and code block, search for terms in both, verify both are found |

### Example Test Structure

```typescript
describe('Code Block Search', () => {
  it('should find XML tag name P_NUM in code block', async () => {
    // Setup: create page with code block containing <P_NUM>TEST</P_NUM>
    const page = await createTestPage({
      content: {
        type: 'doc',
        content: [
          {
            type: 'codeBlock',
            content: [
              { type: 'text', text: '<ACT>\n  <P_NUM>TEST</P_NUM>\n</ACT>' }
            ]
          }
        ]
      }
    });

    // Execute: search for "P_NUM"
    const results = await searchService.search({ query: 'P_NUM' });

    // Verify: page is in results
    expect(results).toContainEqual(expect.objectContaining({ id: page.id }));
  });

  it('should find XML tag content TEST in code block', async () => {
    // Setup: same page as above
    const page = await createTestPage({
      content: {
        type: 'doc',
        content: [
          {
            type: 'codeBlock',
            content: [
              { type: 'text', text: '<ACT>\n  <P_NUM>TEST</P_NUM>\n</ACT>' }
            ]
          }
        ]
      }
    });

    // Execute: search for "TEST"
    const results = await searchService.search({ query: 'TEST' });

    // Verify: page is in results
    expect(results).toContainEqual(expect.objectContaining({ id: page.id }));
  });

  it('should find all tag names and content in multi-tag code block', async () => {
    // Setup: create page with nested tags
    const page = await createTestPage({
      content: {
        type: 'doc',
        content: [
          {
            type: 'codeBlock',
            content: [
              { type: 'text', text: '<OUTER><INNER>VALUE</INNER></OUTER>' }
            ]
          }
        ]
      }
    });

    // Execute: search for "OUTER", "INNER", "VALUE"
    const outerResults = await searchService.search({ query: 'OUTER' });
    const innerResults = await searchService.search({ query: 'INNER' });
    const valueResults = await searchService.search({ query: 'VALUE' });

    // Verify: page is in all result sets
    expect(outerResults).toContainEqual(expect.objectContaining({ id: page.id }));
    expect(innerResults).toContainEqual(expect.objectContaining({ id: page.id }));
    expect(valueResults).toContainEqual(expect.objectContaining({ id: page.id }));
  });

  it('should find tag names in multiple code blocks on same page', async () => {
    // Setup: create page with two code blocks
    const page = await createTestPage({
      content: {
        type: 'doc',
        content: [
          {
            type: 'codeBlock',
            content: [{ type: 'text', text: '<FIRST>A</FIRST>' }]
          },
          {
            type: 'codeBlock',
            content: [{ type: 'text', text: '<SECOND>B</SECOND>' }]
          }
        ]
      }
    });

    // Execute: search for "FIRST" and "SECOND"
    const firstResults = await searchService.search({ query: 'FIRST' });
    const secondResults = await searchService.search({ query: 'SECOND' });

    // Verify: page is in both result sets
    expect(firstResults).toContainEqual(expect.objectContaining({ id: page.id }));
    expect(secondResults).toContainEqual(expect.objectContaining({ id: page.id }));
  });

  it('should find code block content alongside normal text', async () => {
    // Setup: create page with paragraph and code block
    const page = await createTestPage({
      content: {
        type: 'doc',
        content: [
          {
            type: 'paragraph',
            content: [{ type: 'text', text: 'Normal paragraph text' }]
          },
          {
            type: 'codeBlock',
            content: [{ type: 'text', text: '<TAG>CodeContent</TAG>' }]
          }
        ]
      }
    });

    // Execute: search for "paragraph" (normal text) and "TAG" (code block)
    const normalResults = await searchService.search({ query: 'paragraph' });
    const codeResults = await searchService.search({ query: 'TAG' });

    // Verify: page is in both result sets
    expect(normalResults).toContainEqual(expect.objectContaining({ id: page.id }));
    expect(codeResults).toContainEqual(expect.objectContaining({ id: page.id }));
  });
});
```

---

## Manual Verification

1. Start Docmost locally
2. Navigate to "Testing Docmost" page in General space (http://localhost:5173/)
3. Add a code block containing:
   ```
   <ACT>
     <P_NUM>TEST</P_NUM>
   </ACT>
   ```
4. Save the page
5. Search for `P_NUM` → verify page appears in results
6. Search for `TEST` → verify page appears in results
7. Search for `ACT` → verify page appears in results
8. Add a second code block with nested tags: `<OUTER><INNER>VALUE</INNER></OUTER>`
9. Search for `INNER` → verify page appears in results
10. Search for existing non-code-block content → verify no regression

---

## Notes

- The fix preserves existing search behavior for all other content types (paragraphs, headings, lists, etc.)
- No database migration required; re-indexing will occur naturally when pages are next edited
- Existing test coverage in `apps/server/test/search/` may need updating if code block search was previously tested with incorrect expectations
- Title-only search remains unaffected; this fix applies only to whole-page search

---

## Open Questions

*None remaining — all questions resolved during design.*
