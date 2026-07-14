# Directory Structure

foobarjs follows a convention-over-configuration directory layout.

```
my-app/
├── app/
│   ├── controllers/       # Request handlers (extend Controller)
│   ├── models/            # Active Record models (extend Model)
│   ├── views/             # Templates (.html)
│   │   ├── layouts/       # Layout templates
│   │   └── errors/        # 404/403/500/419 overrides
│   ├── admin/             # Admin panel resource configs
│   ├── middleware/        # Custom middleware
│   ├── validators/        # FormRequest subclasses
│   ├── serializers/       # API response shapers
│   ├── jobs/              # Queue jobs
│   ├── events/            # Event classes
│   ├── listeners/         # Event listeners
│   └── notifications/     # Notification classes
├── config/                # Config files (one per subsystem)
├── database/
│   ├── foobar.db          # SQLite database (gitignored)
│   ├── migrations/        # Migration files
│   └── seeders/           # Seeder classes
├── routes/
│   ├── web.js             # Explicit route registration
│   └── api.js             # Optional; same shape as web.js
├── public/                # Static assets (served at /)
│   ├── css/
│   └── uploads/           # Storage default disk root
├── test/                  # Test files
├── .env                   # Environment variables (gitignored)
├── .env.example           # Committed template
└── package.json
```

## Auto-discovery

Foobarjs walks specific directories at boot and wires things up by convention.

| Directory | What happens |
|-----------|--------------|
| `app/controllers/` | Each file becomes a REST resource at `/<filename>`. `home.controller.js` also mounts at `/`. Only methods you define get routes. |
| `app/models/` | Registered with the ORM. The default-exported class from each file is imported and registered. Table name derives from the class name: `User` → `users`, `OrderItem` → `order_items`. Override with `static tableName`. |
| `app/views/` | Available to `this.render('folder/file')` from a controller. |
| `app/admin/` | Registered with the admin panel. `product.admin.js` mounts under `/admin/products`. |
| `app/middleware/` | Available by name for route assignment. |
| `app/jobs/` | Registered with the queue system. |
| `app/events/` | Event classes for dispatch/listener wiring. |
| `app/listeners/` | Auto-discovered via `static events = [...]`. |
| `routes/web.js` | Explicit routes, loaded after the filename convention runs. |
| `routes/api.js` | Same shape as `web.js`. |
| `config/*.js` | Loaded by `ConfigLoader`. Values accessible via `foobar.configLoader.get('app.name')`. |
| `database/migrations/` | Run in filename order by `foobar db:migrate`. |
| `database/seeders/` | `foobar db:seed` looks for `DatabaseSeeder.js` here, or falls back to `seed.js` at the project root. |
| `public/` | Served as static files. |

Folders that are **not** auto-discovered — they're conventional locations, but
you import from them explicitly:

| Folder | Consumed by |
|--------|-------------|
| `app/validators/` | Imported by the controller that calls `this.validate(FooValidator)`. |
| `app/serializers/` | Looked up per model at request time by `autoSerialize(Model, data)` — filename must match the lowercased class name (`Product` → `app/serializers/product.serializer.js`). |
| `app/notifications/` | Imported where dispatched (`Notification.send(user, new OrderShipped(order))`). |

For a complete rundown of every foobarjs convention (URL mapping,
auto-response contract, view lookup rules, framework-owned tables, etc.)
see [Conventions](./conventions.md).

## Next steps

- Read the full convention reference: [Conventions](./conventions.md)
- Add controllers: [Controllers](./controllers.md)
- Wire routes: [Routing](./routing.md)
- Configure the app: [Configuration](./configuration.md)
- Model your data: [ORM: getting started](./orm/getting-started.md)
