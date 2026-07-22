[← Back to docs](./README.md)

# Admin Panel

The `foobarjs/admin` plugin generates a full admin panel for all your models with zero configuration. It uses Bootstrap 5 with offline assets, Edge templates, and supports per-model customization, search, filters, bulk actions, and granular permissions.

## Setup

Add `foobarjs/admin` to the `plugins` array in `config/app.js`:

```js
export default {
  plugins: ['foobarjs/admin'],
}
```

All admin routes are protected by authentication. By default only users with `isAdmin: true` can access the panel. Set `admin.requireAdmin: false` in your config to allow any authenticated user and rely on per-model permissions instead.

```js
// config/admin.js
export default {
  requireAdmin: false,
  theme: {
    brand: 'Foobar Admin',
    logo: '/logo.svg',
    primaryColor: '#0d6efd',
    mode: 'light',
  },
}
```

## Generated Routes

The admin panel registers routes under `/admin` for every discovered model:

| Method | URI | Description |
|--------|-----|-------------|
| GET | `/admin` | Dashboard |
| GET | `/admin/products` | List products |
| GET | `/admin/products/create` | Create product form |
| POST | `/admin/products` | Store product |
| GET | `/admin/products/:id` | Show product detail |
| GET | `/admin/products/:id/edit` | Edit product form |
| PUT | `/admin/products/:id` | Update product |
| DELETE | `/admin/products/:id` | Delete product |
| POST | `/admin/products/bulk` | Bulk actions |
| POST | `/admin/products/:id/action/:name` | Inline action |
| GET | `/admin/search` | Global search |
| GET | `/admin/failed-jobs` | Failed jobs list |

## Features

### Dashboard

The dashboard shows stat cards for every model, resource-specific widgets defined in your admin configs, and a list of recent failed jobs when the queue plugin is enabled.

### List View

List views include:

- Pagination (default 15 items per page)
- Search across configured fields
- Per-column filters
- Sortable columns
- Sticky table headers
- Bulk actions (delete by default)
- Inline action dropdown (view, edit, delete, plus custom actions)

Use query parameters to control the list:

```
/admin/products?page=2&perPage=25
/admin/products?q=laptop
/admin/products?sort=price&order=desc
/admin/products?f[published]=1
```

### Date Display

Date and datetime fields display in a human-readable `MMM D, YYYY h:mm A` format (e.g., "Jul 14, 2026 10:30 AM") in both list and detail views. Edit forms use standard HTML `date` and `datetime-local` inputs.

### Form Fields

Admin automatically renders the appropriate form field type based on your schema:

| Field Type | Form Input |
|------------|------------|
| `string` | Text input |
| `text` | Textarea |
| `json` | Textarea (JSON) |
| `number` | Number input |
| `float` / `decimal` | Number input (step=0.01) |
| `boolean` | Checkbox |
| `date` | Date input |
| `datetime` | Datetime input |
| `email` | Email input |
| `password` | Password input |
| `file` / `image` | File input |
| `belongsTo` | Select with related records |
| `belongsToMany` | Multi-select with related records |
| `hasMany` | Hint to manage from the related model |

Fields marked with `.hidden()` are omitted from admin forms and lists.

### Form Validation

Admin auto-validates every submitted form against the model's schema:

- Rules declared with `.required()`, `.email()`, `.minLength(n)`, `.regex(...)`, etc. run inside `Model.save()`. On failure the form re-renders with a summary panel at the top and inline `is-invalid` styling next to each affected input.
- Database-level constraints (`.unique()`, `NOT NULL`, foreign keys, checks) are translated into the same `ValidationError` shape at the ORM layer. So a duplicate slug in `Field.string().required().unique()` surfaces as *"Slug has already been taken"* next to the slug input — never as a silent 500 or redirect to home.
- The user's previously-submitted values are preserved so they don't have to retype anything.

See [Validation](./validation.md) for the full rule vocabulary and how to add custom messages.

### Relationship Fields

`belongsTo` fields render as a dropdown populated with records from the related model. The display value is `name`, `title`, `slug`, or `id` (in that order). `belongsToMany` fields render as a multi-select.

```js
import { Model, Field } from 'foobarjs/orm'
import Category from './category.model.js'

class Product extends Model {
  static schema = {
    name: Field.string().required(),
    description: Field.text(),
    price: Field.float().required(),
    published: Field.boolean().default(false),
    category: Field.belongsTo(() => Category),
  }
}
```

## Zero-Config Defaults

Models without an `Admin.resource()` config file get sensible CRUD defaults automatically:

### Smart list columns

The auto-generated list view picks a compact set of columns (capped at 7):

1. **`id`** — always first
2. **Title field** — the model's `_title` heuristic (`name`, `title`, `label`, or first string field)
3. **Up to 4 "interesting" fields**, prioritized by type:
   - `belongsTo` relations (shows the related model's display label)
   - Enum fields
   - Boolean fields
   - Number/decimal fields
   - String fields
   - Date/datetime fields
4. **`createdAt`** — always last (when `static timestamps = true`)

Excluded by default: `updatedAt`, `deletedAt`, `text`, `json`, `password`, `file`, `image`, and `belongsToMany` (often noisy as a tag list).

### Auto-generated form fields

Without explicit `form.fields`, the form includes all non-reserved column fields plus `belongsTo` and `belongsToMany` relations. The field list respects mass-assignment declarations:

- **`static fillable`** (allow-list): only those fields appear on the form.
- **`static guarded`** (deny-list): guarded fields are excluded. The default `guarded` is `['id', 'createdAt', 'updatedAt', 'deletedAt']`, which already matches the reserved columns.
- **Explicit `form.fields`** in an admin config always wins.

Relations (`belongsTo`, `belongsToMany`) are always included regardless of `fillable`/`guarded`, since they're set via foreign keys rather than mass assignment.

### Framework-managed fields

`id`, `createdAt`, `updatedAt`, and `deletedAt` are managed by the framework:

- **Detail page**: displayed as regular read-only fields
- **Edit form**: shown in the sidebar meta card (right column) as read-only
- **Create form**: hidden (no values yet)
- **List page**: `id` and `createdAt` are included as columns (see above)

### Default searchable fields

Without explicit `searchable` config, the admin panel auto-selects up to 3 string/text/email fields plus `id` for the search box and global search. This means you can search records by ID out of the box on any model.

### Default filters

Without explicit `filters` config, the admin auto-generates filters for:

- **Enum fields** — rendered as a select dropdown with the enum values
- **Boolean fields** — rendered as a Yes/No select
- **`belongsTo` relations** — rendered as a lazy combobox that searches the related model

Up to 6 auto-generated filters are shown. Explicit `filters` in an admin config always wins.

### Default bulk action confirmation

The built-in bulk actions (**Delete**, **Export**, **Restore**) require confirmation before executing. When a user clicks Apply, they are taken to a review page showing the selected records and a confirm/cancel prompt. This uses the same `.confirm()` mechanism available to custom bulk actions:

```js
Action.bulk('archive', 'Archive selected')
  .confirm('Archive all selected records?')
  .handler(async (ids, { Model }) => { /* ... */ })
```

### Opting out

Hide a model from the admin panel entirely with a static property:

```js
class InternalLog extends Model {
  static admin = false
  // ...
}
```

Models with `static admin = false` are excluded from:
- Sidebar navigation
- Admin route registration (all CRUD routes return 404)
- Admin config attachment

The `hidden` option in admin config continues to work for backward compatibility.

## Custom Admin Configuration

Create files in `app/admin/` to customize admin behavior per model. The file can export either a plain config object or a fluent `Admin.resource()` builder.

### Plain Object Config

```js
// app/admin/product.admin.js
export default {
  model: 'Product',
  label: 'Products',
  singular: 'Product',
  icon: 'bi-box-seam',
  list: {
    columns: [
      { name: 'name', label: 'Name', sortable: true },
      { name: 'price', label: 'Price', sortable: true },
      { name: 'stock', label: 'Stock' },
    ],
  },
  searchable: ['name', 'slug', 'description'],
  filters: [
    {
      name: 'published',
      label: 'Published',
      type: 'select',
      options: [
        { value: '1', label: 'Yes' },
        { value: '0', label: 'No' },
      ],
    },
  ],
  form: {
    fields: ['name', 'slug', 'description', 'price', 'stock', 'published', 'category'],
  },
  permissions: {
    view: true,
    create: true,
    edit: true,
    delete: false,
  },
}
```

### Fluent Resource Builder

The same config expressed with the chainable builder API:

```js
// app/admin/product.admin.js
import { Admin, Column, Filter, Field, Section, Action } from 'foobarjs/admin'
import Product from '../models/product.model.js'

export default Admin.resource(Product)
  .label('Products', 'Product')
  .icon('bi-box-seam')
  .searchable('name', 'slug', 'description')
  .list(list => list
    .columns([
      Column.text('name').sortable().searchable(),
      Column.money('price').sortable(),
      Column.number('stock'),
    ])
    .filters([
      Filter.boolean('published'),
    ])
  )
  .form(form => form
    .fields([
      Field.text('name').required(),
      Field.textarea('description'),
      Field.belongsTo('category').label('Category'),
    ])
    .sections([
      Section.make('Overview').fields(['name', 'description']).columns(2),
      Section.make('Details').fields(['price', 'stock', 'published', 'category']).columns(2),
    ])
  )
  .permissions({ view: true, create: true, edit: true, delete: false })
```

### Configuration Options

| Option | Type | Description |
|--------|------|-------------|
| `model` | `string` | Model class name this config applies to |
| `label` | `string` | Plural label shown in the UI |
| `singular` | `string` | Singular label shown in the UI |
| `icon` | `string` | Bootstrap Icons class for the sidebar |
| `hidden` | `boolean` | Hide this model from the admin panel (prefer `static admin = false` on the Model) |
| `list.columns` | `array` | Columns to display on the list page |
| `list.fields` | `array` | Deprecated alias for `list.columns` |
| `searchable` | `array` | Fields to include in full-text search |
| `filters` | `array` | Filter fields shown above the list |
| `form.fields` | `array` | Fields to include on create/edit forms |
| `form.exclude` | `array` | Fields to exclude from forms |
| `defaultSort` | `string` | Default sort column |
| `defaultOrder` | `string` | Default sort direction (`asc` or `desc`) |
| `bulkActions` | `array` | Custom bulk actions |
| `bulkHandlers` | `object` | Handlers for custom bulk actions |
| `permissions` | `object` | Per-action permissions |
| `form.sections` | `array` | Group form fields into titled sections |
| `form.columns` | `number` | Number of columns for the form layout (default 1) |
| `detail.fields` | `array` | Fields shown on the detail page |
| `detail.sections` | `array` | Group detail fields into titled sections |
| `widgets` | `array` | Dashboard widgets for this resource |

### Permissions

By default admin users (`isAdmin: true`) can perform any action.

#### Boolean permissions

For simple configs, set each action to `true` or `false`:

```js
export default {
  model: 'Product',
  permissions: {
    view: true,
    create: false,
    edit: false,
    delete: false,
  },
}
```

#### Role-based permissions

For finer control, assign an array of allowed roles to each action. Users must have at least one of the listed roles to perform the action:

```js
export default {
  model: 'Product',
  permissions: {
    view: ['admin', 'editor', 'viewer'],
    create: ['admin', 'editor'],
    edit: ['admin', 'editor'],
    delete: ['admin'],
  },
}
```

The same API works with the fluent builder:

```js
export default Admin.resource(Product)
  .permissions({
    view: ['admin', 'editor', 'viewer'],
    create: ['admin', 'editor'],
    edit: ['admin', 'editor'],
    delete: ['admin'],
  })
  // ...
```

Roles are read from `user.roles`, which can be an array or a JSON-encoded array stored in the database:

```js
class User extends AuthenticableModel {
  static schema = {
    // ...
    roles: Field.json().nullable(),
  }
}
```

#### Admin access

By default only `isAdmin: true` users **or** users with at least one role can enter the admin panel. Two config knobs tune this in `config/admin.js`:

```js
export default {
  requireAdmin: true,   // enforce the entry check at all (default true)
  strictAdmin: false,   // when true, ONLY isAdmin users enter. Roles alone are not enough.
}
```

- `requireAdmin: false` — any authenticated user can enter; per-resource `.permissions({ view: [...] })` becomes the only gate.
- `strictAdmin: true` — locks the panel to `isAdmin` users. Organizers with `roles: ['organizer']` are redirected to `/`. Use this when your app runs organizer workflows outside the admin panel (e.g. a dedicated `/organizer/*` area) and you want a clean separation.

When a permission is denied, the user is redirected back to the admin dashboard with a flash message. When a record id in a URL doesn't resolve (e.g. `/admin/attendees/999999`), the panel flashes `"<Model> not found."` and redirects to the resource index — no 500 crash, no stale-URL leak.

Custom row and bulk [actions](#inline--bulk-actions) are authorized the same way — see `.can()` there.

### Builder Components

When using the fluent API, the following components are available:

#### Column

```js
import { Column } from 'foobarjs/admin'

Column.text('name').sortable().searchable()
Column.number('stock')
Column.money('price', '$')
Column.boolean('published')
Column.badge('status', { active: 'success', inactive: 'secondary' })
Column.image('thumbnail')
Column.date('published_at')
Column.belongsTo('category')
Column.belongsToMany('tags')
```

**Model-aware columns** — when you use a string column name or `Column.make()`, the column type is auto-resolved from the model field. Enum fields become `badge` columns with auto-generated color mappings, boolean fields become `boolean` columns, and date/datetime fields get proper formatting:

```js
// These are equivalent when `status` is an enum field on the model:
Column.make('status')                 // auto-detects badge + generates color map
Column.badge('status', { ... })       // explicit badge with custom colors
```

#### Filter

```js
import { Filter } from 'foobarjs/admin'

Filter.text('sku')
Filter.boolean('published')
Filter.select('status', [
  { value: 'draft', label: 'Draft' },
  { value: 'published', label: 'Published' },
])
```

**Model-aware filters** — if the model field has `.enum()` values or is a boolean, the filter auto-resolves options without you specifying them:

```js
// Auto-resolves options from Field.string().enum('draft', 'published', ...)
Filter.select('status')

// Explicit options still work when you need custom labels or a subset
Filter.select('status', [{ value: 'active', label: 'Active' }])

// Boolean fields auto-resolve to Yes/No
Filter.make('published')  // auto-detects boolean, renders as select
```

#### Field

```js
import { Field } from 'foobarjs/admin'

Field.text('name').required().placeholder('Product name')
Field.textarea('description')
Field.email('contact_email')
Field.select('status', [{ value: 'draft', label: 'Draft' }])
Field.json('metadata')  // monospace textarea with JSON validation hint; auto-parses on submit
Field.belongsTo('category').label('Category')
Field.belongsToMany('tags').label('Tags')
```

Additional builder methods for constraints and input attributes:

```js
Field.number('quantity').min(1).max(100).step(1)
Field.text('notes').helpText('Internal notes, not shown to customers')
Field.email('contact').readonly()
Field.text('code').disabled()
```

#### Form Sections

```js
import { Section } from 'foobarjs/admin'

.form(form => form
  .sections([
    Section.make('Overview').fields(['name', 'slug', 'description']).columns(2),
    Section.make('Pricing & Inventory').fields(['price', 'stock', 'published']),
    Section.make('Media & Relations').fields(['image', 'category', 'tags']),
  ])
)
```

#### Widgets

```js
import { Widget } from 'foobarjs/admin'

.widgets([
  Widget.sum('total-stock', Product, 'stock').label('Total Stock'),
  Widget.count('draft-products', Product.where('published', false)).label('Drafts'),
  Widget.trend('signups-30d', User, { metric: 'count', bucket: 'day', range: 30 })
    .label('Signups (30d)'),
])
```

See the [Dashboard Widgets](#dashboard-widgets) section below for the full widget API.

### Custom Field & Column Types

You can register custom partial templates for field or column types using the component registry in a service provider or boot script:

```js
import { ComponentRegistry } from 'foobarjs/admin'

const registry = new ComponentRegistry()
registry.registerField('rich', 'admin.fields.rich')
registry.registerColumn('sparkline', 'admin.columns.sparkline')
```

Create the partials in `app/views/admin/fields/rich.html` and `app/views/admin/columns/sparkline.html`.

## Template Overrides

Admin templates live in `foobarjs/admin/src/views/`. You can override any template by creating a file with the same path in `app/views/admin/`:

```
app/views/admin/layout.html
app/views/admin/dashboard.html
app/views/admin/models/list.html
app/views/admin/models/show.html
app/views/admin/models/form.html
app/views/admin/failed-jobs.html
app/views/admin/components/flash.html
app/views/admin/components/pagination.html
```

The package templates use the same Edge directives as the rest of the framework (`@layout`, `@section`, `@yield`, `@foreach`, `@if`, `@include`).

## System Group

Framework-owned models (`FailedJob`, `QueueJob`, `NotificationModel`, `PersonalAccessToken`, `AdminExport`) automatically appear in the sidebar under a **System** group with friendly labels and icons. They are `dashboard: false` by default so they don't clutter the dashboard, and their default permissions block create/edit — you can only view and delete rows. All framework models also set `static api = false` so they are not exposed via the REST API.

Each framework model declares a `static adminDefaults = { ... }` object. `AdminPlugin` merges these defaults with any user-defined resource config so your overrides always win:

```js
// framework/packages/queue/src/FailedJob.js
static adminDefaults = {
  group: 'System',
  label: 'Failed Jobs',
  singular: 'Failed Job',
  icon: 'bi-exclamation-triangle',
  dashboard: false,
  permissions: { view: true, create: false, edit: false, delete: true },
  list: {
    columns: [ /* ... */ ],
    actions: [{
      name: 'retry',
      label: 'Retry',
      icon: 'bi-arrow-clockwise',
      handler: async (job) => { await job.retry(); return 'Retried.' },
    }],
    bulkActions: [{
      name: 'retry',
      label: 'Retry selected',
      bulk: true,
      handler: async (ids, { Model }) => { /* ... */ },
    }],
  },
}
```

Your own models can adopt the same pattern to group themselves without importing anything from `foobarjs/admin`.

## Failed Jobs

Failed jobs live at `/admin/failed_jobs` alongside every other model. Retry is available as a per-row action ("Retry") and a bulk action ("Retry selected"). Delete row(s) to clear failed job records.

The old bespoke `/admin/failed-jobs` route has been retired.

### Global Search

Use the search box in the top bar or visit `/admin/search?q=...` to search across all resources. Results are grouped by model and link directly to the record detail page.

### Inline & Bulk Actions

Custom per-row actions can be added to the list config:

```js
import { Admin, Action } from 'foobarjs/admin'
import Order from '../models/order.model.js'

export default Admin.resource(Order)
  .list(list => list
    .actions([
      Action.make('ship', 'Mark as shipped')
        .icon('bi-truck')
        .handler(async (order, { c }) => {
          order.status = 'shipped'
          await order.save()
          return 'Order shipped.'
        }),
    ])
    .bulkActions([
      Action.bulk('archive', 'Archive selected')
        .handler(async (ids, { Model, c }) => {
          // custom bulk logic
        }),
    ])
  )
```

Inline action handlers receive the record and a context object `{ foobar, Model, c, formData, flash, redirect }`. Returning a plain string still works as a success flash (backward compatible). For more control, use the context methods:

- `ctx.flash(type, message)` — return a typed flash message. `type` is one of `'success'`, `'warning'`, `'danger'`, or `'info'`.
- `ctx.redirect(url)` — redirect to a URL instead of back to the list.

Bulk action handlers receive the selected ids and the same context object.

```js
// Return a string — renders as a success flash
.handler(async (item, ctx) => {
  return 'Done!'
})

// Return a typed flash message
.handler(async (item, ctx) => {
  return ctx.flash('warning', 'Completed with warnings.')
})

// Redirect to a different URL (e.g. a download)
.handler(async (item, ctx) => {
  return ctx.redirect(`/admin/exports/${item.id}/download`)
})
```

#### Authorizing actions

Custom actions are authorized **server-side** before the handler runs — a hidden button is not a security boundary. By default an action requires the resource's `edit` permission (actions mutate data, so `view` is too weak and `delete` too strong). Declare a different requirement with `.can()`:

```js
Action.make('refund', 'Refund order')
  .can(['admin', 'finance'])          // an array of allowed roles
  .handler(async (order, { c }) => { /* ... */ }),

Action.make('publish', 'Publish')
  .can('create')                      // a CRUD permission name
  .handler(async (post, { c }) => { /* ... */ }),

Action.make('close', 'Close')
  .can((user, order) => order.status === 'open')   // a predicate
  .handler(async (order, { c }) => { /* ... */ }),
```

`.can()` accepts a CRUD permission name (`'view'`, `'create'`, `'edit'`, `'delete'`), an array of role names, a boolean, or a `fn(user, item)` predicate. `isAdmin: true` users bypass the check. A user who fails is redirected back with a flash message and the handler never runs.

The action's `.visible(fn)` predicate is enforced server-side too: an action hidden from a user cannot be invoked by crafting the request directly.

Built-in bulk actions carry sensible permissions — **Export** requires `view`, **Delete** requires `delete`, **Restore** requires `edit`. All three require confirmation before executing (the user sees a review page with the selected records).

#### Action forms

Actions can collect input from the user before running. Use `.form()` with admin `Field` builders for a unified API:

```js
import { Action, Field } from 'foobarjs/admin'

Action.make('changePassword', 'Change Password')
  .confirm('Set a new password for this user.')
  .form([
    Field.password('password').required().helpText('Minimum 8 characters'),
  ])
  .handler(async (user, { formData }) => {
    user.password = formData.password
    await user.save()
    return 'Password updated.'
  })
```

Action form fields are **model-aware** — if a field name matches a model field, the type and options are auto-resolved:

```js
Action.make('changeStatus', 'Change Status')
  .form([
    Field.make('status'),  // auto-resolves to select with enum options from the model
  ])
  .handler(async (order, { formData }) => {
    order.status = formData.status
    await order.save()
  })
```

Plain objects still work for simple cases: `{ name: 'reason', label: 'Reason', type: 'text' }`.

#### Queued bulk actions

Bulk actions that are slow or touch external services can be dispatched to the background queue instead of running inline. Add `.queued()` to a named action:

```js
import { Admin, Action } from 'foobarjs/admin'
import Order from '../models/order.model.js'

export default Admin.resource(Order)
  .list(list => list
    .bulkActions([
      Action.make('sync-warehouse', 'Sync to Warehouse')
        .icon('bi-cloud-upload')
        .confirm('Sync selected orders to the warehouse?')
        .queued()
        .handler(async (items, { foobar, Model }) => {
          for (const order of items) {
            await warehouseApi.sync(order)
          }
          return `Synced ${items.length} orders.`
        }),
    ])
  )
```

How it works:

- When **foobarjs/queue** is installed, the action is dispatched as a `BulkActionJob`. The user sees a flash message ("has been queued") and is redirected back immediately.
- Without a queue, the action runs inline as usual (synchronously in the request).
- The handler receives hydrated model instances (not raw IDs). When "select all" is used, the job rebuilds the query from the stored filter state instead of serializing thousands of IDs.
- Items are loaded in chunks of 500 to keep memory usage constant regardless of result set size.
- If **foobarjs/notifications** is installed, the user receives a notification when the job completes.

Queued actions **must have a name** (the first argument to `Action.make()`). The name is used as a registry key so the job can resolve the handler function at execution time — anonymous queued actions fall back to inline execution with a warning.

| Method | Description |
|--------|-------------|
| `.queued()` | Dispatch via background job when a queue is available |
| `.handler(fn)` | `fn(items, { foobar, Model })` — receives hydrated model instances |
| `.confirm(msg?)` | Show confirmation page before dispatching |

## CRUD Hooks

Hook into the admin panel's create, update, and delete lifecycle with `beforeStore`, `afterStore`, `beforeUpdate`, `afterUpdate`, `beforeDestroy`, and `afterDestroy`:

```js
import { Admin } from 'foobarjs/admin'
import Order from '../models/order.model.js'

export default Admin.resource(Order)
  .beforeStore(async ({ data, Model, c, foobar }) => {
    if (!data.status) data.status = 'pending'
  })
  .afterStore(async ({ item, data, Model, c, foobar }) => {
    await notifySlack(`New order #${item.id} created`)
  })
  .beforeUpdate(async ({ data, item, Model, c, foobar }) => {
    // `item` is the existing record, `data` is the incoming values
  })
  .afterUpdate(async ({ item, data, Model, c, foobar }) => {
    // `item` has already been saved at this point
  })
  .beforeDestroy(async ({ item, Model, c, foobar }) => {
    await archiveRecord(item)
  })
  .afterDestroy(async ({ item, Model, c, foobar }) => {
    await notifySlack(`Order #${item.id} deleted`)
  })
```

Each hook receives a context object:

| Property | Available in | Description |
|----------|-------------|-------------|
| `data` | before/afterStore, before/afterUpdate | The form data being saved |
| `item` | afterStore, before/afterUpdate, before/afterDestroy | The model instance |
| `Model` | All hooks | The model class |
| `c` | All hooks | The Hono request context |
| `foobar` | All hooks | The app instance |

Hooks are **augmenting** — they run around the existing CRUD logic, not replacing it. Errors thrown in hooks propagate to the standard error handler. `beforeStore` can modify `data` before the record is created. `beforeUpdate` can modify `data` or `item` before saving.

## Data Export

Every list view exposes a first-class **Export selected** bulk action that renders the currently-selected rows (or, when no rows are selected, the currently-filtered query) as CSV.

Behavior:

- If **foobarjs/queue** is installed and the export would touch more than `admin.export.syncThreshold` rows (default **500**), the export is dispatched as a background `ExportJob`. The user is redirected back with a flash message; the completed file appears under **System → Exports** at `/admin/admin_exports`.
- Otherwise the export runs inline and streams the CSV back in the response.

Each export produces an `AdminExport` row with `status` (`pending → processing → complete | failed`), `file_path`, `total_rows`, and a `queryState` blob. Only the initiating user can download the file (or an admin user).

Global configuration:

```js
// config/admin.js
export default {
  export: {
    enabled: true,          // default true; set false to remove the export bulk action
    mode: 'auto',           // 'sync', 'queue', or 'auto' (default)
    syncThreshold: 500,     // rows above which auto-mode uses the queue
  },
}
```

Per-resource overrides with `ExportAction`:

```js
import { Admin, ExportAction } from 'foobarjs/admin'
import Order from '../models/order.model.js'

Admin.resource(Order)
  .list(list => list
    .bulkActions([
      ExportAction.make('export-orders', 'Export Orders')
        .filename('orders-export')
        .columns(['id', 'status', 'total', 'shippingAddress', 'paidAt'])
        .delimiter(';')
        .dateFormat('YYYY-MM-DD HH:mm')
        .confirm('Export all selected orders?'),
    ])
  )
```

`ExportAction` extends `Action` with export-specific fluent methods:

| Method | Description |
|--------|-------------|
| `make(name?, label?)` | Factory. Optional `name` identifies the action for queued exports (default `'export'`). Optional `label` sets the button text (default `'Export'`). |
| `label(string)` | Override the button label after construction. |
| `filename(string \| fn)` | Custom filename (without `.csv`). Accepts a string or a `(query) => string` function. An ISO timestamp is automatically appended to prevent overwriting (e.g. `orders-export-2026-07-18T05-06-51.csv`). |
| `columns(array \| object)` | Columns to export. Array of field names (or mixed with computed columns), or an object where keys are headers and values are field names or formatter functions. |
| `delimiter(string)` | CSV delimiter (default `,`). Use `';'` for European locales. |
| `dateFormat(string)` | Date format string (e.g. `'YYYY-MM-DD'`). Uses dayjs format tokens. Default is ISO 8601. |

**Computed columns** — add derived values to exports using formatter functions. Mix regular field names with `{ label: formatterFn }` objects:

```js
ExportAction.make('export-orders', 'Export Orders')
  .columns([
    'id', 'status', 'total',
    { 'QR Code': (record) => `https://qr.example.com/${record.id}` },
    { 'Full Name': (record) => `${record.firstName} ${record.lastName}` },
  ])
```

Computed columns appear in the column selector on the confirm page. When a named `ExportAction` is used, the framework registers it at boot so queued exports can resolve the formatter functions at execution time.

The object form also supports renaming: `{ 'Order #': 'id', 'Amount': (val, record) => `$${val}` }`.

Without an `ExportAction`, the built-in export uses the model's table name as filename, all list columns, comma delimiter, and ISO dates.

When **foobarjs/notifications** is registered, completing exports create a database notification for the initiating user with a `downloadUrl`.

### Extending AdminExport

Userland apps can extend framework-defined models like `AdminExport` using `Admin.resource()`. Your config layers on top of the model's `adminDefaults` via the merge system — actions are concatenated, not replaced:

```js
// app/admin/export.admin.js
import { Admin, Action, Field } from 'foobarjs/admin'
import { AdminExport } from 'foobarjs/admin'

export default Admin.resource(AdminExport)
  .list(list => list
    .actions([
      Action.make('download', 'Download')
        .icon('bi-download')
        .can('view')
        .visible(({ item }) => item.status === 'complete' && !!item.filePath)
        .handler((item, ctx) => ctx.redirect(`/admin/exports/${item.id}/download`))
        .confirm(),
    ])
  )
```

## Dashboard Widgets

Resources can define async widgets that appear on the dashboard under **Insights**:

Resources can define widgets that appear on the dashboard under **Insights**. Widgets fall into three types — `value`, `trend`, and `chart` — and each has a fluent factory.

### Value widgets (KPI cards)

Query-driven factories build a widget from a `Model` or `RelationQueryBuilder`:

```js
import { Admin, Widget } from 'foobarjs/admin'
import Order from '../models/order.model.js'

export default Admin.resource(Order)
  .widgets([
    Widget.count('pending-orders', Order.where('status', 'pending'))
      .label('Pending Orders')
      .icon('bi-hourglass-split'),

    Widget.sum('total-revenue', Order, 'total')
      .format('currency', { currency: 'USD' })
      .label('Total Revenue')
      .icon('bi-currency-dollar'),

    Widget.avg('avg-order', Order, 'total')
      .format('currency')
      .label('Average Order Value'),

    Widget.exists('has-shipped', Order.where('status', 'shipped'))
      .label('Any Shipped?'),
  ])
```

Available factories:

| Factory | Signature | SQL |
|---------|-----------|-----|
| `Widget.count(name, query)` | Row count | `SELECT COUNT(*)` |
| `Widget.sum(name, query, col)` | Sum of a column | `SELECT SUM(col)` |
| `Widget.avg(name, query, col)` | Average | `SELECT AVG(col)` |
| `Widget.min(name, query, col)` | Minimum | `SELECT MIN(col)` |
| `Widget.max(name, query, col)` | Maximum | `SELECT MAX(col)` |
| `Widget.exists(name, query)` | Boolean | `SELECT 1 ... LIMIT 1` |
| `Widget.value(name, resolver)` | Custom async resolver | (whatever you write) |

All factories accept either a Model class (`Order`) or a pre-built query (`Order.where(...)`). Both work identically; the query is executed at dashboard render time.

### Formatting

Chain `.format(kind, options)` to format the resolved value:

```js
Widget.sum('rev', Order, 'total').format('currency', { currency: 'EUR' })
Widget.count('users', User).format('number')                // 1,234
Widget.avg('rate', Order, 'rating').format('decimal', { precision: 2 })
Widget.value('adoption', () => 0.72).format('percent', { fromRatio: true })
Widget.value('size', () => 1_500_000).format('bytes')       // 1.43 MB
Widget.value('runtime', () => 12500).format('duration', { unit: 'ms' })
Widget.value('custom', () => 42).format(v => `→ ${v}`)      // callback
```

Available formats: `number`, `currency`, `decimal`, `percent`, `bytes`, `duration`. Or pass a function for full control.

### Layout: width and order

```js
Widget.sum(...).width('sm')    // col-lg-3 (default)
Widget.chart(...).width('md')  // col-lg-4
Widget.chart(...).width('lg')  // col-lg-6
Widget.chart(...).width('xl')  // col-lg-12
Widget.count(...).order(1)     // sort ascending; default 0
```

### Trend widgets (time-series sparkline)

```js
Widget.trend('orders-7d', Order, {
  metric: 'count',      // 'count' | 'sum' | 'avg' | 'min' | 'max'
  column: null,         // required for sum/avg/etc
  bucket: 'day',        // 'day' | 'week' | 'month' | 'year'
  range: 7,             // last N buckets
  dateColumn: 'createdAt',
})
  .label('Orders (last 7 days)')
  .icon('bi-graph-up')
```

Trend widgets render a value plus a sparkline chart underneath. The resolver returns `{ value, current, previous, series, labels }` with `previous`/`current` split at the series midpoint so the card can display an up/down delta automatically.

### Chart widgets

```js
Widget.chart('orders-by-status', Order, {
  chart: 'doughnut',       // 'bar' | 'line' | 'pie' | 'doughnut'
  groupBy: 'status',       // categorical grouping
  metric: 'count',
})

Widget.chart('revenue-30d', Order, {
  chart: 'line',
  bucket: 'day',           // time-series grouping
  range: 30,
  metric: 'sum',
  column: 'total',
  dateColumn: 'createdAt',
})
```

`groupBy: 'column'` produces a categorical breakdown, `bucket: 'day'/'week'/...` produces a time series. Optional `limit` caps the number of groups; `labels: { rawValue: 'Display' }` re-labels categorical values.

### Chart rendering

Chart widgets render as a `<canvas>` element and are drawn client-side by [Chart.js](https://www.chartjs.org/). Chart.js is vendored with `foobarjs/admin` (no CDN required) and injected into the admin layout automatically. Values, labels, and colors are theme-aware — bar/line/pie chart colors use `--fb-primary` and complement it with a curated palette.

### Custom resolver context

Custom `Widget.value()` resolvers receive a context object:

```js
Widget.value('per-user-orders', async ({ user, foobar, c }) => {
  if (!user) return 0
  return Order.where('user', user.id).count()
})
```

`user` is the logged-in admin, `c` is the request context (useful for reading query params like `?range=30d`), `foobar` is the app instance. Factory-based widgets (`Widget.count`, `Widget.sum`, ...) can also accept a function that returns a query dynamically per request:

```js
Widget.count('current-user-orders', ({ user }) => Order.where('user', user.id))
```

### Global dashboard widgets

Register widgets on the dashboard that aren't tied to a resource by adding them to `config/admin.js`:

```js
export default {
  theme: { ... },
  dashboard: {
    hideModelCards: false,   // set true to hide auto per-model count cards
    hideFailedJobs: false,   // hide the recent failed jobs table
    widgets: [               // extra widgets shown at the bottom of Insights
      Widget.count('active-users', User.where('lastLoginAt', '>=', last30d))
        .label('Active users (30d)'),
    ],
  },
}
```

## Legacy widget usage

The old `Widget.value(name, async () => ...)` async-resolver API still works and is unchanged. If a widget factory can express your metric (`count`, `sum`, `avg`, `exists`, ...), prefer it — it runs a single aggregate SQL query rather than materializing rows and iterating in JavaScript.

### ❌ Don't do this

```js
// Anti-pattern: fetch all rows and count in JS
Widget.value('pending-orders', async () => {
  const orders = await Order.where('status', 'pending').get()  // fetches ALL rows
  return orders.length                                          // then counts them
})

// Anti-pattern: fetch all rows and sum in JS
Widget.value('total-revenue', async () => {
  const orders = await Order.all()                              // fetches EVERY order
  return orders.reduce((sum, o) => sum + o.total, 0)            // then sums
})
```

### ✓ Do this

```js
Widget.count('pending-orders', Order.where('status', 'pending'))
  .label('Pending Orders')

Widget.sum('total-revenue', Order, 'total')
  .format('currency')
  .label('Total Revenue')
```

Each produces a single `SELECT COUNT(*)` / `SELECT SUM(total)` at the database, avoiding transfer and computation for tens of thousands of rows.

## Custom Data Sources (CustomModel)

Use `CustomModel` to add non-database data sources to the admin panel. Instead of querying the ORM, a `CustomModel` routes all CRUD operations through static methods you implement: `$list`, `$find`, `$create`, `$update`, and `$delete`.

```js
// app/models/system-log.model.js
import { CustomModel, Field } from 'foobarjs/orm'

export default class SystemLog extends CustomModel {
  static tableName = 'system_logs'

  static schema = {
    level: Field.string().enum('info', 'warning', 'error').required(),
    message: Field.string().required(),
    source: Field.string().required(),
    createdAt: Field.date().nullable(),
  }

  static async $list({ page, perPage, filters, sort, search }) {
    const res = await fetch(`https://logging-api.example.com/logs?page=${page}`)
    const json = await res.json()
    return { data: json.items, meta: { total: json.total, page, perPage } }
  }

  static async $find(id) {
    const res = await fetch(`https://logging-api.example.com/logs/${id}`)
    return await res.json()
  }

  static async $create(data) {
    const res = await fetch('https://logging-api.example.com/logs', {
      method: 'POST',
      body: JSON.stringify(data),
      headers: { 'Content-Type': 'application/json' },
    })
    return await res.json()
  }

  static async $update(id, data) {
    const res = await fetch(`https://logging-api.example.com/logs/${id}`, {
      method: 'PUT',
      body: JSON.stringify(data),
      headers: { 'Content-Type': 'application/json' },
    })
    return await res.json()
  }

  static async $delete(id) {
    await fetch(`https://logging-api.example.com/logs/${id}`, { method: 'DELETE' })
  }
}
```

Register it in the admin panel like any other model:

```js
// app/admin/system-log.admin.js
import { Admin, Column, Filter } from 'foobarjs/admin'
import SystemLog from '../models/system-log.model.js'

export default Admin.resource(SystemLog)
  .label('System Logs', 'Log')
  .icon('bi-terminal')
  .group('System')
  .searchable('message', 'source')
  .list(list => list
    .columns([
      Column.text('id').sortable(),
      Column.badge('level', { info: 'info', warning: 'warning', error: 'danger' }),
      Column.text('message'),
      Column.text('source').sortable(),
    ])
    .filters([Filter.select('level')])
  )
```

### How it works

- `CustomModel` extends `Model` but bypasses MikroORM entirely — no database table is created.
- `$list()` receives `{ page, perPage, filters, sort, search }` and must return `{ data, meta: { total, page, perPage } }`.
- `$find(id)` returns a plain object or `null`.
- `$create(data)` and `$update(id, data)` return the created/updated object.
- `$delete(id)` removes the record.
- The admin panel's list, show, create, edit, and delete views all work transparently.
- `CustomModel` instances support schema validation — your `static schema` field definitions are enforced on `create()` and `save()`.

### Limitations

- No relation queries (`.where().orderBy().get()`) — list uses `$list()` directly.
- No eager loading or relations between custom and DB-backed models.
- `$list()` returns pre-paginated data — the admin passes filter/sort/search params for the implementation to handle.

## Theme and Branding

Configure the admin theme in `config/admin.js`:

```js
export default {
  theme: {
    brand: 'My App Admin',
    logo: '/admin-logo.svg',
    primaryColor: '#6366f1',
    mode: 'dark',
  },
}
```

Bootstrap 5 CSS, JavaScript, and Bootstrap Icons are bundled locally in `foobarjs/admin` so the admin panel works without an internet connection.

## Design System

The admin panel ships with a shadcn-inspired design system built on top of Bootstrap 5. All colors, radii, spacing, and typography are defined as CSS custom properties in `public/css/admin.css`, so overriding them from your own stylesheet is a one-line change.

```css
:root {
  --fb-primary: 240 5.9% 10%;        /* HSL triple */
  --fb-primary-foreground: 0 0% 98%;
  --fb-radius: 0.5rem;
  --fb-sidebar-width: 260px;
}
```

To override, add your own stylesheet after `admin.css`:

```html
<link rel="stylesheet" href="/admin-assets/css/admin.css">
<link rel="stylesheet" href="/css/admin-overrides.css">
```

## Dark Mode

Dark mode is fully supported. Users can toggle between light and dark themes with the topbar theme button (persisted in the `foobar_admin_theme` cookie). Set the default in `config/admin.js`:

```js
theme: { mode: 'light' }  // or 'dark'
```

The dark theme reuses the same tokens with dark equivalents, so custom brand color overrides work in both modes.

## Sidebar Groups

Group models in the sidebar with `.group(name)`:

```js
Admin.resource(Product).group('Catalog')
Admin.resource(Category).group('Catalog')
Admin.resource(Order).group('Sales')
Admin.resource(User).group('People')
```

Ungrouped models fall into the "Main" group. Framework models are pinned under **System** (rendered last). Users can collapse the sidebar with the topbar toggle (persisted per-user).

## Dashboard Cards Are Opt-In

Stat cards on the dashboard are opt-in per resource:

```js
Admin.resource(Product).dashboard({})
Admin.resource(Order).dashboard({ icon: 'bi-cart', color: 'primary' })
```

Passing `.dashboard({})` enables a card with defaults derived from the resource label/icon. Passing an options object lets you override `label`, `icon`, `link`, or `color`. Resources without `.dashboard()` are excluded from the dashboard entirely — this keeps the dashboard focused on the metrics that matter, not a list of every table in the database.

If no resources opt in and no widgets are configured, the dashboard shows a friendly empty state explaining how to enable cards or widgets.

Global widgets (not tied to a specific resource) can be defined in `config/admin.js` under `dashboard.widgets`.

## Filters

Filters live in a **popover** triggered by a "Filters" button in the toolbar. When any filter is active, a chip strip renders directly under the toolbar with one removable pill per active filter.

Available on `Filter`:

```js
Filter.select('status', [
  { value: 'pending', label: 'Pending' },
  { value: 'shipped', label: 'Shipped' },
])
  .default('pending')          // apply this value when no URL param is set
  .label('Order status')       // override auto-derived label
  .placeholder('All')          // shown in the empty option

Filter.belongsTo('user')       // renders as a lazy <fb-combobox>
Filter.boolean('published')    // renders as a Yes/No select
```

Available on `ListConfig`:

```js
Admin.resource(Product).list(list => list
  .deferFilters(true)          // in-progress: don't apply filters until the user clicks Apply
)
```

> **Note:** `belongsTo` filters are now auto-generated by default for all models (even without an `Admin.resource()` config). The `.autoFilters()` option is no longer needed.

Filter URL contract is unchanged: `f[key]=value`. The URL is the single source of truth for active filters — removing a chip navigates to a URL with `f[key]=` (empty) which explicitly clears that filter.

## Command Palette

Press `Cmd+K` (macOS) or `Ctrl+K` (elsewhere) to open the global command palette. It uses the same `/admin/search` endpoint with `?format=json` and lets users jump to any record across all resources with the keyboard.

## Relationships

### Lazy-loaded belongsTo and belongsToMany

Every `belongsTo` and `belongsToMany` field renders as a searchable `<fb-combobox>` fed by a per-resource lookup endpoint:

```
GET /admin/:table/lookup?q=<term>&page=1&perPage=25
GET /admin/:table/lookup?ids=1,2,3      # hydrate labels for known ids
```

Response shape:

```json
{
  "data": [{ "value": 42, "label": "Electronics" }],
  "meta": { "page": 1, "perPage": 25, "total": 120, "hasMore": true }
}
```

No related records are eagerly loaded when rendering forms or lists. The list-view lookup batches into a single `WHERE id IN (...)` query per belongsTo column, so a page of 25 orders no longer loads the entire users table.

The endpoint reuses the resource's `view` permission and honors `.searchable(...)` fields for the `q` filter.

### Real belongsTo filters

Filter the list by a related record:

```
GET /admin/products?f[category]=42
```

The controller detects `category` is a relation and applies `WHERE category_id = 42` (not `LIKE %42%`). Filter templates use the same combobox UI.

Enable auto-generation for every belongsTo relation in a list:

```js
.list(list => list.autoFilters(true))
```

### Custom relation display

By default related labels fall back to `name → title → slug → #<id>`. Override at the resource level:

```js
Admin.resource(Order).displayLabel(o => `Order #${o.id} (${o.status})`)
```

Or per column:

```js
Column.belongsTo('user').display(u => `${u.name} <${u.email}>`)
```

Or per field:

```js
Field.belongsTo('category').display(c => c.name.toUpperCase())
```

`displayLabel` is used by the lookup endpoint, the list-view chip rendering, and the detail-view link label.

### hasMany link-outs

`hasMany` fields render as a compact table on the parent form with links to the child edit pages and an **Add** button that navigates to the child create form with the parent's foreign key pre-filled:

```
GET /admin/order-items/create?order=42
```

The controller reads the query string and prefills recognized field names (columns or relation FKs). No inline CRUD is performed on the parent form — child records are created and edited via their own admin pages.

## Detail Page Sections and Tabs

Group detail fields into cards or tabs:

```js
Admin.resource(Order).detail(detail => detail
  .fields(['id', 'user', 'status', 'total', 'paidAt', 'shippingAddress'])
  .sections([
    Section.make('Overview').fields(['id', 'user', 'status']).columns(2),
    Section.make('Payment').fields(['total', 'paidAt']).columns(2).icon('bi-cash-coin'),
    Section.make('Shipping').fields(['shippingAddress']).icon('bi-truck'),
  ])
  .tabs(true)  // Render sections as tabs instead of stacked cards
)
```

On the detail page, `belongsTo` values are rendered as clickable chip links to the related record. `belongsToMany` values become a chip list, capped at 10 with a "+N more" indicator.

### Custom field display

Use `Field.display(fn)` to render custom HTML for a field in the detail view. The callback receives the model instance and returns an HTML string:

```js
Field.make('filePath').display((item) => {
  return `<a href="/download/${item.id}">${item.filePath}</a>`
})
```

## Prefs Endpoint

Client-side UI state (theme, sidebar collapse, table density, column visibility) is persisted via `POST /admin/prefs`. The request accepts:

- `theme=light|dark`
- `sidebar=expanded|collapsed`
- `density=compact|cozy|comfortable`
- `table_cols_<table>=col1,col2,col3` (comma list of hidden columns)

All values are validated server-side and stored as scoped cookies (`Path=/admin`).

## Client Custom Elements

The client library at `/admin-assets/js/admin.js` exposes a small set of custom elements you can use in your own admin templates:

- `<fb-combobox endpoint="/admin/foo/lookup" name="foo">` — async searchable select
- `<fb-tabs>` / `<fb-tab label="...">` — accessible tabs
- `<fb-confirm-form>` — replaces `window.confirm` with a themed modal
- `<fb-dialog>` — a plain modal wrapper
- `<fb-toast-host>` — flash message container
- `<fb-command>` — command palette (auto-mounted in the layout)
- `<fb-autosize>` — auto-growing textarea wrapper
- `<fb-password>` — password field with an eye toggle
- `<fb-file>` — file input with drop zone and preview

There is no build step. All elements progressively enhance server-rendered HTML — no-JS clients still get a functional native form fallback.
