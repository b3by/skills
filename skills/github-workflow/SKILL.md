---
name: github-workflow
description: Issue-first GitHub workflow for features and fixes. Covers a GitHub issue with a plan, explicit owner approval, a typed branch per issue, local checks, a squash-merged PR opened by the agent, CI follow-up and review. Opt-in only. Use it when the repository's AGENTS.md or CLAUDE.md says it follows this workflow, or when the user asks for it explicitly. Do not apply it to other repositories, and never merge PRs under it.
---

# GitHub workflow

An agreement between the repository owner and coding agents on how features
and fixes move through GitHub: issue, approved plan, branch, pull request,
merge. It applies **only** to repositories that have adopted it (their
`AGENTS.md` or `CLAUDE.md` says so) or when the user asks for it. Never
impose it elsewhere. The repository's own instructions take precedence, and
editing this skill doesn't adopt or run it.

```text
issue → owner approves the plan → branch → implement and test
      → local checks → PR → CI green → owner reviews and squash-merges
```

## Authorization

- **The owner** approves plans, decides scope, chooses versions and merges.
- **An approved plan** authorizes implementing it on its branch, committing,
  pushing that branch, opening its PR and fixing CI and review findings
  within the approved scope.
- **It does not authorize** merging, enabling auto-merge, pushing to the
  default branch, bypassing branch protection, releasing, tagging, or work
  outside the plan.
- Never infer approval from silence, elapsed time, passing tests, or an
  approval given for another issue. Never fabricate an approval.
- Honor narrower instructions such as "only open the issue", "don't push" or
  "local work only".

Use the `gh` CLI for every GitHub action, and only report an issue, branch
or PR as created when `gh` returned its URL.

## 1. Open an issue

Every feature or fix starts with an issue, before any implementation. If a
matching issue already exists, reuse it instead of opening a duplicate.

- Write the issue from the template in
  [templates.md](references/templates.md): requirements and acceptance
  criteria, proposed solution, code examples, test plan and execution plan
  for features; reproduction, expected and actual behaviour for fixes.
- Label it with exactly one Conventional Commit type:

  | Label | Use for |
  | --- | --- |
  | `feat` | A new capability |
  | `fix` | A behaviour correction |
  | `docs` | Documentation only |
  | `refactor` | Restructuring without behaviour changes |
  | `perf` | A performance improvement |
  | `test` | Test changes only |
  | `build` | Build system or dependencies |
  | `ci` | Continuous integration |
  | `chore` | Other maintenance |
  | `style` | Formatting without behaviour changes |
  | `revert` | Reverting an earlier change |

  Create a missing type label (`gh label create feat --description ...`)
  and don't change any other repository setting.
- Split work that won't fit one reviewable PR into sub-issues. Each gets its
  own label, branch and PR, and refers to the parent with `Refs #<parent>`.

```bash
gh issue create --title 'Add record search' --label feat \
  --body-file /path/to/issue.md
```

Write the body file in a temporary directory outside the repository, and
delete it afterwards. Never commit issue drafts.

Then **stop and wait for the owner to approve the plan**, in the
conversation or on the issue. Before approval, only read-only investigation
is allowed: no code, scaffolding, dependency changes or migrations. When the
approval comes from the conversation, record it with a comment on the issue
(`Plan approved by the owner in the working session.`).

If the plan has to change materially during the work (scope, approach,
public API, data model), update the issue and get the change approved before
implementing it.

## 2. Create the branch

Branch from an up-to-date default branch, named
`<type>/<issue-number>-<slug>`, where the slug is the issue title in
lowercase with hyphens:

```bash
git switch main
git pull --ff-only origin main
git switch -c feat/42-add-record-search
```

- Find the default branch with `gh repo view --json defaultBranchRef`
  instead of assuming `main`.
- Preserve any uncommitted work you find. Don't stash, reset or discard
  changes you didn't make.
- When resuming work, switch to the existing branch for the issue instead
  of creating a new one.
- Never implement on the default branch.

## 3. Implement and test

- Implement only the approved plan. Share progress so the owner can comment,
  and fold feedback in within the approved scope.
- Write commit messages as Conventional Commits (`feat(search): add query
  parser`). Branch history is squashed on merge, so small focused commits are
  fine, but keep them meaningful.
- Cover every acceptance criterion with tests, using the project's test
  conventions (for Python, see the `python-tools` skill).
- For a fix, first write a regression test that reproduces the bug and
  fails, then make it pass.
- Update the documentation the change affects (README, docstrings, usage
  examples) in the same branch.

## 4. Check before publishing

Run the project's full checks and make them pass. For Python projects, run
the commands in
[`python-tools`](../python-tools/SKILL.md#before-handing-off).

Then review your own diff against the base (`git diff main...HEAD`):

- Every acceptance criterion in the issue is met and tested.
- There are no unrelated changes, reformatting of untouched code, debug
  output, commented-out code or leftover TODOs from this work.
- There are no secrets, tokens, credentials or personal data, in the code,
  fixtures or test output.
- Dependency and lockfile changes are intended and explained.
- Database schema changes come with a migration.
- Breaking changes to public APIs, configuration or data formats are
  intended and called out.
- The branch is up to date with the default branch. If it isn't, rebase it
  (`git rebase origin/main`), rerun the checks, and push with
  `--force-with-lease`. Only ever force-push the agent's own feature branch.

Tests that couldn't run are not passing. If a check can't run (a missing
service, tool or permission), report it as a blocker instead of publishing.

## 5. Open the pull request

```bash
git push -u origin feat/42-add-record-search
gh pr create --base main --title 'feat(search): add record search' \
  --body-file /path/to/pr.md
```

- PRs are **squash-merged**, so the PR title becomes the commit on the
  default branch. Write it as a Conventional Commit with the issue's type:
  `<type>(<optional scope>): <lowercase imperative summary>`.
- Mark breaking changes with `!` in the title (`feat(api)!: ...`) and a
  `BREAKING CHANGE:` paragraph in the body.
- Write the body from the PR template in
  [templates.md](references/templates.md): summary, changes, test evidence
  and notes for the reviewer.
- End the body with `Closes #<number>` for each issue the PR fully resolves
  and `Refs #<number>` for related ones. Don't close a parent issue that
  still has open sub-issues.
- Never bump the version or edit the changelog. release-please derives both
  from the squashed PR titles, and releases belong to the owner.

## 6. Follow CI and review

```bash
gh pr checks --watch
```

- A PR is ready for review only when CI is green. If a check fails, read
  its log, fix the cause on the branch and push again. Never skip, delete or
  weaken tests or checks to get a green run.
- Answer every review comment, either with a fix or with a reason. Push
  fixes to the same branch and rerun the checks. Leave resolving threads to
  the reviewer.
- Then leave the PR to the owner. Never merge it, enable auto-merge, or
  delete the branch before it's merged.

Closing keywords take effect when the PR is merged into the default branch.

## Handoffs

At every stop, report:

- the issue and PR links (only ones `gh` actually returned)
- the branch name
- the checks that ran, and their real results
- the stage: awaiting plan approval, implementing, PR open with CI running,
  CI failing, awaiting review, or awaiting merge
- any blocker, and what's needed to clear it
