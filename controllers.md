# Controllers

Controllers group related request handling logic into a single class. They
live in `app/controllers/` and are named with the `.controller.js` suffix.

## The Controller base class

Every controller extends `Controller` from `foobarjs/core`. The base class
provides response helpers (`this.render`, `this.json`, `this.redirect`, ...),
convenience accessors (`this.getLoggedInUser()`, `this.params()`,
`this.body()`), and the request context as `this.c`.

```js
// app/controllers/home.controller.js
import { Controller } from 'foobarjs/core'

class HomeController extends Controller {
  async index() {
    return this.render('home/index', { title: 'Welcome' })
  }
}

export default HomeController
```

Controller methods do not take the context as a parameter — it's already on
`this.c`. Access request data via `this.params()`, `this.query()`,
`this.body()`, or drop down to `this.c.req.header('X-Custom')` when you need
Direct access to the underlying request context.

## Response helpers

| Helper | Returns |
|--------|---------|
| `this.render(template, data)` | HTML rendered from `app/views/<template>.html`. |
| `this.json(data)` | JSON response (status 200). |
| `this.json(data, statusCode)` | JSON response with an explicit status. |
| `this.text(body)` / `this.text(body, status)` | Plain text response. |
| `this.html(body)` | Raw HTML string. |
| `this.html(template, data)` | Same as `this.render(template, data)`. |
| `this.redirect(path)` / `this.redirect(path, status)` | Returns a `RedirectResponse` builder (default 302). |
| `this.back()` / `this.back(status)` | Redirect to the `Referer` header (or `/`). |
| `this.view(template, data)` | Alias for `this.render()`. |
| `this.flash(key, message)` | Set a one-shot session flash. Chainable. |
| `this.share(key, value)` | Share a value with all views rendered during this request. Chainable. |
| `this.wantsJson()` | `true` when the request accepts JSON (checks `Accept` and `Content-Type`). |

## Convenience accessors

| Accessor | What it returns |
|----------|-----------------|
| `this.c` | Request context. Drop down here for anything the helpers don't cover. |
| `this.request` | Alias for `this.c.req`. |
| `this.response` | Alias for `this.c.res`. |
| `this.params()` | Route parameters (`{ id: '5' }`). |
| `this.query()` | Query string (`{ page: '2' }`). |
| `this.body()` | Parsed request body. Tries JSON first, then form-encoded. |
| `this.getLoggedInUser()` | The authenticated user, or `null`. |
| `this.isLoggedIn()` | Boolean. |
| `this.validate(FormRequestClass)` | Runs the given `FormRequest`, returns it. |
| `this.config(key, default)` | Read a config value by dot-notation (e.g. `'app.name'`). |
| `this.env(key, default)` | Read an environment variable from `process.env`. |

## Controller Middleware

Declare middleware on a controller with a static `middleware` property. The middleware runs before every action on that controller — whether the route was registered explicitly or via convention.

### Apply to all actions

```js
import { Controller } from 'foobarjs/core'
import { RequireAuthMiddleware } from 'foobarjs/auth'

class DashboardController extends Controller {
  static middleware = [RequireAuthMiddleware]

  async index() {
    return this.render('dashboard/index', { user: this.getLoggedInUser() })
  }
}
```

### Apply to specific actions with `only`

```js
class PostsController extends Controller {
  static middleware = {
    use: [RequireAuthMiddleware],
    only: ['store', 'update', 'destroy'],
  }

  index() { /* public — no auth */ }
  show() { /* public — no auth */ }
  store() { /* auth required */ }
  update() { /* auth required */ }
  destroy() { /* auth required */ }
}
```

### Exclude specific actions with `except`

```js
class ArticlesController extends Controller {
  static middleware = {
    use: [RequireAuthMiddleware],
    except: ['index', 'show'],
  }
  // Auth runs on store, update, destroy — but not index or show
}
```

### Multiple middleware

Stack middleware by adding more classes to the array:

```js
class AdminReportsController extends Controller {
  static middleware = [RequireAuthMiddleware, AdminOnlyMiddleware]
}
```

Controller middleware stacks with group middleware from `router.group()` — group middleware runs first, then controller middleware.

### Opting out of auto-applied middleware

Use `static withoutMiddleware` to skip middleware that was auto-discovered from `app/middlewares/`. Pass middleware names (derived from the filename) or imported references:

```js
class WebhookController extends Controller {
  static withoutMiddleware = ['Csrf', 'CartShare']
}
```

```js
import CartShare from '../app/middlewares/CartShare.js'

class ApiController extends Controller {
  static withoutMiddleware = [CartShare]
}
```

See [Middleware](./middleware.md) for more on the folder convention and opt-out mechanism.

### Writing a middleware class

Any class with a `handle(c, next)` method works as middleware:

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

Pass the class itself — foobarjs instantiates it automatically. You can also pass an already-instantiated object if you need constructor arguments.

## Convention-based views

If a controller action returns data (not a `Response`), foobarjs looks for a
matching view template and renders it. Fallback is JSON.

| Action | View path | Data key |
|--------|-----------|----------|
| `index` | `app/views/<controller>/index.html` | plural (`products`) |
| `show` | `app/views/<controller>/show.html` | singular (`product`) |
| `edit` | `app/views/<controller>/edit.html` | singular |
| `new` | `app/views/<controller>/new.html` | singular |
| `destroy` | — | typically redirects |
| `store`, `update` | — | typically redirect |

Example:

```js
class ProductsController extends Controller {
  async index() {
    return Product.all()
  }

  async show() {
    return Product.find(this.params().id)
  }
}
```

Automatically renders `products/index.html` with `{ products }` and
`products/show.html` with `{ product }` if those views exist, otherwise
returns JSON. This makes the same controller action serve HTML browsers
and API clients from one implementation.

## Auto response contract

The value returned from a controller action is coerced into an HTTP response
by the framework:

| Return value | Framework behavior |
|--------------|--------------------|
| A `Response` object | Sent as-is. |
| `undefined` or `null` | `204 No Content`. |
| An object or array | Looks for `app/views/<controller>/<action>.html`. If found, renders it with the value bound to a data key (see mapping above). Otherwise returns JSON. |
| A string or primitive | Sent as plain text (`c.text(String(value))`). |

The `<controller>` folder name is derived from the class name with the
`Controller` suffix stripped and lowercased. `ProductsController` →
`products`. This is independent of the filename.

## RESTful actions

Controllers follow REST by convention. Define only the actions you need —
missing actions produce no route (a 404), not a 500.

| Method | HTTP + URL |
|--------|------------|
| `index()` | `GET /base` |
| `new()` | `GET /base/new` |
| `store()` | `POST /base` |
| `show()` | `GET /base/:id` |
| `edit()` | `GET /base/:id/edit` |
| `update()` | `PUT /base/:id` |
| `destroy()` | `DELETE /base/:id` |

The `/base` prefix comes from the filename (`products.controller.js` → `/products`)
or from an explicit registration in `routes/web.js`. See [Routing](./routing.md).

## Working with the raw context

For anything the helpers don't cover, use `this.c` directly:

```js
async show() {
  const ifNoneMatch = this.c.req.header('If-None-Match')
  const url = new URL(this.c.req.url)
  this.c.header('Cache-Control', 'max-age=60')
  return this.render('products/show', { product })
}
```

## Redirect Builder

`this.redirect()` and `this.back()` return a `RedirectResponse` builder that
can flash errors and old input before the redirect is sent:

```js
import { Controller } from 'foobarjs/core'
import { ValidationError } from 'foobarjs/orm'

class OrdersController extends Controller {
  async store() {
    let request
    try {
      request = await this.validate(StoreOrderValidator)
    } catch (err) {
      if (err.name === 'ValidationError') {
        if (this.wantsJson()) {
          return this.json({ errors: err.errors, message: err.message }, 422)
        }
        return this.back().withErrors(err).withInput(err.input)
      }
      throw err
    }
    const order = await Order.create(request.validated())
    return this.redirect('/orders').with('success', 'Order placed!')
  }
}
```

| Method | Description |
|--------|-------------|
| `.withErrors(err)` | Flashes `err.errors` (or `err` itself if it's a plain map) to the session as `errors`. |
| `.withInput(data)` | Flashes the given data to the session as `old`, so form fields can be repopulated. |
| `.with(key, value)` | Flash an arbitrary key/value to the session. |
| `.toResponse()` | Materialise the redirect. Called automatically by the framework — you only need it in tests. |

The framework does **not** auto-redirect on validation errors. Your controller
decides what to do: redirect back for HTML forms, return JSON for APIs, or
handle it any other way. This keeps behaviour explicit and predictable,
especially for custom API endpoints.

## Next steps

- Add routes for these controllers: [Routing](./routing.md)
- Render templates: [Views](./views.md)
- Validate input: [Validation](./validation.md)
- Handle errors: [Error handling](./error-handling.md)
