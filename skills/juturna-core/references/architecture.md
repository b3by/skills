# Juturna architecture

How a pipeline is built and run. Paths are relative to the `juturna/`
package. Check the code before relying on any detail here.

## Pipeline configuration

```json
{
  "version": "3.0.0",
  "pipeline": {
    "name": "my_pipeline",
    "id": "1234",
    "folder": "./running_pipelines",
    "transport": "threading",
    "telemetry": { "name": "memory" },
    "nodes": [
      { "name": "src", "type": "AudioRtp", "configuration": { "port": 0 } },
      {
        "name": "filter",
        "type": "contrib.my_org.nodes.ThresholdFilter",
        "configuration": { "threshold": 0.7 },
        "warped": true,
        "sticky": false,
        "warp_configuration": { "grpc_host": "filters.internal" }
      }
    ],
    "links": [{ "from": "src", "to": "filter" }]
  }
}
```

- `transport` is a backend name or `{"name": ..., <options>}`; the default
  is `threading`. `telemetry` is off when absent.
- Node roles (source, processing, sink) come only from the graph shape.
- A node with `"warped": true` is replaced by the `Warp` node of the
  official collection, configured by `warp_configuration`, which sends each
  message to a `juturna remotize` service over gRPC. The node's own
  `configuration` becomes `remote_config`, and `sticky` (default `true`)
  says whether the service keeps the node state (see "Remote service").
- The HTTP service validates configurations with a pydantic model
  (`cli/commands/models/api/`) before building them.

## Lifecycle (`components/_pipeline.py`)

`NEW → warmup() → READY → start() → RUNNING → stop() → STOPPED →
destroy() → DESTROYED`. Each method enforces its transition and raises a
`PipelineStateError` subclass otherwise.

- `warmup()` writes the configuration to the pipeline folder, gives every
  node its own subfolder, builds the nodes, links them (`add_destination`
  and `origins`), rejects duplicate node names and cycles (`_dag.py`), sets up
  telemetry, and calls each node's `warmup()`. A failure rolls back what
  was built.
- `start()` walks the DAG layers in reverse (sinks first, sources last), so
  no node receives data before it runs. `stop()` walks them forward, one
  layer at a time, so upstream nodes stop before downstream ones.
- `update_node(name, prop, value)` calls the node's `set_on_config`.

## Node building (`components/_node_builder/`)

1. Resolve the `type`: a path starting with `extensions` or `contrib` is
   imported under `juturna.`, anything else under `juturna.nodes.`. The last
   segment is the class name, read from the package registry.
2. Read `config.toml` next to the class module (`importlib.resources`).
3. Merge: take each key of `[arguments]` from the pipeline `configuration`
   if present, otherwise the default. Undeclared keys are dropped.
4. Resolve values starting with `$JT_ENV_` from the environment, cast to the
   type of the default; an unset variable is an error.
5. Instantiate with the arguments plus `node_name`, `pipe_name` and
   `transport`, then call `configure()`.

`juturna/__init__.py` extends its `__path__` with `pkgutil.extend_path`, so
other installed distributions can provide `juturna.nodes`,
`juturna.contrib.<org>` and `juturna.extensions`. Each node package's
`__init__.py` maps class names to modules in `_AVAILABLE_PLUGINS` and
imports them lazily through `_lazy_node_loaders`.

## Node runtime (`components/_node.py`)

`Node[T_Input, T_Output]` runs up to two workers, created through the
transport:

- `_worker` takes each message from the input queue, records the `rx`
  telemetry, and calls `update(message, state=...)` directly;
- `_source` runs only after `set_source()`: it calls the source function in
  a loop and feeds the results to the node's own queue, with an exponential
  backoff on errors.

There is no buffer or synchroniser: messages from every upstream node reach
`update()` one at a time, in queue order, and `message.creator` tells them
apart. A node that joins several inputs keeps the pending messages in its
`State`, which the pipeline creates for each node at warmup and which
outlives the node's `stop()` and `kill()`.

`transmit()` freezes the message and puts it into every destination's
queue. Exceptions in user code are caught, counted in `health`, and logged.

Two hooks serve the remote service, and aren't meant for node authors:

- `link_state_resolver(fn)` picks the state for each message instead of the
  pipeline's one; returning `Node.SKIP_UPDATE` drops the message without
  calling `update()`.
- `link_update_observer(fn)` is called after every `update()`, with the
  message and the exception it raised, or `None`.

## Registries

| Registry | Where | Entries |
| --- | --- | --- |
| Transports | `transport/_registry.py` (`_TRANSPORTS`) | `threading`, `browser` |
| Telemetry managers | `telemetry/_registry.py` (`_MANAGERS`) | `memory`, `csv`, `browser` |
| CLI commands | `cli/commands/__init__.py` (`_MODULES`) | `launch`, `validate`, `serve`, `create`, `stub`, `remotize` |

CLI commands register only when their module imports, so a command whose
extra isn't installed simply doesn't appear. Each command module provides
`setup_parser(subparsers)` and `_execute(args)`.

## Transports (`transport/`)

`TransportBackend` (`_base.py`) provides `new_queue`, `new_event`,
`new_lock`, `new_condition`, `spawn` and `is_current`, plus `node_scope` and
`remote_destination` for backends where nodes live in separate workers.
Queue timeouts raise `juturna.transport.Empty`, and full queues
`juturna.transport.Full`.

- `ThreadingTransport` (`threads/`): threads and standard queues.
- `BrowserTransport` (`browser/`): Pyodide workers communicating through
  `SharedArrayBuffer` rings and `Atomics`, made blocking by JSPI. Queue
  slots have a fixed size; image, audio and bytes payloads take a binary
  path, other payloads are pickled. Blocking calls must be reached from JS
  through `callPromising()`.

## Constants (`meta/_constants.py`)

Each tunable (`JUTURNA_THREAD_JOIN_TIMEOUT`, `JUTURNA_MAX_QUEUE_SIZE`,
`JUTURNA_SOURCE_MAX_BACKOFF`, `JUTURNA_TELEMETRY_BATCH_SIZE`,
`JUTURNA_PAYLOAD_SIZE_MAX_VISITS`, `JUTURNA_ENV_VAR_PREFIX`, ...) has a
default and can be overridden by an environment variable of the same name,
cast to the default's type.

## HTTP service (`cli/commands/`)

`juturna serve` runs a FastAPI app (the `httpwrapper` extra) backed by the
`PipelineManager` singleton. It authenticates requests with a bearer token,
validates configurations, confines pipeline folders to its base folder, and
resolves only allowlisted environment variables.

## Remote service (`cli/commands/_juturna_remote_service.py`, `remotizer/`)

`juturna remotize --node-name ... --node-type ...` (the `warp` extra) builds
one node and serves it over gRPC to `Warp` clients. `remotizer/protos/`
defines the wire format, `remotizer/c_protos/` holds the generated code,
and `remotizer/utils.py` converts messages, payloads and states.

- Each request carries one message. The service processes it with the
  client's state, through the node's state resolver, and answers when
  `update()` returns (through the update observer), with the first message
  the node transmitted or none.
- Each client gets its own state. A sticky client's state is kept by the
  service and sent back with each response; the client resends its copy
  only when the service answers `FAILED_PRECONDITION` (after a restart, or
  once an idle state was dropped). A stateless client sends its state with
  every request, so any replica behind a load balancer can serve it.
- Values keep their types on the wire. What can't be sent (a value of an
  unsupported type, a second message from one `update()`, a message
  transmitted after `update()` returned) is left out with a warning, once
  per client and kind of loss. With `--strict`, the request fails with
  `DATA_LOSS` instead.
