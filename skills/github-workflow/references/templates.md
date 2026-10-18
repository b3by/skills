# Issue and PR templates

Fill in every section. Write "None" instead of deleting a section that
doesn't apply. If the repository has its own templates in
`.github/ISSUE_TEMPLATE/` or `.github/pull_request_template.md`, use those
instead.

## Feature issue

Labelled `feat` (or `refactor`, `perf`, `docs`, ... for non-feature work).

````markdown
## Requirements

The user need and the expected behaviour, in a few sentences.

Acceptance criteria:

- [ ] Searching by name returns matching records, case-insensitively.
- [ ] An empty query returns a validation error.

Out of scope: what this issue deliberately doesn't cover.

## Proposed solution

The approach, the affected modules, public API or data model changes, new
dependencies, trade-offs, and open questions for the owner.

## Code examples

```python
# Proposed, not yet implemented
results = store.search('alpha', limit=10)
```

## Test plan

The test scenarios that prove each acceptance criterion, including failure
cases.

## Execution plan

1. Ordered implementation steps.
2. Tests for every acceptance criterion.
3. Documentation updates.
4. Local checks, then a PR closing this issue.

## Approval

Awaiting the owner's approval of this plan.
````

## Bug issue

Labelled `fix`.

````markdown
## Problem

What goes wrong, in one or two sentences.

## Reproduction

1. Steps, commands or a minimal snippet that triggers the bug.
2. ...

Version, commit or environment where it happens.

## Expected behaviour

What should happen.

## Actual behaviour

What happens instead, with the relevant error or log lines (no secrets).

## Proposed fix

The likely cause and the planned change, and the regression test that will
reproduce the bug.

## Approval

Awaiting the owner's approval of this plan.
````

## Pull request

Title: `<type>(<optional scope>): <lowercase imperative summary>`, such as
`feat(search): add record search`.

````markdown
## Summary

What the PR does and why, in a few sentences.

## Changes

- The main changes, grouped by area.

## Tests

- The tests added or changed, mapped to the acceptance criteria.
- The checks run locally and their results:

  ```text
  uv run ruff check .    passed
  uv run mypy            passed
  uv run pytest          42 passed
  ```

## Notes for the reviewer

Breaking changes, migrations, new dependencies, follow-up work, or "None".

Closes #42
Refs #40
````
