# Contributing to the official node collection

The official nodes live in the `juturna-nodes` repository, in the
`juturna.nodes` namespace, and are used in a pipeline by class name
(`"type": "AudioRtp"`). Follow that repository's `CONTRIBUTING.md` and
`CLAUDE.md`; this file summarizes how it differs from a contrib repository.
Contribute a node here only when it is meant for everyone. Nodes for one
project or organisation belong in their own `juturna.contrib.<org>`
repository.

## Layout and packaging

```text
src/juturna/nodes/
  __init__.py              # registry: _AVAILABLE_PLUGINS, by role
  _my_node/
    my_node.py
    config.toml
    README.md
    *.template             # optional, compiled at runtime
```

- The repository ships only `juturna/nodes`, with hatchling:

  ```toml
  [tool.hatch.build.targets.wheel]
  only-include = ["src/juturna/nodes"]
  sources = ["src"]
  ```

  Never add `src/juturna/__init__.py`.
- Subpackages are allowed (`src/juturna/nodes/my_group/`, used as
  `my_group.MyNode`). Each needs its own `__init__.py` with the same
  lazy-loader registry.
- `juturna stub` generates contrib nodes. Here, copy a similar existing node
  (`_passthrough_identity` is the smallest) and register it by hand, in the
  section of `_AVAILABLE_PLUGINS` that matches its role.

## Dependencies

- No `requirements.txt` in node folders: declare dependencies in
  `pyproject.toml`, with a minimum version you tested.
- Small pure-Python packages (HTTP clients, websockets, database drivers) go
  in the main `dependencies`.
- Heavy packages (ML frameworks, models, media libraries) go in an optional
  extra: reuse an existing one (`av`, `warp`, `audio-ml`, `vision`) when the
  node fits, otherwise add one, and include it in `all`.
- The core only depends on `numpy`. Nothing else comes with it.
- During development, the repository may point `juturna` at a local
  checkout of the core through `[tool.uv.sources]`. Check the node against
  the core version it will be released with: an API the node uses may have
  changed or been removed there.

## Tests

Tests live in `tests/`, one file per node (`tests/test_warp.py`), and run
with `pytest` after installing the `test` extra. A test that needs a node's
optional dependencies skips without them, so the suite runs with any set of
extras:

```python
grpc = pytest.importorskip('grpc')
```

## Documentation

A new node needs, besides its folder `README.md`:

- an entry in the "available nodes" list of the main `README.md`;
- a page in `docs/source/nodes/`, listed in the toctree of its role and in
  `overview.rst`.

## Checklist

- [ ] `_<name>/<name>.py` layout, registered in `_AVAILABLE_PLUGINS` under
      the exact class name.
- [ ] `config.toml` declares every constructor argument, plus `[meta]`.
- [ ] Dependencies in `pyproject.toml`, heavy ones in an extra included in
      `all`.
- [ ] `from juturna.nodes import MyNode` works after
      `pip install -e ".[all,dev_full]"`.
- [ ] The node runs in a small pipeline (`juturna validate` and
      `juturna launch`), and has tests.
- [ ] Node `README.md`, main README entry and docs page are complete.
- [ ] `ruff check` and `ruff format --check` pass.
