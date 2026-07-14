# Conventions

foobarjs is **convention-over-configuration**. Put a file in the right folder
with the right name and it gets picked up. This page is a one-stop reference to
every convention the framework applies at boot and at request time. Each
section links to the deeper doc.

Guiding principles:

- **Auto-discovery** — the framework walks specific folders and imports what
  it finds. There is no central manifest to keep in sync.
- **Filename drives behavior** — routes come from controller filenames,
  entities from model filenames, views from `<controller>/<action>.html`.
- **Reasonable defaults, explicit overrides** — every default (table name,
  primary key, view path, admin URL, etc.) can be overridden with a `static`
  property on the class.
- **Escape hatches** — `foobar.app` exposes the underlying HTTP router,
  `this.c` is the request context,
  `Model.query().getQueryBuilder()` gives you the raw query builder. When
  convention doesn't fit, drop down one layer.

## Directory conventions

| Folder / file | What foobarjs does |
|---|---|
| `app/controllers/**/*.controller.js` | Mounts a REST resource per file. Only actions you define get routes. `home.controller.js` also mounts at `/`. Sub-folders become URL prefixes (`admin/orders.controller.js` → `/admin/orders`). |
| `app/models/**/*.model.js` | Imports the default export and registers it with the ORM. Class extends `Model` (or `AuthenticableModel`). |
| `app/admin/**/*.admin.js` | Registers an admin panel resource config. |
| `app/middleware/**/*.js` | Available for route assignment by name. |
| `app/jobs/**/*.js` | Registered with the queue system. |
| `app/events/**/*.js` | Registered as event classes. |
| `app/listeners/**/*.js` | Auto-discovered via `static events = [...]`. |
| `app/notifications/**/*.js` | Plain userland classes — imported explicitly where dispatched. No auto-discovery. |
| `app/serializers/<model>.serializer.js` | Auto-looked-up per request by `autoSerialize(Model, data)`. Filename must match the lowercased model class name. |
| `app/validators/**/*.js` | Plain userland classes — imported by `FormRequest` subclasses. No auto-discovery. |
| `app/views/**/*.html` | Rendered via `this.render('folder/file')`. Also used for the auto-render fallback. |
| `app/views/layouts/*.html` | Convention only. Reference from templates via `@layout('layouts/app')`. |
| `app/views/errors/*.html` | Auto-lookup for HTTP error rendering (see [Error handling](./error-handling.md)). |
| `app/views/auth/*.html` | Optional user overrides for the auth plugin's login/register templates. If absent, foobarjs ships a self-contained default. |
| `app/views/admin/*.html` | Optional user overrides for admin panel templates. |
| `app/views/components/*.html` | Auto-lookup for `@component('name', ...)`. |
| `routes/web.js` | Explicit route registration. Loaded after filename-convention routes. |
| `routes/api.js` | Same shape as `web.js`. |
| `config/*.js` | Auto-loaded by filename. Access via `foobar.configLoader.get('<file>.<key>')`. |
| `database/migrations/*` | Run in filename order by `foobar db:migrate`. Tracked in `_foobar_migrations`. See [Database migrations](./database/migrations.md). |
| `database/seeders/*.seeder.js` | Run alphabetically by `foobar db:seed`. |
| `public/**` | Served as static files at `/`. |
| `.env`, `.env.<NODE_ENV>` | Loaded by `ConfigLoader` before any config file. |

See [Directory structure](./directory-structure.md) for the tree view.

## Routes

- **Filename** — `app/controllers/products.controller.js` mounts a REST
  resource at `/products`. The framework calls
  `router.resource('/products', ProductsController)` under the hood.
- **`resource()` only registers verbs whose method exists.** If the
  controller has only `index()`, only `GET /products` is registered.
  Missing routes return real 404, not 500.
- **`home` special case** — `app/controllers/home.controller.js` also
  mounts at `/`. Equivalent to declaring
  `router.get('/', HomeController, 'index')` explicitly.
- **Sub-folders become URL prefixes.**
- **`routes/web.js`** and **`routes/api.js`** are loaded after the filename
  convention. Use them for anything the convention can't express (root
  paths, inline callbacks, non-REST verbs, custom paths).

Verb → action map when using `router.resource(path, ControllerClass)`:

| Verb | URL | Action |
|------|-----|--------|
| GET | `/base` | `index` |
| GET | `/base/new` | `new` |
| POST | `/base` | `store` |
| GET | `/base/:id` | `show` |
| GET | `/base/:id/edit` | `edit` |
| PUT | `/base/:id` | `update` |
| DELETE | `/base/:id` | `destroy` |

See [Routing](./routing.md).

## Controllers

- Extend `Controller` from `foobarjs/core`.
- Action methods take no argument; use `this.c`, `this.params()`,
  `this.query()`, `this.body()`, `this.getLoggedInUser()`, etc.
- Return values are converted to responses by the framework — see
  **Auto response contract** below.

See [Controllers](./controllers.md).

## Auto response contract

Whatever a controller action returns is coerced into a response:

| Return value | Framework behavior |
|--------------|--------------------|
| A `Response` object | Sent as-is. |
| `undefined` or `null` | `204 No Content`. |
| An object or array | Looks for `app/views/<controller>/<action>.html`. If found, renders it with the value bound to a data key (see below). Otherwise returns JSON. |
| A string or primitive | `c.text(String(value))`. |

**Data key naming for the auto-render fallback:**

- `index` uses the plural form (from the controller name, lowercased).
  `ProductsController.index → {products: [...]}`.
- `show`, `edit`, `new`, `destroy` use the singular.
  `ProductsController.show → {product: {...}}`.
- `store` and `update` typically redirect, so they don't hit this path.

The controller name for view lookup and data key comes from the class name
with the `Controller` suffix stripped and lowercased. `ProductsController` →
`products`, `WidgetController` → `widget`. This is independent of the
filename.

## Models

- Extend `Model` from `foobarjs/orm` (or `AuthenticableModel` from
  `foobarjs/auth` for user models).
- Declare `static schema = { fieldName: Field.string()... }`.
- Primary key is always `id` (auto-increment integer).
- Column names are auto-derived by camelCase → snake_case:
  `firstName` → `first_name`, `orderItemId` → `order_item_id`.
- Table name defaults to `ClassName.replace(/([A-Z])/g, '_$1').toLowerCase().replace(/^_/, '') + 's'` —
  which produces `users` from `User`, `order_items` from `OrderItem`,
  and `categorys` from `Category`. **Set `static tableName = 'categories'`
  for words that don't pluralize with a naive `+s`.**
- `static timestamps = true` (default) → `createdAt`, `updatedAt` are
  auto-managed on save.
- `static softDelete = true` opts in to a `deleted_at` column. Queries
  auto-exclude soft-deleted rows unless `.withTrashed()` is called.
- Foreign keys declared via `Field.belongsTo(() => X)` get an auto-index by
  default. Opt out per-field with `.noIndex()` or globally with
  `config.database.autoIndexForeignKeys = false`.
- `Field.string().hidden()` → excluded from `toJSON()`.
- `static appends = ['url']` → adds computed getters to `toJSON()`.
- Lifecycle hooks fire by method presence: `beforeSave`, `afterSave`,
  `beforeCreate`, `afterCreate`, `beforeUpdate`, `afterUpdate`,
  `beforeDelete`, `afterDelete`.

For `AuthenticableModel`:

- Default identifier is `email`, default password field is `password`.
  Override with `static authIdentifierName` / `static authPasswordName`.
- Assigning plaintext to `user.password` hashes it via scrypt automatically.
  Never pre-hash.
- The `foobarjs/auth` plugin looks for the user model at
  `app/models/user.model.js`. Renaming the file breaks auth.

See [ORM: getting started](./orm/getting-started.md) and
[Authentication](./authentication.md).

## Auto admin

- Every registered model gets a CRUD UI at `/admin/<tableName>`.
  `Product` → `/admin/products`, `OrderItem` → `/admin/order_items`.
- Hide a model with `Model.adminDefaults = { hidden: true }`.
- Configure per-model via `app/admin/<name>.admin.js` (fluent builder
  preferred).
- Field types map to widgets automatically:
  `Field.string()` → text input, `Field.text()` → textarea,
  `Field.boolean()` → checkbox, `Field.date()` → date picker,
  `Field.enum(...)` → select, `Field.belongsTo(() => X)` → related-record combobox.
- Labels default to a humanized version of the field name:
  `firstName` → "First name".
- `user.isAdmin === true` bypasses every permission check.
- `user.roles` (array or JSON-encoded array) drives role-based checks.
- Soft-delete models automatically get restore and force-delete
  actions in the admin.
- Framework plugins register their own admin resources under a "System"
  group when enabled: `personal_access_tokens` (auth),
  `queue_jobs` + `failed_jobs` (queue with database driver),
  `cache_entries` (cache with database driver), `notifications`
  (notifications), `admin_exports` (admin).

See [Admin panel](./admin-panel.md).

## Auto REST API

- Every registered model gets JSON CRUD endpoints at `/api/<tableName>`.
  Configurable prefix via `new ApiPlugin({prefix: '/v2'})`.
- Endpoints per model:
  - `GET /api/products` — list. Query strings: `include`, `filter[key]`,
    `sort`, `page`, `perPage`.
  - `GET /api/products/:id` — show.
  - `POST /api/products` — create, returns 201.
  - `PUT /api/products/:id` — update.
  - `DELETE /api/products/:id` — delete.
- **Response shape** is whatever `autoSerialize(Model, result)` returns —
  a bare array for lists (or a `{data, meta}` object when the ORM
  returns a paginated result via `?page=`), a bare object for `show`
  and `create`, and `{message: 'Deleted'}` for `delete`.
- Validation failures return 422 with `{error, errors}`.
- Serializers auto-load from
  `app/serializers/<modelname>.serializer.js` (lowercased class name).
  If no file exists, the model's `toJSON()` is used (respects `hidden`
  fields and `appends`).

See [API](./api.md) and [Serialization](./serialization.md).

## Auto validation

- `ValidationError` from `foobarjs/orm` always renders as HTTP 422.
- `Model.save()` auto-validates the schema and translates DB constraint
  violations (unique / not-null / foreign-key / check) into `ValidationError`.
  Never catch database constraint exceptions directly — catch
  `ValidationError`.
- `this.validate(FormRequestClass)` (or `c.validate(...)`) runs a
  `FormRequest`, throws `ValidationError` on failure.
- `FormRequest.authorize()` returning `false` → HTTP 403.
- On a 422 for an HTML request, foobarjs redirects back and flashes
  `errors` and `old` into the session. Templates read them via
  `errors.email` / `old('email')` / `@error('email') ... @enderror`.
  See [Views](./views.md#view-globals).
- Field labels in error messages are auto-humanized:
  `first_name` / `firstName` → "First name".
- Non-field errors (e.g. "record has related records") go into the
  `_form` bucket in the errors object.

See [Validation](./validation.md).

## Views

- Rendered via `this.render('folder/file', data)` or the auto-render
  fallback from a returned object.
- Template path is relative to `app/views/`, no extension. Both
  `products.index` (dotted) and `products/index` (slashed) work.
- Extensions searched: `.html`, `.edge`, `.eta`.
- **Auto-injected globals in every render:** `user`, `loggedIn`,
  `cartCount`, `flash`, `errors`, `old`.
- Errors view auto-lookup order: `errors/N.html` (exact) →
  `errors/Nxx.html` → `errors/error.html` → `errors/500.html`
  (5xx only) → built-in fallback / dev diagnostic page.
- Directive processing order (matters for nesting): `@error` blocks →
  loops (`@foreach`) → conditionals (`@if`) → variable interpolation.
- Auth plugin renders `foobarjs/auth`'s bundled login/register templates
  by default. Override by creating `app/views/auth/login.html` and/or
  `app/views/auth/register.html`.

See [Views](./views.md).

## Auth and session

- `foobarjs/auth` in `config.app.plugins` auto-registers:
  - session middleware,
  - auth middleware (populates `c.get('user')` / `c.get('loggedIn')`),
  - routes: `GET /login`, `POST /login`, `GET /register`, `POST /register`,
    `POST /logout`.
- Session cookie name: `foobar_session`, HMAC-SHA256 signed with
  `APP_SECRET`. Missing `APP_SECRET` → hard error at boot.
- `Bearer <token>` in the `Authorization` header authenticates via
  `PersonalAccessToken`. No cookie needed.
- Flash messages are one-shot: they're delivered on the next request
  and cleared on the response after that. Use `session.reflash()` or
  `session.keep('errors')` to carry entries over one more request.
- After a failed auth check on a protected route, the requested URL
  is stashed in `session._intended` and the user is redirected to
  `/login`. On successful login, they are redirected back to the
  original URL.

See [Authentication](./authentication.md) and [Session](./session.md).

## Notifications

- Dispatched via `Notification.send(notifiable, notification)`.
- Channels come from `notification.via(notifiable)` — defaults to
  `['database']`. Framework channels: `database`, `mail`, `broadcast`.
- Notification `type` in the database is `notification.constructor.name`.
- Mail channel assumes the notifiable has an `email` field.
- Broadcast channel comes from `notification.broadcastOn(notifiable)`,
  event name from `notification.broadcastAs(notifiable)`.

See [Notifications](./notifications.md).

## Events and listeners

- `app/events/*.js` and `app/listeners/*.js` are auto-imported.
- Listeners register themselves via `static events = [OrderPlaced]`
  (class reference or string name).
- `Event.dispatch(new SomeEvent(...))` calls each registered listener's
  `handle(event)`. Errors thrown inside a listener are logged (via
  `console.error`) but do not propagate — the dispatch does not throw.
- `SomeEvent.broadcastOn()` (static, returns an array of channel names)
  is called automatically during dispatch and broadcasts via
  `foobarjs/broadcast`.

See [Events](./events.md).

## Queues

- `foobarjs/queue` in plugins auto-registers `QueueJob` and `FailedJob`
  models **only when `config.queue.default === 'database'`**. Under `sync`
  or `redis`, those tables are not created.
- Database driver stores queued jobs in the `queue_jobs` table and failed
  jobs in `failed_jobs`.
- Redis driver uses list keys `queues:<queue>`.
- `sync` driver runs jobs in-process, immediately, on dispatch.
- `Job.dispatch(...args)` schedules a job. Per-job overrides via
  `static queue = 'emails'`, `static tries = 5`, `static timeout = 30`.
- Failed jobs surface in the admin at `/admin/failed_jobs`.
- `foobar queue:work` runs a worker that pulls from the configured queue.

See [Queues](./queues.md).

## Cache

- Driver comes from `config.cache.default`. Options: `memory`, `file`,
  `database`, `redis`.
- The database driver auto-registers a `CacheEntry` model
  (`cache_entries` table).
- File driver defaults to `storage/framework/cache` and sanitizes keys
  containing anything other than `[A-Za-z0-9_-]` to `_`.
- Higher-level helpers: `Cache.remember(key, ttl, callback)`,
  `Cache.forever(key, value)`, `Cache.pull(key)`, `Cache.add(key, value, ttl)`.

See [Cache](./cache.md).

## Storage

- `Storage.disk()` returns the default disk (`local` unless configured
  otherwise).
- The bundled `local` disk stores files under `public/uploads/`. The
  `config.storage.disks.local.root` value is only honored if you replace
  the disk via `Storage.registerDisk('local', new LocalDisk('/custom/root'))`.
- `Storage.url(path)` for the local disk returns `/<path>` (paths are
  already relative to `public/`).
- `Storage.image(file)` uses `sharp` for resize/crop/format conversion
  if the optional `sharp` dependency is present.
- Register a custom disk (e.g. S3) with `Storage.registerDisk(name, diskInstance)`.

See [Storage](./storage.md).

## Config

- Any `config/foo.js` becomes accessible as `foobar.configLoader.get('foo.bar')`.
- `.env` is loaded before configs. `.env.<NODE_ENV>` is loaded after
  (values override plain `.env` unless already set in the shell environment).
- `configLoader.isDebug()` resolution order:
  `config.app.debug` → `APP_DEBUG` env → `NODE_ENV !== 'production'`.
- Plugins in `config.app.plugins` are loaded by dynamic
  `import(pluginPath)`. If that fails, foobarjs falls back to
  `createRequire(basePath)` and tries again.

See [Configuration](./configuration.md).

## Framework-owned tables

When you enable these plugins, foobarjs creates and manages the following
tables for you:

| Plugin | Table(s) |
|--------|----------|
| `foobarjs/auth` | `personal_access_tokens` |
| `foobarjs/queue` (`driver: 'database'` only) | `queue_jobs`, `failed_jobs` |
| `foobarjs/cache` (`store: 'database'` only) | `cache_entries` |
| `foobarjs/notifications` | `notifications` |
| `foobarjs/admin` (export feature) | `admin_exports` |
| Migrations (whenever any migration runs) | `_foobar_migrations` |

Each of these also appears in the admin panel under the "System" group when
`foobarjs/admin` is enabled.

## Migrations

- **File-based** is the recommended workflow. `foobar db:make` writes a
  migration file with a timestamp prefix; `foobar db:migrate` runs
  pending files in order and tracks them in `_foobar_migrations`;
  `foobar db:rollback` undoes the last batch.
- **Auto-diff** generation. `foobar db:make` compares your current model
  schemas to `database/.foobar-schema.json` and writes only the delta.
  Destructive changes (column drops, type changes, table drops) are
  commented out for manual review unless you pass `--allow-destructive`.
- **Blank migrations** for hand-written data changes:
  `foobar db:make --name backfill_country_codes`.
- **Auto schema-sync on boot is disabled** whenever `database/migrations/`
  contains at least one migration file. Explicitly disable regardless with
  `database.autoSync = false` in `config/database.js`.
- **`foobar db:sync`** is a dev-only shortcut that applies the model diff
  by diffing your models against the database. Refuses to run with `NODE_ENV=production`
  and refuses destructive changes without `--force`.

See [Database migrations](./database/migrations.md).

## See also

- [Directory structure](./directory-structure.md)
- [Routing](./routing.md)
- [Controllers](./controllers.md)
- [ORM: getting started](./orm/getting-started.md)
- [Admin panel](./admin-panel.md)
- [Configuration](./configuration.md)
