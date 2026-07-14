# Database migrations

foobarjs supports **file-based migrations** — versioned, reviewable,
reversible SQL changes tracked in a `_foobar_migrations` table. This is the
recommended workflow for anything you'll deploy.

There's also a **schema-sync** shortcut (`foobar db:sync`) for local
prototyping. It reads your model definitions and applies the diff directly,
no files involved. Use it while sketching; use migrations when the schema
starts to matter.

## The commands

| Command | What it does |
|---------|--------------|
| `foobar db:make` | Auto-diff. Compares model schemas to the last-known snapshot and writes a migration file with the delta. Prints `No model changes detected.` if nothing changed. |
| `foobar db:make --name <name>` | Blank migration. Writes a timestamped stub with empty `up` and `down` for you to fill in (data backfills, ad-hoc SQL, edge cases the DSL doesn't cover). |
| `foobar db:make --diff --name <name>` | Auto-diff, but with a specific name instead of the default `auto_diff`. |
| `foobar db:make --diff --allow-destructive` | Auto-diff, allowing operations that would drop columns/tables. Destructive `up` steps are still commented out for you to review; you must uncomment them consciously. |
| `foobar db:migrate` | Runs pending migrations in filename order. Records each in `_foobar_migrations` with a batch number. |
| `foobar db:rollback [--step N]` | Rolls back the last N batches (default 1). Each batch is undone in reverse order. |
| `foobar db:status` | Lists every migration file and whether it's been applied. |
| `foobar db:sync` | Dev-only shortcut: applies the model diff directly against the database. **Refuses to run in production**, and refuses to drop columns/tables without `--force`. |
| `foobar db:seed` | Runs the seeder(s). |
| `foobar db:reset` | Drops all tables and recreates them from models. Loses data. Not migration-aware — use for hard resets. |
| `foobar db:fresh` | Drop all tables, then run all migrations from scratch (if any exist) or `schema.create()` (if none), then seed. |

## Anatomy of a migration file

Every migration lives at `database/migrations/<timestamp>_<name>.js` and
exports a default object with `up` and `down` functions. Both receive a
`schema` builder:

```js
// database/migrations/2025_10_15_083012_create_products.js
export default {
  async up(schema) {
    await schema.createTable('products', (t) => {
      t.id()
      t.string('name').notNullable()
      t.string('slug').unique()
      t.text('description').nullable()
      t.decimal('price').default(0)
      t.integer('category_id').references('categories', 'id').onDelete('SET NULL')
      t.boolean('published').default(false)
      t.timestamps()
      t.softDeletes()
      t.index(['category_id', 'published'])
    })
  },

  async down(schema) {
    await schema.dropTable('products')
  },
}
```

`up` is what runs on `foobar db:migrate`. `down` is what runs on
`foobar db:rollback`. Always write both.

## Schema builder DSL

The `schema` argument is a `Schema` instance from `foobarjs/orm`.

### Table-level ops

```js
schema.createTable(name, (t) => { ... })
schema.createTableIfNotExists(name, (t) => { ... })
schema.alterTable(name, (t) => { ... })
schema.dropTable(name)
schema.renameTable(from, to)
schema.raw(sql, params)      // escape hatch for anything the DSL misses
```

### Column types

Inside a `createTable` or `alterTable` callback:

| Method | SQL type (sqlite / mysql / postgres) |
|--------|--------------------------------------|
| `t.id()` | `INTEGER PRIMARY KEY AUTOINCREMENT` (always adds `id`) |
| `t.string(name, length = 255)` | `TEXT` / `VARCHAR(N)` / `VARCHAR(N)` |
| `t.text(name)` | `TEXT` |
| `t.integer(name)` | `INTEGER` |
| `t.bigInteger(name)` | `INTEGER` / `BIGINT` / `BIGINT` |
| `t.float(name)` | `REAL` / `FLOAT` / `FLOAT` |
| `t.decimal(name, precision, scale)` | `REAL` / `DECIMAL(p,s)` / `DECIMAL(p,s)` |
| `t.boolean(name)` | `INTEGER` / `TINYINT(1)` / `BOOLEAN` |
| `t.date(name)` | `TEXT` / `DATE` / `DATE` |
| `t.datetime(name)` / `t.timestamp(name)` | `TEXT` / `DATETIME` / `TIMESTAMP` |
| `t.json(name)` | `TEXT` / `JSON` / `JSONB` |
| `t.uuid(name)` | `TEXT` / `CHAR(36)` / `UUID` |
| `t.enum(name, [values])` | `TEXT` / `ENUM(...)` / `VARCHAR(255)` |
| `t.timestamps()` | Adds nullable `created_at` and `updated_at` |
| `t.softDeletes()` | Adds nullable `deleted_at` |

### Column modifiers

Every column method returns a builder; chain modifiers on it.

```js
t.string('email').notNullable().unique()
t.integer('user_id').references('users', 'id').onDelete('CASCADE')
t.decimal('price').default(0)
t.text('bio').nullable()
t.string('slug').index('slug_lookup')      // named index
```

Available modifiers: `.nullable()`, `.notNullable()`, `.default(value)`,
`.unique([name])`, `.index([name])`, `.primary()`,
`.references(table, column)`, `.onDelete(action)`, `.comment(text)`.

### Compound indexes and uniques

```js
t.index(['user_id', 'created_at'])          // implicit name: idx_<table>_user_id_created_at
t.index(['user_id', 'status'], 'orders_user_status')
t.unique(['product_id', 'variant_id'])
```

### Altering existing tables

```js
schema.alterTable('users', (t) => {
  t.string('avatar_url').nullable()          // add column
  t.dropColumn('legacy_field')               // drop column
  t.renameColumn('bio', 'about')             // rename column
  t.index(['country', 'created_at'])         // add index
  t.renameTo('accounts')                     // rename the table
})
```

### Raw SQL for what the DSL doesn't cover

```js
await schema.raw(
  'UPDATE users SET country = ? WHERE country IS NULL',
  ['US']
)
```

Use `raw()` for data backfills, database-specific features (partial
indexes, JSONB operators, `GENERATED ALWAYS AS`), or anything the DSL
doesn't natively support.

## The auto-diff feature

`foobar db:make` compares two snapshots:

- **Previous snapshot** — stored at `database/.foobar-schema.json`, updated
  automatically after every successful `db:migrate` or `db:sync`.
- **Current snapshot** — derived from your model classes at the moment you
  run `db make`.

It produces a migration file with only the delta.

**Example:** you add a field to a model.

```js
class Product extends Model {
  static schema = {
    name: Field.string().required(),
    price: Field.decimal(10, 2).required(),
    stock: Field.number().default(0),   // NEW
  }
}
```

Then:

```bash
foobar db:make --diff --name add_stock_to_products
```

Produces:

```js
// database/migrations/2025_10_15_101245_add_stock_to_products.js
export default {
  async up(schema) {
    await schema.alterTable('products', (t) => {
      t.integer('stock').default(0)
    })
  },
  async down(schema) {
    await schema.alterTable('products', (t) => {
      t.dropColumn('stock')
    })
  },
}
```

Review the file, then `foobar db:migrate` applies it.

### Destructive changes are refused by default

If your model diff includes column drops, table drops, or column type
changes, `foobar db:make --diff` will refuse:

```bash
$ foobar db:make --diff
Refusing to write a migration because it would drop columns/tables or
change existing column types.
Re-run with --allow-destructive to generate the file (destructive up
steps will be commented out for you to review).
```

Pass `--allow-destructive` and the generator writes the file with the
destructive operations commented out:

```js
await schema.alterTable('products', (t) => {
  // DESTRUCTIVE: t.dropColumn('legacy_sku')
})
```

You have to un-comment those lines yourself before running `db:migrate`.
There is no fully-automated path to data loss.

## Blank migrations for hand-written work

For anything the auto-diff can't produce — data backfills, seed data
edits, driver-specific SQL, complex alterations — generate a blank
migration and fill it in:

```bash
foobar db:make --name backfill_country_codes
```

Produces:

```js
// database/migrations/2025_10_15_101800_backfill_country_codes.js
export default {
  async up(schema) {
    // await schema.raw("UPDATE users SET country = 'US' WHERE country IS NULL")
  },
  async down(schema) {
    // TODO: undo the change.
  },
}
```

## The migration tracking table

foobarjs creates `_foobar_migrations` on first run:

| Column | Meaning |
|--------|---------|
| `id` | Auto-increment. |
| `name` | Filename of the migration, e.g. `2025_10_15_083012_create_products.js`. Unique. |
| `batch` | Which `foobar db:migrate` invocation applied it. Used for rollback grouping. |
| `ran_at` | ISO 8601 timestamp. |

The table is prefixed with an underscore so it doesn't collide with any
user model and sorts low in the admin panel. You can inspect it with any
SQLite client or via `foobar db:explain`.

## Batches

Every `foobar db:migrate` invocation that applies at least one migration
gets a new batch number (starts at 1). `foobar db:rollback` undoes the
most recent batch by default. `--step N` rolls back N most recent batches.

If you migrated three files together, they share a batch, so one
`db:rollback` undoes all three (in reverse order).

## The dev shortcut: `foobar db:sync`

For local prototyping — before you care about migration files at all — you
can skip the file workflow entirely:

```bash
foobar db:sync
```

This diffs your current models against the database schema and applies changes. Fast,
zero ceremony.

`db:sync` has two safety guards:

- **Destructive-change refusal.** If the diff would drop a column/table or
  change an existing column's type, `db:sync` refuses to run without
  `--force`. Same rationale as `db make`.
- **Production refusal.** If `NODE_ENV=production`, `db:sync` refuses to
  run at all (still with `--force` override, but you shouldn't).

Once you have any migration files in `database/migrations/`, the
framework's boot-time auto-sync is disabled — the assumption is that if
you've committed to file migrations, you don't want implicit schema
changes on server boot. To opt out explicitly regardless of migration
files, set `database.autoSync = false` in `config/database.js`.

## Configuration

```js
// config/database.js
export default {
  connection: process.env.DB_CONNECTION || 'sqlite',
  database: process.env.DB_DATABASE || 'foobar.db',
  autoSync: false,                       // never sync on boot
  autoIndexForeignKeys: true,            // FK columns get auto-indexed (default true)
}
```

## Recommended workflow

1. **Locally, while sketching:** use `foobar db:sync` — no ceremony,
   change models, restart, keep going.
2. **When the schema starts to matter** (e.g., a teammate is going to
   check out the branch, or you're about to deploy): run
   `foobar db:make --diff --name <describe_change>` to capture the
   history as a migration file.
3. **In CI/prod deploys:** never `db:sync`. Always `foobar db:migrate`.

## Manual migrations for data changes

Structural changes (add/drop tables and columns) can be auto-generated.
Data changes cannot — foobarjs has no idea what `UPDATE users SET ...`
should be. For those:

```bash
foobar db:make --name backfill_country_codes
```

Fill in the blank migration:

```js
export default {
  async up(schema) {
    await schema.raw(`UPDATE users SET country = 'US' WHERE country IS NULL`)
  },
  async down(schema) {
    // Data migration; typically not reversible.
  },
}
```

Commit the file. `foobar db:migrate` applies it in order alongside your
structural migrations.

## See also

- [CLI](./../cli.md)
- [ORM: getting started](./../orm/getting-started.md)
- [Conventions](./../conventions.md)
