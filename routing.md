# Routing

foobarjs uses convention-based routing. Controllers are auto-discovered from `app/controllers/` and their methods are mapped to routes automatically.

## Convention Routes

Given a controller at `app/controllers/products.controller.js` exporting a `ProductsController` class, the following routes are auto-generated:

| HTTP Method | URI | Controller Method |
|------------|-----|-------------------|
| GET | `/products` | `index` |
| GET | `/products/:id` | `show` |
| GET | `/products/new` | `new` |
| POST | `/products` | `store` |
| GET | `/products/:id/edit` | `edit` |
| PUT | `/products/:id` | `update` |
| DELETE | `/products/:id` | `destroy` |

## Controllers

Controllers are classes with methods that return Hono-compatible responses:

```js
import { Controller } from 'foobarjs/core'

export default class ProductsController extends Controller {
  async index(c) {
    const products = await Product.all()
    return c.render('products/index', { products })
  }

  async show(c) {
    const product = await Product.find(c.req.param('id'))
    if (!product) return c.notFound()
    return c.render('products/show', { product })
  }
}
```

## Route Parameters

Route parameters are accessed via `c.req.param('name')`:

```js
async show(c) {
  const id = c.req.param('id')
}
```

## Form Method Spoofing

HTML forms don't support PUT, PATCH, or DELETE methods. Use a `_method` hidden field:

```html
<form action="/products/{{ product.id }}" method="POST">
  <input type="hidden" name="_method" value="PUT">
</form>
```

The method override middleware handles this transparently.

## Accessing the Router

The underlying `Hono` router is accessible via `foobar.app` if you need to define custom routes:

```js
foobar.app.get('/custom/route', (c) => c.text('Hello'))
```

This is typically done inside a plugin's `register()` method.
