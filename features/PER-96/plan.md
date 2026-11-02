# Plan — PER-96 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Search bug where XML tag names inside code blocks are not found by full-text search; quick_fix loop fits because the issue is isolated to search indexing logic in one repo with clear behavior.

## Steps

1. [ ] **analyst (inline_story)** — Turn the bug report into a single implementable story with clear acceptance criteria and tech investigation notes.
   - focus: Focus on the search indexing behavior for code block content; include AC for both P_NUM finding the page and TEST continuing to work. Add tech notes on investigating Postgres full-text indexing of code blocks.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Loop delivery.
4. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [ ] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Single inline story with no other artifacts to cross-check.

## Planner notes

- attempt 1: designer is not a step of loop quick_fix
