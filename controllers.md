# Controllers

Controllers group related request handling logic into a single class. They
live in `app/controllers/` and are named with the `.controller.js` suffix.

## The Controller base class

Every controller extends `Controller` from `foobarjs/core`. The base class
provides response helpers (`this.render`, `this.json`, `this.redirect`, ...),
convenience accessors (`this.getLoggedInUser()`, `this.params()`,
`this.body()`), and the Hono context itself as `this.c`.

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
Hono-native access.

## Response helpers

| Helper | Returns |
|--------|---------|
| `this.render(template, data)` | HTML rendered from `app/views/<template>.html`. |
| `this.json(data)` | JSON response (status 200). |
| `this.json(data, statusCode)` | JSON response with an explicit status. |
| `this.text(body)` / `this.text(body, status)` | Plain text response. |
| `this.html(body)` | Raw HTML string. |
| `this.html(template, data)` | Same as `this.render(template, data)`. |
| `this.redirect(path)` / `this.redirect(path, status)` | HTTP redirect (default 302). |
| `this.view(template, data)` | Alias for `this.render()`. |
| `this.flash(key, message)` | Set a one-shot session flash. Chainable. |

## Convenience accessors

| Accessor | What it returns |
|----------|-----------------|
| `this.c` | Raw Hono context. Drop down here for anything the helpers don't cover. |
| `this.request` | Alias for `this.c.req`. |
| `this.response` | Alias for `this.c.res`. |
| `this.params()` | Route parameters (`{ id: '5' }`). |
| `this.query()` | Query string (`{ page: '2' }`). |
| `this.body()` | Parsed request body. Tries JSON first, then form-encoded. |
| `this.getLoggedInUser()` | The authenticated user, or `null`. |
| `this.isLoggedIn()` | Boolean. |
| `this.validate(FormRequestClass)` | Runs the given `FormRequest`, returns it. |

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

Any Hono API you need is on `this.c`:

```js
async show() {
  const ifNoneMatch = this.c.req.header('If-None-Match')
  const url = new URL(this.c.req.url)
  this.c.header('Cache-Control', 'max-age=60')
  return this.render('products/show', { product })
}
```

## Next steps

- Add routes for these controllers: [Routing](./routing.md)
- Render templates: [Views](./views.md)
- Validate input: [Validation](./validation.md)
- Handle errors: [Error handling](./error-handling.md)
