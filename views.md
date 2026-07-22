[← Back to docs](./README.md)

{% raw %}
# Views

foobarjs uses **JSX** as its view engine. Views live in `app/views/` and are
rendered by controllers via `this.render(...)` or by convention. The resolution
order is: `.jsx` → `.tsx`.

## Rendering views

Inside a controller method (extending `Controller`):

```js
async index() {
  return this.render('products/index', {
    products: await Product.all(),
    title: 'Products',
  })
}
```

Templates are referenced by path relative to `app/views/`, without the
extension. `'products/index'` resolves to `app/views/products/index.jsx` (or
`.tsx`).

Outside a controller (e.g. from an inline callback in `routes/web.js`):

```js
router.get('/pricing', (c) => c.render('marketing/pricing', { plans }))
```

`c.render(template, data)` is the underlying primitive — `this.render()` is a
thin wrapper around it.

Views must be explicitly rendered from a controller action via `this.render()`.
Returning a plain object from a controller action always returns JSON — there
is no automatic view lookup.

See [Controllers](./controllers.md) for the auto response contract.

## JSX views

JSX views are function components that receive props and return JSX. They use
[Hono's JSX runtime](https://hono.dev/docs/guides/jsx) — no React required.

Class names in foobarjs JSX use `class` (not `className`) — the JSX runtime
treats them as plain HTML attributes.

### Writing a JSX view

`foobar serve` registers the JSX loader for you — no extra script wiring
in `package.json` is required. Any `.jsx` (or `.tsx`) file under
`app/views/` is importable and renderable via `Controller#render`.

```jsx
// app/views/products/index.jsx
export default function ProductsIndex({ products, title }) {
  return (
    <div>
      <h1>{title}</h1>
      {products.map(p => (
        <div key={p.id}>
          <h3>{p.name}</h3>
          <p>${p.price}</p>
        </div>
      ))}
    </div>
  )
}
```

Each `.jsx` file exports a default function component. The data object passed to
`this.render()` becomes the component's props.

### Layouts with composition

Layouts are just components that render `children`:

```jsx
// app/views/layouts/App.jsx
export default function AppLayout({ title, children }) {
  return (
    <html lang="en">
      <head>
        <meta charset="UTF-8" />
        <title>{title || 'My App'}</title>
        <link rel="stylesheet" href="/css/app.css" />
      </head>
      <body>
        <nav>/* ... */</nav>
        <main>{children}</main>
        <script src="/js/app.js"></script>
      </body>
    </html>
  )
}
```

```jsx
// app/views/products/index.jsx
import AppLayout from '../layouts/App.jsx'

export default function ProductsIndex({ products }) {
  return (
    <AppLayout title="Products">
      <h1>Products</h1>
      {products.map(p => <ProductCard product={p} />)}
    </AppLayout>
  )
}
```

### Raw HTML

Use `raw()` from `hono/html` to inject unescaped HTML:

```jsx
import { raw } from 'hono/html'

export default function Post({ post }) {
  return (
    <article>
      <h1>{post.title}</h1>
      <div>{raw(post.htmlContent)}</div>
    </article>
  )
}
```

Never call `raw()` on user-supplied content — it bypasses HTML escaping.

### Inline scripts

Use `dangerouslySetInnerHTML` for inline `<script>` blocks:

```jsx
<script dangerouslySetInnerHTML={{ __html: `
  document.addEventListener('DOMContentLoaded', () => {
    // client-side code
  })
`}} />
```

### Shared data and view globals

Data shared via `c.share()` or middleware is available in components. Reach
for the `useView()` hook from `foobarjs/jsx` to read view globals (`user`,
`loggedIn`, `flash`, `errors`, `old`) and any values added via `c.share()`:

```jsx
import { useView } from 'foobarjs/jsx'

export default function Header({ cartCount }) {
  const { user, loggedIn } = useView()
  return (
    <nav>
      <a href="/">Home</a>
      {loggedIn
        ? <span>Welcome, {user.name}</span>
        : <a href="/login">Login</a>}
      <a href="/cart">Cart ({cartCount})</a>
    </nav>
  )
}
```

The data object you pass to `this.render()` becomes the component's props;
view globals and shared data flow through `useView()`.

## Error views

Override the built-in error pages by dropping `.jsx` files in `app/views/errors/`:

```
app/views/errors/404.jsx
app/views/errors/500.jsx
app/views/errors/403.jsx
app/views/errors/419.jsx
```

```jsx
export default function NotFound({ status, message, requestId }) {
  return (
    <div>
      <h1>{status} — Page Not Found</h1>
      <p>{message}</p>
      {requestId && <p>Request ID: {requestId}</p>}
    </div>
  )
}
```

Lookup order for an error of status `N`:

1. `errors/N.jsx` (exact)
2. `errors/<class>xx.jsx` (e.g. `5xx.jsx` for any 5-series)
3. `errors/error.jsx`
4. `errors/500.jsx` (only for 5xx)
5. Built-in fallback page (production) or the diagnostic page (development)

Variables available in error views:

| Variable | Description |
|----------|-------------|
| `status` | HTTP status code |
| `message` | Publicly-safe error message |
| `requestId` | Correlation ID (also on `X-Request-Id` response header) |
| `error` | The original error object |
| `stack` | Stack trace (development only) |

See [Error handling](./error-handling.md) for the full pipeline.

## View globals

Every render exposes these through `useView()`:

| Global | Description |
|--------|-------------|
| `user` | Current authenticated user, or `null` |
| `loggedIn` | Boolean |
| `flash` | Flash messages (`{ success, error, ... }`) |
| `errors` | Flashed validation errors (`{ field: ['msg'], ... }`) |
| `old(key, fallback = '')` | Read previously-submitted input |
| *shared data* | Any values added via `c.share(key, value)` in middleware |

Values shared via `c.share()` in middleware are merged into every view's
context automatically. See [Middleware: Sharing data](./middleware.md#sharing-data-with-views)
for details.

`errors` and `old` are populated automatically after a `ValidationError`
redirect. Use them to re-populate forms and render inline messages:

```jsx
import { useView } from 'foobarjs/jsx'

export default function LoginForm() {
  const { errors, old } = useView()
  return (
    <form method="post" action="/login">
      <input name="email" value={old ? old('email') : ''} />
      {errors?.email && <div class="form-error">{errors.email[0]}</div>}
      <button type="submit">Login</button>
    </form>
  )
}
```

## CSRF

foobarjs protects against CSRF at the middleware layer using an **origin check**
(built-in CSRF middleware): unsafe requests (`POST`/`PUT`/`PATCH`/`DELETE`) must
come from an allowed origin. Same-origin form submissions are accepted
automatically, so forms need no per-request token — just submit them normally:

```jsx
<form method="POST" action="/posts">
  <input name="title" />
  <button type="submit">Save</button>
</form>
```

## Client-side interactivity

foobarjs is unopinionated about client-side JavaScript. Add whichever
library you prefer (Alpine.js, htmx, Stimulus, Vue islands, plain vanilla)
by including it in your layout or bundling it into `public/`.

## Next steps

- Add controllers that render these views: [Controllers](./controllers.md)
- Handle form submissions and validation: [Validation](./validation.md)
- Sharing data with views from middleware: [Middleware](./middleware.md#sharing-data-with-views)
- Sessions and flash messages: [Session](./session.md)
{% endraw %}
