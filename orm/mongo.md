[← Back to docs](../README.md)

---
title: MongoDB support
layout: default
nav_order: 60
---

# MongoDB support

foobarjs is built on MikroORM. MikroORM ships first-class MongoDB
support, and the framework has taken care to keep its ORM abstraction
compatible with document databases — most notably by never emitting
JOINs. Relationships load via IN-batched hydration, not JOINs; that
one design decision is what makes MongoDB support tractable at all.

As of **v0.5.2**, MongoDB dispatch covers the full query-builder
surface. `Model.query()`, `.first()`, `.get()`, `.paginate()`,
`.pluck()`, `.value()`, `.updateAll()`, `.deleteAll()`, `.with(...)`,
`.withCount / withSum / withAvg / withMin / withMax / withExists`
(including dotted-path through-aggregates), `.has() / doesntHave() /
whereHas / whereDoesntHave`, and the widget aggregate primitives all
work on mongo. `Db.boot()` bootstraps a mongo-driven connection end-
to-end.

### Newly wired in v0.5.2

- **`.whereColumn(a, op, b)` / `.orWhereColumn(a, op, b)`** — translates
  to `{$expr: {[op]: ['$a', '$b']}}`.
- **`.select([cols])`** — projection pushed as MikroORM `fields`.
- **`.distinct()`** — row-dedup post-fetch; `.distinct().pluck(col)`
  routes through the native mongo `distinct(col, match)`.
- **`.cursor()`** — streams via native `Collection.find(...)` cursor,
  yielding hydrated models.
- **`.cursorPaginate({perPage, cursor, column, direction})`** — keyset
  pagination using `$gt` / `$lt` on the key column.
- **`.having(col, op, val)` + `groupBy(col) + selectSum/Avg/Min/Max/Count`** —
  a full `$match → $group → $match(having) → $sort → $limit` pipeline
  supporting multiple named aggregates in one group.
- **`.whereYear / whereMonth / whereDay / whereDate`** — translated via
  `$expr` + `$year / $month / $dayOfMonth / $dateToString`.
- **`.whereMongo(filter) / .orWhereMongo(filter)`** — NEW mongo-side
  raw filter hatch, symmetric with `.whereRaw()`. Merges any native
  MongoDB filter into the generated `$match`. Throws on SQL
  connections with a pointer at `.whereRaw()`.

---

## The mental model

foobarjs treats MongoDB as one of MikroORM's supported drivers — same
`config('database.driver')` knob, same `Model.query()` API, same
connection registry. Two ways to use it:

**Whole-app mongo** — set `database.driver: 'mongodb'` in
`config/database.js`. Every model lives in mongo. Verified as of
v0.5.1: `Db.boot()` bootstraps the mongo schema (ObjectId `_id` with
serialized `id`), and CRUD + query round-trip through the public API.

**Named connection** — leave the default on SQL/SQLite, opt specific
models into mongo via `static connection = 'audit'` on the Model
class. Both engines coexist and cross-connection eager loading works
(each hop uses its own connection's adapter).

## Why the ORM design is mongo-compatible in principle

- **No JOINs.** Every relation loads with a separate `WHERE fk IN (?,
  ?, ...)` per hop. Documented in [database/workflow.md](../database/workflow.md).
  Mongo's `$in` operator maps 1:1 — the framework's IN-batched pattern
  IS the correct pattern for mongo.
- **Model id handling is type-agnostic.** IDs pass through as opaque
  tokens. Integer on SQL, ObjectId on mongo, both exposed as `.id`.
  The framework doesn't do arithmetic on ids; it passes them to
  `find()`, interpolates into URLs, stores in FK columns. Works.
- **Cross-connection relations work by IN-batching.** A mongo-backed
  `AuditLog` with a `userId` field can eager-load Users from a SQL-
  backed User model: fetch audit logs, collect user ids, fetch Users
  with `WHERE id IN (…)`, stitch in JS. Two round trips, two engines,
  one relation.

## What works today (via MikroORM + MongoAdapter)

Verified by parity tests under `framework/test/orm/mongo/parity/`:

- Basic CRUD — `Model.find(id)`, `.findOrFail(id)`, `item.save()`,
  `item.delete()`, `item.forceFill()`.
- Filters — `where(col, op, val)`, `whereIn`, `whereNotIn`,
  `whereBetween`, `whereNotBetween`, `whereNull`, `whereNotNull`,
  `orWhere`, nested `where(callback)` blocks.
- Ordering, limit, offset, take/skip, `latest()`, `oldest()`,
  `reorder()`.
- Pagination — `.paginate(page, perPage).get()` returns `{data, meta}`
  with `total`, `lastPage`, `from`, `to`.
- Terminal reads — `.first()`, `.get()`, `.pluck(col)`, `.value(col)`,
  `.findBy(col, val)`, `.findMany(ids)`.
- Bulk writes — `.updateAll(patch)` (native update), `.deleteAll()`
  (native delete), soft-delete via `deleted_at`.
- Eager loading — `.with('rel')` for belongsTo / hasMany / hasOne /
  belongsToMany (pivot collections). Nested subrelations work.
- Through-aggregates — `.withCount / withSum / withAvg / withMin /
  withMax / withExists`, one-hop and dotted-path (e.g.
  `.withSum('events.orders', 'total')`).
- Constraint callbacks — `.withCount('orders', q => q.where(...))`,
  `.whereHas('rel', q => ...)`, `.has()`, `.doesntHave()`.
- Widgets — `Widget.value / count / sum / avg / min / max / exists /
  trend / chart({bucket|groupBy})`. All route through the adapter.
- Admin dashboard — the dashboard's `Model.query().count()` sweep works
  on mongo-connected models.

## Genuinely SQL-only

Two escape hatches have no mongo counterpart — they explicitly hand you
a SQL construct. Each throws with a pointer at the mongo equivalent.

| Method | Why | Mongo alternative |
|---|---|---|
| `.toSQL()` / `.getQueryBuilder()` | returns MikroORM's SQL QueryBuilder | `.getCollection()` + aggregation pipeline |
| `.whereRaw(sql, bindings)` / `.orWhereRaw()` | raw SQL fragment | `.whereMongo(filter)` or `.getCollection()` |

Everything else on the query builder — filtering, ordering, projection,
grouping, having, aggregates, cursor, keyset pagination, date-part
wheres, column-vs-column comparisons, eager loads,
through-aggregates — dispatches to the `MongoAdapter` on mongo-
connected models.

### `foobar db:sync`, `foobar db:make`, `foobar db:migrate`

SQL-shaped. Not usable on mongo. Mongo is schemaless — collections
are auto-created on first write. If you want indexes, use
`Model.query().getCollection().createIndex(...)` in a boot script or
seeder.

## Escape hatches

Two symmetric escape hatches let you drop below the framework's ORM
for driver-specific work. Both throw a helpful error when called
against the wrong connection type, so you can't accidentally reach
for the wrong one.

### SQL side — `.getQueryBuilder()`

```js
// SQL-connected model
const qb = Order.query().getQueryBuilder()
// MikroORM's SQL QueryBuilder. Joins, subqueries, raw SQL, knex escape hatch.
```

On a mongo-connected model:

```
Error: Order.query().getQueryBuilder() is a SQL-only escape hatch.
For MongoDB-specific queries, use .getCollection() and compose an
aggregation pipeline via the native MongoDB Collection API. See
docs/orm/mongo.md.
```

### MongoDB side — `.getCollection()`

```js
// Mongo-connected model
const coll = AuditLog.query().getCollection()
const results = await coll.aggregate([
  { $match: { userId } },
  { $group: { _id: '$action', count: { $sum: 1 } } },
  { $sort: { count: -1 } },
]).toArray()
```

`.getCollection()` returns MikroORM's underlying MongoDB `Collection`.
The full driver-level API is available: `aggregate`, `distinct`,
`createIndex`, `bulkWrite`, `findOneAndUpdate`, etc.

On a SQL-connected model:

```
Error: Order.query().getCollection() is a MongoDB-only escape hatch.
For SQL-specific queries, use .getQueryBuilder() and compose a query
via MikroORM's SQL QueryBuilder. See docs/orm/mongo.md.
```

`.toSQL()` throws the equivalent error on mongo. Symmetric behavior
across the whole boundary — no silent pass-through, no opaque driver-
level errors.

## Enabling MongoDB in your app

**Whole-app**:

```js
// config/database.js
export default {
  driver: 'mongodb',
  database: 'foobar',
  host: process.env.MONGO_HOST || 'localhost',
  port: 27017,
  // OR: url: 'mongodb://user:pass@host:27017'
}
```

Bring your own mongo. `brew services start mongodb-community` on
macOS, or `mongod --dbpath ...` if you prefer.

**Named connection** (recommended today):

```js
// config/database.js
export default {
  driver: 'sqlite',
  database: 'foobar.db',
  connections: {
    audit: {
      driver: 'mongodb',
      database: 'foobar_audit',
      host: process.env.MONGO_HOST || 'localhost',
      port: 27017,
    },
  },
}

// app/models/audit-log.model.js
class AuditLog extends Model {
  static connection = 'audit'
  static schema = {
    action: Field.string().required(),
    userId: Field.string(),
    payload: Field.json(),
  }
  static timestamps = true
}
```

`AuditLog` reads/writes route to mongo; every other model stays on
SQLite. No cross-engine JOIN needed — because the framework doesn't
emit JOINs.

## MongoAdapter (implementation)

The "admin works on mongo" story ships as `framework/src/orm/mongo/
MongoAdapter.js`. Same pattern as this doc's escape-hatch symmetry:
**compose over MikroORM, don't extend or monkey-patch it.**

The adapter holds a reference to MikroORM's EntityManager and exposes
framework-native query methods, one per SQL code path the
`RelationQueryBuilder` used to hardcode:

| Method | Implementation |
|---|---|
| `runAggregate('count', col)` | `em.getConnection().countDocuments(where)` |
| `runAggregate('sum'|'avg'|'min'|'max', col)` | `em.aggregate([$match, $group])` |
| `runExists(where)` | `countDocuments(where, {limit:1}) > 0` |
| `runGroupedAggregate(col, kind)` | `em.aggregate([$match, $group: {_id: '$col', value: ...}])` |
| `runDateBucketAggregate(dateCol, bucket, kind)` | `em.aggregate([$match, $group])` + JS bucket-label rollup |
| `runIn(entity, fkCol, ids, opts)` | `em.find(entity, {[fkCol]: {$in: ids}}, opts)` |
| `runBelongsToMany(pivotColl, parentKey, ids)` | `em.getCollection(pivot).find({[parentKey]: {$in: ids}})` |
| `runWithCount(fkCol, parentIds)` | `em.aggregate([$match, $group: {_id: '$fkCol', count: {$sum: 1}}])` |
| `runWithAggregate(fkCol, parentIds, kind, col)` | `em.aggregate([$match, $group: {_id, agg}])` |

The `RelationQueryBuilder` dispatches per-model: on each terminal call
that touches a related model, it checks the RelatedModel's connection
and picks the SQL or mongo branch. Cross-engine relations (SQL parent
with a mongo child, or vice versa) route each hop through its own
adapter — no single pipeline crosses engines.

### Dotted-path aggregates

`.withCount('events.orders.attendees')` walks each hop separately and
uses the adapter of that hop's model. A three-hop path is three
round trips regardless of connection type — same shape as the SQL
path's IN-batched hops. No `$lookup` pipelines and no cross-engine
JOINs.

Not needed:

- A custom MikroORM driver. `em.aggregate` and `em.getCollection` are
  already exposed by MikroORM's mongo driver — the framework just
  needs to route to them instead of `em.createQueryBuilder`.
- Any change to MikroORM's driver loading. The adapter composes over
  the EntityManager MikroORM returns; MikroORM's own boot is
  unchanged.

## Known limitations

- **Cross-connection through-aggregates are per-hop, not one pipeline.**
  `AuditLog` (mongo) with `.withCount('user.actions')` where `User` is
  on SQL runs two round trips: the mongo hop through the mongo adapter,
  the SQL hop through the SQL path. No single aggregation pipeline
  crosses engines — that would require a distributed query planner.
  For typical cardinalities this is fine; for wide fan-out at each hop
  consider denormalizing.
- **No cross-engine transactions.** MikroORM's transaction contexts
  are per-connection. If you write to both a SQL model and a mongo
  model in one action, they're two independent operations; if one
  fails after the other succeeded, you own the compensation logic.
- **`foobar db:make` / `db:migrate` don't apply to mongo connections.**
  Mongo is schemaless. For indexes on mongo-connected models, use
  `Model.query().getCollection().createIndex(...)` in a boot script.

## See also

- [Database workflow](../database/workflow.md) — the schema-sync
  modes for SQL connections
- [ORM: getting started](./getting-started.md) — the query reference
