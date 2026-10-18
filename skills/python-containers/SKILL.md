---
name: python-containers
description: Containerized development environment for Python libraries, packages and services with Docker Compose. The package source is bind-mounted into the container's site-packages so host edits are live, secrets live only in .env, backing services such as PostgreSQL are Compose services, tests run inside the container, and one multi-stage Dockerfile also builds the production image. Use when creating a new Python project, or when working in a repository that already uses this setup (compose.yaml with a dev override mounting the package). Don't convert projects with a different container setup unless asked.
---

# Python containers

A Docker Compose development environment for Python libraries, packages and
services. Use it for new projects, and follow it in repositories that
already use it. The project's own instructions and configuration take
precedence. Code style is in the [`python`](../python/SKILL.md) skill and
tooling in [`python-tools`](../python-tools/SKILL.md).

Three principles:

1. **Live code.** The image installs the package and its locked dependencies.
   In development, the package source is bind-mounted over the installed copy
   in site-packages, so every edit on the host is what the container runs.
2. **Secrets only in `.env`.** Passwords, tokens and keys live in an
   untracked `.env` file and reach containers through Compose. They are never
   committed, copied into images or written in Compose files.
3. **Services in Compose.** Databases, caches and other dependencies are
   Compose services next to the app, never installed on the host.

## Layout

```text
my_package/
  Dockerfile
  .dockerignore
  compose.yaml            # base: app (production image) and services
  compose.override.yaml   # development: dev image, mounts, reload, ports
  .env                    # secrets and local settings, never committed
  .env.example            # committed template with placeholder values
  pyproject.toml
  uv.lock
  src/my_package/
  tests/
  migrations/             # if the project has a database
```

Add `.env` to `.gitignore`. Compose loads `compose.override.yaml`
automatically, so `docker compose ...` runs the development setup and
`docker compose -f compose.yaml ...` runs the production one.

## Dockerfile

One Dockerfile with three targets: `dev` for development and tests, `build`
to assemble the production environment, and `runtime` for the production
image.

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13.16-slim-trixie AS base

COPY --from=ghcr.io/astral-sh/uv:0.13.1 /uv /uvx /bin/

ENV UV_PROJECT_ENVIRONMENT=/opt/venv \
    UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never \
    PATH=/opt/venv/bin:$PATH \
    PYTHONUNBUFFERED=1

RUN useradd --create-home --uid 1000 app

WORKDIR /app


FROM base AS dev

ENV PYTHONPYCACHEPREFIX=/tmp/pycache

COPY pyproject.toml uv.lock README.md ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --all-extras --no-install-project

COPY src ./src
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --all-extras --no-editable \
    && chown app:app /app

USER app


FROM base AS build

COPY pyproject.toml uv.lock README.md ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev --no-install-project

COPY src ./src
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev --no-editable


FROM python:3.13.16-slim-trixie AS runtime

ENV PATH=/opt/venv/bin:$PATH \
    PYTHONUNBUFFERED=1

RUN useradd --create-home --uid 1000 app

COPY --from=build /opt/venv /opt/venv
COPY pyproject.toml alembic.ini /app/
COPY migrations /app/migrations

WORKDIR /app
USER app

CMD ["python", "-m", "my_package"]
```

- Pin exact image versions (`python:3.13.16-slim-trixie`, `uv:0.13.1`),
  never floating tags, and use the same Python image in `base` and
  `runtime`, so the copied virtual environment works. Look up the current
  patch releases when creating or updating the pins.
- Install the dependencies before copying the source, so source edits don't
  invalidate the dependency layer.
- `--no-editable` installs the package as a normal wheel into
  `/opt/venv/lib/python3.13/site-packages/my_package`, with its metadata and
  entry points. That is the directory the development setup mounts over.
- `PYTHONPYCACHEPREFIX` keeps bytecode out of the mounted source, and the
  non-root `app` user (UID 1000, like the usual host user) keeps every file
  the container creates owned by you on the host.
- The `runtime` image has no dev dependencies, no uv, no tests and no
  secrets, and runs as non-root. Copy the migrations only if the project has
  a database.
- A library that isn't deployed doesn't need the `build` and `runtime`
  targets. Point the `app` service at the `dev` target in `compose.yaml` and
  leave out the override.

Exclude everything else from the build context:

```text
# .dockerignore
.git
.venv
.env
.env.*
!.env.example
**/__pycache__
.pytest_cache
.ruff_cache
.mypy_cache
.coverage
dist
```

## Compose files

The base file defines the production setup and every backing service:

```yaml
# compose.yaml
name: my-package

services:
  app:
    build:
      context: .
      target: runtime
    environment:
      MY_APP_DATABASE_URL: postgresql+psycopg://${POSTGRES_USER:?}:${POSTGRES_PASSWORD:?}@postgres:5432/${POSTGRES_DB:?}
      MY_APP_API_TOKEN: ${MY_APP_API_TOKEN:?}
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:18.6-trixie
    environment:
      POSTGRES_USER: ${POSTGRES_USER:?}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?}
      POSTGRES_DB: ${POSTGRES_DB:?}
    volumes:
      - postgres-data:/var/lib/postgresql
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U "$$POSTGRES_USER" -d "$$POSTGRES_DB"']
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  postgres-data:
```

The override turns the app into the development container:

```yaml
# compose.override.yaml
services:
  app:
    build:
      target: dev
    command: >-
      watchfiles --filter python 'python -m my_package'
      /opt/venv/lib/python3.13/site-packages/my_package
    volumes:
      - ./src/my_package:/opt/venv/lib/python3.13/site-packages/my_package
      - ./tests:/app/tests
      - ./migrations:/app/migrations
    ports:
      - 127.0.0.1:8000:8000
```

- Mount the package directory exactly over its installed location. The path
  contains the Python minor version (`python3.13`), so update it together
  with the image. For a namespace package, mount the subpackage you own
  (`./src/my_namespace/my_package` onto `.../site-packages/my_namespace/my_package`).
- Mount `tests/` too, and `migrations/` when the project has a database.
  Leave out the mounts, ports and services the project doesn't use.
- Restart services on code changes with `watchfiles` (a dev dependency)
  watching the mounted directory, or with the framework's own reloader
  pointed at it (for example `uvicorn --reload --reload-dir <path>`).
- Publish only the ports you need from the host, bound to `127.0.0.1`.
  Containers reach each other by service name (`postgres:5432`) on the
  Compose network, so backing services usually publish nothing.
- Give every backing service a `healthcheck`, and make the app wait for it
  with `condition: service_healthy`.
- Keep persistent data in named volumes (`postgres-data`), never inside the
  repository.

## Environment and secrets

Compose reads `.env` from the project directory for `${...}` interpolation.

```text
# .env.example: copy to .env and replace every value
POSTGRES_USER=my_package
POSTGRES_PASSWORD=change-me
POSTGRES_DB=my_package
MY_APP_API_TOKEN=change-me
```

- Put every password, token, key and private URL in `.env`, and only there.
  Keep `.env.example` complete, with placeholders, so a new checkout knows
  what to fill in.
- Pass variables to each container explicitly through `environment`, and
  give each container only the values it needs. Don't hand the whole `.env`
  to every service with `env_file`.
- Mark required values with `${VAR:?}`, so Compose fails at startup when one
  is missing instead of starting with an empty value.
- Name the app's variables with the settings prefix from `python-tools`
  (`MY_APP_...`), so pydantic-settings reads them.
- Never copy `.env` into an image (`.dockerignore` excludes it), never pass
  secrets as build arguments, and never print the resolved configuration
  (`docker compose config` shows secrets).

## Daily commands

```bash
cp .env.example .env                     # first time: then fill in .env
docker compose up -d --build --wait      # start the stack, wait for health
docker compose logs -f app               # follow the app
docker compose run --rm app pytest       # run the tests in the container
docker compose run --rm app alembic upgrade head
docker compose exec app python           # shell into the running app
docker compose down                      # stop; volumes are kept
```

- Run tests and anything that needs the backing services inside the
  container: `docker compose run --rm app <command>`. It uses the same
  Python, dependencies and network as the app. ruff and mypy can also run on
  the host with `uv run`.
- Manage dependencies on the host with uv (`uv add`, `uv lock`), then
  rebuild: `docker compose up -d --build`. The lockfile is what the image
  installs.
- Rebuild also after changing `pyproject.toml` in other ways (entry points,
  extras, tool configuration) or the Dockerfile. Code changes under the
  mounted directories need no rebuild.
- `docker compose down -v` deletes the volumes and their data. Run it only
  when the owner asks for a clean database.

## Production image

```bash
docker compose -f compose.yaml build
docker compose -f compose.yaml up -d --wait
docker compose -f compose.yaml run --rm app alembic upgrade head
```

The production stack uses the `runtime` target, mounts no source and
publishes no development ports. Run migrations as an explicit deployment
step, as described in
[`python-tools`](../python-tools/references/postgres.md).

## Verifying the environment

After setting it up or changing it, check that:

- `docker compose config --quiet` succeeds.
- The stack starts healthy with `docker compose up -d --build --wait`.
- The app imports the mounted package:
  `docker compose run --rm app python -c 'import my_package; print(my_package.__file__)'`
  prints a path under `site-packages`.
- An edit on the host is picked up without a rebuild.
- The tests pass with `docker compose run --rm app pytest`.
- No files owned by root appeared in the checkout
  (`find . -user root -not -path './.git/*'`).

Report which of these you ran. If Docker isn't available, say so instead of
reporting the environment as working.
