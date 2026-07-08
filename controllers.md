# Controllers

Controllers group related request handling logic into a single class. They are placed in `app/controllers/` and named with the `.controller.js` suffix.

## Basic Controller

```js
// app/controllers/home.controller.js
import { Controller } from 'foobarjs/core'

export default class HomeController extends Controller {
  async index(c) {
    return c.render('home/index', {
      title: 'Welcome',
    })
  }
}
```

## Controller Methods

Each method receives the Hono `Context` object (`c`) and should return a response. Available response types:

```js
// Render a view
return c.render('products/index', { products })

// Return JSON
return c.json({ message: 'Hello' })

// Redirect
return c.redirect('/login')

// Text response
return c.text('Not found', 404)

// Empty body with status
return c.body(null, 204)
```

## Convention-Based Views

If a controller method returns data instead of a `Response`, foobarjs checks for a matching view and renders it automatically. This follows the convention:

| Controller | Action | View Path | Data Key |
|------------|--------|-----------|----------|
| `ProductsController` | `index` | `app/views/products/index.html` | `products` |
| `ProductsController` | `show` | `app/views/products/show.html` | `product` |
| `ProductsController` | `edit` | `app/views/products/edit.html` | `product` |
| `ProductsController` | `new` | `app/views/products/new.html` | `product` |

For example, this controller:

```js
class ProductsController {
  async index(c) {
    return Product.all()
  }

  async show(c) {
    return Product.find(c.req.param('id'))
  }
}
```

Automatically renders `products/index.html` with `{ products }` and `products/show.html` with `{ product }`.

If no matching view exists, the data is returned as JSON instead. This makes the same controller action serve both HTML and API clients depending on whether the view file exists.

### Action-to-Data-Key Mapping

- `index` uses the controller name as-is (plural): `products`
- `show`, `edit`, `new`, and `destroy` use the singular form: `product`
- `store` and `update` typically redirect, so they are not auto-rendered

## Request Data

Access request data from the Hono context:

```js
// Route parameters
c.req.param('id')

// Query parameters
c.req.query('page')

// Body (parsed from JSON or form data)
const body = await c.req.parseBody()
// or for JSON:
const json = await c.req.json()
```

## Controller Conventions

Controllers follow a RESTful pattern with these methods:

| Method | Description |
|--------|-------------|
| `index(c)` | List all resources |
| `show(c)` | Show a single resource |
| `new(c)` | Show create form |
| `store(c)` | Handle create submission |
| `edit(c)` | Show edit form |
| `update(c)` | Handle update submission |
| `destroy(c)` | Handle deletion |

Not all methods are required. Only defined methods get routes registered.
