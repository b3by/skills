# Agent guide

This repository is a collection of agent skills. It contains no application
code: the product is the Markdown in `skills/`.

## Layout

```text
skills/
  <skill-name>/
    SKILL.md            # required: frontmatter + instructions
    references/         # optional: detail loaded on demand
```

- One directory per skill. The directory name equals the frontmatter `name`
  (lowercase, hyphenated).
- `SKILL.md` links to supporting files with relative paths.

## Current skills

| Skill | Kind | Trigger |
| --- | --- | --- |
| `python` | Style guide | Writing, editing or reviewing Python code |
| `python-containers` | Environment guide | Creating a Python project, or working in one with the Compose development setup |
| `python-tools` | Tooling guide | Setting up, configuring, testing, building or shipping a Python project; pydantic; PostgreSQL |
| `github-workflow` | Process agreement (opt-in) | Feature or fix work in a repository that has adopted the issue-first workflow |
| `prerelease-review` | Process agreement (explicit request) | A request to review a release candidate, and its release follow-up |

`python-tools` owns the ruff configuration that enforces the `python` style,
and `python` links to it. When you change a style rule that ruff can check,
update both skills in the same change.

The Conventional Commit types appear in `github-workflow` (issue labels) and
`python-tools` (commit-msg hook). Keep the two lists identical.

Process agreements such as `github-workflow` must say they are opt-in, spell
out what an approval authorizes, and leave merging and releases to the
owner.

Releases go through release-please in every skill: agents never bump versions,
edit changelogs or create tags by hand. `python-tools` holds the release
workflow, `github-workflow` and `prerelease-review` rely on it.

## Writing a skill

- The frontmatter `description` is all an agent sees before loading the
  skill: say what it covers, when to use it and when not to.
- Start the body with a short paragraph on scope, and state that the
  consuming project's own configuration and instructions take precedence.
- Write rules in the imperative mood and give the reason when it isn't
  obvious. Prefer a short example over a paragraph of description.
- Keep examples generic and fictional (`my_package`, `Record`). Never include
  real credentials, hosts or customer data.
- Wrap prose at about 80 columns.
- Keep `SKILL.md` to what an agent needs every time the skill fires; move
  long or rarely needed material into `references/`.

## Verifying a change

1. Check that each `name` matches its directory:

   ```bash
   for f in skills/*/SKILL.md; do
     d=$(basename "$(dirname "$f")")
     n=$(sed -n 's/^name: //p' "$f" | head -1)
     [ "$d" = "$n" ] || echo "name mismatch: $f ($n)"
   done
   ```

2. Re-read the `description` and ask whether an agent would load the skill
   for the tasks it is meant for, and not for the ones it excludes.
3. If you added, renamed or removed a skill, update the tables in
   `README.md` and in this file.

## Working in this repository

- Commits use Conventional Commits: skill edits are usually `docs`, a new
  skill is `feat`.
- Preserve the tone and structure of a skill when making targeted edits.
