---
title: Middleware
layout: default
nav_order: 7
---

# Middleware

Middleware runs before your route handler, letting you inspect or modify the request, short-circuit with a response, or run code after the handler completes.

## Writing Middleware

A middleware is any class with a `handle(c, next)` method:

```js
// app/middleware/admin-only.js
export default class AdminOnlyMiddleware {
  async handle(c, next) {
    const user = c.get('user')
    if (!user?.isAdmin) {
      return c.json({ error: 'Forbidden' }, 403)
    }
    await next()
  }
}
```

Call `await next()` to pass control to the next middleware or handler. Return a response early to short-circuit the chain.

You can also run code after the handler:

```js
export default class TimingMiddleware {
  async handle(c, next) {
    const start = Date.now()
    await next()
    c.header('X-Response-Time', `${Date.now() - start}ms`)
  }
}
```

Plain functions also work as middleware:

```js
const logRequest = async (c, next) => {
  console.log(`${c.req.method} ${c.req.url}`)
  await next()
}
```

## Applying Middleware

### On a controller

Use `static middleware` to apply middleware to all actions on a controller:

```js
import { Controller } from 'foobarjs/core'
import { RequireAuthMiddleware } from 'foobarjs/auth'

class OrdersController extends Controller {
  static middleware = [RequireAuthMiddleware]

  index() { /* ... */ }
  store() { /* ... */ }
}
```

Limit to specific actions with `only` or `except`:

```js
class PostsController extends Controller {
  static middleware = {
    use: [RequireAuthMiddleware],
    only: ['store', 'update', 'destroy'],
  }

  index() { /* public */ }
  show() { /* public */ }
  store() { /* auth required */ }
}
```

```js
class ArticlesController extends Controller {
  static middleware = {
    use: [RequireAuthMiddleware],
    except: ['index', 'show'],
  }
}
```

### On a route

Pass middleware between the path and handler:

```js
// routes/web.js
import { RequireAuthMiddleware } from 'foobarjs/auth'
import AdminOnlyMiddleware from '../app/middleware/admin-only.js'

export default function (router) {
  router.get('/dashboard', RequireAuthMiddleware, (c) => {
    return c.json({ user: c.get('user') })
  })

  router.get('/admin/stats', RequireAuthMiddleware, AdminOnlyMiddleware, (c) => {
    return c.json({ stats: true })
  })
}
```

### On a group of routes

Use `router.group()` to apply middleware to multiple routes at once:

```js
import { RequireAuthMiddleware } from 'foobarjs/auth'

export default function (router) {
  // Public
  router.get('/health', (c) => c.json({ status: 'ok' }))

  // Protected
  router.group({ middleware: [RequireAuthMiddleware] }, (router) => {
    router.get('/dashboard', DashboardController, 'index')
    router.get('/settings', SettingsController, 'index')
    router.resource('orders', OrdersController)
  })
}
```

Groups support `prefix` to namespace URLs:

```js
router.group({ prefix: '/api/v1', middleware: [RequireAuthMiddleware] }, (router) => {
  router.get('/users', UsersController, 'index')    // → GET /api/v1/users
  router.get('/posts', PostsController, 'index')    // → GET /api/v1/posts
})
```

Groups can be nested — prefixes concatenate and middleware stacks:

```js
router.group({ prefix: '/api', middleware: [RequireAuthMiddleware] }, (router) => {
  router.group({ prefix: '/v1' }, (router) => {
    router.get('/items', ItemsController, 'index')  // → GET /api/v1/items
  })
})
```

## Middleware Stacking Order

When multiple middleware sources apply to a route, they run in this order:

1. **Global middleware** (session, auth loading, CSRF, CORS — applied by the framework)
2. **Group middleware** (from `router.group()`, outer groups first)
3. **Controller middleware** (from `static middleware`)
4. **Inline middleware** (passed directly on the route)

## Built-in Middleware

These are applied globally by the framework and plugins:

| Middleware | Source | Description |
|------------|--------|-------------|
| Session | `foobarjs/auth` plugin | Reads/writes signed session cookies |
| Auth loader | `foobarjs/auth` plugin | Resolves the current user from session or Bearer token |
| CSRF | Framework core | Validates CSRF tokens on state-changing requests |
| CORS | Framework core | Cross-origin resource sharing headers |
| Rate limiting | Framework core | Request throttling (configurable in `config/security.js`) |
| Secure headers | Framework core | Security-related HTTP headers |

These are available for you to apply where needed:

| Class | Import | Description |
|-------|--------|-------------|
| `RequireAuthMiddleware` | `foobarjs/auth` | Rejects unauthenticated requests (redirect or 401) |
| `GuestMiddleware` | `foobarjs/auth` | Redirects authenticated users away |

## File Convention

| Pattern | Location |
|---------|----------|
| `*.js` | `app/middleware/` |

Place your middleware files in `app/middleware/` and import them where needed.

## Next Steps

- Apply middleware to routes: [Routing](./routing.md)
- Apply middleware to controllers: [Controllers](./controllers.md)
- Auth middleware details: [Authentication](./authentication.md)
