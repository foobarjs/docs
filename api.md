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
| GET | `/api/<tableName>` | List (paginated when `?page=` is provided) |
| GET | `/api/<tableName>/:id` | Show |
| POST | `/api/<tableName>` | Create — returns 201 |
| PUT | `/api/<tableName>/:id` | Update |
| DELETE | `/api/<tableName>/:id` | Delete |

The URL segment is the model's `getTableName()`. `Product` → `/api/products`,
`OrderItem` → `/api/order_items`.

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
| `page` | Page number — triggers paginated response shape |
| `perPage` | Items per page (default: 15, only used when `page` is set) |

## Response shapes

### List — without `?page=`

Returns a bare array of the model (or an array of serializer output if a
serializer is defined for the model):

```json
[
  {
    "id": 1,
    "name": "Wireless Headphones",
    "price": 79.99,
    "category": 1
  }
]
```

### List — with `?page=`

Returns a `{ data, meta }` envelope:

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
    "currentPage": 2,
    "lastPage": 5,
    "perPage": 20,
    "total": 92,
    "from": 21,
    "to": 40
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

### Validation errors

Validation failures return 422:

```json
{
  "error": "Validation failed",
  "errors": {
    "name": ["The name field is required."],
    "price": ["The price must be a number."]
  }
}
```

See [Validation](./validation.md) for how the underlying errors are produced.

### Not found

`GET /api/products/9999` on a missing id returns 404:

```json
{ "error": "Not found" }
```

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
