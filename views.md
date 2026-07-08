# Views

foobarjs uses Edge.js templates (`.html` files) with a Blade-like syntax. Views are stored in `app/views/` and use `@` directives for control structures.

## Rendering Views

Inside a controller method, use `c.render()`:

```js
async index(c) {
  return c.render('products/index', {
    products: await Product.all(),
    title: 'Products',
  })
}
```

Templates are referenced by their path relative to `app/views/`, without the extension.

## Convention-Based Rendering

If a controller method returns data instead of explicitly calling `c.render()`, foobarjs looks for a matching view based on the controller and action names.

Given `ProductsController.show` returning a single product, the framework looks for `app/views/products/show.html` and renders it with `{ product }`.

See [Controllers](./controllers.md#convention-based-views) for the full mapping.

## Error Views

You can override the built-in error pages by creating views in `app/views/errors/`:

```
app/views/errors/404.html
app/views/errors/500.html
app/views/errors/403.html
app/views/errors/419.html
```

For example:

```html
@layout('layouts/app')

@section('content')
  <h1>{{ status }} - Page Not Found</h1>
  <p>The page you requested could not be found.</p>
  @if(requestId)<p>Request ID: {{ requestId }}</p>@endif
@endsection
```

Lookup order for an error of status `N`:

1. `errors/N.html` (exact)
2. `errors/<class>xx.html` (e.g. `5xx.html` for any 5-series)
3. `errors/error.html`
4. `errors/500.html` (only for 5xx)
5. Built-in fallback page (production) or the Whoops-style dev page (development)

The following variables are available in error views:

| Variable | Description |
|----------|-------------|
| `status` | HTTP status code |
| `message` | Publicly-safe error message |
| `requestId` | Correlation ID (also exposed via `X-Request-Id` header) |
| `error` | The original error object |
| `stack` | Stack trace (in development only) |

See [Error Handling](./error-handling.md) for the full pipeline.

## View Globals

Every render receives these globals automatically:

| Global | Description |
|--------|-------------|
| `user` | Current authenticated user or `null` |
| `loggedIn` | Boolean |
| `cartCount` | Sum of `quantity` across `session.cart` |
| `flash` | Object of session flash messages (`{ success, error, ... }`) |
| `errors` | Flashed validation errors (`{ field: ['msg'], ... }`) |
| `old(key, fallback = '')` | Helper for reading previously-submitted input |

`errors` and `old` are populated automatically after a `ValidationError` redirect. Use them for form re-population and inline messages:

```html
<input name="email"
  value="{{ old('email') }}"
  class="{{ errors.email ? 'is-invalid' : '' }}">

@error('email')
  <div class="form-error">{{ message }}</div>
@enderror
```

The `@error('field') ... @enderror` directive renders its body only when the field has one or more errors. Inside the block a `message` local is bound to the first error message.

## Template Syntax

### Variables

```html
<h1>{{ title }}</h1>
<p>{{ product.description }}</p>
```

Variables are automatically HTML-escaped. For raw (unescaped) output:

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

```html
<form method="POST">
  @csrf
  <input name="email">
</form>
@csrf renders as: <input type="hidden" name="_csrf" value="...">
```

## Auto-Injected Variables

The following variables are automatically available in every view:

| Variable | Description |
|----------|-------------|
| `user` | The currently authenticated user (or null) |
| `loggedIn` | Boolean, whether the user is authenticated |
| `cartCount` | Number of items in the cart (from session) |
| `flash` | Object containing flash messages from the session |

## Components

```html
@component('alert', { type: 'success' })
  Operation completed successfully!
@endcomponent
```

Components are rendered by matching component files or inline handlers. Alpine.js component classes can provide reactive data.

## Alpine.js Integration

foobarjs is designed to work with Alpine.js for client-side interactivity:

```html
<div x-data="{ open: false }">
  <button @click="open = !open">Toggle</button>
  <div x-show="open" x-transition>
    Content
  </div>
</div>
```

Include Alpine.js via CDN in your layout or bundle it with your assets.

```html
<script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
```
