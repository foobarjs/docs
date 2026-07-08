# Logging

foobarjs ships with a lightweight `Logger` that writes structured logs to stdout (pretty in TTY, JSON when piped) and optionally to a file. There are no external dependencies.

## Configuration

```js
// config/app.js
export default {
  // ...
  log: {
    level: process.env.LOG_LEVEL || 'info',
    file: process.env.LOG_FILE || null,      // path to append JSON log lines
    pretty: undefined,                        // override auto-detect
    silent: false,                            // suppress stdout/stderr (file still written)
  },
}
```

`.env`:

```
LOG_LEVEL=info
LOG_FILE=storage/logs/app.log
```

## Levels

Ordered from highest severity to lowest:

- `error` — unexpected failures, 5xx responses
- `warn` — recoverable issues, 4xx responses
- `info` — normal operational messages (default)
- `debug` — verbose developer output

A log call is only emitted if its level is at or below the configured threshold.

## Using the Logger

```js
import { Logger } from 'foobarjs/core'

const log = Logger.instance()

log.info('Server started', { port: 3000 })
log.warn('Slow query', { sql, ms: 320 })
log.error(new Error('boom'), { requestId: 'abc' })
log.debug('Cache miss', { key: 'user:42' })
```

The second argument is any JSON-serialisable context object. When you pass an `Error` as the first argument, the logger extracts `name`, `message`, `stack`, `status`, and `code` into the entry's `error` sub-object.

## Entry Shape

```json
{
  "ts": "2026-01-01T12:00:00.000Z",
  "level": "error",
  "message": "boom",
  "requestId": "01H...",
  "error": {
    "name": "TypeError",
    "message": "boom",
    "stack": "TypeError: boom\n    at ..."
  }
}
```

Every entry has `ts`, `level`, and `message`. Context keys are merged at the top level.

## Automatic Error Logging

`ErrorHandler` calls `Logger.instance().error(...)` on every unhandled 5xx and `warn(...)` on every 4xx, with `requestId`, `method`, `url`, and (if a session is present) `userId`. There is nothing to configure to get this behaviour — just make sure `Logger` is initialised (it is, automatically, when `Foobar.boot()` runs).

## Process-Level Handlers

`Foobar.boot()` also installs:

```js
process.on('unhandledRejection', (reason) => Logger.instance().error(reason, { source: 'unhandledRejection' }))
process.on('uncaughtException', (err) => {
  Logger.instance().error(err, { source: 'uncaughtException' })
  if (process.env.NODE_ENV === 'production') process.exit(1)
})
```

so promise rejections and uncaught exceptions don't disappear silently.

## Custom Configuration

Reconfigure at any time:

```js
Logger.configure({ level: 'debug', file: '/var/log/app.log', pretty: false })
```

Call `Logger.reset()` in tests to clear the singleton.

## Silent Mode (Tests)

To suppress console output but still write to a file:

```js
Logger.configure({ silent: true, file: '/tmp/test.log' })
```

## Redirecting Framework Logs

Framework packages (auth, admin, api, etc.) call `Logger.instance()` directly, so once you configure the singleton every subsystem uses your settings automatically.
