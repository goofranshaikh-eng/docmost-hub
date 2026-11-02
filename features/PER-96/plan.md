# Plan — PER-96 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Bug fix for search not finding XML tag names inside code blocks; single-repo behavioral fix with clear repro and acceptance criteria.

## Steps

1. [x] **analyst (inline_story)** — Turn the ticket into an implementable story with AC and technical notes.
   - focus: Focus on the search indexing behavior for code block content. The AC must verify both P_NUM (currently broken) and TEST (currently working) are found after the fix.
2. [x] **designer (lld)** — Need concrete file plan for the search path mentioned in repo context.
   - focus: Examine how page text extraction handles code blocks in collaboration.util.ts and the search indexing pipeline. Provide a specific fix approach for including code block content in searchable text.
3. [x] **test_architect (tests)** — Edge cases around code block search need explicit test coverage.
   - focus: Map tests to both acceptance criteria (P_NUM found, TEST still found), and add cases for nested tags, multiple code blocks, and mixed content/code.
4. [x] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Loop delivery.
6. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
7. [ ] **open_pr** _(auto)_ — Loop delivery.
8. [ ] **pr_review** _(auto)_ — Loop delivery.
9. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are clear: search must find text inside code blocks.
- **critic** — Only one story artifact; no cross-document consistency to check.

## Planner notes

- attempt 1: designer is not a step of loop quick_fix; test_architect is not a step of loop quick_fix
