# Directory Structure

foobarjs follows a convention-over-configuration directory structure inspired by Laravel.

```
my-app/
├── app/
│   ├── controllers/       # Route handlers
│   ├── models/            # Active Record models
│   ├── views/             # Edge.js templates (.html)
│   │   └── layouts/       # Layout templates
│   ├── middleware/         # Custom Hono middleware
│   ├── validators/        # Validation rulesets
│   ├── serializers/       # API serialization classes
│   ├── jobs/              # Queue job classes
│   ├── events/            # Event classes
│   └── listeners/         # Event listener classes
├── config/                # Configuration files (.js)
├── database/              # Database files
│   ├── migrations/        # Migration files
│   └── seeders/           # Seeder classes
├── public/                # Static assets (served at /)
│   └── css/
├── resources/             # Uncompiled assets
├── test/                  # Test files
├── .env                   # Environment variables
└── package.json
```

## Convention Discovery

| Directory | Auto-Discovery | Convention |
|-----------|---------------|------------|
| `app/controllers/` | Routes built from controller names | `products.controller.js` → `ProductsController` → `/products` |
| `app/models/` | ORM entity registration | `user.model.js` → `User` model → `users` table |
| `app/views/` | Template lookup by name | `products/index.html` → `products.index` |
| `app/middleware/` | Available for route assignment | Name from filename |
| `app/validators/` | Custom validation rules | `user.validator.js` → `UserValidator` |
| `app/serializers/` | Auto-used by API plugin | `product.serializer.js` → `ProductSerializer` |
| `app/listeners/` | Events auto-discovered from `static events` | Registration on boot |
