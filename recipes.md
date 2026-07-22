[← Back to docs](./README.md)

{% raw %}
# Recipes & Recommendations

This guide addresses the most common "where do I put this?" questions in
foobarjs. Each section covers overlapping features, shows when to use each,
and ends with an opinionated recommendation.

All examples use an e-commerce app (products, orders, checkout) as the
running scenario.

---

## 1. Request Data Access

foobarjs gives you multiple ways to read incoming data. They serve different
purposes.

### When to use what

| What you need | In a controller | In a closure / middleware |
|---------------|-----------------|--------------------------|
| Route parameter (`/products/:id`) | `this.param('id')` | `c.req.param('id')` |
| Query string (`?page=2`) | `this.query('page')` | `c.req.query('page')` |
| Parsed body (form or JSON) | `this.body` | `await c.req.parseBody()` or `await c.req.json()` |
| Single body field | `this.input('email')` | n/a -- destructure from body |
| Subset of body fields | `this.only('name', 'email')` | n/a |
| Uploaded file | `this.file('avatar')` | `await c.req.parseBody()` |

### Real-world example: checkout flow

```js
class CheckoutController extends Controller {
  async store() {
    const orderId = this.param('id')              // route param
    const promo = this.query('promo')             // query string
    const { address, notes } = this.only('address', 'notes')  // body fields
  }
}
```

### Common mistake: calling `c.req.json()` in a controller

```js
// WRONG -- body stream already consumed by the framework
const data = await this.c.req.json()

// RIGHT -- use the pre-parsed body
const data = this.body
```

The framework parses the body once and stores it on the context. Calling
`c.req.json()` again fails because the stream is consumed. It is only safe
in raw middleware that runs before the framework's body parser.

**Foobar recommends:** Use `this.body` for the full parsed body, `this.input()`
for a single field, and `this.only()` when you want to whitelist fields.
Reserve `this.param()` and `this.query()` for URL data. Never call
`c.req.json()` inside a controller.

---

## 2. Authentication

Three mechanisms control whether a route requires a logged-in user. They
look similar but work at different levels.

### When to use what

| Mechanism | Scope | Effect |
|-----------|-------|--------|
| `static auth = false` on controller | All actions in that controller | Marks the controller as public (convention routes) |
| `.public()` on a route | Single route or group | Removes `'auth'` middleware from that route |
| `auth` in `config/api.js` | Auto-generated API endpoints | Controls API auth per-model or globally |

### How they compose

The auth system is auth-first by default (web layer) (`config/auth.js` `guard: 'required'`).
Every route requires authentication unless explicitly opted out.

```js
// Convention controller -- static auth = false makes ALL actions public
class ProductsController extends Controller {
  static auth = false   // browsing is public
  index() { /* public */ }
  show()  { /* public */ }
  store() { /* also public -- probably not what you want */ }
}

// Explicit route -- .public() targets one route only
router.get('/', HomeController, 'index').public()

// API plugin -- config/api.js controls auto-generated REST
export default { auth: { read: false, write: 'session' } }
```

### Common mistake: `static auth = false` on a controller with write actions

```js
// DANGEROUS -- all seven REST actions become public
class OrdersController extends Controller {
  static auth = false
  index() { /* ok -- public listing */ }
  store() { /* anyone can create orders! */ }
  destroy() { /* anyone can delete orders! */ }
}
```

If you need some actions public and some protected, use controller middleware
with `only`/`except` instead:

```js
class OrdersController extends Controller {
  static middleware = {
    use: ['auth'],
    except: ['index', 'show'],
  }
}
```

**Foobar recommends:** Use `static auth = false` only for fully public
controllers (landing pages, product browsing). Use `.public()` on explicit
routes for one-off public endpoints. Use `config/api.js` for API-level auth
rules. When mixing public and private actions, use `static middleware` with
`except`.

---

## 3. Authorization

Gates are the single authorization system. They work everywhere: admin panel,
API, controllers, and routes.

### When to use what

| Tool | Best for | Checks |
|------|----------|--------|
| Gates (`app/gates/`) | Per-model CRUD rules, scoping | Ownership, roles, custom logic |
| `.permissions()` on `Admin.resource()` | Quick admin role setup | Generates a gate under the hood |
| `.can()` on routes | Per-route gate enforcement | Wires a gate check into the route pipeline |
| `this.authorize()` in controllers | In-controller gate check | Explicit gate check, throws 401/403 |

### Decision tree

```
Is this about the admin panel UI?
  YES -> Define a Gate (or use .permissions() shorthand)
  NO  -> Is this a model-level rule (who can view/edit/delete)?
           YES -> Write a Gate, use .can() on routes or this.authorize() in controllers
           NO  -> Is this a non-resource check (e.g., "can view analytics")?
                    YES -> Write a standalone Gate
                    NO  -> Handle in controller logic
```

### Real-world example: order management

```js
// app/gates/order.gate.js -- the single source of truth
export default Gate({
  model: Order,
  viewAny(user) { return user.hasRole('staff', 'admin') },
  view(user, order) { return order.user === user.id || user.hasRole('staff') },
  update(user, order) { return order.user === user.id && order.status === 'pending' },
  delete(user, order) { return user.hasRole('admin') },
  scope(user, query) {
    if (user.hasRole('admin')) return query
    return query.where('user', user.id)
  },
})

// routes/web.js -- enforce the gate on web routes
router.put('/orders/:order/cancel', OrderController, 'cancel')
  .bind(Order).can('update', Order)

// app/admin/order.admin.js -- control admin panel buttons
Admin.resource(Order).permissions({
  view: ['admin', 'staff'], edit: ['admin'], delete: ['admin'],
})
```

### Edge case: gates vs admin permissions conflict

When both a gate and `.permissions()` exist for the same model, the gate
takes precedence. The admin panel checks the gate first; if no gate exists,
it falls back to the `.permissions()` role check.

This means if your gate says `delete` returns `false` for everyone, the
admin delete button is hidden even if `.permissions()` allows it.

**Foobar recommends:** Write gates for real authorization logic. Use admin
`.permissions()` only for simple role-based UI hiding when you do not need
a gate. Use `.can()` on routes to enforce gates in the web layer. The API
plugin enforces gates automatically -- you do not need to wire anything.

---

## 4. Response Methods

Multiple ways to send a response back. Each has different behavior.

### When to use what

| Method | Returns | When to use |
|--------|---------|-------------|
| `this.render(template, data)` | Rendered HTML from a Blade template | Server-rendered pages |
| `this.html(string)` | Raw HTML string | Inline HTML snippets, no template needed |
| `this.json(data)` | JSON response | API endpoints, AJAX responses |
| `this.json(data, status)` | JSON with status code | Error responses, 201 created |
| `this.redirect(path)` | 302 redirect (builder) | After form submission |
| `this.text(body)` | Plain text | Health checks, simple responses |
| `return object` | Auto-JSON | Quick returns (auto response contract) |
| `return null` | 204 No Content | Delete actions, fire-and-forget |

### Real-world example: dual HTML/JSON endpoint

```js
class OrdersController extends Controller {
  async store() {
    let request
    try {
      request = await this.validate(StoreOrderValidator)
    } catch (err) {
      if (err.name === 'ValidationError') {
        if (this.wantsJson()) return this.json({ errors: err.errors }, 422)
        return this.back().withErrors(err).withInput(err.input)
      }
      throw err
    }
    const order = await Order.create(request.validated())
    if (this.wantsJson()) return this.json(order, 201)
    return this.redirect('/orders').with('success', 'Order placed!')
  }
}
```

### Common mistake: returning a plain object expecting HTML

```js
// WRONG -- auto response contract coerces objects to JSON
return { products }  // -> {"products": [...]}

// RIGHT -- explicitly render a template for HTML
return this.render('products/index', { products })
```

**Foobar recommends:** Use `this.render()` for pages, `this.json()` for
explicit API responses, and `return object` only when you want quick JSON
returns (e.g., a health check controller). Always use `this.wantsJson()` to
branch between HTML and JSON in dual-purpose endpoints.

---

## 5. Middleware Registration

Three ways to register middleware. They stack in a defined order.

### When to use what

| Method | Scope | Auto-applied? |
|--------|-------|---------------|
| File in `app/middlewares/` | Global (or scoped to web/api) | Yes (async functions and classes) |
| `router.group({ middleware: [...] })` | Routes in the group | No -- explicit |
| `.middleware()` on a route | Single route | No -- explicit |
| `static middleware` on controller | All actions (or `only`/`except`) | No -- explicit |

### Execution order

```
1. Framework middleware (session, auth, CSRF, CORS, rate limit)
2. Auto-discovered middleware (app/middlewares/*.js)
3. Group middleware (outer groups first)
4. Controller static middleware
5. Inline route middleware
6. Fluent .middleware() additions
```

### Real-world example: cart count for storefront

```js
// app/middlewares/CartShare.js -- auto-applied to every request
export default async function CartShare(c, next) {
  const cart = c.get('session')?.get('cart') || []
  c.share('cartCount', cart.reduce((sum, i) => sum + i.quantity, 0))
  await next()
}
```

No registration needed. Subfolder scoping: `app/middlewares/api/` for
API-only, `app/middlewares/web/` for web-only.

### Common mistake: factory middleware in the auto-discover folder

```js
// app/middlewares/RateLimit.js -- sync function = factory, NOT auto-applied
export default function RateLimit(options = {}) {
  return async (c, next) => { /* ... */ await next() }
}
// Must be used explicitly: .middleware([RateLimit({ max: 30 })])
```

### Controlling order

Files load alphabetically. Prefix with numbers (`01-Timing.js`, `02-CartShare.js`)
for explicit ordering.

**Foobar recommends:** Use auto-discovery for middleware that should run on
every request (cart sharing, analytics, timing). Use `router.group()` for
middleware shared by a set of routes (admin section, API version). Use
controller `static middleware` for controller-specific concerns. Use
`.middleware()` on individual routes only for one-off needs like custom rate
limits.

---

## 6. Auth Configuration

Three places to configure whether auth is required. They have a clear
precedence order.

### Precedence (highest wins)

```
1. .public() or .middleware() on the specific route
2. static auth = false on the controller
3. router.group({ middleware: [...] })
4. config/auth.js guard setting ('required' or 'manual')
```

For the API plugin, the precedence is different:

```
1. Api.resource(Model).middleware('auth')  (app/api/*.api.js)
2. Api.resource(Model).auth(...)  (app/api/*.api.js)
3. config/api.js -> models[ModelName]
4. config/api.js -> top-level auth (default: false)
```

### Real-world example: mixed public and private

```js
// config/auth.js -- auth-first baseline
export default { guard: 'required' }

// routes/web.js -- opt specific routes out
export default function (router) {
  router.get('/', HomeController, 'index').public()
  router.get('/products', ProductsController, 'index').public()
  // Everything else requires auth automatically
}

// config/api.js -- API reads public, writes need auth
export default {
  auth: { read: false, write: 'session' },
  models: { users: 'session' },
}
```

### Edge case: `static auth = false` travels with the controller

A controller's `static auth = false` applies regardless of how it is
registered (convention or explicit). To override it for a specific route,
add `'auth'` middleware inline:

```js
router.get('/pages/secret', 'auth', PagesController, 'secret')
```

**Foobar recommends:** Set `guard: 'required'` in `config/auth.js` (the
default) and opt out selectively. Use `.public()` on explicit routes, and
`static auth = false` only for controllers where every action is truly
public. For the API, configure auth in `config/api.js` and use
`Api.resource().auth()` for per-model overrides.

---

## 7. Validation

Three validation approaches overlap. Each serves a different layer.

### When to use what

| Approach | Where | Runs when | Best for |
|----------|-------|-----------|----------|
| `Field` rules in `static schema` | Model | Every `save()` | Data integrity, DB-level guarantees |
| `FormRequest` via `this.validate()` | Controller | Before save, in controller | Request-specific rules, authorization |
| `Validator` standalone | Anywhere | When you call `.validate()` | Jobs, services, scripts |

### Decision tree

```
Is this a rule about the data itself (always true, regardless of context)?
  YES -> Put it in the model schema
  NO  -> Is this a request-specific rule (e.g., "confirmed" password)?
           YES -> Put it in a FormRequest
           NO  -> Use standalone Validator
```

### Real-world example: product creation

```js
// app/models/product.model.js -- always-enforced rules
class Product extends Model {
  static schema = {
    name: Field.string().required().maxLength(255),
    slug: Field.string().required().unique(),
    price: Field.float().required().min(0),
    stock: Field.number().default(0).min(0),
  }
}

// app/validators/store-product.validator.js -- request-specific rules
class StoreProductValidator extends FormRequest {
  authorize() {
    return this.c.get('user')?.hasRole('admin', 'editor')
  }
  rules() {
    return {
      name: Field.string().required().maxLength(255),
      slug: Field.string().required().regex(/^[a-z0-9-]+$/, 'Must be kebab-case'),
      price: Field.float().required().min(0),
    }
  }
  messages() {
    return { 'slug.regex': 'Slug must be lowercase letters, numbers, and dashes only' }
  }
}
```

### Common mistake: duplicating model rules in FormRequest

The model validates on `save()` regardless. Focus the FormRequest on
request-specific concerns the model does not cover:

```js
// BETTER -- only validate what the request adds beyond the model
class StoreOrderValidator extends FormRequest {
  rules() {
    return {
      address: Field.string().required().minLength(5),  // not in model schema
    }
  }
}
```

### Edge case: admin panel validation

The admin panel does not use FormRequest. It saves the model directly, so
only `static schema` rules and DB constraint translations run. If you need
admin-specific validation beyond what the model schema provides, add the
rules to the model schema itself.

**Foobar recommends:** Put data-integrity rules in the model schema (they
run everywhere -- controllers, admin, API, jobs). Use FormRequest for
request-specific validation (authorization checks, cross-field rules like
`confirmed`, custom error messages for a specific form). Use standalone
`Validator` in jobs and services.

---

## 8. Configuration Access

Three ways to read configuration. Each works in different contexts.

### When to use what

| Method | Where it works | When available |
|--------|----------------|----------------|
| `process.env.KEY` | Everywhere | Immediately |
| `env('KEY', default)` | Everywhere | Immediately (with auto-casting) |
| `config('dot.path', default)` | After boot | Controllers, jobs, middleware, listeners |
| `this.config('dot.path')` | Controllers only | During request handling |
| `c.get('configLoader').get()` | Middleware, closures | During request handling |

### Decision tree

```
Are you inside a config file (config/*.js)?
  YES -> Use process.env directly
  NO  -> Do you need a config value (from config/*.js files)?
           YES -> Use config('dot.path')
           NO  -> Do you need an env var?
                    YES -> Use env('KEY', default) for auto-casting
                    NO  -> You probably want config()
```

### Real-world example: payment integration

```js
// config/payments.js -- use process.env (config not yet loaded)
export default {
  stripe: {
    key: process.env.STRIPE_KEY,
    secret: process.env.STRIPE_SECRET,
  },
}

// app/controllers/checkout.controller.js -- use config()
class CheckoutController extends Controller {
  async store() {
    const stripeKey = this.config('payments.stripe.key')
  }
}

// app/jobs/process-payment.job.js -- config() works in jobs too
import { config, env } from 'foobarjs/core'

class ProcessPaymentJob extends QueueJob {
  async handle() {
    const secret = config('payments.stripe.secret')
    const debug = env('APP_DEBUG', false)  // auto-casts 'true' -> true
  }
}
```

### Common mistake: using `process.env` in controllers

```js
// FRAGILE -- no defaults, no dot-notation
const appName = process.env.APP_NAME   // might be undefined

// BETTER -- config layer provides defaults and structure
const appName = this.config('app.name', 'My App')
```

### Edge case: `env()` auto-casting

`env()` casts `'true'`/`'false'` to booleans and numeric strings to numbers.
This can surprise you if a value looks numeric but should stay a string:

```js
// .env: API_VERSION=2
env('API_VERSION')   // returns number 2, not string '2'
// Use process.env.API_VERSION if you need the raw string
```

**Foobar recommends:** Use `config()` everywhere after boot. Use
`process.env` only inside config files. Use `env()` when you need
auto-casting of booleans and numbers. In controllers, prefer `this.config()`
for the cleaner syntax.

---

## 9. Error Handling

Three layers handle errors. Understanding the flow prevents double-handling
or swallowed errors.

### The error flow

```
Controller action throws
  |
  v
Did you catch it in a try/catch?
  YES -> You handle it (return JSON, redirect, etc.)
  NO  -> Falls through to central ErrorHandler
           |
           v
         Is it a ValidationError?
           YES -> 422 response (JSON or HTML error page)
           NO  -> Is it an HttpException (NotFoundError, etc.)?
                    YES -> Renders the appropriate status (404, 403, etc.)
                    NO  -> 500 Internal Server Error
```

### When to use what

| Approach | When |
|----------|------|
| `try/catch` in controller | Validation errors you want to redirect back with |
| `throw new NotFoundError()` | Let the framework render the right error page |
| Custom `ErrorHandler` | Cross-cutting error handling (billing errors, external API failures) |

### Real-world example: three layers in action

```js
class OrdersController extends Controller {
  async store() {
    // LAYER 1: Catch validation to redirect with flash
    let request
    try {
      request = await this.validate(StoreOrderValidator)
    } catch (err) {
      if (err.name === 'ValidationError') {
        return this.back().withErrors(err).withInput(err.input)
      }
      throw err
    }

    // LAYER 2: Throw HTTP exceptions for framework handling
    const product = await Product.find(request.validated().productId)
    if (!product) throw new NotFoundError('Product not found')

    // Unexpected errors (DB down, etc.) fall through to ErrorHandler
    const order = await Order.create({ ... })
    return this.redirect('/orders/' + order.id)
  }
}

// LAYER 3: Custom ErrorHandler for cross-cutting concerns
class AppErrorHandler extends ErrorHandler {
  async handle(err, c) {
    if (err.name === 'StripeError') {
      Logger.instance().error(err, { source: 'stripe' })
      return c.json({ error: 'Payment failed' }, 402)
    }
    return super.handle(err, c)
  }
}
```

### Common mistake: catching too broadly

```js
// WRONG -- swallows programming bugs
try {
  const order = await Order.create(this.body)
  return this.json(order)
} catch (err) {
  return this.json({ error: 'Something went wrong' }, 500)
}

// RIGHT -- catch only what you expect
try {
  const order = await Order.create(this.body)
  return this.json(order)
} catch (err) {
  if (err.name === 'ValidationError') {
    return this.json({ errors: err.errors }, 422)
  }
  throw err  // programming errors reach ErrorHandler with full stack
}
```

### Edge case: error views in production

Create `app/views/errors/404.html` and `app/views/errors/500.html` for
branded error pages. The view receives `status`, `message`, and `requestId`.

**Foobar recommends:** Use `try/catch` only for `ValidationError` where you
need to redirect back with form data. Throw `NotFoundError`, `ForbiddenError`,
and `AuthenticationError` for standard HTTP errors -- let the framework
handle rendering. Subclass `ErrorHandler` only for cross-cutting concerns
like third-party API errors or custom logging.

---

## 10. Route Definition

Four ways to define routes. They coexist and each has a sweet spot.

### When to use what

| Method | Best for | Auth default |
|--------|----------|--------------|
| `routes/web.js` explicit | Custom paths, inline handlers, non-REST routes | Required (auth-first) |
| Convention routes (filename) | Standard REST controllers | Required (auth-first) |
| API plugin auto-routes | Model CRUD APIs | Configured in `config/api.js` |
| Admin plugin routes | Admin panel (auto-generated) | Requires admin session |

### Decision tree

```
Is this a REST API for a model?
  YES -> Let the API plugin handle it (config/api.js + app/api/*.api.js)
  NO  -> Is this a standard REST resource (index/show/store/update/destroy)?
           YES -> Use convention: app/controllers/things.controller.js
           NO  -> Is this a custom route (webhook, health check, special path)?
                    YES -> Use routes/web.js
                    NO  -> Is this admin UI?
                             YES -> Use app/admin/*.admin.js (auto-generated)
```

### Real-world example: full e-commerce routing

```
app/controllers/
  home.controller.js       <- convention: GET /
  products.controller.js   <- convention: REST at /products
  cart.controller.js        <- convention: REST at /cart
  checkout.controller.js    <- convention: REST at /checkout
routes/web.js              <- explicit: custom routes only
  router.get('/', HomeController, 'index').public()
  router.get('/health', (c) => c.json({ status: 'ok' })).public()
config/api.js              <- API plugin: auto REST for all models
app/admin/
  product.admin.js          <- admin panel: auto-generated CRUD UI
```

### Common mistake: defining routes in both places

Convention already registers REST routes from `app/controllers/products.controller.js`.
Adding `router.get('/products', ProductsController, 'index')` in `routes/web.js`
creates a duplicate. Explicit routes are for paths that do not match the
convention pattern.

### Edge case: convention and explicit routes for the same controller

Explicit routes take precedence for the paths they define. Convention routes
still register for uncovered actions. This can create duplicates:

```js
// Convention registers GET /products (index). You also register:
router.get('/shop/products', ProductsController, 'index').public()
// Now GET /products AND GET /shop/products both call index()
```

### When to use `routes/api.js`

Use it for custom JSON endpoints that do not map to a model:

```js
// routes/api.js
export default function (router) {
  router.post('/api/v1/webhooks/stripe', WebhookController, 'handle')
    .withoutMiddleware(['Csrf'])
  router.get('/api/v1/search', SearchController, 'index')
    .middleware([RateLimit({ max: 30 })])
}
```

**Foobar recommends:** Let convention routes handle standard REST
controllers. Use `routes/web.js` for non-REST routes (root path, health
checks, webhooks, custom paths). Let the API plugin auto-generate model
APIs -- customize with `config/api.js` and `app/api/*.api.js` files. Only
add `routes/api.js` for custom API endpoints that do not map to a model.
{% endraw %}
