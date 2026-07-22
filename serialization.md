[← Back to docs](./README.md)

# Serialization

foobarjs provides a serialization layer for transforming models and collections into JSON responses.

## Basic Serializer

Serializers are placed in `app/serializers/` and define which fields to include:

```js
// app/serializers/product.serializer.js
import { Serializer } from 'foobarjs/serialization'

export default class ProductSerializer extends Serializer {
  static fields = ['id', 'name', 'price', 'slug']
  static include = {
    category: ['id', 'name'],
    orders: ['id', 'status', 'total'],
  }
}
```

## Using Serializers

```js
import { autoSerialize } from 'foobarjs/serialization'

const products = await Product.all()
const data = autoSerialize(Product, products)
```

### Serializer Methods

| Method | Description |
|--------|-------------|
| `serializer.only(record)` | Serialize a single record |
| `serializer.index(records)` | Serialize a collection |
| `serializer.show(record)` | Serialize for show view |

## Auto-Discovery

When using the API plugin, serializers are auto-discovered from `app/serializers/` and applied to API responses. The naming convention matches the model name (e.g., `Product` → `product.serializer.js`).

## Default Serialization

Without a custom serializer, models are serialized using `toJSON()`, which:
- Includes all non-hidden fields
- Field names (camelCase) map directly to JSON keys
- Respects the `hidden()` field option
- Includes the `id` field
- Includes any `appends` attributes

```js
const json = product.toJSON()
// { id: 1, name: '...', price: 29.99, description: '...' }
```
