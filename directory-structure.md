# Directory Structure

foobarjs follows a convention-over-configuration directory structure inspired
by Laravel.

```
my-app/
├── app/
│   ├── controllers/       # Request handlers (extend Controller)
│   ├── models/            # Active Record models (extend Model)
│   ├── views/             # Templates (.html)
│   │   ├── layouts/       # Layout templates
│   │   └── errors/        # 404/403/500/419 overrides
│   ├── admin/             # Admin panel resource configs
│   ├── middleware/        # Custom Hono middleware
│   ├── validators/        # FormRequest subclasses
│   ├── serializers/       # API response shapers
│   ├── jobs/              # Queue jobs
│   ├── events/            # Event classes
│   ├── listeners/         # Event listeners
│   └── notifications/     # Notification classes
├── config/                # Config files (one per subsystem)
├── database/
│   ├── migrations/        # Migration files
│   └── seeders/           # Seeder classes (or seed.js at root)
├── routes/
│   ├── web.js             # Explicit route registration
│   └── api.js             # Optional; same shape as web.js
├── public/                # Static assets (served at /)
│   ├── css/
│   └── uploads/           # Storage default disk root
├── test/                  # Test files
├── .env                   # Environment variables (gitignored)
├── .env.example           # Committed template
├── seed.js                # Optional database seeder (root-level shortcut)
└── package.json
```

## Auto-discovery

Foobarjs walks specific directories at boot and wires things up by convention.

| Directory | What happens |
|-----------|--------------|
| `app/controllers/` | Each file becomes a REST resource at `/<filename>`. `home.controller.js` also mounts at `/`. Only methods you define get routes. |
| `app/models/` | Registered with the ORM. `user.model.js` → `User` class → `users` table (unless `static tableName` overrides). |
| `app/views/` | Available to `this.render('folder/file')` from a controller. |
| `app/admin/` | Registered with the admin panel. `product.admin.js` mounts under `/admin/products`. |
| `app/middleware/` | Available by name for route assignment. |
| `app/validators/` | `FormRequest` subclasses. Used by `this.validate(FormRequestClass)`. |
| `app/serializers/` | Auto-used by the `foobarjs/api` plugin for matching models. |
| `app/jobs/` | Registered with the queue system. |
| `app/events/` | Event classes for dispatch/listener wiring. |
| `app/listeners/` | Auto-discovered via `static events = [...]`. |
| `app/notifications/` | `Notification` subclasses. |
| `routes/web.js` | Explicit routes, loaded after the filename convention runs. |
| `routes/api.js` | Same shape as `web.js`. |
| `config/*.js` | Loaded by `ConfigLoader`. Values accessible via `foobar.configLoader.get('app.name')`. |
| `database/migrations/` | Run in filename order by `foobar db migrate`. |
| `database/seeders/` | `foobar db seed` looks for `DatabaseSeeder.js` here, or falls back to `seed.js` at the project root. |
| `public/` | Served as static files. |

## Next steps

- Add controllers: [Controllers](./controllers.md)
- Wire routes: [Routing](./routing.md)
- Configure the app: [Configuration](./configuration.md)
- Model your data: [ORM: getting started](./orm/getting-started.md)
