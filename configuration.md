# Configuration

foobarjs uses a layered configuration system: built-in defaults → environment variables → config files in `config/`.

## Built-in Defaults

The framework ships with sensible defaults for every config key. You only need to create config files for values you want to override. A minimal app with no `config/` directory at all will boot with:

| Key | Default | Notes |
|-----|---------|-------|
| `app.name` | `'Foobar App'` | |
| `app.port` | `3000` | `PORT` env |
| `app.debug` | `true` (false in production) | `APP_DEBUG` env |
| `app.log.level` | `'info'` | `LOG_LEVEL` env |
| `app.log.file` | `'storage/logs/app.log'` | `LOG_FILE` env; set to empty string to disable |
| `app.log.maxSize` | `10485760` (10 MB) | Rotate when daily log exceeds this size |
| `app.log.maxFiles` | `5` | Max rotated files to keep per day |
| `app.log.maxDays` | `14` | Auto-delete daily logs older than this |
| `database.driver` | `'sqlite'` | `DB_CONNECTION` env |
| `database.database` | `'foobar.db'` | `DB_DATABASE` env |
| `session.driver` | `'cookie'` | `SESSION_DRIVER` env |
| `session.lifetime` | `120` (minutes) | |
| `session.secure` | `true` in production | |
| `security.rateLimit.max` | `100` | Requests per window |
| `security.rateLimit.windowMs` | `60000` | 1 minute |
| `cors.origin` | `APP_URL` or `'http://localhost:3000'` | |
| `cors.credentials` | `true` | |
| `queue.default` | `'sync'` | `QUEUE_CONNECTION` env |
| `cache.default` | `'memory'` | `CACHE_STORE` env |
| `mail.driver` | `'log'` | `MAIL_DRIVER` env |
| `mail.from` | `'hello@example.com'` | `MAIL_FROM` env |
| `storage.default` | `'local'` | |
| `storage.disks.local.root` | `'public/uploads'` | |

User config files are deep-merged on top of these defaults — any value you set takes precedence.

## Environment Files

`.env` files are loaded first. Environment-specific overrides can be placed in `.env.{NODE_ENV}` (e.g., `.env.development`, `.env.production`, `.env.test`).

```env
APP_URL=http://localhost:3000
PORT=3000
NODE_ENV=development
DB_CONNECTION=sqlite
DB_DATABASE=foobar.db
SESSION_DRIVER=cookie
LOG_FILE=storage/logs/app.log
```

Variables set in `.env` are injected into `process.env` and will not override already-set environment variables. Variables in `.env.{NODE_ENV}` **do** override `.env` values.

### Test Environment

The `foobar new` scaffold generates a `.env.test` file that isolates the test database:

```env
NODE_ENV=test
DB_DATABASE=test.db
LOG_FILE=
```

This ensures `foobar test` uses `test.db` instead of your development database, preventing test data from polluting your dev environment. `LOG_FILE=` disables file logging during tests.

## Config Files

Config files are JavaScript modules located in the `config/` directory. Each file exports an object that becomes available under its filename as a namespace.

### `config/app.js`

```js
export default {
  name: process.env.APP_NAME || 'Foobar App',
  url: process.env.APP_URL || 'http://localhost:3000',
  port: parseInt(process.env.PORT || '3000'),
  env: process.env.NODE_ENV || 'development',
  debug: process.env.APP_DEBUG === undefined
    ? process.env.NODE_ENV !== 'production'
    : (process.env.APP_DEBUG === 'true' || process.env.APP_DEBUG === '1'),
  secret: process.env.APP_SECRET,
  log: {
    level: process.env.LOG_LEVEL || 'info',
    file: process.env.LOG_FILE || 'storage/logs/app.log',
    maxSize: 10 * 1024 * 1024, // 10 MB — rotate when a daily log exceeds this
    maxFiles: 5,               // rotated files to keep per day (.1, .2, …)
    maxDays: 14,               // auto-delete daily logs older than this
  },
  plugins: ['foobarjs/auth', 'foobarjs/admin', 'foobarjs/api', 'foobarjs/api-docs'],
}
```

The `plugins` key defines which first-party plugins to load at boot.

### `config/database.js`

The `driver` key determines the database driver. Supported values:
`sqlite`, `postgres`, `mysql`, `mongodb`.

**SQLite** (default):

```js
export default {
  driver: process.env.DB_CONNECTION || 'sqlite',
  database: process.env.DB_DATABASE || 'foobar.db',
}
```

**PostgreSQL**:

```js
export default {
  driver: 'postgres',
  host: process.env.DB_HOST || 'localhost',
  port: Number(process.env.DB_PORT || 5432),
  database: process.env.DB_DATABASE || 'myapp',
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD || '',
}
```

**MySQL**:

```js
export default {
  driver: 'mysql',
  host: process.env.DB_HOST || 'localhost',
  port: Number(process.env.DB_PORT || 3306),
  database: process.env.DB_DATABASE || 'myapp',
  user: process.env.DB_USER || 'root',
  password: process.env.DB_PASSWORD || '',
}
```

**Multiple connections** — add named connections under `connections`. Each
named connection specifies its own `driver`:

```js
export default {
  driver: 'postgres',
  host: process.env.DB_HOST || 'localhost',
  port: Number(process.env.DB_PORT || 5432),
  database: process.env.DB_DATABASE || 'myapp',
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD || '',

  connections: {
    analytics: {
      driver: 'mysql',
      host: process.env.ANALYTICS_DB_HOST || 'localhost',
      port: Number(process.env.ANALYTICS_DB_PORT || 3306),
      database: 'analytics',
      user: process.env.ANALYTICS_DB_USER || 'root',
      password: process.env.ANALYTICS_DB_PASSWORD || '',
    },
    logs: {
      driver: 'sqlite',
      database: 'logs.db',
    },
  },
}
```

Point a model at a named connection with `static connection`:

```js
class PageView extends Model {
  static connection = 'analytics'
}
```

See [ORM: Multiple Connections](./orm/getting-started.md#multiple-connections)
for details on cross-connection relationships, transactions, and read-only
connections.

### `config/session.js`

```js
export default {
  driver: process.env.SESSION_DRIVER || 'cookie',
  lifetime: 120,
  secure: process.env.NODE_ENV === 'production',
}
```

### `config/cors.js`

```js
export default {
  origin: process.env.APP_URL || 'http://localhost:3000',
  credentials: true,
}
```

### `config/security.js`

```js
export default {
  helmet: {
    contentSecurityPolicy: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:"],
      fontSrc: ["'self'"],
      connectSrc: ["'self'", "ws:", "wss:"],
    },
  },
  rateLimit: { max: 100, windowMs: 60000 },
}
```

See [Security](./security.md) for CSP customization and rate limiting details.

### `config/storage.js`

```js
export default {
  default: 'local',
  disks: {
    local: { driver: 'local', root: 'public/uploads' },
  },
}
```

### `config/mail.js`

```js
export default {
  driver: process.env.MAIL_DRIVER || 'log',
  from: process.env.MAIL_FROM || 'hello@example.com',
  smtp: {
    host: process.env.MAIL_HOST || 'localhost',
    port: parseInt(process.env.MAIL_PORT || '1025'),
  },
}
```

### `config/queue.js`

```js
export default {
  default: process.env.QUEUE_CONNECTION || 'sync',
  connections: {
    sync: { driver: 'sync' },
    database: { driver: 'database', table: 'jobs', queue: 'default' },
  },
}
```

### `config/cache.js`

```js
export default {
  default: process.env.CACHE_STORE || 'memory',
  stores: {
    memory: { driver: 'memory' },
    database: { driver: 'database', table: 'cache' },
  },
}
```

## Available Plugins

| Plugin | Package | Description |
|--------|---------|-------------|
| Auth | `foobarjs/auth` | Session-based authentication with login/register |
| Admin | `foobarjs/admin` | Auto-generated admin panel for all models |
| API | `foobarjs/api` | RESTful JSON API for all models |
| API Docs | `foobarjs/api-docs` | OpenAPI documentation and Scalar UI |

## Accessing Configuration

foobarjs provides `env()` and `config()` helpers that work **everywhere** — controllers, jobs, listeners, middleware, or any module:

```js
import { env, config } from 'foobarjs/core'

const port = env('PORT', 3000)             // reads process.env with auto-casting
const debug = env('APP_DEBUG', false)       // 'true'/'false' → boolean, numeric strings → number
const secret = env('APP_SECRET')            // raw string or undefined

const appName = config('app.name', 'Foobar')   // reads config/app.js → { name: ... }
const mailFrom = config('mail.from')            // reads config/mail.js → { from: ... }
const dbName = config('database.database')      // dot-notation path into any config file
```

`env()` is available immediately (`.env` is loaded before any CLI command runs). `config()` is available after boot — i.e. in controllers, jobs, listeners, and event handlers. Before boot (e.g. inside config files themselves), use `process.env` directly.

### In controllers

Controllers also have instance-method shortcuts:

```js
class ProductController extends Controller {
  async index() {
    const perPage = this.config('app.perPage', 25)
    const cdn = this.env('CDN_URL', '/assets')
    // ...
  }
}
```

### Raw context access

In route handlers or middleware, the `ConfigLoader` is available on the request context:

```js
app.get('/health', (c) => {
  const appName = c.get('configLoader').get('app.name')
  return c.json({ app: appName, status: 'ok' })
})
```
