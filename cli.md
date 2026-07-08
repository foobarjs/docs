# CLI

foobarjs provides a command-line tool called `foobar` for common development tasks.

## Usage

```bash
foobar [command] [options]
```

## Available Commands

### `foobar new <name>`

Create a new foobarjs project. Handles the full setup end-to-end:

1. Prompts for admin name, email, and password.
2. Writes the project skeleton.
3. Generates a secure `APP_SECRET` into `.env`.
4. Runs `npm install`.
5. Creates the database schema and inserts the admin user.

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

**Opt-out flags:**

| Flag | Effect |
|------|--------|
| `-y, --yes` | Skip prompts, use defaults for admin fields not passed as flags. |
| `--admin-name <name>` | Admin display name. |
| `--admin-email <email>` | Admin email (login identifier). |
| `--admin-password <pass>` | Admin password (min 8 chars in interactive mode). |
| `--skip-install` | Don't run `npm install`. Implies `--skip-admin` (admin creation needs the framework installed). |
| `--skip-admin` | Scaffold and install, but don't create the admin user. Register users yourself via `/register` or a seeder. |

**What the skeleton includes:** `package.json`, `.env`, `.gitignore`,
`config/`, `app/controllers/`, `app/models/`, `app/views/` (layout, home,
error pages), `routes/web.js`, `public/`, `test/`, `db/`, `database/`.

### `foobar key:generate`

Generate a secure `APP_SECRET` for your `.env` file:

```bash
foobar key:generate
# APP_SECRET=...
```

Copy the printed value into your `.env` file. `foobar new` runs this
automatically; use `key:generate` for key rotation or when setting up an
app that was scaffolded with `--skip-install`.

### `foobar serve`

Start the development server:

```bash
foobar serve          # port 3000
foobar serve --port 8080
foobar serve --watch  # auto-restart on file changes
```

### `foobar generate` (`foobar g`)

Generate framework components:

```bash
# Model
foobar generate model Product --fields name:string,price:float,description:text

# Controller
foobar generate controller Products

# View
foobar generate view products/index

# Full scaffold (model + controller + views + serializer + validator)
foobar generate scaffold Product --fields name:string,price:float

# Other components
foobar generate validator Product
foobar generate serializer Product
foobar generate job SendWelcomeEmail
foobar generate event UserRegistered
foobar generate listener SendWelcomeEmail --event UserRegistered
foobar generate middleware LogRequests
```

### `foobar db`

Database management:

```bash
foobar db migrate   # Sync schema with models
foobar db seed      # Run seeders
foobar db reset     # Drop and recreate all tables
foobar db fresh     # Reset and seed

# Inspect indexes
foobar db indexes                    # List declared vs. actual indexes
foobar db indexes --missing          # Only show missing; exit 1 if any (CI)
foobar db indexes --extra            # Only show indexes present but not declared
foobar db indexes --json             # Machine-readable output

# Explain a query plan
foobar db explain 'SELECT * FROM orders WHERE user_id = ? AND status = ?' \
  --bind 42 --bind pending
foobar db explain 'SELECT ...' --analyze   # EXPLAIN ANALYZE on Postgres
```

`foobar db indexes` compares indexes declared in your models (via `Field.index()`, `Field.unique()`, `static indexes`, `static uniques`, and auto-FK indexes) against `sqlite_master` / `pg_indexes` / `SHOW INDEX` on the running database. Combined with `--missing` and a non-zero exit code, it works as a CI guard against forgetting to create declared indexes.

`foobar db explain` runs the driver-appropriate EXPLAIN and prints the query plan. Use it to confirm that a query is hitting the index you expect. On SQLite, plans look like `SEARCH orders USING INDEX idx_orders_user_id (user_id=?)`.

See [`docs/orm/getting-started.md#indexes`](./orm/getting-started.md#indexes) for the model-side index API.

## Options

| Option | Description |
|--------|-------------|
| `--help` | Show help for any command |
| `--version` | Show framework version |
| `-s, --step <n>` | Number of migration steps (future) |
| `-p, --port <port>` | Server port |
| `-w, --watch` | Watch mode for dev server |
| `-f, --fields <fields>` | Comma-separated field:type pairs for generators |
| `-e, --event <event>` | Event name for listener generation |
