[← Back to docs](./README.md)

{% raw %}
# Authorization

foobarjs provides a **gate** system for authorization. Gates are pure functions that answer one question: *"Is this user allowed to do this?"*

They work across the admin panel, API plugin, and custom web routes — write the rule once, enforce it everywhere.

## Quick Start

Generate a gate:

```bash
foobar generate gate order
```

This creates `app/gates/order.gate.js`:

```js
import { Gate } from 'foobarjs/auth'
import Order from '../models/order.model.js'

export default Gate({
  model: Order,

  viewAny(user) {
    return user.hasRole('staff', 'admin')
  },

  view(user, order) {
    return order.user_id === user.id || user.hasRole('staff')
  },

  create(user) {
    return true
  },

  update(user, order) {
    return order.user_id === user.id || user.hasRole('admin')
  },

  delete(user, order) {
    return user.hasRole('admin')
  },

  scope(user, query) {
    if (user.hasRole('admin')) return query
    return query.where('user_id', user.id)
  },
})
```

Gates are auto-discovered from `app/gates/` on boot — no manual registration needed.

## Resolution Order

When a gate check runs, the system follows this order:

1. **`before(user, action)`** — if defined, runs first. Return `true` to allow, `false` to hard deny (even super-admins), or `undefined` to continue.
2. **Super-admin bypass** — if `user.isAdmin` is `true`, the action is allowed.
3. **Gate action function** — the specific function (e.g., `view`, `update`) runs and returns `true` or `false`.

If no gate exists for a model, the check returns `null` (no opinion), and the caller decides the fallback.

## Gate Actions

Standard actions match CRUD operations:

| Action | When checked |
|--------|-------------|
| `viewAny` | Listing resources (index) |
| `view` | Viewing a single resource |
| `create` | Creating a new resource |
| `update` | Updating an existing resource |
| `delete` | Deleting a resource |

Define custom actions for domain-specific operations:

```js
export default Gate({
  model: Ticket,

  'check-in'(user, ticket) {
    return user.hasRole('staff') && ticket.status === 'assigned'
  },

  refund(user, ticket) {
    return user.hasRole('finance')
  },

  assign(user, ticket) {
    return ticket.order.email === user.email || user.hasRole('staff')
  },
})
```

## Scoping

The `scope` function filters queries so users only see records they're authorized to access:

```js
scope(user, query) {
  if (user.hasRole('admin')) return query
  return query.where('organization_id', user.organization_id)
}
```

Scopes are automatically applied to API list endpoints and can be manually applied in controllers.

## The `before()` Hook

Use `before()` for cross-cutting rules that apply to all actions:

```js
export default Gate({
  model: Order,

  async before(user, action) {
    // Suspended users can never do anything
    if (user.suspended) return false

    // Compliance team can view but not modify
    if (user.hasRole('compliance') && action === 'view') return true
  },

  // ... action functions
})
```

Returning `false` from `before()` is a **hard deny** — it blocks even super-admins. This is useful for compliance, legal holds, or account suspension.

`before()` can be `async` — the framework awaits it, so you can perform database lookups or external checks safely.

## Standalone Gates

Gates without a `model` handle non-resource authorization. The framework reads the function names automatically — just define the functions:

```js
// app/gates/analytics.gate.js
import { Gate } from 'foobarjs/auth'

export default Gate({
  view(user) {
    return user.hasRole('analyst', 'admin')
  },

  export(user) {
    return user.hasRole('admin')
  },
})
```

## Integration

### Route Model Binding

Use `.bind()` on routes to automatically resolve a model from a route parameter. The param name matches the model name (lowercase):

```js
import Order from '../app/models/order.model.js'

router.put('/orders/:order/refund', OrderController, 'refund')
  .bind(Order)
```

The resolved instance is available via `this.bound('order')` in controllers or `c.req.bound('order')` in closures. If the record isn't found, a `404 Not Found` is thrown automatically.

#### Combining `.bind()` with `.can()`

This is the recommended pattern for per-item authorization on routes:

```js
// The gate receives the bound Order instance — per-item check works
router.put('/orders/:order/refund', OrderController, 'refund')
  .bind(Order)
  .can('refund', Order)
```

Without `.bind()`, the gate runs with no item (`null`) — only useful for user-level checks like "can this user access the analytics dashboard?" With `.bind()`, the gate gets the actual record to check ownership, status, or other per-item rules.

#### In the controller

```js
class OrderController extends Controller {
  async refund() {
    const order = this.bound('order')   // already loaded and authorized
    const { reason } = this.body        // pre-parsed request body

    order.status = 'refunded'
    order.refund_reason = reason
    await order.save()

    return this.json({ message: 'Order refunded', order })
  }
}
```

#### Auto-binding with `resource()`

Pass a Model as the third argument to `router.resource()` to auto-bind on detail routes:

```js
router.resource('/orders', OrderController, Order)
// GET /orders/:id → Order auto-bound, available as this.bound('order')
```

Convention controllers can declare `static model` to auto-bind without any route configuration:

```js
class OrderController extends Controller {
  static model = Order

  async show() {
    const order = this.bound('order')  // auto-loaded from :id param
    return this.json(order)
  }
}
```

### Admin Panel

The admin panel uses gates for all authorization. One system everywhere.

**If you define a gate file** (`app/gates/order.gate.js`), the admin uses it directly — menu visibility, create/edit/delete buttons, and action permissions are all driven by the gate.

**If you use `.permissions()` on `Admin.resource()`**, the framework generates an equivalent gate automatically:

```js
// This admin config...
Admin.resource(Order).permissions({
  view: ['admin', 'editor'],
  create: ['admin'],
  edit: ['admin', 'editor'],
  delete: ['admin'],
})

// ...generates a gate equivalent to:
Gate({
  model: Order,
  viewAny(user) { return user.hasRole('admin', 'editor') },
  view(user) { return user.hasRole('admin', 'editor') },
  create(user) { return user.hasRole('admin') },
  update(user) { return user.hasRole('admin', 'editor') },
  delete(user) { return user.hasRole('admin') },
})
```

**If neither exists**, the admin generates a default gate that only allows `isAdmin` users.

If both a gate file and `.permissions()` exist, the gate file wins.

The admin maps its actions to gate actions: `view` → `viewAny`, `edit` → `update`. Custom admin actions (e.g. `Action.make('ship')`) check the gate's `ship()` function if defined, falling back to `update()` if not.

### API Plugin

The API plugin checks gates on every endpoint:

- **List** (`GET /api/orders`): checks `viewAny`, applies `scope`
- **Show** (`GET /api/orders/:id`): checks `view` per item
- **Create** (`POST /api/orders`): checks `create`
- **Update** (`PUT /api/orders/:id`): checks `update` per item
- **Delete** (`DELETE /api/orders/:id`): checks `delete` per item

If a gate returns `false`, the API responds with `403 Forbidden`. If no gate exists for a model with authenticated endpoints, the API returns `403 Forbidden`. Public endpoints (`auth: false`) bypass gate checks. This enforces the gate requirement: every authenticated API model must have a gate.

### Controllers

Use `this.authorize()` inside any controller action to check a gate explicitly. This works in both convention (CoC) and custom controllers:

```js
class OrderController extends Controller {
  async show() {
    const order = await Order.find(this.param('id'))
    await this.authorize('view', order)      // throws 403 if denied
    return this.render('orders/show', { order })
  }

  async index() {
    await this.authorize('viewAny', Order)   // pass the Model class for collection checks
    const orders = await Order.all()
    return this.render('orders/index', { orders })
  }

  async update() {
    const order = this.bound('order')
    await this.authorize('update', order)    // pass an instance for per-item checks
    order.fill(this.body)
    await order.save()
    return this.json(order)
  }
}
```

`this.authorize(action, modelOrInstance)` accepts either a **Model class** (for collection-level checks like `viewAny`, `create`) or a **model instance** (for per-item checks like `view`, `update`, `delete`). When you pass an instance, the gate receives it as the second argument.

If no user is authenticated, it throws `AuthenticationError` (401). If the gate denies or no gate exists, it throws `ForbiddenError` (403).

### Web Routes

Use `.can()` on routes to require gate authorization:

```js
// Standalone gate — user-level check, no item needed
router.get('/dashboard', DashboardController, 'index')
  .can('view', 'analytics')

// Model gate with binding — per-item authorization
router.put('/orders/:order/refund', OrderController, 'refund')
  .bind(Order)
  .can('refund', Order)
```

The gate is checked after authentication middleware and model binding. If the check fails, a `403 Forbidden` error is thrown.

> **Note:** `.can()` requires an authenticated user. If no user is present (e.g. on a public route), foobarjs throws an `AuthenticationError` (HTTP 401).

### Gates auto-apply on both admin and API

Any Model that has a registered Gate (`app/gates/*.gate.js`) is automatically
checked on both the admin panel and the auto-mounted API. The API plugin maps
each REST verb to a gate action:

| REST verb | Gate action |
|-----------|-------------|
| `index`   | `viewAny`   |
| `show`    | `view`      |
| `store`   | `create`    |
| `update`  | `update`    |
| `destroy` | `delete`    |

Attach additional or custom gates with the fluent `.can()` on the resource:

```js
Api.resource(Order)
  .can('refund', Order, 'update')   // extra gate on PUT/PATCH only
```

**Gates fire only when there is a user.** For `.public()` verbs no user is
resolved, so the API skips the gate entirely — the same request on the same
model returns data anonymously when public and enforces the gate as soon as a
user is present.

## Roles

### `hasRole()`

`AuthenticableModel` provides `hasRole()` for checking roles stored in the `roles` JSON column:

```js
user.hasRole('admin')                    // single role
user.hasRole('staff', 'admin')           // any of these roles
```

### Role Picker in Admin

Use `Field.tags()` in your admin config to provide a role picker:

```js
// app/admin/user.admin.js
import { Field } from 'foobarjs/orm'

export default {
  fields: [
    Field.text('name'),
    Field.email('email'),
    Field.tags('roles').placeholder('Add roles...'),
    // Or with suggestions:
    // Field.tags('roles').options(['admin', 'editor', 'staff', 'viewer']),
  ],
}
```

This renders a tag input with chips — type a value and press Enter to add, click the X to remove. The field stores its value as a JSON array.

## Inspect CLI

View all routes with their auth and gate status:

```bash
foobar inspect:routes
```

```
METHOD  PATH                AUTH      GATE           HANDLER
GET     /                   public    -              HomeController#index
GET     /api/orders         required  -              api/auto
POST    /api/orders         required  -              api/auto
GET     /dashboard          required  analytics.view DashboardController#index
```

Use `--strict` to exit with a non-zero code if any authenticated route lacks a gate (useful in CI):

```bash
foobar inspect:routes --strict
```

Use `--json` for machine-readable output.

## Business Logic vs Authorization

Gates answer "Is this user **allowed** to do this?" They do not enforce business rules.

**Litmus test:** Would a super-admin also be blocked? If yes, it's business logic, not authorization.

| Gate (authorization) | Business logic |
|---------------------|---------------|
| Can this user edit orders? | Is this order still editable (not shipped)? |
| Can this user refund tickets? | Is the refund window still open? |
| Can this user view this report? | Does the report have data yet? |

Business rules belong in controllers, models, or validators — not in gates.
{% endraw %}
