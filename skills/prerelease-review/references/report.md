# Report template

The report is a GitHub issue. Keep every section, even when it's blocked or
empty, so that missing work stays visible. Link to permalinks at the
candidate SHA, never to branch heads.

Title: `chore(release): prerelease review for <version line> at <short SHA>`,
labelled `chore` (or the project's release label).

````markdown
## Candidate and conclusion

- Repository and branch: `org/my-package`, `main`
- Candidate: <full SHA>, reviewed on <date>, report revision <n>
- Baseline: `v1.3.0` (<SHA>), [compare](<compare link>), and why it applies
- Environment: OS, Python versions, uv and tool versions
- Conclusion: ready for the owner's decision / fixes recommended /
  review incomplete
- Required checks: all passed / some failed / some blocked
- Version: recommended <patch|minor|major> because <reason>; release-please
  will compute <bump>. The owner chooses.
- Main risks and unverified areas.

## Changes

What this release delivers, in plain language, linked to its issues.

| Change | Issues and PRs | Commits | Effect for users |
| --- | --- | --- | --- |
| `feat(search): add record search` | #42, #51 | abc1234 | New `Store.search()` |
| `fix(store)!: reject empty names` | #47 | def5678 | Breaking: empty names now raise |

Explain every breaking change and what users have to do. Then add the full
commit inventory, including unlinked commits, reverts and superseded work.

## Requirements and tests

| Issue and requirement | Implementation | Test and assertion | Result | Gap |
| --- | --- | --- | --- | --- |

## Check results

| Check | Command | Result | Counts | Notes |
| --- | --- | --- | --- | --- |
| Lint | `uv run ruff check .` | passed | | |
| Types | `uv run mypy` | passed | | |
| Tests (3.13) | `uv run --python 3.13 pytest` | failed | 210 passed, 2 failed, 3 skipped | R4 |
| Package | `uvx twine check --strict dist/*` | passed | | |
| API | `uvx griffe check ...` | 2 breaking changes | | R2 |
| Postgres tests | `uv run pytest -m postgres` | blocked | | no database server |

List every expected check, so that a check that didn't run is visible.

## Code review and security

Reviewed areas, areas not reviewed, white-box scope, dependency audit
results and black-box cases, each summarized separately.

| ID | Area | Severity | Confidence | Summary | Evidence | Proposed action |
| --- | --- | --- | --- | --- | --- | --- |

For each finding: location, the requirement it violates (if any), how to
reproduce it, impact, proposed fix and how to verify the fix. Mark which
findings should block the release, which are other defects, and which are
optional cleanup. Mark findings that predate this release.

## Owner decision

Awaiting the owner's review.

Candidate follow-up issues (not created yet): finding IDs with proposed
acceptance criteria.
````

## Updating the report

- Record the owner's actual decisions with a link to where they were made:
  the selected findings and their fix issues, the accepted revision and
  candidate SHA, accepted remaining risks, and the chosen version.
- On each refresh, increase the revision number, move the candidate, keep
  the finding IDs and record each finding's resolution.
- At release time, add the release PR link, the release commit and tag, the
  GitHub Release link and the publish results.
