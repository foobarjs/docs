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

Throughout your application, use the `config.get()` method via the `ConfigLoader`:

```js
const url = configLoader.get('app.url')
const dbName = configLoader.get('database.database')
```
