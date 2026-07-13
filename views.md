# Views

foobarjs ships an Edge-inspired template engine (`.html` files with `@`
directives and `{{ }}` interpolation). Templates live in `app/views/` and are
rendered by controllers via `this.render(...)` or by convention.

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
extension. `'products/index'` renders `app/views/products/index.html`.

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
`app/views/products/show.html` and renders it with `{ product }`.

If no matching view exists, the data becomes JSON.

See [Controllers](./controllers.md#convention-based-views) for the full mapping.

## Error views

Override the built-in error pages by dropping files in `app/views/errors/`:

```
app/views/errors/404.html
app/views/errors/500.html
app/views/errors/403.html
app/views/errors/419.html
```

Example:

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
| `cartCount` | Sum of `quantity` across `session.cart` |
| `flash` | Flash messages (`{ success, error, ... }`) |
| `errors` | Flashed validation errors (`{ field: ['msg'], ... }`) |
| `old(key, fallback = '')` | Read previously-submitted input |

`errors` and `old` are populated automatically after a `ValidationError`
redirect. Use them to re-populate forms and render inline messages:

```html
<input name="email"
  value="{{ old('email') }}"
  class="{{ errors.email ? 'is-invalid' : '' }}">

@error('email')
  <div class="form-error">{{ message }}</div>
@enderror
```

The `@error('field') ... @enderror` directive renders its body only when the
field has one or more errors. Inside the block, a `message` local is bound to
the first error message.

## Template syntax

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

## Components

```html
@component('alert', { type: 'success' })
  Operation completed successfully!
@endcomponent
```

Components live in `app/views/components/` and receive the passed data plus
`{{ $slot }}` for the wrapped body.

## Client-side interactivity

foobarjs is unopinionated about client-side JavaScript. Add whichever
library you prefer (Alpine.js, htmx, Stimulus, Vue islands, plain vanilla)
by including it in your layout or bundling it into `public/`.

## Next steps

- Add controllers that render these views: [Controllers](./controllers.md)
- Handle form submissions and validation: [Validation](./validation.md)
- Sessions and flash messages: [Session](./session.md)
