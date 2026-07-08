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

## Raw Hono access

For anything the router API doesn't cover, drop down to Hono directly:

```js
// Inside a plugin's register(foobar) or from routes/web.js if you accept
// the second argument:
export default function (router, foobar) {
  foobar.app.get('/exotic/*', (c) => { /* ... */ })
}
```

## Next steps

- Write the controllers your routes point at: [Controllers](./controllers.md)
- Render responses: [Views](./views.md)
- Auto-generated REST API for models: [API](./api.md)
