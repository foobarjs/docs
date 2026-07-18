{% raw %}
# Routing

foobarjs supports two ways to register routes:

1. **Explicit** — declare routes in `routes/web.js`.
2. **Convention** — drop a file in `app/controllers/` and REST routes are
   auto-registered from its filename.

Both work in the same app. Use the explicit form for routes that don't fit
the REST pattern (root path, inline callbacks, custom paths); rely on the
convention for standard resource controllers.

## Explicit routes — `routes/web.js`

Create `routes/web.js` and export a function that receives the `router`:

```js
// routes/web.js
import HomeController from '../app/controllers/home.controller.js'
import ProductsController from '../app/controllers/products.controller.js'

export default function (router) {
  // Root path.
  router.get('/', HomeController, 'index')

  // A full REST resource — the same seven routes the filename convention
  // would produce, but declared explicitly.
  router.resource('/products', ProductsController)

  // Inline callback — no controller class needed.
  router.get('/health', (c) => c.json({ status: 'ok' }))

  // Any HTTP verb.
  router.post('/webhooks/stripe', StripeWebhookController, 'handle')
}
```

### Router API

```js
router.get(path, ControllerClass, 'action')   // controller + method
router.get(path, (c) => ...)                  // inline callback
router.post(path, ...)
router.put(path, ...)
router.patch(path, ...)
router.delete(path, ...)
router.resource(path, ControllerClass)        // full REST resource
```

For each verb, the two-arg form `(path, callback)` and the three-arg form
`(path, ControllerClass, 'actionName')` both work.

You can also pass middleware classes or instances between the path and handler:

```js
import { RequireAuthMiddleware } from 'foobarjs/auth'

router.get('/dashboard', RequireAuthMiddleware, (c) => {
  return c.json({ user: c.get('user') })
})
```

Any class with a `handle(c, next)` method works as middleware — pass the class itself (it will be instantiated automatically) or an instance.

### Fluent middleware API

Route methods return a chainable object with `.withoutMiddleware()` and `.middleware()`:

```js
import RateLimit from '../app/middlewares/RateLimit.js'

// Opt out of auto-applied middleware
router.post('/webhook/stripe', WebhookController, 'handle')
  .withoutMiddleware(['Csrf'])

// Add a factory middleware
router.get('/dashboard', DashboardController, 'index')
  .middleware([RateLimit({ max: 100 })])
  .withoutMiddleware(['Analytics'])
```

Pass middleware names (strings, derived from filenames) or imported references.

### `router.resource(path, ControllerClass)`

Registers up to seven routes, but only for actions the controller actually
defines. Missing methods produce no route.

| Verb | URL | Action |
|------|-----|--------|
| GET | `/base` | `index` |
| GET | `/base/new` | `new` |
| POST | `/base` | `store` |
| GET | `/base/:id` | `show` |
| GET | `/base/:id/edit` | `edit` |
| PUT | `/base/:id` | `update` |
| DELETE | `/base/:id` | `destroy` |

Also available: `routes/api.js`, loaded the same way. Convention is to use it
for JSON API routes, e.g. `router.get('/api/v1/status', ...)`, but the file
just runs — put whatever routes you like in it.

### `router.group(options, callback)`

Group routes that share a prefix, middleware, or both:

```js
import { RequireAuthMiddleware } from 'foobarjs/auth'
import DashboardController from '../app/controllers/dashboard.controller.js'
import SettingsController from '../app/controllers/settings.controller.js'

export default function (router) {
  // Public routes
  router.get('/health', (c) => c.json({ status: 'ok' }))

  // Protected routes — RequireAuthMiddleware applies to everything inside
  router.group({ middleware: [RequireAuthMiddleware] }, (router) => {
    router.get('/dashboard', DashboardController, 'index')
    router.get('/settings', SettingsController, 'index')
    router.post('/settings', SettingsController, 'update')
  })
}
```

#### Prefix

```js
router.group({ prefix: '/api/v1' }, (router) => {
  router.get('/users', UsersController, 'index')    // → GET /api/v1/users
  router.get('/posts', PostsController, 'index')    // → GET /api/v1/posts
})
```

#### Middleware + Prefix

```js
router.group({ prefix: '/api', middleware: [RequireAuthMiddleware] }, (router) => {
  router.resource('orders', OrdersController)       // → /api/orders/*
})
```

#### Nesting

Groups can be nested. Prefixes concatenate and middleware stacks:

```js
router.group({ prefix: '/api', middleware: [RequireAuthMiddleware] }, (router) => {
  router.group({ prefix: '/v1', middleware: [RateLimitMiddleware] }, (router) => {
    router.get('/items', ItemsController, 'index')  // → GET /api/v1/items
    // Both RequireAuthMiddleware and RateLimitMiddleware run
  })
})
```

Middleware only applies inside the group — routes defined outside are unaffected.

Groups also support fluent `.withoutMiddleware()` and `.middleware()`:

```js
router.group({ prefix: '/webhooks' }, (r) => {
  r.post('/stripe', WebhookController, 'stripe')
  r.post('/github', WebhookController, 'github')
}).withoutMiddleware(['Csrf'])
```

## Convention routes

Any controller in `app/controllers/` is auto-mounted as a REST resource at
`/<filename-without-suffix>`. Only methods you actually define get routes.

Given `app/controllers/products.controller.js`:

| HTTP | URL | Method (must exist on controller) |
|------|-----|------|
| GET | `/products` | `index` |
| GET | `/products/new` | `new` |
| POST | `/products` | `store` |
| GET | `/products/:id` | `show` |
| GET | `/products/:id/edit` | `edit` |
| PUT | `/products/:id` | `update` |
| DELETE | `/products/:id` | `destroy` |

The `home.controller.js` filename is a special case: its `index()` also
mounts at `/`. This is a shortcut — declaring `router.get('/', HomeController, 'index')`
in `routes/web.js` is equivalent and clearer.

Sub-folders work: `app/controllers/admin/orders.controller.js` mounts at
`/admin/orders`.

## Route parameters

Access via `this.params()` (Controller) or `c.req.param('name')`:

```js
async show() {
  const id = this.params().id
  const product = await Product.find(id)
  return this.render('products/show', { product })
}
```

## Form method spoofing

HTML forms only support `GET` and `POST`. Use a hidden `_method` field for
the other verbs:

```html
<form action="/products/{{ product.id }}" method="POST">
  <input type="hidden" name="_method" value="PUT">
  <!-- ... -->
</form>
```

The method-override middleware handles this transparently.

## Raw router access

For anything the router API doesn't cover, drop down to the underlying router:

```js
// Inside a plugin's register(foobar) or from routes/web.js if you accept
// the second argument:
export default function (router, foobar) {
  foobar.app.get('/exotic/*', (c) => { /* ... */ })
}
```

## Building URLs — the `Uri` helper

`Uri` is a fluent, immutable URL builder — a fluent, immutable URL builder.
Use it whenever you're constructing a link that preserves or mutates the current
query string (sortable table headers, filter chips, pagination, tab links):

```js
import { Uri } from 'foobarjs'

// Build from scratch
Uri.of('/admin/products').withQuery({ sort: 'name', order: 'asc' }).toString()
// → "/admin/products?sort=name&order=asc"

// Mutate the current request's URL
const url = Uri.current(c)
  .withQuery({ page: 2 })
  .without('f_reset')
  .toString()
```

### API

| Method | Purpose |
|--------|---------|
| `Uri.of(input)` | Parse a string, `URL`, or another `Uri`. Relative paths stay relative. |
| `Uri.current(c)` | Convenience for `Uri.of(c.req.url)`. |
| `.withQuery({ k: v, ... })` | Merge params. `undefined`, `null`, or `''` **remove** the key. |
| `.withQueryIf(cond, params)` | Merge only when `cond` is truthy. |
| `.without('a', 'b')` | Remove one or more keys (varargs or an array). |
| `.replaceQuery({ ... })` | Clear the entire query, then apply the given params. |
| `.fragment(value)` | Set or clear the hash. |
| `.query()` | Return the query as a plain object. |
| `.searchParams()` | Return a fresh `URLSearchParams` copy (mutations don't leak). |
| `.path()` | Path portion only. |
| `.hasQuery(key)` | Is the key present? |
| `.toString()` / `.toJSON()` | Serialize. |

Every mutation returns a **new** instance — safe to share, safe to chain.

```js
// Preserve every existing query param, just bump the page:
Uri.current(c).withQuery({ page: nextPage }).toString()

// Replace one filter, clear another, and reset the page counter:
Uri.current(c)
  .withQuery({ 'f[status]': 'open', page: undefined })
  .without('f_reset')
  .toString()
```

## Next steps

- Write the controllers your routes point at: [Controllers](./controllers.md)
- Render responses: [Views](./views.md)
- Auto-generated REST API for models: [API](./api.md)
{% endraw %}
