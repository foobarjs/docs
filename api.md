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

**The API is authenticated by default.** Out of the box every endpoint requires
an authenticated request (`auth: 'session'`), so registering `foobarjs/api`
without any config does *not* expose your data publicly.

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
[Authentication](./authentication.md)); the API gate just requires one to be
present. Clients obtain a Bearer token by posting credentials to
`POST /api/auth/token` and inspect the current user with
`GET /api/auth/me` (see [Authentication](./authentication.md#api-login-endpoint)).

### Access rules

Configure access in `config/api.js`. Config values override the plugin defaults:

```js
// config/api.js
export default {
  // Storefront catalog: anyone can read, only authenticated users can write.
  auth: { read: false, write: 'session' },

  // Per-resource overrides, keyed by model name.
  models: {
    User: 'session',
    PersonalAccessToken: 'session',
  },
}
```

A rule can be:

| Rule | Meaning |
|------|---------|
| `'session'` / `true` / `'any'` | Require any authenticated user (session cookie **or** Bearer token) |
| `'token'` | Require a Bearer personal access token specifically |
| `false` / `'public'` / `'none'` | Public — no authentication |
| `{ read, write }` | Different rules for reads (`GET`) vs writes (`POST`/`PUT`/`DELETE`) |

Resolution precedence (most specific wins):

1. `models[ModelName]` in `config/api.js`
2. `static apiAuth` on the model class
3. the top-level `auth` option

The gate **fails closed**: if `foobarjs/auth` is not registered there is never a
current user, so every non-public rule rejects with `401`.

> **Zero-trust gates:** Authenticated API models must have a registered gate. If no gate exists for an authenticated model, the API returns `403 Forbidden`. Public endpoints (`auth: false`) bypass gate checks.

```js
// Per-model override on the model itself
class AuditLog extends Model {
  static apiAuth = 'token'   // this resource always requires a Bearer token
  static schema = { /* ... */ }
}
```

A request that fails its rule gets HTTP `401`:

```json
{ "error": "Unauthenticated" }
```

## Per-Resource Configuration (Api.resource)

Create files in `app/api/` to customize API behavior per model using the fluent `Api.resource()` builder:

```js
// app/api/order.api.js
import { Api } from 'foobarjs/api'
import Order from '../models/order.model.js'

export default Api.resource(Order)
  .only(['index', 'show', 'store'])       // only these routes
  .fillable(['name', 'status', 'total'])   // writable fields (POST/PUT)
  .hidden(['internal_notes', 'cost'])      // stripped from responses
  .filterable(['status', 'total'])         // allowed filter[field] params
  .sortable(['id', 'total', 'created_at']) // allowed sort fields
  .includable(['items', 'customer'])       // allowed include relations
  .perPage(50)                             // default page size
  .maxPerPage(200)                         // max allowed page size
  .auth({ read: 'public', write: 'token' })
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
| `auth(rule)` | Auth rule for this resource (highest precedence). Accepts the same values as `config/api.js` auth. |

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

When `Api.resource()` sets `.auth()`, it takes the highest precedence:

1. `Api.resource(Model).auth(...)` — per-resource config file
2. `config/api.js` → `models[ModelName]` — global per-model override
3. `static apiAuth` on the model class
4. `config/api.js` → `auth` — global default

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
