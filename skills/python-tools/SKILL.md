---
name: python-tools
description: Tooling for Python packages, libraries and services. Covers uv projects and dependencies, ruff, mypy, pytest, pre-commit, GitHub Actions CI, building and publishing, pydantic for data models and settings, PostgreSQL with SQLAlchemy and Alembic, and secure coding. Use when creating or configuring a Python project, editing pyproject.toml, adding dependencies, running or configuring ruff/mypy/pytest/pre-commit, writing CI, defining pydantic models, or working with a Postgres database from Python. For how the code itself should look, use the `python` skill.
---

# Python tools

The tools and libraries used to build, check, test and ship Python packages,
libraries and services, and how to configure them. How the code itself
should look is in the [`python`](../python/SKILL.md) skill. If the project
already has its own configuration (`pyproject.toml`, `ruff.toml`,
`.pre-commit-config.yaml`, CI workflows), **that configuration wins**: read it
first, follow it, and use this skill only for what it leaves undefined.

All tool configuration goes in `pyproject.toml` unless a tool can't read it.
Target Python 3.13 or later.

## Project setup (uv)

Use [uv](https://docs.astral.sh/uv/) for interpreters, virtual environments,
dependencies, builds and publishing.

```bash
uv init --lib my_package          # src layout, uv_build backend, py.typed
cd my_package
uv python pin 3.13                # writes .python-version
uv add 'pydantic>=2.14,<3'        # runtime dependency
uv add --dev ruff mypy pytest pytest-cov pre-commit
uv sync                           # create .venv from uv.lock
```

Use `uv init --app` for a service that won't be imported by other code, or
`uv init --package` for an application with a console script entry point.
Commit `pyproject.toml`, `uv.lock` and `.python-version`. Never commit
`.venv`.

```text
my_package/
  pyproject.toml
  uv.lock
  src/my_package/
    __init__.py
    py.typed
    _models.py
  tests/
    test_models.py
```

Daily commands:

```bash
uv sync                           # bring .venv in line with uv.lock
uv sync --all-extras              # include optional extras
uv run pytest                     # run anything inside the project env
uv run python -m my_package       # run the package
uv add --optional server fastapi  # add to an optional extra
uv remove httpx
uv lock --upgrade-package httpx   # upgrade one dependency deliberately
uv tree                           # inspect the resolved dependency graph
uv run --python 3.14 pytest       # test on another interpreter
```

Always run tools through `uv run`, so they use the project's environment and
pinned versions. Don't activate the virtual environment or call `pip`.

## Dependencies

- Keep runtime dependencies **minimal**. Put anything only some users need
  into an optional extra (`[project.optional-dependencies]`) and import it
  lazily inside the feature that uses it.
- Libraries declare **bounded ranges** (`httpx>=0.28,<1`), never exact pins.
  Applications and services also rely on `uv.lock` for exact versions.
- Dev tools go in `[dependency-groups]` (`uv add --dev`), never in runtime
  dependencies or extras.
- Upgrade deliberately (`uv lock --upgrade-package X`) and review the lockfile
  diff. Don't run a blanket `uv lock --upgrade` together with unrelated
  changes.
- Before adding a dependency, check that it is maintained, how much it pulls
  in transitively, and whether the standard library already covers the need.
- Never install from untrusted indexes, or from VCS URLs without pinning a
  commit.

## Ruff

Ruff is the only linter and formatter. Don't add black, isort, flake8 or
pylint next to it. This configuration enforces the `python` skill's style:

```toml
[tool.ruff]
line-length = 80
indent-width = 4
target-version = "py313"

[tool.ruff.format]
quote-style = "single"
indent-style = "space"
docstring-code-format = true
docstring-code-line-length = 80

[tool.ruff.lint]
task-tags = ["TODO", "HACK"]
select = [
    "E", "W",  # pycodestyle
    "F",       # pyflakes
    "D",       # pydocstyle
    "I",       # isort
    "N",       # pep8-naming
    "UP",      # pyupgrade
    "B",       # flake8-bugbear
    "SIM",     # flake8-simplify
    "PTH",     # flake8-use-pathlib
    "S",       # flake8-bandit (security)
]
ignore = ["D203", "D205", "D212", "D400", "D401", "D415"]

[tool.ruff.lint.pydocstyle]
convention = "numpy"

[tool.ruff.lint.isort]
force-single-line = true
lines-between-types = 1
lines-after-imports = 2

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["D", "S101"]
"migrations/**" = ["D"]
```

```bash
uv run ruff check .               # lint
uv run ruff check . --fix         # lint and apply safe fixes
uv run ruff format .              # format
uv run ruff format --check .      # CI: fail if not formatted
uv run ruff rule B006             # explain a rule
```

Never turn a rule off globally to get a change through. If one line must
break a rule, use a targeted `# noqa: <CODE>` with the reason.

## Type checking (mypy)

Run mypy in strict mode on the package. Add the pydantic plugin when the
project uses pydantic. SQLAlchemy 2 needs no plugin.

```toml
[tool.mypy]
python_version = "3.13"
files = ["src"]
strict = true
plugins = ["pydantic.mypy"]

[tool.pydantic-mypy]
init_forbid_extra = true
init_typed = true
```

```bash
uv run mypy
```

- Ship a `py.typed` marker in every library package (`uv init --lib` creates
  it), so users' type checkers read your annotations.
- In an existing untyped project, start without `strict` and tighten module
  by module with `[[tool.mypy.overrides]]`, rather than skipping mypy.
- Silence a single line with `# type: ignore[<code>]` and the reason. Never
  use a bare `# type: ignore`.
- For a third-party package without type information, add its stubs
  (`uv add --dev types-<package>`) or an override with
  `ignore_missing_imports = true` for that module only.

## Testing (pytest)

```toml
[tool.pytest]
minversion = "9.0"
testpaths = ["tests"]
addopts = ["-ra", "--import-mode=importlib"]
strict = true
filterwarnings = ["error"]
markers = ["postgres: requires a running PostgreSQL server"]
```

```bash
uv run pytest                                  # all tests
uv run pytest tests/test_models.py::test_name  # one test
uv run pytest --cov=my_package --cov-report=term-missing
```

- Use plain `assert`s and fixtures in `conftest.py`. Name tests
  `test_<unit>_<behaviour>`.
- Keep tests independent and order-agnostic. Reset global state (counters,
  singletons, environment variables) with fixtures or `monkeypatch`.
- Use `tmp_path` for anything that writes files. Never write to the
  repository tree.
- Use `pytest.mark.parametrize` instead of copy-pasted tests, and
  `pytest.raises(Error, match=...)` for error paths.
- Mock at the boundary (HTTP, clock, filesystem), not internal helpers. Test
  database code against a real PostgreSQL server, never SQLite (see
  [postgres.md](references/postgres.md)).
- Don't sleep a fixed time and hope something finished. Poll for the
  condition with a deadline.
- Every bug fix comes with a regression test.

```python
@pytest.mark.parametrize(
    ('raw', 'expected'),
    [(' Alpha ', 'alpha'), ('BETA', 'beta')],
)
def test_record_in_normalises_name(raw, expected):
    assert RecordIn(name=raw).name == expected


def test_record_in_reads_json_file(tmp_path):
    path = pathlib.Path(tmp_path, 'record.json')
    path.write_text('{"name": "alpha", "size": 3}', encoding='utf-8')

    record = RecordIn.model_validate_json(path.read_text(encoding='utf-8'))

    assert record.size == 3
```

## Pre-commit

```yaml
# .pre-commit-config.yaml
default_install_hook_types: [pre-commit, commit-msg]
default_stages: [pre-commit]

repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v6.0.0
    hooks:
      - id: check-merge-conflict
      - id: check-yaml
      - id: check-json
      - id: check-toml
      - id: end-of-file-fixer
      - id: trailing-whitespace
      - id: detect-private-key

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.17.1
    hooks:
      - id: ruff-check
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/astral-sh/uv-pre-commit
    rev: 0.13.1
    hooks:
      - id: uv-lock

  - repo: local
    hooks:
      - id: mypy
        name: mypy
        entry: uv run mypy
        language: system
        types: [python]
        pass_filenames: false

  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v4.4.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
```

```bash
uv run pre-commit install         # installs both hook types
uv run pre-commit run --all-files
uv run pre-commit autoupdate      # bump hook revisions, then review
```

- mypy runs as a local hook through `uv run`, so it sees the project's real
  dependencies. The mirrored mypy hook runs in an isolated environment and
  misses them.
- Commit messages follow Conventional Commits:
  `feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert`, an optional
  scope, and a lowercase imperative subject, such as
  `fix(store): reject empty record names`.

## CI (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: ci

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  check:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        python-version: ['3.13', '3.14']
    steps:
      - uses: actions/checkout@v7
      - uses: astral-sh/setup-uv@v10
        with:
          python-version: ${{ matrix.python-version }}
          enable-cache: true
      - run: uv sync --locked --all-extras
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run mypy
      - run: uv run pytest --cov=my_package --cov-report=term-missing
```

- `--locked` fails the job if `uv.lock` is out of date with
  `pyproject.toml`.
- Give workflows the least `permissions` they need.
- For a database service, add the `postgres` service shown in
  [postgres.md](references/postgres.md).

## Building and publishing

- Keep `uv_build` as the build backend. Switch to `hatchling` only when you
  need a feature `uv_build` lacks, such as dynamic versions or custom file
  selection.
- Build and check the artifacts locally:

  ```bash
  uv build                          # sdist and wheel in dist/
  uvx twine check --strict dist/*   # metadata and README render
  ```

- Version and release with
  [release-please](https://github.com/googleapis/release-please). It reads
  the Conventional Commits on the default branch, opens a release PR that
  bumps the version in `pyproject.toml` (and any `extra-files`) and updates
  `CHANGELOG.md`, and tags and creates the GitHub Release when that PR is
  merged. Never bump versions, edit the changelog or create tags by hand.
  The owner decides when to release and which version.
- Publish to PyPI from the same workflow, with trusted publishing, never
  with a token stored in the repository. A release created with
  `GITHUB_TOKEN` doesn't trigger other workflows, so a separate
  `on: release` workflow would never run:

  ```yaml
  # .github/workflows/release-please.yml
  name: release-please

  on:
    workflow_dispatch:
      inputs:
        release-as:
          description: Exact version to release (optional)
          required: false

  jobs:
    release-please:
      runs-on: ubuntu-latest
      permissions:
        contents: write
        pull-requests: write
      outputs:
        release_created: ${{ steps.release.outputs.release_created }}
      steps:
        - id: release
          uses: googleapis/release-please-action@v5
          with:
            release-as: ${{ inputs.release-as }}

    publish:
      needs: release-please
      if: needs.release-please.outputs.release_created == 'true'
      runs-on: ubuntu-latest
      environment: pypi
      permissions:
        id-token: write
        contents: read
      steps:
        - uses: actions/checkout@v7
        - uses: astral-sh/setup-uv@v10
        - run: uv build
        - run: uv publish
  ```

  Configure it in `release-please-config.json`. List `uv.lock` in
  `extra-files`, because it records the project's own version and
  release-please doesn't update it otherwise; a stale lockfile fails
  `uv sync --locked`:

  ```json
  {
    "release-type": "python",
    "packages": {
      ".": {
        "package-name": "my-package",
        "changelog-path": "CHANGELOG.md",
        "extra-files": [
          {
            "type": "toml",
            "path": "uv.lock",
            "jsonpath": "$.package[?(@.name.value=='my-package')].version"
          }
        ]
      }
    }
  }
  ```

  Add `.release-please-manifest.json` with the current version
  (`{".": "0.1.0"}`).
- Before a release, the [`prerelease-review`](../prerelease-review/SKILL.md)
  skill reviews the candidate, if the owner asks for it.
- Never trigger a release, publish, tag or create a GitHub Release unless
  the owner asked for it.

## Data models and settings (pydantic)

Use pydantic v2 for data that crosses a boundary: request and response
bodies, messages, files, configuration and environment variables. Keep
frozen dataclasses (see the `python` skill) for internal values that are
already valid.

- Make models strict about their input: `ConfigDict(frozen=True,
  extra='forbid')`.
- Express constraints with types and `Field` (`Annotated[int, Field(ge=0)]`),
  and use validators only for what types can't say.
- Read settings with `pydantic-settings`, from environment variables with a
  project prefix, and store secrets as `SecretStr`.

Read [pydantic.md](references/pydantic.md) when defining models, validators
or settings.

## PostgreSQL (SQLAlchemy and Alembic)

Use PostgreSQL through SQLAlchemy 2 with the psycopg 3 driver, and manage the
schema with Alembic migrations.

- Libraries depend on `sqlalchemy` and `psycopg`. Applications and services
  add `psycopg[binary]` (or `psycopg[c]`) and `alembic`.
- Use typed declarative models (`Mapped`, `mapped_column`), 2.0-style
  `select()` queries and one transaction per unit of work.
- Change the schema only with a reviewed Alembic migration.

Read [postgres.md](references/postgres.md) before writing models, queries,
migrations or database tests.

## Security

The ruff `S` rules check the code on every lint run. Also audit the
dependencies before a release and after adding new ones:

```bash
uv run --with pip-audit pip-audit --skip-editable
```

Read [security.md](references/security.md) when the code handles untrusted
input, files, subprocesses, network access, secrets or randomness.

## Before handing off

Run the full set of checks and report the real results:

```bash
uv run ruff check . && uv run ruff format --check . && uv run mypy \
  && uv run pytest
```

In projects that use the
[`python-containers`](../python-containers/SKILL.md) setup, run the tests in
the container instead: `docker compose run --rm app pytest`.

If a check couldn't run (for example, no database server was available), say
so. Don't report it as passing.
