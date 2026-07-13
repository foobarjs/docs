# Cache

foobarjs provides a simple caching layer with multiple storage drivers.

## Setup

Add `foobarjs/cache` to the plugins array in `config/app.js`:

```js
export default {
  plugins: ['foobarjs/cache'],
}
```

Create `config/cache.js`:

```js
export default {
  default: process.env.CACHE_STORE || 'database',
  stores: {
    memory: { driver: 'memory' },
    file: { driver: 'file', path: 'storage/framework/cache' },
    database: { driver: 'database', table: 'cache' },
    redis: {
      driver: 'redis',
      connection: process.env.REDIS_CACHE_CONNECTION || 'cache',
    },
  },
}
```

The `database` driver stores cache entries in a `cache` table auto-created by the ORM. The `redis` driver requires `foobarjs/redis` and a running Redis server.

## Redis Connection

When using the Redis driver, also create `config/redis.js`:

```js
export default {
  default: process.env.REDIS_CONNECTION || 'default',
  connections: {
    cache: {
      host: process.env.REDIS_CACHE_HOST || process.env.REDIS_HOST || '127.0.0.1',
      port: parseInt(process.env.REDIS_CACHE_PORT || process.env.REDIS_PORT || '6379'),
      db: parseInt(process.env.REDIS_CACHE_DB || process.env.REDIS_DB || '1'),
      password: process.env.REDIS_CACHE_PASSWORD || process.env.REDIS_PASSWORD || undefined,
    },
  },
}
```

## Basic Usage

```js
import { Cache } from 'foobarjs/cache'

// Store a value (TTL in seconds)
await Cache.put('user:1', { name: 'Alice' }, 300)

// Retrieve a value
const user = await Cache.get('user:1')

// Retrieve with default
const user = await Cache.get('user:2', { name: 'Guest' })

// Remove a value
await Cache.forget('user:1')

// Clear all cached values
await Cache.flush()
```

## remember

`remember` stores the result of a callback if the key is missing:

```js
const products = await Cache.remember('products', 60, async () => {
  return Product.all()
})
```

The callback only runs when the key is missing or expired.

## forever

Store a value without expiration:

```js
await Cache.forever('app-version', '1.0.0')
```

## Switching Stores

```js
const value = await Cache.store('memory').get('key')
await Cache.store('file').put('key', 'value', 60)
await Cache.store('redis').put('key', 'value', 60)
```

## Drivers

| Driver | Description |
|--------|-------------|
| `memory` | In-process Map. Lost on restart. Good for tests. |
| `file` | JSON files in `storage/framework/cache`. |
| `database` | SQL table via the ORM. |
| `redis` | Redis strings with TTL. Fast and shared across processes. |
