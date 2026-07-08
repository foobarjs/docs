# API

The `foobarjs/api` plugin generates a RESTful JSON API for all your models automatically.

## Setup

```json
{
  "plugins": ["foobarjs/api"]
}
```

## Generated Endpoints

All API routes are prefixed with `/api` (configurable):

| Method | URI | Description |
|--------|-----|-------------|
| GET | `/api/products` | List products |
| GET | `/api/products/:id` | Show product |
| POST | `/api/products` | Create product |
| PUT | `/api/products/:id` | Update product |
| DELETE | `/api/products/:id` | Delete product |

## Query Parameters

### Listing

```
GET /api/products?include=category&fields=id,name,price&filter[published]=true&sort=-price&page=2&perPage=20
```

| Parameter | Description |
|-----------|-------------|
| `include` | Comma-separated relation names to eager load |
| `fields` | Comma-separated columns to return |
| `filter[field]` | Filter by field value |
| `sort` | Field to sort by. Prefix with `-` for descending |
| `page` | Page number (default: 1) |
| `perPage` | Items per page (default: 15) |

### Response Format

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
    "total": 10,
    "page": 1,
    "perPage": 15,
    "lastPage": 1,
    "from": 1,
    "to": 10
  }
}
```

### Single Resource

```json
{
  "data": {
    "id": 1,
    "name": "Wireless Headphones",
    "price": 79.99,
    "description": "Bluetooth noise-cancelling headphones",
    "category": 1,
    "created_at": "...",
    "updated_at": "..."
  }
}
```

## Creating & Updating

Send a JSON body with the appropriate fields:

```json
POST /api/products
{
  "name": "New Product",
  "slug": "new-product",
  "price": 29.99,
  "category_id": 1
}
```

### Validation Errors

Validation failures return 422 with error details:

```json
{
  "error": "Validation failed",
  "errors": {
    "name": ["The name field is required."],
    "price": ["The price must be a number."]
  }
}
```

## Serializers

Create serializer classes in `app/serializers/` to customize API output:

```js
// app/serializers/product.serializer.js
import { Serializer } from 'foobarjs/serialization'

export default class ProductSerializer extends Serializer {
  static fields = ['id', 'name', 'price', 'slug']
  static include = {
    category: ['id', 'name'],
  }
}
```

Serializers are auto-discovered and applied by the API plugin.

## API Documentation

The `foobarjs/api-docs` plugin generates OpenAPI 3.1.0 documentation automatically:

```json
{
  "plugins": ["foobarjs/api-docs"]
}
```

Access the docs at `/api/docs`. A machine-readable OpenAPI spec is available at `/api/docs.json`. The documentation UI uses Scalar for an interactive experience.
