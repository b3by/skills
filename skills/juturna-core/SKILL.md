---
name: juturna-core
description: Contribute to the Juturna core library, the Python framework for real-time data pipelines built as DAGs of nodes. Covers the development setup, the checks CI runs (including free-threaded Python), the architecture, the rules for concurrency through the transport layer, the pipeline lifecycle, the node contract that external node packages depend on, the remote service and its wire protocol, registries, tests and docs. Use when changing the Juturna library or its tests. For writing nodes, use juturna-nodes.
---

# Juturna core

How to work on the Juturna library itself: components, payloads, transports,
telemetry, the remotizer and the CLI. The repository's own `CLAUDE.md`,
`CONTRIBUTING.md`, `ruff.toml` and `pyproject.toml` take precedence over this
skill, and over the generic [`python`](../python/SKILL.md) and
[`python-tools`](../python-tools/SKILL.md) skills: Juturna supports Python
3.12 and later, installs with pip extras, and keeps its own ruff
configuration.

The code is the source of truth. Juturna changes quickly, and the repository
documentation and the Sphinx docs sometimes describe older behaviour (such
as the `mark` and `plugins` pipeline keys, or a `plugins/` folder). Before
relying on a description, check it against the code, and fix the
documentation in the same change when they disagree.

## Setup and checks

```bash
pip install -U -e ".[full,dev_full]"   # all features and dev tools
pre-commit install && pre-commit install --hook-type commit-msg
```

Run what CI runs before handing off:

```bash
ruff check .
ruff format --check .
pytest tests/ -q -W error::RuntimeWarning --cov=juturna --cov-report=term-missing
pre-commit run --hook-stage manual bandit --all-files
```

- CI runs the tests on Python 3.12, 3.13, 3.14 and **free-threaded 3.14t**.
  Run at least the affected tests on `3.14t` when you touch concurrency.
- `-W error::RuntimeWarning` turns unawaited coroutines and similar
  warnings into failures.
- CI installs `ffmpeg`, which the media tests need. Tests for optional
  features skip when their extra isn't installed: say which ones skipped.
- `pytest --keep-data` keeps the pipeline folders the tests write under
  `tests/running_pipelines`, for debugging.

## Architecture

```text
juturna/
  components/   Pipeline, Node, Message, State, DAG, node builder,
                pipeline exceptions
  payloads/     frozen payload dataclasses and Draft
  transport/    TransportBackend protocol; threading and browser backends
  telemetry/    telemetry managers: memory, csv, browser
  remotizer/    protobuf wire format and converters for remote nodes
  cli/          python -m juturna commands, HTTP service, PipelineManager,
                remote node service (remotize)
  meta/         tunable constants, overridable by environment variables
  names/        StrEnum status names
  utils/        logging, env vars, safe names, networking helpers
```

Read [architecture.md](references/architecture.md) for how pipelines build
and run nodes, and for the registries. Read it before changing components,
transports, telemetry or the builder.

## Rules

### Concurrency goes through the transport

- Components never import `threading`, `queue` or `multiprocessing`. They
  get queues, events, locks, conditions and workers from the
  `TransportBackend` passed to them. The browser backend (Pyodide, shared
  memory and JSPI) and future backends depend on it.
- Code must be correct **without the GIL**. Protect every piece of state
  shared between workers with a transport lock or condition, and don't
  rely on the atomicity of operations on lists, dicts or counters.
- Put a timeout on every blocking call (`JUTURNA_THREAD_JOIN_TIMEOUT`), and
  make every worker loop check its stop event, so that `stop()` always
  returns.
- Keep the browser backend importable on a desktop interpreter: import `js`
  and `pyodide` lazily, inside the methods that need them.

### Lifecycle and errors

- `Pipeline` is a state machine (NEW, READY, RUNNING, STOPPED, DESTROYED).
  Each method checks the current state and raises a subclass of
  `PipelineStateError` (a `RuntimeError`) for an illegal transition. Keep
  every new operation inside that model.
- A failing `warmup()` or `start()` rolls back what it already did. Keep the
  rollback paths working, and test them.
- Exceptions in user code (`update()`, sources, workers) are caught, logged
  and counted in `health`, and never kill the node. Don't swallow
  exceptions anywhere else.

### The node contract is public API

External packages (`juturna.nodes` and every `juturna.contrib.<org>`
repository) depend on:

- the `Node` hooks and helpers (`__init__(**kwargs)`, `configure`, `warmup`,
  `start`, `stop`, `update(message, **kwargs)`, `set_on_config`,
  `destroy`, `transmit`, `put`, `set_source`, `prepare_template`,
  `dump_json`, `pipe_path`, `static_path`, `logger`);
- the delivery model: `update()` gets every message one at a time, in queue
  order, with the pipeline-owned `State` in `kwargs['state']`, which
  outlives the node's own `stop()` and `kill()`;
- `Message`, the payload classes and `Draft`;
- the builder rules: `config.toml` `[arguments]` as the argument whitelist,
  `$JT_ENV_` resolution, and the `juturna.nodes`, `juturna.contrib` and
  `juturna.extensions` namespaces;
- `juturna.components._node_builder._utils._lazy_node_loaders` and
  `juturna.meta` constants, even where their names are private;
- the remote protocol: the `Warp` node of the official collection talks to
  `juturna remotize` services of the same protocol version.

A change to any of them is a breaking change: mark the commit with `!` and
a `BREAKING CHANGE:` footer, update the docs, and list the affected node
packages so their maintainers can follow. Removing a constant or module
that nodes import breaks them at import time.

### Data and configuration

- Messages are frozen by `transmit()`. Payloads are frozen dataclasses that
  expose `size_bytes` for telemetry. A new payload type also needs
  `serialize`, `Draft` support, a path through every transport, and a
  protobuf message with its converters in the remotizer.
- Add tunables to `juturna/meta/_constants.py`, so they can be overridden
  by an environment variable with the same name. Don't hardcode timeouts or
  sizes.
- Pipeline names, node names and telemetry file names become paths: check
  them with `juturna.utils.jt_utils.check_name`. The HTTP service only
  resolves environment variables from its allowlist; keep it that way.
- Add a backend (transport, telemetry manager, CLI command) by
  implementing its protocol and adding it to its registry. Don't
  special-case it in callers.

### Remote nodes

- Messages, `meta`, `ObjectPayload` values and node states keep their exact
  types on the wire. A value the protocol can't carry is left out with one
  warning per client and kind of loss, or fails the request with
  `DATA_LOSS` when the service runs with `--strict`. Never drop data
  silently.
- The service answers each request when the node's `update()` returns, with
  the first message it transmitted or none, through the node's update
  observer. Keep `Node.link_update_observer` and `link_state_resolver`
  working: the service relies on them to swap the state per request.
- Sticky services keep each client's state; stateless ones receive and
  return it with every request. Support both in every change.
- After editing `remotizer/protos/*.proto`, regenerate `remotizer/c_protos/`
  from the `remotizer` folder with `./compile_protos.sh` (needs
  `grpcio-tools`), and commit the generated files with the change. A
  protocol change is breaking for every `Warp` client.

### Public API and style

- Implementations live in private modules (`_pipeline.py`), and each
  package's `__init__.py` re-exports the public names through `__all__`
  and starts with `# noqa: D104`.
- `ruff.toml` is the style source of truth: 80 columns, single quotes,
  NumPy docstrings. It doesn't enable the import-sorting, naming or pathlib
  rules, so don't reformat imports you aren't otherwise changing.

## Tests

- Tests live flat in `tests/`, one file per area (`test_pipeline.py`,
  `test_transport_registry.py`, ...).
- Test-only nodes live in `tests/_nodes/juturna/contrib/jt_test/nodes/`
  (sequencer, crasher, dumper, aggregator, ...), found through
  `pythonpath = ["tests/_nodes"]`. Use them in pipeline configurations as
  `contrib.jt_test.nodes.<Class>`, and add a new one there when no existing
  node fits.
- Pipeline JSON fixtures live in `tests/test_pipelines/`.
- An autouse fixture resets the global `Message` id counter. Don't depend
  on message ids across tests.
- Wait for conditions with the `wait_for_condition` fixture and a timeout,
  never with a fixed `sleep`.
- Every bug fix gets a regression test, and concurrency fixes get a test
  that fails without the fix under load or on `3.14t`.

## Docs and releases

- The Sphinx docs are in `docs/source/` (`how_to/`, `explain/`, `create/`,
  `built_in/`) and are published from `main`. Update the pages that
  describe what you changed:

  ```bash
  sphinx-build -b html docs/source/ docs/build/html
  ```

- Commits follow Conventional Commits, checked by a commit-msg hook and in
  CI. release-please builds the changelog and the version from them, so
  never bump the version or edit `CHANGELOG.md` by hand.
