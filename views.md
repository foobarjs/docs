# Views

foobarjs supports two view engines: **JSX** (recommended) and an Edge-inspired
**Blade** template engine (`.html` files with `@` directives). Both live in
`app/views/` and are rendered by controllers via `this.render(...)` or by
convention.

When both a `.jsx` and `.html` file exist for the same template, **JSX takes
priority**. The resolution order is: `.jsx` → `.tsx` → `.html`.

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
`.tsx`, or `.html`).

Outside a controller (e.g. from an inline callback in `routes/web.js`):

```js
router.get('/pricing', (c) => c.render('marketing/pricing', { plans }))
```

`c.render(template, data)` is the underlying primitive — `this.render()` is a
thin wrapper around it.

## Convention-based rendering

If a controller action returns data instead of a `Response`, foobarjs looks
for a matching view and renders it automatically.

`ProductsController.show` returning a single product → looks for
`app/views/products/show.jsx` (or `.html`) and renders it with `{ product }`.

If no matching view exists, the data becomes JSON.

See [Controllers](./controllers.md#convention-based-views) for the full mapping.

## JSX views

JSX views are function components that receive props and return JSX. They use
[Hono's JSX runtime](https://hono.dev/docs/guides/jsx) — no React required.

### Setup

Add the JSX loader to your dev and start scripts so Node can import `.jsx` files:

```json
{
  "scripts": {
    "dev": "node --import foobarjs/jsx-loader foobar serve --dev",
    "start": "node --import foobarjs/jsx-loader foobar serve"
  }
}
```

### Writing a JSX view

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

Blade's `@layout`/`@section`/`@yield` pattern maps to component composition
with `children`:

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

Use `raw()` from `hono/html` to inject unescaped HTML (equivalent to Blade's
`{!! !!}`):

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

### Inline scripts

Use `dangerouslySetInnerHTML` for inline `<script>` blocks:

```jsx
<script dangerouslySetInnerHTML={{ __html: `
  document.addEventListener('DOMContentLoaded', () => {
    // client-side code
  })
`}} />
```

### Shared data in JSX

Data shared via `c.share()` or middleware is available through props. View
globals (`user`, `loggedIn`, `flash`, `errors`, `old`) are merged into the
props passed to your component:

```jsx
export default function Header({ user, loggedIn, cartCount }) {
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

## Error views

Override the built-in error pages by dropping files in `app/views/errors/`.
Both JSX and Blade formats work:

```
app/views/errors/404.jsx   (or .html)
app/views/errors/500.jsx
app/views/errors/403.jsx
app/views/errors/419.jsx
```

JSX example:

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

Blade example:

```html
@layout('layouts/app')

@section('content')
  <h1>{{ status }} — Page Not Found</h1>
  <p>The page you requested could not be found.</p>
  @if(requestId)<p>Request ID: {{ requestId }}</p>@endif
@endsection
```

Lookup order for an error of status `N`:

1. `errors/N.html` (exact)
2. `errors/<class>xx.html` (e.g. `5xx.html` for any 5-series)
3. `errors/error.html`
4. `errors/500.html` (only for 5xx)
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

Every render receives these automatically:

| Global | Description |
|--------|-------------|
| `user` | Current authenticated user, or `null` |
| `loggedIn` | Boolean |
| `flash` | Flash messages (`{ success, error, ... }`) |
| `errors` | Flashed validation errors (`{ field: ['msg'], ... }`) |
| `old(key, fallback = '')` | Read previously-submitted input |
| *shared data* | Any values added via `c.share(key, value)` in middleware |

Values shared via `c.share()` in middleware are merged into every view's
data automatically. See [Middleware: Sharing data](./middleware.md#sharing-data-with-views)
for details.

`errors` and `old` are populated automatically after a `ValidationError`
redirect. Use them to re-populate forms and render inline messages.

In Blade:

```html
<input name="email"
  value="{{ old('email') }}"
  class="{{ errors.email ? 'is-invalid' : '' }}">

@error('email')
  <div class="form-error">{{ message }}</div>
@enderror
```

In JSX:

```jsx
export default function LoginForm({ errors, old }) {
  return (
    <form method="post" action="/login">
      <input name="email" value={old ? old('email') : ''} />
      {errors?.email && <div class="form-error">{errors.email[0]}</div>}
      <button type="submit">Login</button>
    </form>
  )
}
```

The Blade `@error('field') ... @enderror` directive renders its body only when
the field has one or more errors. Inside the block, a `message` local is bound
to the first error message.

## Blade template syntax

### Variables

```html
<h1>{{ title }}</h1>
<p>{{ product.description }}</p>
```

Variables are HTML-escaped. For raw output:

```html
{!! html_content !!}
```

### Layouts

```html
<!-- app/views/products/index.html -->
@layout('layouts/app')

@section('content')
  <h1>Products</h1>
  @foreach(products as product)
    <div>{{ product.name }}</div>
  @endforeach
@endsection
```

```html
<!-- app/views/layouts/app.html -->
<!DOCTYPE html>
<html>
<head>
  <title>@yield('title', 'My App')</title>
</head>
<body>
  @yield('content')
</body>
</html>
```

### Conditionals

```html
@if(loggedIn)
  <p>Welcome, {{ user.name }}</p>
@else
  <a href="/login">Login</a>
@endif
```

### Loops

```html
@foreach(products as product)
  <div>
    <h3>{{ product.name }}</h3>
    <p>${{ product.price }}</p>
  </div>
@endforeach
```

### Includes

```html
@include('components/product-card', { product })
```

### CSRF

foobarjs protects against CSRF at the middleware layer using an **origin check**
(built-in CSRF middleware): unsafe requests (`POST`/`PUT`/`PATCH`/`DELETE`) must come from an
allowed origin. Same-origin form submissions are accepted automatically, so
forms need no per-request token — just submit them normally:

```html
<form method="POST" action="/posts">
  <input name="title">
  <button type="submit">Save</button>
</form>
```

> The old `@csrf` directive has been removed. It emitted an inert hidden field
> whose token was never populated; origin-based protection replaces it. If a
> template still contains `@csrf` it now renders nothing, so you can delete it.

## Blade components

```html
@component('alert', { type: 'success' })
  Operation completed successfully!
@endcomponent
```

Components live in `app/views/components/` and receive the passed data plus
`{{ $slot }}` for the wrapped body.

## Choosing between JSX and Blade

| | JSX | Blade |
|---|---|---|
| **Syntax** | Standard JavaScript/JSX | `@` directives, `{{ }}` interpolation |
| **Layouts** | Component composition with `children` | `@layout` / `@section` / `@yield` |
| **Components** | Import and render directly | `@include` / `@component` |
| **Raw HTML** | `raw()` from `hono/html` | `{!! expr !!}` |
| **Type safety** | Full IDE support, autocomplete | String-based |
| **Reuse** | Standard ES module imports | Template includes |

Both engines work side-by-side. You can migrate incrementally — JSX files
take priority over `.html` files with the same name, so drop in a `.jsx`
replacement and the old template is bypassed automatically.

## Client-side interactivity

foobarjs is unopinionated about client-side JavaScript. Add whichever
library you prefer (Alpine.js, htmx, Stimulus, Vue islands, plain vanilla)
by including it in your layout or bundling it into `public/`.

## Next steps

- Add controllers that render these views: [Controllers](./controllers.md)
- Handle form submissions and validation: [Validation](./validation.md)
- Sharing data with views from middleware: [Middleware](./middleware.md#sharing-data-with-views)
- Sessions and flash messages: [Session](./session.md)
