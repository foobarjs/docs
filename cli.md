---
title: CLI
layout: default
nav_order: 3
---

# CLI

foobarjs provides a command-line tool called `foobar` for common development tasks.

```bash
foobar [command] [options]
foobar help              # list all commands
foobar help <command>    # help for a specific command
```

## Command Naming Convention

Commands follow the **colon-namespaced** convention:

- **Simple commands** — standalone verbs: `new`, `serve`, `test`, `routes`, `repl`
- **Namespaced commands** — group related actions: `db:migrate`, `queue:work`, `user:create`
- **Generator** — `generate` (alias `g`) with a type argument: `foobar g model Product`

---

## Project

### `foobar new <name>`

Create a new foobarjs project with full setup:

```bash
foobar new my-blog
cd my-blog
foobar serve
```

Then log in at <http://localhost:3000/admin>.

**Non-interactive** (CI, scripts):

```bash
foobar new my-blog \
  --admin-name "Alice" \
  --admin-email "alice@example.com" \
  --admin-password "supersecret"

# Or accept defaults (admin@example.com / password):
foobar new my-blog --yes
```

| Option | Description |
|--------|-------------|
| `-y, --yes` | Skip prompts, use defaults |
| `--admin-name <name>` | Admin display name |
| `--admin-email <email>` | Admin email |
| `--admin-password <pass>` | Admin password |
| `--skip-install` | Don't run `npm install` |
| `--skip-admin` | Don't create the admin user |

### `foobar serve`

Start the development server:

```bash
foobar serve              # port 3000
foobar serve --port 8080
foobar serve --watch      # auto-restart on file changes
foobar serve --dev        # watch + debug + browser live-reload
```

### `foobar routes`

List all registered routes:

```bash
foobar routes
foobar routes --json     # machine-readable output
```

### `foobar inspect:routes`

List routes with auth and gate details:

```bash
foobar inspect:routes
foobar inspect:routes --json     # machine-readable output
foobar inspect:routes --strict   # exit non-zero if authenticated routes lack a gate
```

The `--strict` flag is useful in CI to ensure all authenticated routes have gate authorization.

### `foobar repl`

Start an interactive console with the booted application:

```bash
foobar repl
```

The REPL boots your app (loads config, discovers models, connects to the database) and drops you into a Node.js prompt with the `app` instance available.

**Pretty output** — ORM results are auto-formatted:

```
foobar> await User.find(1)
  User #1
  ──────────────────────
  name       │ "Alice"
  email      │ "alice@example.com"
  isAdmin    │ true

foobar> await Tag.all()
┌────┬─────────┐
│ id │ name    │
├────┼─────────┤
│  1 │ alpha   │
│  2 │ beta    │
└────┴─────────┘
2 rows
```

Single models render as key-value cards. Arrays of models render as ASCII tables. Paginated results include meta (page, total, per page). Scalars are color-coded: strings green, numbers yellow, booleans cyan, null/undefined dim.

**Top-level await** works natively:

```
foobar> const user = await User.find(1)
foobar> await user.profile
```

**Importing models** — models are not auto-imported; use dynamic imports:

```
foobar> const { default: User } = await import('./app/models/user.model.js')
```

**Dot-commands:**

| Command | Description |
|---------|-------------|
| `.models` | List all registered model classes and their table names |
| `.routes` | List all registered routes |
| `.reload` | Close the database connection, re-boot the app, and refresh context |
| `.clear` | Reset the REPL context (built-in) |
| `.exit` | Exit the REPL (built-in) |
| `.help` | Show all available commands (built-in) |

**History** — commands persist across sessions in `.foobar/.repl_history`. Press up/down arrows to navigate previous commands.

**Tab completion** — the default Node.js completer provides completion for `app.`, JavaScript globals, and dot-commands.

### `foobar test`

Run tests using Node.js built-in test runner:

```bash
foobar test                        # run all tests
foobar test test/models/           # specific directory
foobar test --filter "user"        # filter by test name
foobar test --watch                # watch mode
```

---

## Generators

### `foobar generate <type> <name>` (alias: `foobar g`)

Generate framework components:

```bash
# Model
foobar g model Product --fields name:string,price:float,description:text

# Controller
foobar g controller Products

# View
foobar g view products/index

# Full scaffold (model + controller + views + serializer + validator)
foobar g scaffold Product --fields name:string,price:float

# Other components
foobar g validator Product
foobar g serializer Product
foobar g job SendWelcomeEmail
foobar g event UserRegistered
foobar g listener SendWelcomeEmail --event UserRegistered
foobar g middleware LogRequests
foobar g test Product
foobar g seeder products
foobar g command sync:inventory
foobar g admin Product
foobar g gate Order
```

Available types: `controller`, `model`, `scaffold`, `view`, `serializer`, `validator`, `job`, `event`, `listener`, `middleware`, `test`, `seeder`, `command`, `admin`, `gate`.

---

## Database

### `foobar db:make`

Create a new migration file:

```bash
foobar db:make                               # auto-diff from model changes
foobar db:make --name add_stock              # blank migration for hand-written SQL
foobar db:make --diff --name add_stock       # auto-diff with a specific name
foobar db:make --allow-destructive           # include drop column/table in generated diff
```

See [Database migrations](./database/migrations.md) for the full workflow.

### `foobar db:migrate`

Apply pending migrations:

```bash
foobar db:migrate
```

### `foobar db:rollback`

Roll back the last batch of migrations:

```bash
foobar db:rollback            # roll back last batch
foobar db:rollback --step 3   # roll back last 3 batches
```

### `foobar db:status`

Show migration status:

```bash
foobar db:status
```

### `foobar db:sync`

Sync database schema directly from models (dev only):

```bash
foobar db:sync                # apply model diff directly
foobar db:sync --force        # bypass destructive-change guard
```

Refuses to run with `NODE_ENV=production`. Use `db:migrate` with real migration files instead.

### `foobar db:seed`

Run database seeders:

```bash
foobar db:seed                      # run all seeders in database/seeders/
foobar db:seed --name products      # run database/seeders/products.seeder.js
```

Seeders are `*.seeder.js` files in `database/seeders/`, executed alphabetically.

### `foobar db:reset`

Drop and recreate all tables:

```bash
foobar db:reset
```

### `foobar db:fresh`

Drop all tables, re-migrate (or recreate from models), and re-seed:

```bash
foobar db:fresh
```

### `foobar db:indexes`

Compare declared vs actual database indexes:

```bash
foobar db:indexes                 # full comparison table
foobar db:indexes --missing       # exit 1 if any declared index is missing (CI)
foobar db:indexes --extra         # show indexes present in DB but not declared
foobar db:indexes --json          # machine-readable output
```

### `foobar db:explain`

Run EXPLAIN on a SQL query:

```bash
foobar db:explain 'SELECT * FROM orders WHERE user_id = ? AND status = ?' \
  --bind 42 --bind pending
foobar db:explain 'SELECT ...' --analyze   # EXPLAIN ANALYZE (Postgres)
```

---

## Auth

### `foobar key:generate`

Generate a secure `APP_SECRET`:

```bash
foobar key:generate
```

Copy the printed value into your `.env` file. `foobar new` runs this automatically.

### `foobar user:create`

Create a user account from the command line:

```bash
foobar user:create --email admin@example.com --password secret123
foobar user:create --name "Jane" --email jane@example.com --password secret123 --admin
foobar user:create --email bot@example.com --password secret123 --token "ci-server"
```

| Option | Description |
|--------|-------------|
| `-n, --name <name>` | User display name |
| `-e, --email <email>` | User email (required) |
| `-p, --password <password>` | User password (required) |
| `-a, --admin` | Grant admin access |
| `-t, --token [name]` | Generate an API token (optional device name) |

---

## Queue

### `foobar queue:work`

Run the queue worker:

```bash
foobar queue:work
foobar queue:work --queue emails --tries 5 --timeout 120
foobar queue:work --once       # process one job and exit
```

| Option | Description |
|--------|-------------|
| `-q, --queue <name>` | Queue name(s), comma-separated (default: all discovered) |
| `-c, --connection <name>` | Queue connection |
| `-s, --sleep <seconds>` | Seconds to sleep when no jobs (default: `3`) |
| `-t, --tries <n>` | Number of retries (default: `3`) |
| `--timeout <seconds>` | Job timeout in seconds (default: `60`) |
| `--once` | Process one job and exit |

### `foobar queue:retry`

Retry all failed jobs:

```bash
foobar queue:retry
foobar queue:retry --queue emails
```

---

## Schedule

### `foobar schedule:work`

Run the schedule worker as a long-running process (recommended for pm2, systemd, Docker):

```bash
foobar schedule:work
```

Checks for due tasks every minute. No OS cron entry needed.

### `foobar schedule:run`

Run all due scheduled tasks once and exit (for OS cron):

```bash
foobar schedule:run
```

### `foobar schedule:list`

List all registered scheduled tasks:

```bash
foobar schedule:list
```

See [Queues — Task Scheduling](./queues.md#task-scheduling) for how to define your schedule in `config/schedule.js`.

---

## Custom Commands

You can add your own CLI commands by creating files in `app/commands/`. Each command is a `*.command.js` file that exports a definition object.

### Creating a Custom Command

Generate a command scaffold:

```bash
foobar g command sync:inventory
```

This creates `app/commands/sync:inventory.command.js`:

```js
export default {
  name: 'sync:inventory',
  description: 'Sync product inventory from external API',

  arguments: [
    { name: '<source>', description: 'inventory source to sync from' },
  ],

  options: [
    { flags: '-d, --dry-run', description: 'preview without making changes' },
    { flags: '-l, --limit <n>', description: 'max items to process', default: '100' },
  ],

  async run(source, options) {
    console.log(`Syncing from ${source}...`)
    if (options.dryRun) {
      console.log('(dry run — no changes made)')
      return
    }
    // Your logic here — import models, services, etc.
  },
}
```

Run it:

```bash
foobar sync:inventory warehouse-api --dry-run
foobar sync:inventory warehouse-api --limit 50
```

### Command Definition

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | `string` | yes | Command name (use colon-namespace for grouped commands) |
| `description` | `string` | no | Help text shown in `foobar help` |
| `alias` | `string` | no | Short alias for the command |
| `arguments` | `array` | no | Positional arguments (see below) |
| `options` | `array` | no | Named options (see below) |
| `run` | `function` | yes | Async function called when the command runs |

**Arguments** are objects with `{ name, description }`. Use `<name>` for required, `[name]` for optional:

```js
arguments: [
  { name: '<action>', description: 'action to perform' },
  { name: '[target]', description: 'optional target' },
]
```

**Options** are objects with `{ flags, description, default, required }`:

```js
options: [
  { flags: '-v, --verbose', description: 'verbose output' },
  { flags: '-o, --output <path>', description: 'output file', default: './out' },
  { flags: '-e, --email <email>', description: 'notification email', required: true },
]
```

### Examples

**Data import command:**

```js
// app/commands/import:csv.command.js
import Product from '../models/product.model.js'

export default {
  name: 'import:csv',
  description: 'Import products from a CSV file',

  arguments: [
    { name: '<file>', description: 'path to CSV file' },
  ],

  options: [
    { flags: '-d, --dry-run', description: 'validate without importing' },
  ],

  async run(file, options) {
    const { readFileSync } = await import('node:fs')
    const lines = readFileSync(file, 'utf8').split('\n').slice(1)

    console.log(`Processing ${lines.length} rows...`)
    for (const line of lines) {
      const [name, price] = line.split(',')
      if (!options.dryRun) {
        await new Product({ name, price: Number(price) }).save()
      }
    }
    console.log(options.dryRun ? 'Dry run complete' : `Imported ${lines.length} products`)
  },
}
```

**Cleanup command:**

```js
// app/commands/cleanup:sessions.command.js
import { dayjs } from 'foobarjs/support'

export default {
  name: 'cleanup:sessions',
  description: 'Remove expired sessions older than N days',

  options: [
    { flags: '--days <n>', description: 'days to keep', default: '30' },
  ],

  async run(options) {
    const cutoff = dayjs().subtract(Number(options.days), 'day')
    // Your cleanup logic here
    console.log(`Cleaned up sessions older than ${cutoff.format('YYYY-MM-DD')}`)
  },
}
```

### File Convention

| Pattern | Location | Example |
|---------|----------|---------|
| `*.command.js` | `app/commands/` | `app/commands/sync:inventory.command.js` |

Custom commands are auto-discovered at startup — no registration needed. They appear in `foobar help` alongside built-in commands.
