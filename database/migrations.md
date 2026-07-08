# Database: Migrations

foobarjs uses MikroORM's schema synchronization for database management. When the application boots or when `foobar db:migrate` is run, the framework compares your model schemas against the actual database and creates/updates tables automatically.

## Running Migrations

```bash
foobar db:migrate
```

This loads your application configuration, discovers all model classes, and synchronizes the database schema to match your model definitions.

### What Happens

1. Loads environment and config files
2. Discovers models from `app/models/`
3. Initializes the ORM with SQLite
4. Creates any missing tables
5. Adds any missing columns
6. Creates indexes for unique fields and foreign keys
7. Applies declared indexes, uniques, and CHECK constraints from models (see [ORM: Indexes](../orm/getting-started.md#indexes))

### Indexes

Schema updates pick up new or removed indexes automatically. When you add `.index()` to a field or a new entry to `static indexes`, the next boot (or the next `foobar db migrate`) creates the index. When you remove one, MikroORM drops it.

For zero-downtime index creation on large Postgres tables, drop to a raw migration and use `CREATE INDEX CONCURRENTLY`. `orm.schema.update()` always issues blocking `CREATE INDEX` statements.

Verify what's in the database with:

```bash
foobar db indexes
foobar db explain 'SELECT ...'
```

## Migration Generator

The framework includes a `MigrationGenerator` that can produce raw SQL from your model schemas. This is used internally for schema sync and can also generate SQL for other dialects (MySQL, PostgreSQL).

```js
import { MigrationGenerator } from 'foobarjs/orm'

const generator = new MigrationGenerator()
generator.generate(modelClasses)
const sql = generator.toSQL('sqlite')  // returns array of SQL statements
```

## Command Reference

| Command | Description |
|---------|-------------|
| `foobar db:migrate` | Sync database schema with models |
| `foobar db:reset` | Drop all tables and recreate them |
| `foobar db:fresh` | Drop all tables, recreate, and run seeders |
| `foobar db:seed` | Run database seeders |
| `foobar db:rollback` | Not yet implemented |

## Notes

- Migrations are determined by your model schemas — there are no hand-written migration files
- Schema synchronization is idempotent: running it multiple times is safe
- For SQLite, the entire schema is synchronized in one pass
- For production, review the generated schema before running on critical data
