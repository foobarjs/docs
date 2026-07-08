# Error Handling

foobarjs ships with a full error-handling pipeline: a central `ErrorHandler` catches every unhandled error, decides whether to render an HTML error page, a JSON response, or a redirect (for validation errors), and logs the incident.

## Debug Mode

The debug flag controls whether stack traces and the rich diagnostic page are exposed.

Resolution order (first non-null wins):
1. `config.app.debug` in `config/app.js`
2. `APP_DEBUG` environment variable (`true`/`1` enables)
3. `NODE_ENV !== 'production'`

```js
// config/app.js
export default {
  debug: process.env.APP_DEBUG === undefined
    ? process.env.NODE_ENV !== 'production'
    : (process.env.APP_DEBUG === 'true' || process.env.APP_DEBUG === '1'),
  // ...
}
```

When debug is on:
- HTML responses show the Whoops-style dev page (see below).
- JSON responses include the real error message and stack trace.

When debug is off (production):
- HTML responses render `app/views/errors/<status>.html` (if present) or a minimal generic page.
- JSON responses expose the status, request ID, and a redacted message; stack traces are never leaked.

## HTTP Exceptions

Throw one of the following to signal a specific HTTP outcome:

```js
import { HttpException, NotFoundError, ForbiddenError, AuthenticationError } from 'foobarjs/core'

throw new HttpException(418, "I'm a teapot")
throw new NotFoundError('Product does not exist')
throw new ForbiddenError()
throw new AuthenticationError()
```

All of them:
- Set `status` / `statusCode`
- Are rendered through the error views described below
- Are logged as warnings (4xx) or errors (5xx)

Custom subclasses work the same way — just set `status` on the error you throw.

## Error Views

If a request accepts HTML, the handler looks for a template in this order:

1. `app/views/errors/<status>.html` (e.g. `404.html`)
2. `app/views/errors/<class>xx.html` (e.g. `5xx.html` for any 500-series)
3. `app/views/errors/error.html`
4. `app/views/errors/500.html` (only for 5xx)
5. Built-in minimal fallback page

Every view receives:

```js
{
  status,        // the HTTP status
  message,       // publicly-safe message
  requestId,     // correlation ID
  error,         // the raw error (available in debug)
  stack,         // only in debug
}
```

Example `app/views/errors/404.html`:

```html
@layout('layouts.app')

@section('title'){{ status }} - Page Not Found@endsection

@section('content')
  <div class="error-page">
    <h1>{{ status }}</h1>
    <h2>{{ message || "Page Not Found" }}</h2>
    <p>The page you're looking for doesn't exist.</p>
    <a href="/">Go home</a>
    @if(requestId)
      <p style="font-family:monospace;font-size:11px;">Request ID: {{ requestId }}</p>
    @endif
  </div>
@endsection
```

## JSON Error Responses

If `Accept: application/json` (or the request has a JSON body), the handler returns:

```json
{
  "error": "Not Found",
  "status": 404,
  "requestId": "01H..."
}
```

Validation errors also include:

```json
{
  "error": "Validation failed",
  "status": 422,
  "errors": { "email": ["Email has already been taken"] },
  "requestId": "..."
}
```

In debug mode, the response additionally has:

```json
{
  "exception": "TypeError",
  "stack": ["at foo (...)", "at bar (...)"],
  "cause": "underlying error message"
}
```

## Validation Errors — HTML Redirect

For non-JSON requests where an error's `name === 'ValidationError'`, the handler:

1. Flashes `errors` and `old` (previously-submitted input) into the session.
2. Redirects to `Referer` (or `/` as a fallback).
3. Views can read them via the injected `errors` and `old()` globals and the `@error('field')...@enderror` directive.

## The Whoops-style Dev Page

When debug mode is on and no custom error view exists, the framework renders a rich diagnostic page with:

- Header with error class, message, status, and request ID.
- Left sidebar listing every stack frame (app frames highlighted in green, `node_modules` frames dimmed, node internals separated).
- Source code viewer showing the failing frame with 6 lines of context above and below. Click any frame in the sidebar to view its source.
- Tabbed request panel with method, URL, path, route params, query, headers, body, session, user, cookies, environment.

All values are HTML-escaped. Sensitive keys (`password`, `secret`, `token`, `authorization`, `cookie`, `api_key`, ...) are redacted before display.

## Logging

Every error is logged via the built-in `Logger`:

- 4xx errors → `warn`
- 5xx errors → `error`

Log entry shape:

```json
{
  "ts": "2026-01-01T12:00:00.000Z",
  "level": "error",
  "message": "Cannot read properties of undefined (reading 'x')",
  "requestId": "...",
  "status": 500,
  "method": "POST",
  "url": "http://localhost/foo",
  "userId": 42,
  "error": {
    "name": "TypeError",
    "message": "...",
    "stack": "..."
  }
}
```

See [Logging](./logging.md) for configuration.

## Process-Level Error Handlers

`Foobar.boot()` installs `process.on('unhandledRejection')` and `process.on('uncaughtException')` handlers that log via the same Logger, so no error goes unrecorded. In production, `uncaughtException` triggers `process.exit(1)` to avoid corrupt state (per Node's recommendation).

## Extending the Handler

Subclass `ErrorHandler` and swap it in during boot:

```js
import { ErrorHandler, Logger } from 'foobarjs/core'

class MyErrorHandler extends ErrorHandler {
  async handle(err, c) {
    if (err.name === 'CustomBillingError') {
      Logger.instance().error(err, { source: 'billing' })
      return c.json({ error: 'Payment problem', code: err.code }, 402)
    }
    return super.handle(err, c)
  }
}

// In your boot:
foobar.errorHandler = new MyErrorHandler(foobar.configLoader, foobar.basePath, {
  logger: Logger.instance(),
})
```

## Testing Errors

Throw exceptions from a controller and hit the endpoint from a test:

```js
import { test, assert } from 'foobarjs/test'
import { NotFoundError } from 'foobarjs/core'

// app/controllers/orders.controller.js
class OrdersController {
  async show(c) {
    const order = await Order.find(c.req.param('id'))
    if (!order) throw new NotFoundError('Order not found')
    return order
  }
}

// demo/test/orders.test.js
test('missing order returns 404', async ({ request }) => {
  const res = await request.get('/orders/99999')
  assert.strictEqual(res.status, 404)
})
```
