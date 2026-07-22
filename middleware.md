[← Back to docs](./README.md)

{% raw %}
---
title: Middleware
layout: default
nav_order: 7
---

# Middleware

Middleware runs before your route handler, letting you inspect or modify the request, short-circuit with a response, share data with views, or run code after the handler completes.

## What middleware does

Middleware serves three jobs:

| Job | Pattern | Example |
|-----|---------|---------|
| **Guard** | Check a condition, abort or redirect if it fails | Auth gate, role check, IP allowlist |
| **Share** | Enrich the request with data for views | `c.share('cartCount', n)` — available in JSX via `useView()` |
| **Transform** | Modify the response on the way out | Cache headers, compression, timing |

## Writing middleware

### Async function (recommended)

The simplest form — an `async function` that receives the context and a `next` callback:

```js
// app/middlewares/CartShare.js
export default async function CartShare(c, next) {
  const session = c.get('session')
  const cart = session?.get('cart') || []
  const cartCount = cart.reduce((sum, item) => sum + (item.quantity || 0), 0)
  c.share('cartCount', cartCount)
  await next()
}
```

Call `await next()` to pass control to the next middleware or handler. Return a response early to short-circuit the chain.

### Class with `handle()`

A class with a `handle(c, next)` method also works as middleware:

```js
// app/middlewares/AdminOnly.js
export default class AdminOnly {
  async handle(c, next) {
    const user = c.get('user')
    if (!user?.isAdmin) {
      return c.json({ error: 'Forbidden' }, 403)
    }
    await next()
  }
}
```

### Running code after the handler

```js
export default async function Timing(c, next) {
  const start = Date.now()
  await next()
  c.header('X-Response-Time', `${Date.now() - start}ms`)
}
```

### Factory middleware (configurable)

When middleware needs configuration, export a regular (sync) function that returns the actual middleware:

```js
// app/middlewares/RateLimit.js
export default function RateLimit(options = {}) {
  const max = options.max || 100
  return async (c, next) => {
    // rate limiting logic using `max`
    await next()
  }
}
```

Factories are **not auto-applied** — they must be used explicitly in routes or groups (see [Applying middleware](#applying-middleware) below). The framework detects factories by checking if the export is a regular (non-async) function. If you accidentally write a sync middleware that should be auto-applied, the framework logs a warning.

## Sharing data with views

Use `c.share(key, value)` in middleware to pass data to all views rendered during the request. Shared data is available in JSX views via `useView()`.

```js
// app/middlewares/CartShare.js
export default async function CartShare(c, next) {
  const session = c.get('session')
  const cart = session?.get('cart') || []
  c.share('cartCount', cart.reduce((sum, item) => sum + (item.quantity || 0), 0))
  await next()
}
```

In JSX:
```jsx
import { useView } from 'foobarjs/jsx'

function Header() {
  const { cartCount } = useView()
  return <a href="/cart">Cart ({cartCount})</a>
}
```

Controllers can also share data directly:

```js
class DashboardController extends Controller {
  async index() {
    this.share('pageTitle', 'Dashboard')
    return this.render('dashboard/index')
  }
}
```

## Middleware folder convention

Where middleware lives determines its scope:

```
app/middlewares/
  CartShare.js          ← auto-applied to every request
  Analytics.js          ← auto-applied to every request
  RateLimit.js          ← factory — NOT auto-applied, use explicitly
app/middlewares/web/
  ThemeLoader.js        ← auto-applied to web routes only
app/middlewares/api/
  ApiVersion.js         ← auto-applied to API routes only
```

| Location | Scope |
|----------|-------|
| `app/middlewares/*.js` | Every request (global) |
| `app/middlewares/web/*.js` | Web routes (`routes/web.js` + convention routes) |
| `app/middlewares/api/*.js` | API routes (`routes/api.js`) |

Middleware runs in alphabetical order within each folder. Prefix with `01-`, `02-` if you need explicit ordering. Only `.js` files directly in the folder are loaded — subdirectories other than `web/` and `api/` are ignored.

### Auto-apply rules

| Export type | Auto-applied? | Why |
|-------------|---------------|-----|
| `async function` | Yes | Middleware must be async (it calls `await next()`) |
| Class with `handle()` | Yes | Instantiated, `handle(c, next)` is called |
| Class with `handle()` + `static global = false` | **No** — aliased only | For opt-in middleware referenced explicitly on controllers/routes |
| Regular `function` | No — treated as factory | Sync functions return configured middleware |

### Aliased-only middleware (`static global = false`)

Set `static global = false` on a class-based middleware to keep it out of the
global auto-apply chain while still registering its filename alias for
explicit use. This is the right shape for "only run on some routes"
middleware:

```js
// app/middlewares/RequireAttendee.js
import Attendee from '../models/attendee.model.js'

class RequireAttendee {
  static global = false     // do not auto-apply globally

  async handle(c, next) {
    if (!(c.get('user') instanceof Attendee)) return c.redirect('/tickets')
    return next()
  }
}
export default RequireAttendee
```

```js
// In a controller — reference by filename alias:
static middleware = { use: ['auth', 'RequireAttendee'], only: ['my', 'edit'] }
```

Without `static global = false`, `RequireAttendee` would run on *every*
request and redirect non-attendees away — the opposite of what you want.

## Applying middleware

### On a route (inline)

Pass middleware between the path and handler:

```js
// routes/web.js
export default function (router) {
  router.get('/dashboard', 'auth', (c) => {
    return c.json({ user: c.get('user') })
  })
}
```

### On a route (fluent)

Route methods return a chainable object. Use `.middleware()` to add and `.withoutMiddleware()` to remove:

```js
import RateLimit from '../app/middlewares/RateLimit.js'

router.post('/webhook/stripe', WebhookController, 'handle')
  .withoutMiddleware(['Csrf'])

router.get('/dashboard', DashboardController, 'index')
  .middleware([RateLimit({ max: 100 })])
```

### On a group of routes

Use `router.group()` to apply middleware to multiple routes:

```js
export default function (router) {
  router.get('/health', (c) => c.json({ status: 'ok' }))

  router.group({ middleware: ['auth'] }, (router) => {
    router.get('/dashboard', DashboardController, 'index')
    router.resource('orders', OrdersController)
  })
}
```

Groups also support fluent `.withoutMiddleware()` and `.middleware()`:

```js
router.group({ prefix: '/webhooks' }, (r) => {
  r.post('/stripe', WebhookController, 'stripe')
  r.post('/github', WebhookController, 'github')
}).withoutMiddleware(['Csrf'])

router.group({ prefix: '/api/v2' }, (r) => {
  r.resource('items', ItemsController)
}).withoutMiddleware(['Csrf']).middleware([RateLimit({ max: 50 })])
```

### On a controller

Use `static middleware` to apply middleware to all actions:

```js
import { Controller } from 'foobarjs/core'

class OrdersController extends Controller {
  static middleware = ['auth']

  index() { /* ... */ }
  store() { /* ... */ }
}
```

Limit to specific actions with `only` or `except`:

```js
class PostsController extends Controller {
  static middleware = {
    use: ['auth'],
    only: ['store', 'update', 'destroy'],
  }
}
```

## Opting out of middleware

Use `static withoutMiddleware` on a controller to skip auto-applied middleware by name or reference:

```js
// By name (derived from filename)
class WebhookController extends Controller {
  static withoutMiddleware = ['Csrf', 'CartShare']
}

// By reference (imported function/class)
import CartShare from '../app/middlewares/CartShare.js'

class ApiController extends Controller {
  static withoutMiddleware = [CartShare]
}
```

Route-level and group-level opt-out use the fluent `.withoutMiddleware()` method (shown above).

Opt-outs from all three sources (controller, group, route) are combined — if any source opts out of a middleware, it won't run.

## Middleware stacking order

When multiple middleware sources apply to a route, they run in this order:

1. **Framework middleware** (session, auth loading, CSRF, CORS — applied by the framework core)
2. **Auto-discovered user middleware** (from `app/middlewares/` folders, filtered by opt-outs)
3. **Group middleware** (from `router.group()`, outer groups first)
4. **Controller middleware** (from `static middleware`)
5. **Inline middleware** (passed directly on the route)
6. **Fluent `.middleware()`** additions

## Built-in middleware

Applied globally by the framework:

| Middleware | Source | Description |
|------------|--------|-------------|
| Session | `foobarjs/auth` plugin | Reads/writes signed session cookies |
| Auth loader | `foobarjs/auth` plugin | Resolves the current user from session or Bearer token |
| CSRF | Framework core | Validates CSRF tokens on state-changing requests |
| CORS | Framework core | Cross-origin resource sharing headers |
| Rate limiting | Framework core | Request throttling (configurable in `config/security.js`) |
| Secure headers | Framework core | Security-related HTTP headers |

Available for explicit use:

| Class | Import | Description |
|-------|--------|-------------|
| `RequireAuthMiddleware` (alias: `'auth'`) | `foobarjs/auth` | Rejects unauthenticated requests (redirect or 401) |
| `GuestMiddleware` | `foobarjs/auth` | Redirects authenticated users away |

## Middleware aliasing

Middleware can be referenced by string alias instead of importing the class directly.

### Built-in aliases

| Alias | Class |
|-------|-------|
| `'auth'` | `RequireAuthMiddleware` |

The `'auth'` alias is registered at boot regardless of `auth.guard` mode. Use
it on a controller (`static middleware = ['auth']`), on a route
(`.middleware(['auth'])`), or in a group — it resolves consistently whether
your app is auth-first or public-by-default.

### Filename-based aliases

Middleware files in `app/middlewares/` are automatically aliased by their filename stem:

```
app/middlewares/
  RateLimit.js    → 'RateLimit'
  LogRequest.js   → 'LogRequest'
```

### Usage

String aliases work everywhere middleware is accepted:

```js
// Controller
static middleware = ['auth']

// Route
router.get('/dashboard', DashboardController, 'index').middleware(['auth'])

// Opt out
router.get('/pricing', PagesController, 'pricing').withoutMiddleware(['auth'])

// Group
router.group({ middleware: ['auth'] }, (r) => {
  // ...
})
```

## The request context (`c`)

`c` is Hono's `Context` object — the per-request container passed to every middleware and route handler. Controllers access it as `this.c`.

### Core API

| Method | Description |
|--------|-------------|
| `c.req` | The incoming request object |
| `c.req.param('name')` | Route parameter |
| `c.req.query('key')` | Query string parameter |
| `c.req.header('name')` | Request header |
| `c.json(data, status?)` | Return a JSON response |
| `c.html(html)` | Return an HTML response |
| `c.text(body, status?)` | Return a plain text response |
| `c.redirect(url, status?)` | Return a redirect response |
| `c.header(name, value)` | Set a response header |
| `c.set(key, value)` | Store a value for this request |
| `c.get(key)` | Retrieve a stored value |
| `c.share(key, value)` | Share a value with all views rendered during this request |

### Framework-decorated values

These are set by framework middleware and available via `c.get()`:

| Key | Type | Set by |
|-----|------|--------|
| `session` | Session object | Auth plugin |
| `user` | User model instance or `null` | Auth plugin |
| `loggedIn` | `boolean` | Auth plugin |
| `configLoader` | ConfigLoader instance | Framework core |
| `render` | `async (template, data) => html` | View engine |
| `viewEngine` | View engine instance | View engine |

### Controller convenience methods

The `Controller` base class wraps `c` with convenience methods so you rarely need `c` directly:

| Controller method | Equivalent `c` call |
|-------------------|---------------------|
| `this.render(template, data)` | `c.render(template, data)` |
| `this.json(data)` | `c.json(data)` |
| `this.redirect(path)` | Returns a `RedirectResponse` builder |
| `this.share(key, value)` | `c.share(key, value)` |
| `this.flash(key, message)` | `c.get('session').flash(key, message)` |
| `this.params()` | `c.req.param()` |
| `this.query()` | `c.req.query()` |
| `this.body` | Parsed request body (JSON or form) |
| `this.config(key)` | `c.get('configLoader').get(key)` |
| `this.validate(Class)` | `c.validate(Class)` |

## Next steps

- Apply middleware to routes: [Routing](./routing.md)
- Apply middleware to controllers: [Controllers](./controllers.md)
- Auth middleware details: [Authentication](./authentication.md)
{% endraw %}
