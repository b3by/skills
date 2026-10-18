---
name: prerelease-review
description: Review a release candidate before it ships. Inventories every commit since the last release tag, traces linked issues to their tests, reruns every check, verifies the built package and its public API against the version bump, reviews the codebase and its security, and publishes the findings as a GitHub issue. After the owner accepts the report, it checks the release-please release PR and the published release. Use only when the user explicitly asks for a prerelease or release-candidate review, or for its follow-up. Not for ordinary feature work, and creating or editing this skill doesn't run it.
---

# Prerelease review

An evidence-based review of a release candidate, delivered as a GitHub issue,
so the owner can decide whether to fix findings or release. It runs only on
explicit request. The consuming project's instructions, conventions and
release configuration take precedence. Resolve the repository, branches,
commands and version files from the project; don't assume them.

Releases are made with
[release-please](https://github.com/googleapis/release-please): it reads the
Conventional Commits on the default branch, opens a release PR that bumps the
version and updates the changelog, and creates the tag and GitHub Release
when that PR is merged. Agents never bump versions or create tags by hand.

## Authorization and stages

Keep three stages separate. Each one needs its own authorization.

1. **Review.** A request for a prerelease review authorizes the analysis,
   running checks locally, and creating or updating the report issue. It
   doesn't authorize changing application code, tests, dependencies or
   versions. Findings are proposals.
2. **Remediation.** Only the findings the owner selects become work. Each
   gets its own issue, linked to the report, and goes through the project's
   normal workflow (for example the
   [`github-workflow`](../github-workflow/SKILL.md) skill), including its
   plan approval. Afterwards, refresh the report.
3. **Release.** Only after the owner explicitly accepts a specific report
   revision and candidate. The owner chooses patch, minor, major or an exact
   version, and triggers release-please. The agent checks the release PR and,
   after the owner merges it, verifies the published release.

Never infer acceptance, approval or a version choice from silence, passing
tests, a recommendation in the report, or approval of something else. Never
merge PRs, enable auto-merge, push to the default branch, create or move
tags, or publish packages.

Use the `gh` CLI for GitHub actions, and only report an issue or PR as
created when `gh` returned its URL.

## 1. Establish the release range

- Pin the candidate: the full commit SHA of the default branch (or the
  branch the owner named) at the start of the review. Note uncommitted local
  changes separately; they are never part of the candidate.
- Fetch history and tags without force (`git fetch --tags origin`), and
  make sure the clone isn't shallow.
- Find the baseline: the latest release tag reachable from the candidate on
  the same release line, following the project's tag format (such as
  `v1.2.0` or `my-package-v1.2.0`):

  ```bash
  git tag --merged <candidate-sha> --sort=-v:refname --list 'v*' | head -1
  ```

  Check it against `.release-please-manifest.json`. If no release tag exists
  after a full fetch, review the whole history as an initial release. If the
  baseline is ambiguous, ask.
- List **every commit** in `<baseline>..<candidate>`, with full messages.
  Also collect the net diff, renames and deletions, dependency and lockfile
  changes, migrations and configuration changes.
- Extract the issue references (`Closes #N`, `Refs #N`, issue URLs) from
  commit messages and their PRs. Fetch each issue's requirements, acceptance
  criteria and approval comments. List unlinked commits and inaccessible
  issues instead of guessing their intent.
- Account for reverted and superseded work. Never describe reverted
  features as shipped.

## 2. Trace requirements to tests

Build a traceability table: **issue and approved requirement → implementation
→ test and what it asserts → result → gap**.

- Read the test bodies and the fixtures they depend on. A test name or a
  passing run doesn't prove the requirement is covered.
- Check happy paths, error paths, edge cases and regression tests for fixes.
- Flag missing tests, assertions too weak to catch the wrong behaviour,
  skipped or `xfail` tests, and tests that check a different requirement.
- Use the approved requirements as the expected behaviour. Keep proposals,
  open questions and contradictions separate from defects.

## 3. Run every check

Discover the checks from the project's instructions, `pyproject.toml`, CI
workflows and pre-commit configuration, and run **all** of them on the
candidate, with the locked dependencies:

```bash
uv sync --locked --all-extras
uv run ruff check .
uv run ruff format --check .
uv run mypy
uv run pytest --cov=my_package --cov-report=term-missing
```

In projects that use the
[`python-containers`](../python-containers/SKILL.md) setup, run the tests in
the container (`docker compose run --rm app pytest ...`) against a freshly
built stack, and also build the production image
(`docker compose -f compose.yaml build`).

Include the database, integration and slow suites, and test on every Python
version in the CI matrix. Then check what users will install:

```bash
uv build
uvx twine check --strict dist/*
uv run --isolated --no-project --with dist/my_package-1.3.0-py3-none-any.whl \
  python -c 'import my_package'
uvx griffe check my_package -s src -a <baseline-tag>
```

- **Artifacts:** the wheel installs and imports in a clean environment, and
  so does each optional extra. List the wheel and sdist contents and check
  that nothing is missing (`py.typed`, data files) and nothing extra ships
  (tests, secrets, local files).
- **Metadata:** `requires-python`, dependency ranges, license, URLs and the
  README render correctly.
- **Public API:** `griffe check` lists breaking changes since the baseline.
  Each one must be intended and marked in its commit (`!` or
  `BREAKING CHANGE:`), or it's a finding.
- **Migrations:** Alembic migrations upgrade a database at the baseline's
  schema to head, and downgrade back where the project supports it.
- **Docs:** the documentation builds without warnings, if the project has a
  docs build.

Record each command with its working directory, tool versions, candidate
SHA, exit code and pass, fail and skip counts. Classify every check as
passed, failed, blocked (couldn't run) or skipped. A check that couldn't run
is never a pass. Investigate failures without fixing product code or
weakening tests, and keep the original failure in the report even if a
rerun passes.

## 4. Review the codebase

Review the whole first-party codebase as it stands at the candidate, not only
the diff. Use the diff to prioritize where regressions are likely.

- Correctness and integration between modules.
- Consistency with the project's style and architecture (and the `python`
  skill where the project follows it).
- Error handling, resource cleanup, concurrency and data integrity.
- Maintainability: duplication, complexity, unclear names, missing docs.
- Dead code, unused dependencies and unused functionality. Before calling
  something unused, check public exports, entry points, dynamic imports,
  plugin registries and configuration. A failed text search isn't enough.
- Deprecations scheduled for removal in this release.

Tie every finding to a permalink at the candidate SHA, its observable impact,
the evidence and a proportionate fix. Separate defects from optional
cleanup. Say which areas were reviewed and which weren't. Never call a
partial review complete.

## 5. Assess security

Read [security.md](references/security.md) and do both parts, scaled to the
project: a white-box review of the source and dependencies, and black-box
checks through the public interface (the API for a library, the endpoints
for a service). A dependency scan alone covers neither.

## 6. Publish the report

Read [report.md](references/report.md) and publish the report as a GitHub
issue. If a report already exists for this release line, update it instead
of opening a new one. Write the body to a temporary file outside the
repository, and never commit report drafts.

- Give every finding a stable ID (`R1`, `R2`, ...) that stays the same across
  report revisions, so the owner can select them.
- Include a version recommendation based on
  [Semantic Versioning](https://semver.org/), and say whether it matches the
  bump release-please will compute from the commits. The recommendation is
  not the owner's choice.
- In a public repository, keep exploit details, credentials and personal
  data out of the issue. Publish a redacted summary and ask the owner for a
  private channel for the rest.
- If publishing fails, give the report in the conversation and say that no
  issue was created.

Then present the issue link, the most important findings and the real test
results, and **stop**. Don't open fix issues, fix code or start a release
until the owner decides.

## 7. Remediation

When the owner selects findings to fix:

- Open (or reuse) one issue per fix, linking the report with
  `Refs #<report>`, and follow the project's workflow for each.
- After the fixes are merged, refresh the report: move the candidate to the
  new SHA, keep the finding IDs, mark each finding's resolution, rerun every
  check from step 3 and the affected security checks, and present the revised
  report for acceptance.

## 8. Release

Only after the owner explicitly accepts a report revision for a specific
candidate SHA:

1. If the owner hasn't chosen the version, ask: **"Should this be a patch,
   minor, or major release?"** An exact version (`1.4.0`) is also a choice.
2. The owner triggers release-please, passing the exact version when it
   differs from the computed one. Trigger it yourself only when asked:

   ```bash
   gh workflow run release-please.yml -f release-as=1.4.0
   ```

3. Check the release PR that release-please opens:
   - Its base contains exactly the accepted candidate. If commits landed
     after the accepted SHA, the review must be refreshed and accepted again.
   - The version matches the owner's choice in every versioned file
     (`pyproject.toml`, `__init__.py`, docs configuration).
   - The changelog entries match the report, breaking changes are listed,
     and the PR changes nothing but release metadata.
   - CI passes on the release PR.

   Comment the results on the release PR and on the report issue, and leave
   the merge to the owner.
4. After the owner merges, verify the result: the tag exists and points to
   the release commit, the GitHub Release is published, and any publish
   workflow (such as PyPI) succeeded. Report what actually happened,
   including failures and jobs still running.
5. When the release is out, close the report issue with a comment linking
   the tag and the GitHub Release, if the owner asks you to.

## Handoffs

At every stop, report the current stage (reviewing, awaiting the owner's
decision, remediation in progress, awaiting release PR, release verified),
the report and PR links that `gh` returned, the candidate SHA, the real check
results, and any blocker with what's needed to clear it.
