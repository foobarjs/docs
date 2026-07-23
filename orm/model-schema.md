---
title: Model schema — compound indexes & constraints
layout: default
parent: ORM
---

# Compound indexes on both SQL and mongo

`static indexes = [...]` and `static uniques = [...]` on a Model produce
real compound indexes on **all** supported drivers, including mongo
(v0.6.0+).

```js
class Membership extends Model {
  static schema = {
    org: Field.string().required(),
    email: Field.string().required(),
    role: Field.string().default('member'),
  }

  // Compound (multi-column) index.
  static indexes = [
    { columns: ['org', 'role'] },
  ]

  // Compound unique — same {org, email} can't exist twice.
  static uniques = [
    { columns: ['org', 'email'] },
  ]
}
```

On mongo, the compound `{org, email}` unique index is enforced at write
time — a duplicate insert raises the driver's `E11000` duplicate-key
error, which surfaces through `Model.create()` as a validation-style
exception.

## Applying schema changes

- SQL — `foobar db:sync` (or the auto-sync path on boot) creates or
  updates the indexes.
- Mongo — call `Db.orm.schema.ensureIndexes()` explicitly. `db:sync`
  does not yet run this for mongo connections; do it once at deploy
  time, or bake it into your app boot.

## Not supported on mongo

- **CHECK constraints** (`static checks = [...]`) are SQL-only. Mongo
  has no equivalent primitive. Declared checks are silently ignored on
  mongo — this is intentional so the same model class boots on either
  driver.

If you need value-shape guarantees on mongo, either enforce them in
your `Field.string().enum([...])`-style declarations, in `hooks`, or
in a `beforeSave` guard.
