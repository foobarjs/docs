[← Back to docs](../README.md)

---
title: MongoDB support
layout: default
nav_order: 60
---

# MongoDB support

foobarjs treats MongoDB as one of MikroORM's supported drivers. Same
`config('database.driver')` knob, same `Model.query()` API, same
connection registry, same escape-hatch story. The framework's query
builder dispatches per-connection to a `MongoAdapter` for mongo-backed
models — nothing in your controllers, admin actions, or widgets needs
to know which engine is behind a given model.

The per-topic mongo callouts live on the ORM pages next to the features
they modify:

- [ORM: getting started](./getting-started.md) — id handling, query
  builder deltas, aggregates, pagination, cursors, `.distinct()`,
  `.getCollection()` / `.whereMongo()` escape hatches.
- [ORM: relationships](./relationships.md) — why the no-JOINs
  design makes mongo work, IN-batched hydration, cross-connection
  relations, per-hop through-aggregates.
- [Validation](../validation.md) — `Field.number()` vs `Field.string()`
  for FK id rules against ObjectId.

This page is the overview and known-dragons list.

## The mental model

Two ways to run it:

**Whole-app mongo** — set `database.driver: 'mongodb'` in
`config/database.js`. Every model lives in mongo. Dogfooded end-to-end
against the reference demo app (event catalog, checkout with attendee
creation, admin dashboard with widget aggregates, list/show/edit/custom
actions across 7 resources, JSON API, organizer dashboard).

**Named connection** — leave the default on SQL/SQLite, opt specific
models into mongo via `static connection = 'audit'` on the Model class.
Both engines coexist and cross-connection eager loading works — each
hop uses its own connection's adapter. See [Multiple Connections](./getting-started.md#multiple-connections).

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

**Named connection**:

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

## Escape hatches

Two symmetric hatches let you drop below the ORM for driver-specific
work. Each throws a helpful error when called against the wrong
connection type, so you can't accidentally reach for the wrong one.

| Side | Method | Returns |
|---|---|---|
| SQL | `.getQueryBuilder()` | MikroORM's SQL QueryBuilder |
| SQL | `.whereRaw(sql, bindings)` | raw SQL fragment added to WHERE |
| Mongo | `.getCollection()` | MikroORM's underlying `Collection` |
| Mongo | `.whereMongo(filter)` / `.orWhereMongo(filter)` | raw mongo filter merged into `$match` |

```js
// Mongo-connected model
const coll = AuditLog.query().getCollection()
const results = await coll.aggregate([
  { $match: { userId } },
  { $group: { _id: '$action', count: { $sum: 1 } } },
  { $sort: { count: -1 } },
]).toArray()
```

The full driver-level API is available via `.getCollection()`:
`aggregate`, `distinct`, `createIndex`, `bulkWrite`, `findOneAndUpdate`,
etc.

## Known dragons

- **`Model.transaction()` on standalone mongod.** MongoDB transactions
  require a replica set (or mongos). Against a standalone `mongod` — the
  common dev setup — `em.transactional` throws before the callback commits.
  The framework detects this once per connection and transparently
  re-runs the callback without a session, so `Model.transaction(...)` on
  mongo becomes a best-effort wrapper (writes execute sequentially, no
  atomicity). If you need cross-write atomicity on mongo, run a replica
  set (e.g. `run-rs` or Atlas) — the framework picks up transactional
  semantics automatically.
- **`.distinct()` row-dedup is JS-side on mongo.** With no argument,
  `.distinct()` dedups fetched models in JS via `JSON.stringify(model)`.
  Models with non-serializable state (custom getters returning symbols,
  circular refs, functions) may dedup incorrectly. The field-list form
  `.distinct(col).pluck(col)` uses MikroORM's native `Collection.distinct`
  and matches SQL semantics exactly — prefer it when you can.
- **No CHECK constraints.** `static checks = [...]` is SQL-only; mongo
  is schemaless. Enforce range/enum invariants at the app layer.
- **Cross-connection through-aggregates are per-hop, not one pipeline.**
  `AuditLog` (mongo) with `.withCount('user.actions')` where `User` is
  on SQL runs two round trips: the mongo hop through the mongo adapter,
  the SQL hop through the SQL path. For typical cardinalities this is
  fine; for wide fan-out at each hop consider denormalizing.
- **No cross-engine transactions.** MikroORM's transaction contexts
  are per-connection. If you write to both a SQL model and a mongo
  model in one action, they're two independent operations; if one
  fails after the other succeeded, you own the compensation.
- **`foobar db:make` / `db:migrate` / `db:sync` don't apply to mongo
  connections.** Mongo is schemaless — collections are auto-created on
  first write. For indexes, use
  `Model.query().getCollection().createIndex(...)` in a boot script or
  seeder.

## See also

- [Database workflow](../database/workflow.md) — schema-sync modes for
  SQL connections
- [ORM: getting started](./getting-started.md) — the query reference,
  with mongo callouts inlined per topic
- [ORM: relationships](./relationships.md) — the no-JOINs design and
  cross-connection notes
