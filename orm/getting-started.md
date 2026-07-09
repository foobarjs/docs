# Eloquent ORM: Getting Started

foobarjs includes an Active Record ORM called Foobar ORM, inspired by Laravel Eloquent. It wraps MikroORM internally for database operations.

## Defining Models

Models extend the `Model` class and define a `static schema`:

```js
import { Model, Field } from 'foobarjs/orm'

export default class Product extends Model {
  static schema = {
    name: Field.string().required(),
    slug: Field.string().required().unique(),
    description: Field.text().nullable(),
    price: Field.float().required(),
    stock: Field.number().default(0),
    published: Field.boolean().default(false),
    category: Field.belongsTo('Category'),
  }
}
```

### Available Field Types

| Method | Database Type | Description |
|--------|--------------|-------------|
| `Field.string()` | VARCHAR(255) | String value |
| `Field.text()` | TEXT | Long text |
| `Field.number()` | INTEGER | Integer |
| `Field.float()` | FLOAT (REAL in SQLite) | Floating point |
| `Field.decimal(p, s)` | DECIMAL (REAL in SQLite) | Precise decimal |
| `Field.boolean()` | TINYINT(1) | Boolean |
| `Field.date()` | DATE | Date only |
| `Field.datetime()` | DATETIME | Date and time |
| `Field.json()` | JSON | JSON object |

### Field Constraints

```js
Field.string().required()           // Not null
Field.string().nullable()           // Allow null
Field.string().unique()             // Unique index (auto-named)
Field.string().unique('name')       // Unique index with explicit name
Field.string().index()              // Non-unique index (auto-named)
Field.string().index('name')        // Non-unique index with explicit name
Field.string().default('value')     // Default value
Field.string().maxLength(255)       // Max length
Field.string().minLength(3)         // Min length
Field.string().hidden()             // Exclude from JSON
Field.number().unsigned()           // Unsigned integer
Field.enum('a', 'b', 'c')           // Enum values
```

### Indexes

foobarjs surfaces database indexes as a first-class model concern.

#### Field-level indexes

```js
class Post extends Model {
  static schema = {
    title: Field.string().required().index(),           // idx_posts_title
    slug: Field.string().unique(),                       // posts_slug_unique
    email: Field.string().unique('uniq_posts_email'),    // named unique
    publishedAt: Field.datetime().nullable().index(),    // idx_posts_published_at
    author: Field.belongsTo('User'),                     // auto-indexed FK
  }
}
```

- `.index()` — non-unique index on the column. Optional string argument sets the index name.
- `.unique()` — unique constraint. Optional string argument sets the constraint name.

#### Auto-index foreign keys

Every `belongsTo` field is automatically indexed. This mirrors Rails/Ecto and prevents the most common source of admin-panel slowness (unindexed FKs cause full-table scans on `WHERE user_id = ?`).

Opt out per-field:

```js
Field.belongsTo(User).noIndex()
```

Or globally in `config/database.js`:

```js
export default {
  driver: 'sqlite',
  database: 'foobar.db',
  autoIndexForeignKeys: false,   // disable auto-FK-indexing (default: true)
}
```

#### Compound and named indexes

For multi-column, partial, or expression indexes, use `static indexes`:

```js
class Order extends Model {
  static schema = {
    user: Field.belongsTo(User),
    status: Field.string(),
    createdAt: Field.datetime(),
  }

  static indexes = [
    // Compound index on (user_id, created_at) — auto-named
    { columns: ['user', 'createdAt'] },

    // Named compound
    { columns: ['status', 'createdAt'], name: 'idx_orders_status_recent' },

    // Partial index (Postgres, SQLite 3.8+)
    { columns: ['status'], where: { status: 'pending' } },

    // Expression index (Postgres)
    { expression: 'LOWER(email)', name: 'idx_users_email_lower' },

    // Driver-specific type
    { columns: ['searchDoc'], type: 'gin' },        // Postgres
    { columns: ['title', 'body'], type: 'fulltext' },// MySQL
  ]
}
```

For compound unique constraints:

```js
static uniques = [
  { columns: ['user', 'externalId'] },
  { columns: ['tenantId', 'slug'], name: 'uniq_orders_tenant_slug' },
]
```

For CHECK constraints (Postgres, MySQL 8.0.16+, SQLite):

```js
static checks = [
  { expression: 'total >= 0' },
  { expression: "status IN ('pending','shipped','delivered')" },
  { expression: 'valid_from < valid_until', name: 'chk_orders_date_range' },
]
```

#### Naming convention

Auto-generated names follow this pattern (matches Rails/Ecto):

| Kind | Format | Example |
|------|--------|---------|
| Index (field) | `idx_<table>_<col>` | `idx_orders_status` |
| Index (compound) | `idx_<table>_<col1>_<col2>` | `idx_orders_user_id_created_at` |
| Index (FK auto) | `idx_<table>_<col>_id` | `idx_orders_user_id` |
| Unique (field) | `<table>_<col>_unique` | `users_email_unique` |
| Unique (compound) | `uniq_<table>_<col1>_<col2>` | `uniq_orders_user_id_slug` |
| Check | `chk_<table>_<col1>_<col2>` | `chk_orders_total` |

Names longer than 63 characters (Postgres limit) are truncated with an 8-character hash suffix for uniqueness.

#### Introspection

```js
Order.getIndexes()   // [{ columns, dbColumns, name, unique, type, where, source }, ...]
Order.getUniques()
Order.getChecks()
```

`source` is one of `'field'` (from `Field.foo().index()`), `'auto-fk'` (auto-created for `belongsTo`), or `'model'` (from `static indexes` / `static uniques`).

Pass `{ includeAutoFk: false }` to `getIndexes()` to exclude the auto-FK entries.

#### Verifying indexes

The `foobar` CLI lets you compare declared indexes against what's actually in the database, and confirm the query planner is using them:

```bash
# List all declared indexes with presence status
foobar db indexes

# Only show missing indexes; exit 1 if any (great for CI)
foobar db indexes --missing

# Show indexes present in DB but not declared
foobar db indexes --extra

# Machine-readable
foobar db indexes --json

# Explain a query to verify the plan
foobar db explain 'SELECT * FROM orders WHERE user_id = ? AND status = ?' \
  --bind 42 --bind pending
```

Rule of thumb: **index every column you filter (`WHERE`) or sort (`ORDER BY`) on**. Foreign keys are already covered by the auto-FK default; everything else you touch in a hot query should be explicit.

### Table Name

The default table name is derived from the class name: snake-case, plus a
naive trailing `s`. So `Product` → `products`, `OrderItem` → `order_items`,
and `User` → `users`.

The framework does **not** run English pluralization heuristics. Words that
don't pluralize with a bare `+s` need an override:

| Class | Default | Reality |
|-------|---------|---------|
| `Product` | `products` | correct |
| `OrderItem` | `order_items` | correct |
| `Category` | `categorys` | needs override → `categories` |
| `Person` | `persons` | needs override → `people` |
| `Bus` | `buss` | needs override → `buses` |

Set `static tableName` explicitly whenever the default is wrong:

```js
class Category extends Model {
  static tableName = 'categories'
  static schema = { /* ... */ }
}

export default Category
```

### Timestamps & Soft Deletes

Timestamps (`created_at`, `updated_at`) are enabled by default. Soft deletes are opt-in:

```js
class Post extends Model {
  static timestamps = true    // default
  static softDelete = true    // adds deleted_at column
  static schema = { ... }
}
```

### Applying schema changes

Once your models are set up, get their tables into the database:

- **File migrations (recommended):** `foobar db make` scans the diff between
  your models and the last applied snapshot, writes a migration file, and
  `foobar db migrate` applies it. See
  [Database migrations](../database/migrations.md).
- **Dev shortcut:** `foobar db sync` applies the diff directly. Fast for
  local prototyping, refuses to run in production, refuses destructive
  changes without `--force`.

The framework never auto-syncs your schema at server boot when migration
files exist.

## Retrieving Models

```js
// Find by primary key
const product = await Product.find(1)

// Find or fail (throws 404)
const product = await Product.findOrFail(1)

// All records
const products = await Product.all()

// Where clause
const products = await Product.where('published', true).get()

// First match
const product = await Product.where('slug', 'my-product').first()

// Pluck — single column values
const names = await Product.pluck('name')
// → ['Product A', 'Product B', ...]

// Pluck with key — key-value pairs
const map = await Product.pluck('name', 'id')
// → { 1: 'Product A', 2: 'Product B', ... }

// Conditional query building
const products = await Product.when(showInStock, qb => qb.where('stock', '>', 0)).get()
const products = await Product.unless(hideUnpublished, qb => qb.where('published', true)).get()

// Filter by relation existence
const products = await Product.whereHas('category', qb => {
  qb.where('name', 'Electronics')
}).get()
const products = await Product.orWhereHas('tags', qb => {
  qb.where('name', 'sale')
}).get()

// Query builder
const products = await Product.query()
  .where('price', '>', 50)
  .where('published', true)
  .orderBy('name', 'asc')
  .limit(10)
  .get()

// Pagination — call .get() to run the query; returns a { data, meta } envelope
const result = await Product.query().paginate(1, 15).get()
// result.data  → array of models
// result.meta  → { currentPage, lastPage, perPage, total, from, to }

// Chunk — process large result sets in batches
await Product.chunk(100, (records, page) => {
  for (const product of records) {
    // process each batch
  }
})
// Return false to stop early:
await Product.chunk(100, (records, page) => {
  return page < 3  // stop after 3 chunks
})

// Lazy — iterate one record at a time (fetches in batches internally)
for await (const product of Product.lazy(200)) {
  // process one product at a time
}

// Cursor — stream records with a database cursor (memory-efficient)
for await (const product of Product.cursor()) {
  // process one record at a time without loading all into memory
}

// All three work with filters:
for await (const product of Product.where('published', true).lazy(50)) {
  // ...
}
```

## Creating & Updating

```js
// Create
const product = await Product.create({
  name: 'New Product',
  slug: 'new-product',
  price: 29.99,
})

// First or create — find first match or create with merged data
const product = await Product.firstOrCreate(
  { slug: 'my-product' },
  { name: 'My Product', price: 19.99 }
)

// Update or create — find and update, or create
const product = await Product.updateOrCreate(
  { slug: 'my-product' },
  { name: 'Updated Name', price: 24.99 }
)

// Or build and save
const product = new Product({ name: 'Draft', price: 0 })
product.price = 19.99
await product.save()

// Update
const product = await Product.find(1)
product.name = 'Updated Name'
await product.save()

// Increment / decrement (persists immediately)
const product = await Product.find(1)
await product.increment('stock', 5)
await product.decrement('stock', 3)
```

## Mass Assignment

`create()`, `updateOrCreate()`, the `new Model(data)` constructor, and the auto
[API](../api.md)'s `POST`/`PUT` handlers all assign many attributes at once from
an untrusted object. To stop a client from setting columns you never intended
(privilege escalation via `{ "isAdmin": true }`, overwriting `id`, and so on),
mass assignment is filtered by two static lists:

```js
class User extends AuthenticableModel {
  static schema = {
    name: Field.string(),
    email: Field.string(),
    isAdmin: Field.boolean().default(false),
    roles: Field.json().nullable(),
  }

  static guarded = ['id', 'isAdmin', 'roles', 'created_at', 'updated_at']
}
```

- **`static guarded`** (default `['id', 'created_at', 'updated_at', 'deleted_at']`)
  is a deny-list: every field is fillable *except* those named. Use `['*']` to
  guard everything.
- **`static fillable`** (default `null`) is an allow-list. When set to a
  non-empty array it takes precedence over `guarded` — only the listed fields
  are fillable.

Guarded keys (and keys absent from the schema) are silently skipped, not errors:

```js
// isAdmin is guarded → ignored here
const user = await User.create({ name: 'Ada', email: 'a@x.com', isAdmin: true })
user.isAdmin   // → false (the default)

// Direct property assignment is NOT mass assignment — it always works:
user.isAdmin = true
await user.save()
```

### fill() and forceFill()

```js
const product = new Product()
product.fill(req.body)        // respects fillable / guarded
product.forceFill(trusted)    // bypasses the guard (schema fields only)

Product.isFillable('price')   // → true / false
```

Use `fill()` for anything derived from user input. Reach for `forceFill()` only
in trusted server-side code that legitimately needs to set a guarded field — for
example a seeder promoting a user with `forceFill({ isAdmin: true })`.

## Deleting

```js
const product = await Product.find(1)
await product.delete()

// With soft deletes, delete() sets deleted_at instead of removing
```

## Transactions

Run a block of database operations inside a transaction. If the callback throws, everything is rolled back.

```js
import { Model } from 'foobarjs/orm'

await Model.transaction(async () => {
  const order = await Order.create({ user: 1, total: 99.99 })
  await OrderItem.create({ order: order.id, product: 5, quantity: 1 })
  await Payment.create({ order: order.id, amount: 99.99 })
})
```

You can also use `Db.transaction()`:

```js
import { Db } from 'foobarjs/orm'

await Db.transaction(async () => {
  // ...
})
```

Throwing inside the callback automatically rolls back:

```js
try {
  await Model.transaction(async () => {
    await Order.create({ total: 50 })
    throw new Error('Payment failed')
  })
} catch (err) {
  // Order was not created
}
```

## Validation

Validation runs automatically on every `save()`. On failure, a `ValidationError` is thrown:

```js
try {
  await product.save()
} catch (err) {
  if (err.name === 'ValidationError') {
    console.log(err.errors)   // { name: ['Name is required'] }
  }
}
```

## Lazy Loading

Load relations on an existing model after it's already been fetched:

```js
const product = await Product.find(1)
await product.load('category')
console.log(product.category.name)

// Nested
await product.load({ category: ['products'] })

// Multiple
await product.load('category', 'tags')
```

## Fresh & Refresh

```js
// fresh() — get a new instance from the database
const fresh = await product.fresh()

// refresh() — re-fetch and update the current instance in-place
await product.refresh()
```

## Scopes

Define reusable query constraints declaratively with a static `scopes()` method:

```js
class Product extends Model {
  static schema = {
    name: Field.string(),
    price: Field.float(),
    published: Field.boolean().default(false),
    expiresAt: Field.date().nullable(),
  }

  static scopes() {
    return {
      published: (qb) => qb.where('published', true),
      cheap: (qb, maxPrice) => qb.where('price', '<=', maxPrice),
      expired: (qb) => qb.where('expiresAt', '<=', new Date()),
    }
  }
}

// Call scopes as methods on the query builder
const products = await Product.query().published().get()
const cheap = await Product.query().cheap(10).get()
const results = await Product.query().published().cheap(20).get()

// Scopes work with all query builder entry points
const product = await Product.where('id', 1).published().first()
```

The old programmatic API is also available for dynamic registration:

```js
Product.scope('published', qb => qb.where('published', true))
Product.applyScope('published', queryBuilder)
```

## Appends

Include virtual/computed attributes in JSON output by defining `static appends`:

```js
class Product extends Model {
  static appends = ['discountedPrice']

  get discountedPrice() {
    return (this.price * 0.9).toFixed(2)
  }
}

const product = await Product.find(1)
JSON.stringify(product)
// → { "id": 1, "name": "...", "discountedPrice": "..." }
```

Appended values are computed at serialization time via the class getter. The getter can access any model value through `this.$attributes[fieldName]` or via the property shorthand (`this.price`).

## Accessors & Mutators

Define custom getter/setter logic for model attributes using native JavaScript class getters/setters:

```js
class User extends Model {
  static schema = {
    name: Field.string(),
    email: Field.string(),
    password: Field.string().hidden(),
  }

  get password() {
    return this.$attributes.password
  }

  set password(value) {
    this.$attributes.password = bcrypt.hashSync(value, 10)
  }
}

// Mutator fires on direct property assignment:
user.password = 'plaintext'
console.log(user.password)  // → bcrypt hash

// Loading from DB writes directly to $attributes, bypassing the setter:
// _fromEntity sets this.$attributes[key] = entity[colName] without
// going through user-defined getters/setters. This prevents
// double-processing (e.g., hashing an already-hashed password).
```

Model values live in `this.$attributes`, keyed by field name (camelCase). The `_entity` object holds the MikroORM persistence layer and should not be accessed directly in custom accessors.

A custom getter/setter on the prototype replaces the framework-generated one entirely — both reads (`_fromEntity`, `toJSON`, `refresh`) and writes (`save`, `constructor`) go through your implementation.

Custom setters should call `this.markDirty(fieldName)` when the underlying value changes, so the ORM knows which fields to persist on `save()`:

```js
class User extends Model {
  static schema = {
    name: Field.string(),
    status: Field.string(),
  }

  set status(value) {
    this.$attributes.status = value.toLowerCase()
    this.markDirty('status')
  }
}
```

## Dirty Tracking

The ORM tracks which fields have changed since the model was loaded or created. This lets `save()` update only modified columns.

```js
const product = await Product.find(1)
product.name = 'New Name'

product.isDirty()              // true — any field changed
product.isDirty('name')        // true
product.isDirty('price')       // false
product.isDirty(['name', 'price']) // true if either changed
```

You can also mark fields dirty manually from custom setters or business logic:

```js
product.markDirty('metadata')
```

Dirty state is cleared after a successful `save()` or `refresh()`.

## Eager Loading

```js
// belongsTo
const products = await Product.with('category').get()
const product = await Product.with('category', 'orders').find(1)

// hasMany
const categories = await Category.with('products').get()

// hasOne
const user = await User.with('profile').find(1)

// belongsToMany
const products = await Product.with('tags').get()

// Nested relations
const products = await Product.with({ category: { products: [] } }).get()

// With count (hasMany, belongsToMany)
const categories = await Category.withCount('products').get()
const products = await Product.withCount('tags').get()
```

Eager loading batches related queries to avoid N+1 performance issues.

Nested eager loading is also batched — foobarjs collects all child instances across all parents into one array and re-runs a single fetch per depth level, so `Product.with({ category: ['products'] })` executes at most 3 queries no matter how many products or categories are involved.

### Eager-loading constraints

Pass a callback instead of an array to filter the loaded relation set:

```js
const categories = await Category
  .with({ products: qb => qb.where('published', true).orderBy('price', 'desc').limit(5) })
  .get()
```

The callback receives a `RelationQueryBuilder` for the related model. Any `where`/`orderBy`/`limit` chained on it is applied when the related rows are fetched.

## Aggregates

foobarjs surfaces database-side aggregates on both `Model` and its query builder. Every aggregate respects any active `where`, `whereHas`, and soft-delete conditions.

```js
// Static shortcuts on Model
await Product.count()                  // COUNT(*)
await Product.sum('price')             // SUM(price)
await Product.avg('price')
await Product.min('price')
await Product.max('price')
await Product.exists()                 // → boolean
await Product.doesntExist()            // → boolean
await Product.value('name')            // first row's name

// Chained on the query builder
await Product.query().where('published', true).count()
await Product.where('category', catId).sum('price')
await Product.where('featured', true).exists()

// Distinct counts
await Product.query().distinct().count('category')
```

`sum`/`avg`/`min`/`max` return a `Number` (or `null` if there are no rows). `count` always returns a `Number`.

## Filter primitives

Beyond `.where(field, op, value)`, the builder exposes shortcuts for the common SQL predicates:

```js
Product.query().whereIn('id', [1, 2, 3])
Product.query().whereNotIn('id', [1, 2, 3])
Product.query().whereNull('deleted_at')
Product.query().whereNotNull('published_at')
Product.query().whereBetween('price', [10, 100])
Product.query().whereNotBetween('price', [10, 100])

// Date helpers (portable across sqlite/mysql/postgres)
Order.query().whereDate('created_at', '2026-07-08')
Order.query().whereYear('created_at', 2026)
Order.query().whereMonth('created_at', 7)
Order.query().whereDay('created_at', 8)

// Compare two columns
Product.query().whereColumn('price', '=', 'stock')

// Escape hatch: raw SQL fragments (add to WHERE with bindings)
Product.query().whereRaw('price > ? OR stock < ?', [50, 5])
Product.query().orWhereRaw('name LIKE ?', ['%sale%'])
```

### Closure-scoped nested WHERE

Group conditions into `AND (A OR B)` blocks by passing a callback:

```js
Product.query()
  .where('published', true)
  .where(qb => qb
    .where('featured', true)
    .orWhere('price', '<', 5))
  .get()
// → WHERE published = true AND (featured = true OR price < 5)
```

Everything inside the closure is grouped as a single predicate. Mixing `where` and `orWhere` inside the closure produces `(A OR B OR ...)`.

## Multi-column sorting

`orderBy` now appends rather than overwriting, so chaining multiple times produces a compound sort:

```js
Product.query().orderBy('price', 'asc').orderBy('name', 'desc').get()

Product.query().orderByDesc('created_at').get()
Product.query().latest()               // === orderBy('created_at', 'desc')
Product.query().oldest()               // === orderBy('created_at', 'asc')

Product.query().latest().reorder().orderBy('id', 'asc')  // clear then re-add
```

## GroupBy, HAVING, and aggregate SELECT

Group rows and compute aggregates per bucket:

```js
// Count orders per status
const rows = await Order.query()
  .groupBy('status')
  .selectCount('*', 'count')
  .get()
// → [{ status: 'pending', count: 12 }, { status: 'shipped', count: 100 }, ...]

// Sum revenue per category
const rows = await Product.query()
  .groupBy('category')
  .selectSum('price', 'total')
  .get()

// Filter groups with HAVING
const rows = await Order.query()
  .groupBy('status')
  .selectCount('*', 'cnt')
  .having('cnt', '>', 10)
  .get()
```

Grouped queries return **plain row objects** (not `Model` instances) since grouping produces synthetic rows. `selectCount`, `selectSum`, `selectAvg`, `selectMin`, `selectMax` all take `(column, alias)` and add aggregate expressions to the SELECT list. `selectAggregate(kind, column, alias)` is the generic form.

`having(column, operator, value)` filters on aggregated values. `havingRaw(sql, bindings)` is the escape hatch for expressions the DSL can't cover.

### Date bucketing (time-series charts)

Portable date-truncation grouping helpers:

```js
// Orders per day for the last 30 days
await Order.query()
  .whereDate('created_at', '>=', from)
  .groupByDay('created_at')
  .selectCount('*', 'count')
  .get()
// → [{ bucket: '2026-07-01', count: 12 }, { bucket: '2026-07-02', count: 8 }, ...]

// Revenue per month
await Order.query()
  .groupByMonth('created_at')
  .selectSum('total', 'revenue')
  .get()

// Also available: groupByWeek, groupByYear
```

Under the hood, each helper emits the correct dialect-specific expression (`strftime` on SQLite, `DATE_FORMAT` on MySQL, `to_char` on Postgres). MongoDB is not supported for date bucketing — call from a SQL driver.

## Relation aggregates

Fetch relation counts, sums, and other aggregates alongside models — batched into single grouped queries per relation.

```js
// Count of items in the relation
await Category.query().withCount('products').get()
// → each category has a `products_count` number

// Sum of a column across the relation
await Category.query().withSum('products', 'price').get()
// → each category has `products_sum_price`

// Also: withAvg, withMin, withMax
await Category.query().withAvg('products', 'price').get()
// → category.products_avg_price
await Category.query().withMax('products', 'created_at').get()
// → category.products_max_created_at

// Existence flag
await Category.query().withExists('products').get()
// → each category has a boolean `products_exists`
```

All `withCount`/`withSum`/... use a single `GROUP BY` query per relation, so a list of 500 categories with a `withSum('products', 'price')` is 2 queries total: 1 for the categories, 1 for the sums.

### whereHas / has / doesntHave

Filter models by relation existence:

```js
// Products that belong to at least one category matching the callback
await Product.whereHas('category', qb => qb.where('name', 'Electronics')).get()

// Products with any orders (no inner conditions)
await Product.query().has('orders').get()

// Products WITHOUT any orders
await Product.query().doesntHave('orders').get()

// whereDoesntHave with inner constraints
await Product.query().whereDoesntHave('orders', qb => qb.where('status', 'pending')).get()

// orWhereHas
await Product.where('featured', true).orWhereHas('tags', qb => qb.where('name', 'sale')).get()
```

## Bulk operations

Operate on many rows in a single SQL statement without instantiating models. **Hooks and validation are skipped**; use these for high-volume writes where per-model logic is not required.

```js
// Bulk insert (single INSERT with multi-VALUES)
await Product.insertMany([
  { name: 'A', slug: 'a', price: 1 },
  { name: 'B', slug: 'b', price: 2 },
  { name: 'C', slug: 'c', price: 3 },
])

// Bulk update matching rows
await Product.query().where('category', catId).updateAll({ stock: 0 })

// Bulk delete matching rows (soft-delete-aware)
await Product.query().where('discontinued', true).deleteAll()
```

`updateAll` respects `belongsTo` fields (writes `${field}_id`) and updates `updated_at` when timestamps are enabled. `deleteAll` on a model with `static softDelete = true` sets `deleted_at` instead of removing the row.

## Pagination

### Offset pagination

```js
const result = await Product.query().paginate(page, perPage).get()
// result.data → array of models
// result.meta.currentPage, lastPage, perPage, total, from, to
```

### Cursor (keyset) pagination

For large tables or infinite-scroll feeds, cursor pagination is O(1) regardless of page depth:

```js
const page1 = await Product.query()
  .orderBy('id', 'asc')
  .cursorPaginate({ perPage: 20 })
// page1.data, page1.meta.nextCursor, page1.meta.hasMore

const page2 = await Product.query()
  .orderBy('id', 'asc')
  .cursorPaginate({ perPage: 20, cursor: page1.meta.nextCursor })
```

Options: `perPage`, `cursor`, `column` (default `id`), `direction` (`asc`|`desc`).

## Debug helpers

```js
// Inspect the SQL a query would run
const { sql, bindings } = Product.query().where('price', '>', 10).toSQL()

// Log SQL to console mid-chain
Product.query().where('name', 'X').dump().get()
```

## Escape hatch

Drop down to the underlying MikroORM QueryBuilder when the DSL can't express what you need:

```js
const mikroQb = Product.query().where('published', true).getQueryBuilder()
// mikroQb is a MikroORM QueryBuilder with your conditions applied
const rows = await mikroQb.select(['id', 'name']).execute('all')
```

Anything MikroORM supports — CTEs, joins, sub-queries, `INSERT INTO ... SELECT`, `FOR UPDATE` locks — is available on the returned builder.
