# Configuration

foobarjs uses a layered configuration system: environment variables and config files in `config/`.

## Environment Files

`.env` files are loaded first. Environment-specific overrides can be placed in `.env.{NODE_ENV}` (e.g., `.env.development`, `.env.production`).

```env
APP_URL=http://localhost:3000
PORT=3000
NODE_ENV=development
DB_CONNECTION=sqlite
DB_DATABASE=foobar.db
SESSION_DRIVER=cookie
```

Variables set in `.env` are injected into `process.env` and will not override already-set environment variables.

## Config Files

Config files are JavaScript modules located in the `config/` directory. Each file exports an object that becomes available under its filename as a namespace.

### `config/app.js`

```js
export default {
  name: process.env.APP_NAME || 'Foobar',
  url: process.env.APP_URL || 'http://localhost:3000',
  port: parseInt(process.env.PORT || '3000'),
  env: process.env.NODE_ENV || 'development',
  plugins: ['foobarjs/auth', 'foobarjs/admin', 'foobarjs/api', 'foobarjs/api-docs'],
}
```

The `plugins` key defines which first-party plugins to load at boot.

### `config/database.js`

```js
export default {
  connection: process.env.DB_CONNECTION || 'sqlite',
  database: process.env.DB_DATABASE || 'foobar.db',
}
```

### `config/session.js`

```js
export default {
  driver: process.env.SESSION_DRIVER || 'cookie',
  lifetime: 60 * 24 * 7,
  secure: false,
}
```

### `config/cors.js`

```js
export default {
  origin: process.env.APP_URL || 'http://localhost:3000',
  credentials: true,
}
```

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
  default: 'sync',
  connections: {
    sync: { driver: 'sync' },
    database: { driver: 'database', table: 'jobs' },
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
