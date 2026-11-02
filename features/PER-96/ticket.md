---
key: PER-96
type: story
status: todo
blocks: []
blocked_by: []
labels:
- Bug
- repo:app
repos:
- app
summary: Search does not find XML tag names inside code blocks
---

```
Upstream: https://github.com/docmost/docmost/issues/2264

A Docmost page contains a code block:

    <ACT>
      <P_NUM>TEST</P_NUM>
    </ACT>

Confirmed locally on the General space page "Testing Docmost" (http://localhost:5173/).

Search for TEST returns that page.
Search for P_NUM returns "No results found".

P_NUM is written on the page and must be found by whole-page search. Title-only search is off.

Done when a search for P_NUM returns the page, and a search for TEST still returns it.
```
