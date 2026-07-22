[← Back to docs](./README.md)

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
router.get('/dashboard', 'auth', (c) => {
  return c.json({ user: c.get('user') })
})
```

Any class with a `handle(c, next)` method works as middleware — pass the class itself (it will be instantiated automatically) or an instance.

### Fluent middleware API

Route methods return a chainable object with `.public()`, `.withoutMiddleware()`, and `.middleware()`:

```js
import RateLimit from '../app/middlewares/RateLimit.js'

// Mark a route as publicly accessible (no login required)
router.get('/pricing', PagesController, 'pricing').public()

// Opt out of auto-applied middleware
router.post('/webhook/stripe', WebhookController, 'handle')
  .withoutMiddleware(['Csrf'])

// Add a factory middleware
router.get('/dashboard', DashboardController, 'index')
  .middleware([RateLimit({ max: 100 })])
  .withoutMiddleware(['Analytics'])
```

`.public()` is sugar for `.withoutMiddleware(['auth'])`. Pass middleware names (strings, derived from filenames) or imported references.

### Route model binding

Use `.bind()` to automatically resolve a Model from a route parameter. The param name matches the model name (lowercase):

```js
import Order from '../app/models/order.model.js'

// :order param → Order.find(paramValue)
router.put('/orders/:order/refund', OrderController, 'refund')
  .bind(Order)
```

The resolved instance is available via `c.req.bound('order')` in closures or `this.bound('order')` in controllers. If the record isn't found, a `404 Not Found` is thrown automatically.

For `router.resource()`, pass the Model as the third argument to auto-bind on detail routes:

```js
router.resource('/orders', OrderController, Order)
```

Convention controllers can declare `static model` to auto-bind:

```js
class OrderController extends Controller {
  static model = Order

  async show() {
    const order = this.bound('order')  // already loaded
  }
}
```

### Gate authorization

Use `.can()` to require gate authorization on a route:

```js
router.get('/dashboard', DashboardController, 'index')
  .can('view', 'analytics')
```

Combine `.bind()` with `.can()` for per-item authorization — the gate receives the bound model instance:

```js
router.put('/orders/:order/refund', OrderController, 'refund')
  .bind(Order)
  .can('refund', Order)
```

Without `.bind()`, the gate only has the user (useful for user-level checks like standalone gates). With `.bind()`, the gate gets the loaded record for per-item checks.

> **Note:** `.can()` requires an authenticated user. If the request has no user (e.g. on a `.public()` route), foobarjs throws an `AuthenticationError` (HTTP 401).

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

Pass a Model class as the third argument to auto-bind on detail routes (`:id`):

```js
router.resource('/orders', OrderController, Order)
```

Also available: `routes/api.js`, loaded the same way. Convention is to use it
for JSON API routes, e.g. `router.get('/api/v1/status', ...)`, but the file
just runs — put whatever routes you like in it.

### `router.group(options, callback)`

Group routes that share a prefix, middleware, or both:

```js
import DashboardController from '../app/controllers/dashboard.controller.js'
import SettingsController from '../app/controllers/settings.controller.js'

export default function (router) {
  // Public routes
  router.get('/health', (c) => c.json({ status: 'ok' }))

  // Protected routes — 'auth' middleware applies to everything inside
  router.group({ middleware: ['auth'] }, (router) => {
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
router.group({ prefix: '/api', middleware: ['auth'] }, (router) => {
  router.resource('orders', OrdersController)       // → /api/orders/*
})
```

#### Nesting

Groups can be nested. Prefixes concatenate and middleware stacks:

```js
router.group({ prefix: '/api', middleware: ['auth'] }, (router) => {
  router.group({ prefix: '/v1', middleware: [RateLimitMiddleware] }, (router) => {
    router.get('/items', ItemsController, 'index')  // → GET /api/v1/items
    // Both 'auth' and RateLimitMiddleware run
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

Groups also support `.public()`:

```js
router.group({ prefix: '/docs' }, (r) => {
  r.get('/', DocsController, 'index')
  r.get('/:slug', DocsController, 'show')
}).public()
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

Convention routes follow the auth-first policy. To make a convention controller public, add `static auth = false`:

```js
class PagesController extends Controller {
  static auth = false
}
```

### Custom controller actions

Convention routing **only** mounts the seven REST verbs above. Custom methods
(anything that isn't `index`/`new`/`store`/`show`/`edit`/`update`/`destroy`)
must be registered explicitly in `routes/web.js`:

```js
// app/controllers/organizer/dashboard.controller.js has an exportAttendees() method

// routes/web.js
router.get('/organizer/dashboard/export-attendees', DashboardController, 'exportAttendees')
```

If you hit a URL that matches `{controllerPath}/{action}` where the action
exists on the controller but no route is registered, the 404 response tells
you exactly what to add:

```
Cannot GET /organizer/dashboard/exportAttendees

DashboardController.exportAttendees() exists but no route is registered.
Add this to routes/web.js:
  router.get('/organizer/dashboard/exportAttendees', DashboardController, 'exportAttendees')
```

## Route parameters

Access via `this.param('name')` (Controller) or `c.req.param('name')` (closure):

```js
async show() {
  const id = this.param('id')
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

## Named routes

Every route can carry a name. Names are lookup keys — `Url.route('events.show', { id })`
builds a URL for the route without hard-coding the path — which means you can
rename `/events` to `/rentals` in one place and the whole app follows.

```js
router.get('/events/:id', EventsController, 'show').name('events.show')

// Later, in a controller or view:
Url.route('events.show', { id: event.id })
// → "https://app.example/events/42"
```

Duplicate names are non-fatal: the later `.name()` call wins and a warning is
logged. Unknown names throw with a Levenshtein-nearest suggestion list, so a
typo like `evens.show` surfaces immediately at request time.

### Auto-naming

You rarely need to call `.name()` yourself — the framework names routes for
you following a consistent convention:

| Registration | Name pattern | Example |
|---|---|---|
| `router.resource('/events', Ctrl)` | `events.<verb>` | `events.show` |
| `router.resource('/organizer/events', Ctrl)` | `organizer.events.<verb>` | `organizer.events.index` |
| Convention route (`app/controllers/events.controller.js`) | same as `resource()` | `events.index` |
| Convention route (`app/controllers/organizer/events.controller.js`) | dot-joined path segments | `organizer.events.index` |
| Admin resource (`AdminRouter`) | `admin.<table>.<verb>` | `admin.users.edit` |
| Admin custom actions | `admin.<table>.action` | `admin.users.action` |
| API plugin (`ApiPlugin`) | `api.<resource>.<verb>` | `api.users.index` |

`<verb>` is one of `index`, `new`, `store`, `show`, `edit`, `update`, `destroy`
(plus `bulk`, `lookup`, `export`, `restore`, etc. for admin).

To override the derived name for an entire resource:

```js
// Option A — per-registration:
router.resource('/rentals', RentalsController, RentalModel, 'events')
// → 'events.index', 'events.show', ...

// Option B — on the controller (for convention routes too):
class RentalsController {
  static routeName = 'events'
  // ...
}
```

### Introspection

```js
router.namedRoutes()
// → Map<string, { name, method, path, route }>
```

## Explicit routes override conventions

Convention routing backfills every REST verb your controller implements
(`index`, `new`, `store`, `show`, `edit`, `update`, `destroy`) **except** any
action you have already claimed with an explicit
`router.get/post/put/patch/delete(path, ControllerClass, 'action')` in
`routes/web.js`. The claim is on the **(Controller, action)** pair — the path
you register at is irrelevant. Claim once, wherever you like, and convention
will not double-mount that verb at its default REST URL.

**Before v0.3.0** — both routes were live simultaneously, which was almost
never what you wanted:

```js
// routes/web.js
router.get('/tickets/my/:id/edit', TicketsController, 'edit')

// Registered routes at boot:
// GET  /tickets/my/:id/edit  → TicketsController#edit   (your explicit route)
// GET  /tickets/:id/edit     → TicketsController#edit   (auto REST — SURPRISE)
```

**v0.3.0 and later** — the explicit declaration wins; convention skips the
matching REST verb and prints an info line at boot so you can see what got
suppressed:

```
[foobar] Skipping convention mount for TicketsController#edit (GET /tickets/:id/edit) — already claimed by explicit route
```

Every other REST verb on `TicketsController` (`index`, `store`, `show`,
`update`, `destroy`, …) is still auto-mounted at its default path — only the
`edit` action's default is skipped. If you actually want both paths bound to
the same action, declare both explicitly.

You can inspect the claim set at any time via `router.claimedActions()` which
returns a `Set<string>` of `"ControllerName#actionName"` keys.

## Next steps

- Write the controllers your routes point at: [Controllers](./controllers.md)
- Render responses: [Views](./views.md)
- Auto-generated REST API for models: [API](./api.md)
{% endraw %}
