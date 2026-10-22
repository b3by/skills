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
| `update(message, **kwargs)` | for each message from any upstream node, one at a time | process input and `transmit()` results |
| `set_on_config(prop, value)` | on `pipeline.update_node(...)` | change a setting at runtime |
| `stop()` | on `pipeline.stop()` | call `super().stop()`, signal your workers and join them with a timeout |
| `destroy()` | on `pipeline.destroy()` | close connections, free models and other resources |
| `configuration` (property) | by the service and telemetry | extend `super().configuration` with the settings worth reporting |

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
| `Batch` | `messages` (tuple of messages), for a node that transmits several messages as one |
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

## Multiple inputs and state

`update()` receives the messages of every upstream node one at a time, in
the order they reach the node's queue; `message.creator` tells which node
sent each one. There is no buffer or synchroniser. A node that needs
messages from several inputs before producing an output keeps the pending
ones in its state and removes them once they are used:

```python
class Joiner(Node[ObjectPayload, ObjectPayload]):
    """Pair the n-th message of the audio and video inputs"""

    _INPUTS = ('audio', 'video')

    def update(self, message: Message[ObjectPayload], **kwargs):
        """Wait until every input has a message, then transmit a pair"""
        pending = kwargs['state'].setdefault('pending', dict())
        pending.setdefault(message.creator, list()).append(message)

        if not all(pending.get(name) for name in self._INPUTS):
            return

        first = {name: pending[name].pop(0) for name in self._INPUTS}
        to_send = Message[ObjectPayload](
            creator=self.name,
            version=first['audio'].version,
            payload=ObjectPayload.from_dict(
                {name: m.payload['value'] for name, m in first.items()}
            ),
            timers_from=first['audio'],
        )

        self.transmit(to_send)
```

- Bound what you keep: drop pending messages that are too old or too many,
  or the state grows without limit when one input stops.
- The creator is the upstream node's name when that node builds its
  messages with `creator=self.name`, so do that in every node you write.

`kwargs['state']` is a dict-like `State` that the pipeline creates for the
node at warmup and keeps for the whole run, even if the node itself is
stopped. Use it for data that belongs to
the stream (pending messages, counters, the last seen values); keep
settings, models, connections and other objects in private attributes. Keep
only plain values in it: `None`, booleans, numbers, strings, bytes, lists,
tuples, frozensets, dicts, numpy arrays and scalars, and messages. Those are
the types that reach a remote node.

## Remote nodes

A pipeline can run a node on another host by marking its entry
`"warped": true`, with a `warp_configuration`. The pipeline replaces it with
a `Warp` node (from the official collection, `warp` extra) that sends each
message to a service started on the remote host with:

```bash
python -m juturna remotize --node-name transcriber \
  --node-type contrib.my_org.nodes.Transcriber --port 45000
```

The remote host needs the same node package installed, and the
`warp_configuration` points `grpc_host` and `grpc_port` at the service. The
`sticky` key of the warped entry tells where the node state lives:

- `"sticky": true` (the default): the remote service keeps the state, so the
  warp node must always reach the same service.
- `"sticky": false`: the state travels with every request and comes back
  with the response, so any replica behind a load balancer can serve it.
  Keep the state small.

To make a node work remotely:

- Keep only plain values in its state, `meta` and `ObjectPayload` values
  (see above). Other values are left out with a warning, or fail the request
  when the service runs with `--strict`.
- Transmit at most one message per `update()`, from the update thread: the
  service answers with the first message transmitted, when `update()`
  returns. Later messages, and messages transmitted by the node's own
  workers, never reach the pipeline.
- Keep all I/O in the lifecycle hooks, so the node starts the same way
  locally and remotely.
