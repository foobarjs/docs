[← Back to docs](./README.md)

# Redis

foobarjs uses Redis for queue, cache, broadcast, and realtime features. The `foobarjs/redis` package provides connection management.

## Configuration

Create `config/redis.js`:

```js
export default {
  default: process.env.REDIS_CONNECTION || 'default',
  connections: {
    default: {
      host: process.env.REDIS_HOST || '127.0.0.1',
      port: parseInt(process.env.REDIS_PORT || '6379'),
      db: parseInt(process.env.REDIS_DB || '0'),
      password: process.env.REDIS_PASSWORD || undefined,
    },
    cache: {
      host: process.env.REDIS_CACHE_HOST || '127.0.0.1',
      port: parseInt(process.env.REDIS_CACHE_PORT || '6379'),
      db: parseInt(process.env.REDIS_CACHE_DB || '1'),
    },
  },
}
```

## Connection Management

```js
import { RedisManager } from 'foobarjs/redis'

const redis = RedisManager.connection()
await redis.set('key', 'value')
const value = await redis.get('key')
```

Use a named connection:

```js
const cacheRedis = RedisManager.connection('cache')
```

## Health Check

```js
const available = await RedisManager.isAvailable('default', 1000)
```

## Supported Features

- Single-node Redis
- Redis Cluster (`cluster` array in connection config)
- Redis Sentinel (`sentinels` array + `sentinelName`)
- TLS, password, custom DB
