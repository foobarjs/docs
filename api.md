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
`POST /api/auth/token` (see [API Token Authentication](./authentication.md#api-login-endpoint)).

### Access rules

Configure access in `config/api.js`. Config values override the plugin defaults:

```js
// config/api.js
export default {
  // Storefront catalog: anyone can read, only authenticated users can write.
  auth: { read: false, write: 'session' },

  // Per-resource overrides, keyed by table name.
  models: {
    users: 'session',
    personal_access_tokens: 'session',
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

1. `models[tableName]` in `config/api.js`
2. `static apiAuth` on the model class
3. the top-level `auth` option

The gate **fails closed**: if `foobarjs/auth` is not registered there is never a
current user, so every non-public rule rejects with `401`.

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
  "created_at": "...",
  "updated_at": "..."
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
