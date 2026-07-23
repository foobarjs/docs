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

That said, **the framework is not fully mongo-ready today.** The ORM's
aggregate and relation-execution code paths currently use MikroORM's
SQL query builder (`em.createQueryBuilder`) which the mongo driver
doesn't expose. This page describes what works now, what breaks, why,
and the escape hatches that let you drop below the framework for
mongo-native operations.

The full "make admin work on mongo" story ships as a follow-up under
the `MongoAdapter` module (see the roadmap section below).

---

## The mental model

foobarjs treats MongoDB as one of MikroORM's supported drivers — same
`config('database.driver')` knob, same `Model.query()` API, same
connection registry. Two ways to use it:

**Whole-app mongo** — set `database.driver: 'mongodb'` in
`config/database.js`. Every model lives in mongo. Currently limited
(see What works / What breaks below).

**Named connection** (recommended today) — leave the default on
SQL/SQLite, opt specific models into mongo via `static connection =
'audit'` on the Model class. The rest of the app stays on the SQL
path; only the mongo-connected model uses the mongo primitives.

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

## What works today (via MikroORM's mongo primitives)

Around 80% of the admin surface. Verified by tracing every callsite:

- Basic CRUD — `Model.find(id)`, `.findOrFail(id)`, `item.save()`,
  `item.delete()`, `item.forceFill()`. Route via `em.findOne` /
  `em.persist` / `em.remove`.
- Filters — `where(col, op, val)`, `whereIn`, `whereNull`,
  `whereNotNull`, `orWhere`. MikroORM's op map (`$eq`, `$in`, `$gte`,
  `$like`, `$exists`) translates all of these on the mongo driver.
- Ordering, limit, offset.
- Basic pagination (`paginate(perPage)`) — two calls under the hood,
  works on mongo.
- Eager loading (`.with('rel')`) via MikroORM's `populate` option —
  works.

Because these primitives all work, the admin's list, show, create,
edit, delete flows work on a mongo-backed model **as long as the
model has no admin widgets or trend charts pointing at it.**

## What breaks today (SQL-only code paths)

Every one of these routes through `em.createQueryBuilder(...)` — the
SQL query builder that doesn't exist on MikroORM's mongo driver.

### Widgets

| Widget | Status | Why |
|---|---|---|
| `Widget.value(name, resolver)` | ✅ works | Arbitrary user code; whatever the resolver does |
| `Widget.count(name, Model)` | ❌ throws | Uses `_runAggregate` → SQL query builder |
| `Widget.sum/avg/min/max` | ❌ throws | Same code path |
| `Widget.exists` | ❌ throws | Uses `mikroRaw('1 as ex')` |
| `Widget.trend(...)` (bucketed) | ❌ throws | Explicit throw in `_dateExpr` |
| `Widget.chart({bucket})` | ❌ throws | Same bucketing code path |
| `Widget.chart({groupBy})` (no bucket) | ❌ throws | Uses `selectAggregate` with `mikroRaw` SQL fragments |

The framework's own dashboard rendering at `AdminController.js:103`
runs `Model.query().count()` for every model with `adminConfig.dashboard =
true`. On a mongo-connected model, that first count throws — the
dashboard 500s.

### Through-aggregates

`.withCount('orders')`, `.withSum('events.orders', 'total')`, and the
whole dotted-path aggregate family use hand-written SQL fragments via
`mikroRaw` for the grouped subqueries. Not usable on mongo. The admin
uses `.withCount(...)` for detail-page relation badges — those would
throw too.

### Relation loading (IN-batched, but currently SQL-only)

The eager-load pattern is mongo-friendly (`$in`) but the terminal
execution uses `em.createQueryBuilder(...)`. `.with('event')` on a
mongo-connected model with a mongo-connected `Event` will throw.

**Cross-connection eager loading (mongo model → SQL model, or the
reverse)** is a design-level "should work" — but the current
implementation's SQL-side path doesn't know to route the mongo hop
through mongo primitives. Deferred until `MongoAdapter` lands.

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

**Whole-app** (limited today; use for models that don't need widgets):

```js
// config/database.js
export default {
  driver: 'mongodb',
  database: 'foobar',
  host: process.env.MONGO_HOST || 'localhost',
  port: 27017,
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

## Roadmap — MongoAdapter

The "admin works on mongo" story lands as a `framework/src/orm/mongo/
MongoAdapter.js` module. Same pattern as this doc's escape-hatch
symmetry: **compose over MikroORM, don't extend or monkey-patch it.**

The adapter holds a reference to MikroORM's EntityManager and exposes
framework-native query methods:

| Method | Implementation |
|---|---|
| `runAggregate('count', col)` | `em.getCollection().countDocuments(where)` |
| `runAggregate('sum'|'avg'|'min'|'max', col)` | `em.aggregate([$match, $group])` |
| `runExists(where)` | `countDocuments(where, {limit:1}) > 0` |
| `runGroupedAggregate(col, kind)` | `em.aggregate([$match, $group: {_id: '$col', value: ...}])` |
| `runDateBucketAggregate(dateCol, bucket, kind)` | `em.aggregate([$match, $group: {_id: {$dateTrunc: {date: '$col', unit: 'day'}}, value: ...}])` |
| `runIn(entity, fkCol, ids, opts)` | `em.find(entity, {[fkCol]: {$in: ids}}, opts)` |
| `runPaginated(where, opts)` | `em.find(entity, where, {limit,offset})` + `countDocuments(where)` |
| `translateConditions(conditions)` | Framework's `_conditions` op codes → mongo filter shape |

When it lands:

- Widget aggregates, trend/chart widgets, and dashboard cards work on
  mongo-connected models.
- Relation loading (`.with('rel')`) works.
- Through-aggregates (`.withCount('rel')`) will be added in a
  follow-up; degrades to zero in the meantime.

Not needed:

- A custom MikroORM driver. `em.aggregate` and `em.getCollection` are
  already exposed by MikroORM's mongo driver — the framework just
  needs to route to them instead of `em.createQueryBuilder`.
- Any change to MikroORM's driver loading. The adapter composes over
  the EntityManager MikroORM returns; MikroORM's own boot is
  unchanged.

## Known limitations

- **Cross-connection through-aggregates.** `AuditLog` (mongo) with
  `.withCount('user.actions')` where `User` is on SQL — the pipeline
  can't hop engines mid-aggregation. Would require the framework to
  run separate aggregates per hop and stitch in JS. Deferred.
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
