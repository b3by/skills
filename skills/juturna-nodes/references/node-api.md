# Node API

The parts of `juturna.components.Node` a node author uses. Check the
installed version's `juturna/components/_node.py` when in doubt.

## Lifecycle hooks

All hooks are optional to override.

| Hook | Called | Use it to |
| --- | --- | --- |
| `__init__(<args>, **kwargs)` | when the pipeline builds the node | store arguments; call `super().__init__(**kwargs)` first |
| `configure()` | right after `__init__` | resolve derived settings, such as claiming a free port for `port = 0` with `juturna.utils.net_utils.get_available_port()` |
| `warmup()` | on `pipeline.warmup()` | open connections, load models, prepare templates, create (not start) workers, call `set_source()` |
| `start()` | on `pipeline.start()` | start your own workers, then call `super().start()` |
| `update(message, **kwargs)` | for each delivered message or batch | process input and `transmit()` results |
| `set_on_config(prop, value)` | on `pipeline.update_node(...)` | change a setting at runtime |
| `stop()` | on `pipeline.stop()` | call `super().stop()`, signal your workers and join them with a timeout |
| `destroy()` | on `pipeline.destroy()` | close connections, free models and other resources |
| `configuration` (property) | by the service and telemetry | extend `super().configuration` with the settings worth reporting |
| `next_batch(sources)` | by the buffer, for multi-input nodes | release messages from several inputs together (see below) |

Pipelines start nodes from sinks to sources and stop them from sources to
sinks, so a node never receives messages before it has started.

## Attributes and helpers

| Name | What it is |
| --- | --- |
| `self.name`, `self.pipe_name`, `self.pipe_id` | the node name, and the name and id of its pipeline |
| `self.logger` | a logger named `<pipe_name>.<node_name>` |
| `self.pipe_path` | the node's own folder in the pipeline run, for outputs |
| `self.static_path` | the folder of the node's source code, for packaged files |
| `self.prepare_template(name, dest, args)` | compile a `string.Template` file from `static_path` into `pipe_path` |
| `self.dump_json(message, file_name)` | write a message as JSON into `pipe_path` |
| `self.transmit(message)` | freeze a message and send it to every destination |
| `self.put(message)` | feed a message into the node's own input queue |
| `self.health` | failure counters per worker, and stuck workers |
| `self._transport` | the concurrency backend: `spawn()`, `new_queue()`, `new_event()`, `new_lock()`, `new_condition()` |

The transport types (`Queue`, `Event`, `Lock`, `WorkerHandle`, and the
`Empty` and `Full` exceptions) are importable from `juturna.transport`, and
the timeouts and limits from `juturna.meta` (`JUTURNA_THREAD_JOIN_TIMEOUT`,
`JUTURNA_MAX_QUEUE_SIZE`, `JUTURNA_SOURCE_MAX_BACKOFF`).

## Messages

```python
to_send = Message[AudioPayload](
    creator=self.name,
    version=message.version,
    payload=payload,
    timers_from=message,
)
to_send.meta['language'] = 'en'

with to_send.timeit('inference'):
    ...

self.transmit(to_send)
```

- Every message gets an automatic `id` and `created_at`.
- `meta` is a free-form dict for annotations, and `timers` holds timings
  (`timer(name, value)`, or the `timeit(name)` context manager).
  `timers_from` copies the upstream timers, so latency can be traced along
  the pipeline.
- Set `meta` and timers before `transmit()`. After it, the message, its
  `meta` and its timers are read-only.
- `to_dict()` and `to_json()` serialize a message; payloads provide their
  own `serialize`.

## Payloads

Payloads are frozen dataclasses from `juturna.payloads`:

| Payload | Fields |
| --- | --- |
| `AudioPayload` | `audio` (ndarray), `sampling_rate`, `audio_format`, `channels`, `start`, `end` |
| `ImagePayload` | `image` (ndarray), `width`, `height`, `depth`, `pixel_format`, `timestamp` |
| `VideoPayload` | `video` (list of `ImagePayload`), `frames_per_second`, `codec`, `start`, `end` |
| `BytesPayload` | `cnt` (bytes) |
| `ObjectPayload` | an immutable `dict`, built with `ObjectPayload.from_dict(...)` |
| `Batch` | `messages` (tuple of messages) |
| `BasePayload` | the base class, as a placeholder type |

Build a frozen payload step by step with a `Draft`:

```python
draft = Draft(AudioPayload, copy_from=message.payload)
draft.audio = resampled
draft.sampling_rate = 16000

payload = draft.compile()
```

`ObjectPayload` rejects item assignment, and only its top level is frozen.
Copy every level you change:

```python
payload = dict(message.payload)
content = dict(payload['content'])
content['transcript'] = str(transcript_path)
payload['content'] = content

new_payload = ObjectPayload.from_dict(payload)
```

Treat arrays and nested values in a received payload as read-only: other
nodes may hold the same objects.

## Source nodes

A source produces data without upstream links. Use one of two patterns.

Let Juturna call a function in a loop:

```python
def warmup(self):
    self.set_source(self._read_next, by=0.5, mode='post')

def _read_next(self) -> Message[ObjectPayload] | None:
    record = self._client.poll()

    if record is None:
        return None

    return Message[ObjectPayload](
        creator=self.name,
        version=record.version,
        payload=ObjectPayload.from_dict(record.data),
    )

def update(self, message, **kwargs):
    self.transmit(message)
```

- The function returns a `Message`, or `None` when there is nothing to send.
  Its result goes through the node's own queue to `update()`.
- `by` is the interval in seconds; with `by=0` the function is called again
  immediately, so it must block until data arrives. `mode='pre'` waits
  before each call, `'post'` after it.
- Exceptions in the function are logged, counted, and retried with an
  exponential backoff capped by `JUTURNA_SOURCE_MAX_BACKOFF`.

Or run your own worker (a server, a socket reader) and feed the node with
`self.put(message)`. Create the worker in `warmup()` with
`self._transport.spawn(target=..., name=f'{self.name}_reader', daemon=True)`,
start it in `start()`, and stop it in `stop()` with an event and a join
timeout.

## Multiple inputs

A node with several upstream nodes receives their messages through a
synchroniser. The default, `passthrough`, delivers every message as soon as
it arrives. To deliver messages from several inputs together, define
`next_batch`:

```python
def next_batch(self, sources: dict[str, list[Message]]) -> dict[str, list[int]]:
    if not all(sources.get(name) for name in ('audio', 'video')):
        return dict()

    return {'audio': [0], 'video': [0]}
```

- `sources` maps each message `creator` to its pending messages, and the
  result maps creators to the indexes to deliver. The creator is the
  upstream node's name when that node builds its messages with
  `creator=self.name`, so do that in every node you write.
- When one message is released, `update()` receives it as is. When several
  are, it receives a `Message[Batch]` whose `payload.messages` holds them.
- A node that defines `next_batch` uses it by default. A pipeline can force
  the default with `"sync": "passthrough"` on the node entry.

## State

`update()` receives `kwargs['state']`, a dict-like `State` that the pipeline
keeps for the node across messages. Use it for per-node data you want the
pipeline to own; use private attributes for everything else.

## Remote nodes

A pipeline can run a node on another host by marking it `"warped": true`
with a `warp_configuration`: the pipeline replaces it with a `Warp` node that
talks to a server started with `juturna remotize`. The remote host needs the
same node package installed. Nodes that keep all their I/O in the lifecycle
hooks, and only exchange data through messages, work remotely unchanged.
