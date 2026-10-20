---
name: juturna-nodes
description: Write, package and test nodes for Juturna, the Python library for real-time data pipelines built as DAGs of nodes. Covers the contrib node repository layout (juturna.contrib.<org>.nodes), juturna stub, config.toml arguments, the node lifecycle, messages and payloads, sources, concurrency through the transport, packaging as a namespace package, and tests. Also covers contributing nodes to the official juturna.nodes collection. Use when creating or changing a Juturna node or node repository. For changes to the Juturna library itself, use juturna-core.
---

# Juturna nodes

How to write Juturna nodes as a user of the library: in your own repository
under `juturna.contrib.<org>.nodes`, or as a contribution to the official
`juturna.nodes` collection. The node repository's own instructions and
configuration take precedence. Use the [`python`](../python/SKILL.md) and
[`python-tools`](../python-tools/SKILL.md) skills for code style and
tooling, except where this skill says otherwise.

The installed Juturna package is the source of truth for the node API. When
something here disagrees with `juturna/components/_node.py` or the node
builder of the installed version, follow the code and say so.

## Namespaces

Juturna loads nodes only from three namespaces, and the `type` of a node in a
pipeline configuration is its dotted path inside them:

| Namespace | Who maintains it | `type` in a pipeline |
| --- | --- | --- |
| `juturna.nodes` | the Juturna team (official collection) | `AudioRtp`, `sub.MyNode` |
| `juturna.contrib.<org>.nodes` | you, in your own repository | `contrib.my_org.nodes.ThresholdFilter` |
| `juturna.extensions` | external | `extensions.nodes.MyNode` |

Write your own nodes under `juturna.contrib.<org>`, where `<org>` is your
organisation or author name. Contribute to `juturna.nodes` only through a PR
to the official repository: read
[official-nodes.md](references/official-nodes.md) first.

`juturna` and `juturna.contrib` are namespace packages shared by every
installed node repository. **Never create `src/juturna/__init__.py` or
`src/juturna/contrib/__init__.py`**: either would shadow the core package
or the other contrib packages.

## Repository layout

```text
my_org_nodes/
  pyproject.toml
  src/juturna/contrib/my_org/
    __init__.py
    nodes/
      __init__.py                 # registry: _AVAILABLE_PLUGINS
      _threshold_filter/          # one folder per node, underscore prefix
        threshold_filter.py       # the node class
        config.toml               # every argument with its default
        README.md                 # node documentation
  tests/
    _nodes/juturna/contrib/my_org_test/nodes/   # test-only nodes
    test_threshold_filter.py
```

```toml
[project]
name = "my-org-nodes"
requires-python = ">=3.13"
dependencies = ["juturna>=3,<4"]

[build-system]
requires = ["uv_build>=0.9,<0.10"]
build-backend = "uv_build"

[tool.uv.build-backend]
module-name = "juturna.contrib.my_org"
```

The `module-name` makes `uv_build` package only `juturna/contrib/my_org`,
without the shared namespace levels. Check the built wheel (`unzip -l
dist/*.whl`): it must contain no `juturna/__init__.py` and no
`juturna/contrib/__init__.py`, and must include every `config.toml`.

## Create a node

Generate the skeleton from the repository root:

```bash
uv run python -m juturna stub -n threshold_filter -p my_org \
  -a 'Jane Doe' -e jane@example.com
```

It creates `src/juturna/contrib/my_org/nodes/_threshold_filter/` with the
class, an empty `config.toml`, a `README.md` and a `requirements.txt`, and
registers the node in `nodes/__init__.py`. Delete the `requirements.txt`:
dependencies belong in `pyproject.toml`.

The registry lazy-loads each node on first access, so a node's imports only
load when a pipeline uses it:

```python
# noqa: D104
from juturna.components._node_builder._utils import _lazy_node_loaders


_AVAILABLE_PLUGINS: dict[str, str] = {
    'ThresholdFilter': '._threshold_filter.threshold_filter',
}


__getattr__, __dir__ = _lazy_node_loaders(__name__, _AVAILABLE_PLUGINS)
__all__ = list(_AVAILABLE_PLUGINS.keys())
```

The key must be the exact class name, and the module path is relative, with
a leading dot. Forgetting to register a node is the most common mistake.

## Arguments: config.toml

```toml
[arguments]
threshold = 0.5
key = "value"

[meta]
```

- Declare **every** constructor argument with its default. The builder
  passes only the keys declared here: keys in a pipeline's `configuration`
  that aren't declared are **dropped silently**.
- Keep `[meta]`, even when empty.
- A pipeline can read a value from the environment with
  `"api_key": "$JT_ENV_MY_ORG_API_KEY"`: the builder reads the variable
  `MY_ORG_API_KEY`, casts it to the type of the default, and fails if it is
  unset. Use this for every secret, never a literal in a pipeline file.

## Write the node

```python
"""
ThresholdFilter

@author: Jane Doe
@email: jane@example.com

Forward only the messages whose value reaches a threshold.
"""

import typing

from juturna.components import Message
from juturna.components import Node
from juturna.payloads import ObjectPayload


class ThresholdFilter(Node[ObjectPayload, ObjectPayload]):
    """Forward messages whose value reaches the threshold"""

    def __init__(self, threshold: float, key: str, **kwargs):
        """
        Parameters
        ----------
        threshold : float
            Minimum value a message needs to be forwarded.
        key : str
            Payload key holding the value to compare.
        kwargs : dict
            Superclass arguments.

        """
        super().__init__(**kwargs)

        self._threshold = threshold
        self._key = key

    def set_on_config(self, prop: str, value: typing.Any):
        """Change the threshold at runtime"""
        if prop == 'threshold':
            self._threshold = float(value)

    def update(self, message: Message[ObjectPayload], **kwargs):
        """Forward the message if its value reaches the threshold"""
        value = message.payload.get(self._key)

        if value is None or value < self._threshold:
            return

        to_send = Message[ObjectPayload](
            creator=self.name,
            version=message.version,
            payload=message.payload,
            timers_from=message,
        )

        self.transmit(to_send)
```

The rules every node follows:

- **Constructor.** Take each `config.toml` argument as an explicit
  parameter, plus `**kwargs`, and call `super().__init__(**kwargs)` first.
  The builder always passes `node_name`, `pipe_name`, `synchroniser` and
  `transport`. Only store state in `__init__`; don't open anything.
- **Lifecycle.** Open connections, load models, claim ports and create
  workers in `warmup()`. Release everything in `destroy()`. If you override
  `start()` or `stop()`, call `super().start()` / `super().stop()`, and stop
  and join your own workers with a timeout. The full hook list is in
  [node-api.md](references/node-api.md).
- **Messages are immutable.** A transmitted message is frozen, and so is its
  payload. Never modify the incoming message: build a new `Message` with
  `creator=self.name`, the input's `version` and `timers_from=message`, and
  transmit that.
- **Payloads.** Use the payload types from `juturna.payloads`, and type the
  node with them (`Node[AudioPayload, ObjectPayload]`). Build or change a
  frozen payload with `Draft`, and an `ObjectPayload` by copying it to a
  `dict` and calling `ObjectPayload.from_dict(...)`.
- **Concurrency.** Never use `threading`, `queue` or `multiprocessing`
  directly. Get workers, queues, events and locks from the node's transport
  (`self._transport.spawn(...)`, `.new_queue(...)`, `.new_event()`), so the
  node also runs on the browser transport and on free-threaded Python. Give
  every blocking call a timeout (`juturna.meta.JUTURNA_THREAD_JOIN_TIMEOUT`).
- **Errors.** An exception raised in `update()` or a source is logged and
  counted in `health`, and the node keeps running. Still catch the errors
  you expect, log them with `self.logger` and skip the message, rather than
  relying on that safety net.
- **Files.** Write runtime outputs under `self.pipe_path` (the node's folder
  in the pipeline run), and read packaged files from `self.static_path`.
- **Heavy dependencies.** Import large libraries (models, media) inside
  `warmup()`, or at module level only when the node can't work without
  them, and declare them in an optional extra.

Read [node-api.md](references/node-api.md) for sources, multi-input
synchronisation, payload fields, runtime reconfiguration and the other node
helpers.

## Test the node

Install the repository with its extras (`uv sync --all-extras`), and check
that the node resolves through its pipeline type:

```bash
uv run python -c 'from juturna.contrib.my_org.nodes import ThresholdFilter'
uv run python -m juturna validate --config pipeline.json
```

Write two kinds of tests:

- **Unit tests** call `update()` directly and capture what is transmitted:

  ```python
  def test_threshold_filter_drops_low_values(monkeypatch):
      node = ThresholdFilter(threshold=0.5, key='value', node_name='filter')
      sent = list()
      monkeypatch.setattr(node, 'transmit', sent.append)

      node.update(make_message(value=0.4))

      assert sent == list()
  ```

- **Pipeline tests** run the node in a real `juturna.components.Pipeline`
  between test-only source and sink nodes. Put those under
  `tests/_nodes/juturna/contrib/my_org_test/nodes/`, in the same layout, and
  add `pythonpath = ["tests/_nodes"]` to the pytest configuration. Use
  `tmp_path` as the pipeline `folder`, poll the sink with a deadline instead
  of sleeping a fixed time, and read the results after `stop()` but before
  `destroy()`, which releases the nodes.

## Document the node

Each node folder has a `README.md` with: what the node does and its role
(source, processing or sink), its input and output payload types, every
`config.toml` argument with its meaning and default, the extra it needs, and
an example pipeline entry:

```json
{
  "name": "filter",
  "type": "contrib.my_org.nodes.ThresholdFilter",
  "configuration": { "threshold": 0.7 }
}
```

## Checklist

- [ ] The node is in `_<name>/<name>.py`, and its class is registered in
      `_AVAILABLE_PLUGINS` under its exact name.
- [ ] `config.toml` declares every constructor argument, plus `[meta]`.
- [ ] Secrets come from `$JT_ENV_...` values, never from literals.
- [ ] Resources open in `warmup()` and close in `destroy()`; overridden
      `start()`/`stop()` call `super()`.
- [ ] No direct `threading` or `queue` use, and every blocking call has a
      timeout.
- [ ] Dependencies are declared in `pyproject.toml`, heavy ones in an extra.
- [ ] The wheel has no `juturna/__init__.py` or `juturna/contrib/__init__.py`.
- [ ] Unit and pipeline tests pass, and the README is complete.
