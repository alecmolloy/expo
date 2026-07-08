# Structured Event Logging (`2g`)

Expo CLI emits structured JSONL events through [`2g`](https://github.com/kitten/2g), a
low-overhead session logger that automated tooling and agents can discover, replay, tail,
and export. The CLI uses the library directly — there is no in-tree wrapper. This doc covers
how we define and emit events; to _read_ sessions, run `2g --help`, which is self-describing.

## Activation

`installEventLogger()` runs once per process. `src/index.ts` calls it early (so `LOG_EVENTS`
and child-process IPC are honored before any output), and most commands (`start`, `serve`,
`export`, `run:*`) call `installEventLogger({ command, version })` to open a bounded session
under the system temp dir. The first activated destination wins; later calls are no-ops.

`LOG_EVENTS` overrides the destination:

```bash
LOG_EVENTS=events.jsonl npx expo start    # write to a file
LOG_EVENTS=1 npx expo start               # write to stdout (console → stderr)
LOG_EVENTS=2 npx expo start               # write to stderr (console → stdout)
```

## Defining events

Create a logger with `events(category)` and declare its payloads by augmenting `2g`'s
`EventRegistry` via declaration merging. Keys are fully-qualified `category:event_name`
strings; there is no central registry file.

```ts
import { events } from '2g';

declare module '2g' {
  interface EventRegistry {
    'my_module:something_started': { platform: string };
    'my_module:something_finished': { platform: string; duration: number };
  }
}

export const event = events('my_module');
```

Payload fields must not use the reserved wire keys `_e`, `_t`, `_d`, `_l`, or `_w`.

## Emitting events

Event names and payloads are type-checked against the merged registry. When the logger is
inactive, `event()` is a cheap no-op.

```ts
event('something_started', { platform: 'ios' });
event('something_started', { wrong: true }); // TS error
```

Use `event.span()` for a start/end pair with a measured duration (recorded as `_d`, ms):

```ts
const done = event.span('something_started', { platform: 'ios' });
done('something_finished', { platform: 'ios', duration: 500 });
```

### `events` vs `events.debug`

`events.debug(category)` creates a logger for chatty, debug-level events (marked `_l: 1`).
Session output drops them unless `LOG_DEBUG` is set (or `installEventLogger({ debug: true })`),
and `2g tap`/`export` skip them unless passed `--debug`. `LOG_DEBUG=metro:*` also mirrors
matching events to stderr in a readable form. Use `events()` for events worth keeping in the
bounded history; use `events.debug()` for high-volume diagnostics.

## Deferred payload helpers

`event.path(absolutePath)` and `event.error(error)` return `Serialized<T>` wrappers
(`{ toJSON(): T }`) that only do their work when an event is actually written, so inactive
loggers skip the cost. Payloads accept `Serialized<T>` wherever the declared type expects `T`.

- `event.path(p)` — logs a path relative to the log target (used across the CLI, e.g.
  `event('config', { serverRoot: event.path(serverRoot) })`).
- `event.error(err)` — serializes an error to `{ name, message, code, stack, cause }`, with
  cause chains resolved recursively.

## Output format

Each event is one JSON line:

```jsonl
{"_e":"my_module:something_started","_t":1713000000000,"platform":"ios"}
{"_e":"my_module:something_finished","_t":1713000000500,"platform":"ios","duration":500,"_d":500}
```

- `_e` — fully qualified event name (`category:event_name`)
- `_t` — wall-clock timestamp (ms) for cross-process correlation
- `_d` — span duration (ms), only on span-end events
- `_l` — log level, only on debug events (`1`)
- `_w` — worker id, only on events from workers/child processes

## Inspecting logs

The `2g` CLI discovers and reads sessions; its `--help` is self-describing (selectors,
filters, formats). The essentials:

```sh
2g ps -a                                       # sessions running now
2g tap "expo start" --tail                     # replay + follow live events
2g tap "expo start" --filter metro:bundling    # narrow to one area
2g export "expo start" -o trace.json           # export a Chrome trace
```

Selectors match by substring of PID, CWD, or command; if several match and exactly one is
running, it wins, otherwise `2g` errors and lists candidates (use a PID to disambiguate).
`--filter` matches event-name prefixes on whole segments (`metro:bundling` matches
`metro:bundling:started`); discover names by tapping unfiltered or via `2g typegen`.

## Testing

Capture a subprocess's events with `captureEvents` from `2g/api`, which hands the child a
pipe as its `LOG_EVENTS` target:

```ts
import { spawn } from 'node:child_process';
import { captureEvents } from '2g/api';

const capture = captureEvents({ filter: 'metro:*' });
const child = spawn('expo', ['export'], capture.spawnOptions({ env: process.env }));
const events = await capture.attach(child).collect();
```

Events are flushed on natural exit; child code that calls `process.exit()` should
`await flushEventLogger()` first, or trailing events may be lost.

## Reducing terminal noise

`src/utils/interactive.ts` exposes `shouldReduceLogs()` — true when the logger is active and
`EXPO_UNSTABLE_HEADLESS` is set — used to quiet interactive/noisy terminal output in favor of
the event log. It also backs `isInteractive()`.
