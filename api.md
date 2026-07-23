[← Back to docs](./README.md)

# API

The `foobarjs/api` plugin generates a RESTful JSON API for every registered
model automatically.

## Setup

```js
// config/app.js
export default {
  // ...
  plugins: ['foobarjs/api'],
}
```

## Generated endpoints

All API routes are prefixed with `/api` by default. Override with plugin
options — see [Prefix](#prefix) below.

| Method | URI | Description |
|--------|-----|-------------|
| GET | `/api/<tableName>` | List — always returns a paginated `{ data, meta }` envelope |
| GET | `/api/<tableName>/:id` | Show |
| POST | `/api/<tableName>` | Create — returns 201 |
| PUT | `/api/<tableName>/:id` | Update |
| DELETE | `/api/<tableName>/:id` | Delete |

The URL segment is the model's `getTableName()`. `Product` → `/api/products`,
`OrderItem` → `/api/order_items`.

## Authentication

**The API requires authentication by default (v0.4.0+).** Auto-mounted
resources are wrapped with the `'auth'` middleware — an unauthenticated
request returns `401`. Opt individual resources (or specific verbs) out
with `.public()`.

Authentication is resolved by the `foobarjs/auth` plugin, which must be
registered **before** `foobarjs/api`:

```js
// config/app.js
export default {
  plugins: ['foobarjs/auth', 'foobarjs/api'],
}
```

`foobarjs/auth` populates the current user from either a session cookie or an
`Authorization: Bearer <token>` header (see
[Authentication](./authentication.md)). Clients obtain a Bearer token by
posting credentials to `POST /api/auth/token` and inspect the current user
with `GET /api/auth/me`.

### The two-verb model

Every API resource is configured with just two verbs plus a gate helper:

- `.middleware(list, ...verbs?)` — attach middleware to specific verbs (or all if no verbs given).
- `.public(...verbs?)` — sugar for `.withoutMiddleware(['auth'], ...verbs)`. No args = whole resource public.
- `.withoutMiddleware(list, ...verbs?)` — remove named middleware (mirrors the route builder).
- `.can(action, Model, ...verbs?)` — attach gate middleware.

Verbs are the five REST actions: `index`, `show`, `store`, `update`, `destroy`.

### Global default

Set `auth.default` in `config/auth.js` to control the ambient policy:

```js
// config/auth.js
export default {
  default: 'auth',    // Auto-mount is auth-required (this is the v0.4.0 default)
  // default: 'public',  // Auto-mount is fully public (old v0.3.x behavior)
}
```

### Per-resource opt-out

```js
// app/api/pricing.api.js — a public catalog endpoint
import { Api } from 'foobarjs/api'
import Pricing from '../models/pricing.model.js'

export default Api.resource(Pricing).public()
```

```js
// app/api/order.api.js — reads are public, writes require auth
import { Api } from 'foobarjs/api'
import Order from '../models/order.model.js'

export default Api.resource(Order)
  .public('index', 'show')                 // GET routes are open
  .middleware('auth', 'store', 'update', 'destroy')  // writes require login
  .can('refund', Order, 'update')          // extra gate on writes
```

```js
// app/api/analytics.api.js — bearer token only
import { Api } from 'foobarjs/api'
import AnalyticsEvent from '../models/analytics-event.model.js'

export default Api.resource(AnalyticsEvent)
  .middleware('tokenOnly')  // rejects session-only users; requires a bearer token
```

### Why auth-required by default

Prior to v0.4.0 auto-mount was public. That default silently exposed any
model added to `app/models/` on the first request. Under the new default a
new model is invisible until its owner explicitly opts it out with
`.public()` — the safe direction to fail.

### Deprecated shapes (removal: v0.5.0)

| Deprecated | Replacement |
|------------|-------------|
| `ApiResource.auth('session')` / `.auth(true)` / `.auth('auth')` | `.middleware('auth')` |
| `ApiResource.auth('token')` | `.middleware('tokenOnly')` |
| `ApiResource.auth('public')` / `.auth(false)` / `.auth('none')` | `.public()` |
| `ApiResource.auth({ read, write })` | Split into `.public('index','show')` / `.middleware('auth','store','update','destroy')` |
| `config('api.auth')` | `config('auth.default')` + per-resource opt-outs |
| `config('api.models')` | Per-resource `.public()` / `.middleware()` in `app/api/*.api.js` |
| `config('auth.guard', 'required')` | `config('auth.default', 'auth')` |
| `config('auth.guard', 'optional')` | `config('auth.default', 'public')` |
| `static auth = false` on a Controller | `static withoutMiddleware = ['auth']` |

Each deprecated shape is still accepted for the v0.4.x release and emits
a boot-time warning naming the offending resource or config key.

> **Gate enforcement:** Gates auto-fire on both admin and API for models
> with a registered gate. Gates only run when there is an authenticated
> user — for public routes the API skips the gate entirely.

## Per-Resource Configuration (Api.resource)

Create files in `app/api/` to customize API behavior per model using the fluent `Api.resource()` builder:

```js
// app/api/order.api.js
import { Api } from 'foobarjs/api'
import Order from '../models/order.model.js'

export default Api.resource(Order)
  .middleware('auth')                      // protect this resource
  .only('index', 'show', 'store')          // only these routes
  .fillable(['name', 'status', 'total'])   // writable fields (POST/PUT)
  .hidden(['internal_notes', 'cost'])      // stripped from responses
  .filterable(['status', 'total'])         // allowed filter[field] params
  .sortable(['id', 'total', 'created_at']) // allowed sort fields
  .includable(['items', 'customer'])       // allowed include relations
  .perPage(50)                             // default page size
  .maxPerPage(200)                         // max allowed page size
  .public('index', 'show')                 // GETs are public; writes still require auth
```

### Configuration Options

| Method | Description |
|--------|-------------|
| `only(routes)` | Allow only these routes: `'index'`, `'show'`, `'store'`, `'update'`, `'destroy'` |
| `except(routes)` | Exclude these routes (inverse of `only`) |
| `fillable(fields)` | Only these fields are accepted in `POST`/`PUT` bodies |
| `hidden(fields)` | These fields are stripped from all responses |
| `filterable(fields)` | Only these fields can be used with `filter[field]=value` |
| `sortable(fields)` | Only these fields can be used with `?sort=field` |
| `includable(relations)` | Only these relations can be eager loaded with `?include=` |
| `perPage(n)` | Default page size for this resource |
| `maxPerPage(n)` | Maximum allowed `?perPage=` value |
| `middleware(list, ...verbs?)` | Attach middleware to all verbs (no verb args) or the given verbs |
| `public(...verbs?)` | Sugar for `withoutMiddleware(['auth'], ...verbs)`; makes the whole resource (or specific verbs) unauth |
| `withoutMiddleware(list, ...verbs?)` | Remove named middleware from the resolved chain |
| `can(action, Model, ...verbs?)` | Attach gate middleware; verbs restrict application |

### Route filtering

`only` and `except` control which HTTP endpoints are registered. Requests to disabled routes return 404.

```js
// Read-only API: no create, update, or delete
Api.resource(Product).only(['index', 'show'])

// Everything except delete
Api.resource(Order).except(['destroy'])
```

### Field restrictions

`fillable` restricts which fields are accepted in request bodies. Fields not in the list are silently dropped. `hidden` strips fields from all JSON responses (list, show, create, update).

```js
Api.resource(User)
  .fillable(['name', 'email'])    // can't set isAdmin via API
  .hidden(['password', 'roles'])  // never exposed in responses
```

### Query restrictions

`filterable`, `sortable`, and `includable` restrict which query parameters are honored. Unknown filter keys, sort fields, or include relations are silently ignored.

### Auth precedence

The chain is composed at boot in this order:

1. Global `auth.default` — prepends `'auth'` unless the resource marks the verb public.
2. Per-resource `.middleware(list, ...verbs)` — appended in registration order.
3. Per-resource `.can(action, Model, ...verbs)` — gate middleware appended after `.middleware()`.
4. `.public(...verbs)` / `.withoutMiddleware(['auth'], ...verbs)` — removes the default `'auth'` entry from the target verbs only.

### Discovery

API config files are auto-discovered from `app/api/`. The filename convention is `<model>.api.js` (e.g., `order.api.js`). The file must export an `Api.resource()` instance as the default export.

## Opting out

Hide a model from the API entirely with a static property:

```js
class InternalLog extends Model {
  static api = false
  // ...
}
```

Models with `static api = false` are skipped during route registration — all
REST endpoints for that model return 404.

Framework-internal models (`QueueJob`, `FailedJob`, `NotificationModel`,
`PersonalAccessToken`, `AdminExport`) default to `static api = false` since
they are system models not intended for public API access.

## Query parameters

### Listing

```
GET /api/products?include=category&filter[published]=true&sort=-price&page=2&perPage=20
```

| Parameter | Description |
|-----------|-------------|
| `include` | Comma-separated relation names to eager load |
| `filter[field]` | Filter by field value |
| `sort` | Field to sort by. Prefix with `-` for descending |
| `page` | Page number (default: 1) |
| `perPage` | Items per page (default: 15, max: 100) |

## Response shapes

### List

Lists are **always paginated** — the endpoint never loads a whole table into
memory. The response is a `{ data, meta }` envelope; `data` is an array of the
model (or serializer output when a serializer is defined). Page through with
`?page=` and `?perPage=`; `perPage` is capped at 100. Both defaults are
configurable in `config/api.js` (`perPage`, `maxPerPage`).

```json
{
  "data": [
    {
      "id": 1,
      "name": "Wireless Headphones",
      "price": 79.99,
      "category": 1
    }
  ],
  "meta": {
    "currentPage": 1,
    "lastPage": 5,
    "perPage": 15,
    "total": 92,
    "from": 1,
    "to": 15
  }
}
```

### Show / Create / Update

Returns a bare object:

```json
{
  "id": 1,
  "name": "Wireless Headphones",
  "price": 79.99,
  "description": "Bluetooth noise-cancelling headphones",
  "category": 1,
  "createdAt": "...",
  "updatedAt": "..."
}
```

### Delete

```json
{ "message": "Deleted" }
```

## Creating and updating

Send a JSON body with the appropriate fields:

```
POST /api/products
Content-Type: application/json

{
  "name": "New Product",
  "slug": "new-product",
  "price": 29.99,
  "category_id": 1
}
```

Returns HTTP 201 with the created object.

> **Mass assignment is guarded.** Both `POST` and `PUT` only accept fields the
> model marks fillable. Structural and privileged columns (`id`, timestamps, and
> anything in a model's `static guarded`) are silently ignored, so a client
> cannot escalate by sending e.g. `{ "isAdmin": true }`. See
> [Mass assignment](./orm/getting-started.md#mass-assignment).

### Errors

Every API error uses the framework's **canonical envelope** — the same shape the
central error handler emits everywhere else — and is always JSON (the API marks
each request so error rendering never depends on the client's `Accept` header).
Each error response also carries an `X-Request-Id` header matching the body's
`requestId`, so a client-reported id ties directly to the server log line.

```json
{ "error": "Unauthenticated", "status": 401, "requestId": "3f2a…-uuid" }
```

| Status | When |
|--------|------|
| 401 | Missing/invalid authentication for a gated resource |
| 404 | No record with the given id |
| 422 | Validation failed (adds an `errors` map) |
| 500 | Unhandled server error (message masked in production) |

Validation failures (422) add the field `errors` map:

```json
{
  "error": "Validation failed",
  "status": 422,
  "requestId": "3f2a…-uuid",
  "errors": {
    "name": ["The name field is required."],
    "price": ["The price must be a number."]
  }
}
```

In debug mode the envelope also includes `exception`, `stack`, and (when present)
`cause`. See [Validation](./validation.md) for how validation errors are produced.

## Prefix

Instantiate `ApiPlugin` explicitly with options to override the prefix:

```js
// config/app.js
import ApiPlugin from 'foobarjs/api'

export default {
  // ...
  plugins: [
    'foobarjs/auth',
    new ApiPlugin({ prefix: '/v2' }),
  ],
}
```

Endpoints move to `/v2/<tableName>`.

## Serializers

Create serializer classes in `app/serializers/` to shape API output. The
`foobarjs/api` plugin auto-loads a serializer per model based on filename:
`Product` → `app/serializers/product.serializer.js` (lowercased class name).

```js
// app/serializers/product.serializer.js
import { Serializer } from 'foobarjs/serialization'

class ProductSerializer extends Serializer {
  static fields = ['id', 'name', 'price', 'slug']
  static include = {
    category: ['id', 'name'],
  }
}

export default ProductSerializer
```

If no serializer file exists, the model's `toJSON()` is used, which respects
`Field.string().hidden()` and `static appends`.

## API documentation

The `foobarjs/api-docs` plugin generates OpenAPI 3.1.0 documentation
automatically:

```js
plugins: ['foobarjs/api', 'foobarjs/api-docs']
```

Access the docs at `/api/docs` (Scalar UI). A machine-readable OpenAPI spec
is available at `/api/docs.json`.

## See also

- [Serialization](./serialization.md)
- [Validation](./validation.md)
- [Conventions](./conventions.md#auto-rest-api)
