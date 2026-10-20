# Skills

Agent skills for building Python packages, libraries and services with coding
agents such as Claude Code. Each skill is a directory with a `SKILL.md` file
that an agent loads when a task matches the skill's description.

## Skills in this collection

| Skill | Use it for |
| --- | --- |
| [`python`](skills/python/SKILL.md) | Python coding style: formatting, imports, naming, typing, data classes, NumPy docstrings, errors, logging and file handling |
| [`python-tools`](skills/python-tools/SKILL.md) | Python tooling: uv, ruff, mypy, pytest, pre-commit, GitHub Actions, building and publishing, pydantic, PostgreSQL with SQLAlchemy and Alembic, secure coding |
| [`python-containers`](skills/python-containers/SKILL.md) | Docker Compose development environment: package source mounted into site-packages for live edits, secrets only in `.env`, backing services such as PostgreSQL in Compose, tests in the container, one multi-stage Dockerfile for dev and production |
| [`juturna-nodes`](skills/juturna-nodes/SKILL.md) | Writing Juturna nodes: contrib node repositories, `juturna stub`, `config.toml` arguments, lifecycle, messages and payloads, transport-based concurrency, namespace packaging, unit and pipeline tests, contributing to the official collection |
| [`juturna-core`](skills/juturna-core/SKILL.md) | Contributing to the Juturna library: setup and CI checks (including free-threaded Python), architecture, transport and lifecycle rules, the public node contract, registries, tests and docs |
| [`github-workflow`](skills/github-workflow/SKILL.md) | Issue-first GitHub process (opt-in): issue with a plan → owner approval → typed branch per issue → local checks → squash-merged PR opened by the agent → CI green → the owner merges |
| [`prerelease-review`](skills/prerelease-review/SKILL.md) | Release candidate review (explicit request): commit inventory since the last tag, requirement → test traceability, every check rerun, package and public API checks, code and security review, report as a GitHub issue; the release itself goes through release-please after the owner accepts |

The Python skills are meant to be installed together: `python` decides how
the code looks, `python-tools` how it is built, checked, tested and shipped,
and `python-containers` the environment it is developed and run in.

`github-workflow` is a process agreement: install it, or declare it in the
project's `AGENTS.md`, only in repositories that have adopted it.
`prerelease-review` runs only when explicitly requested.

The Juturna skills are project-specific: install `juturna-nodes` where you
write nodes, and `juturna-core` in the Juturna repository.

## Installation

For Claude Code, for all your projects:

```bash
git clone https://github.com/b3by/skills.git
mkdir -p ~/.claude/skills
ln -s "$PWD/skills/skills/python" ~/.claude/skills/python
ln -s "$PWD/skills/skills/python-tools" ~/.claude/skills/python-tools
ln -s "$PWD/skills/skills/python-containers" ~/.claude/skills/python-containers
```

Install `github-workflow` the same way, but only for repositories that
follow it, in the project's `.claude/skills/` directory.

For a single project, link or copy the skills into the project's
`.claude/skills/` directory instead. For agents that don't support the
`SKILL.md` format, reference the file from the project's `AGENTS.md`.

A project's own configuration and instructions always take precedence over
a skill.

## Contributing

See [AGENTS.md](AGENTS.md) for the layout and authoring conventions.
