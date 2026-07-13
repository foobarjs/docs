# Queues

foobarjs provides a queue system for deferring work out of the request cycle.

## Setup

Add `foobarjs/queue` to the plugins array in `config/app.js`:

```js
export default {
  plugins: ['foobarjs/queue'],
}
```

Create `config/queue.js`:

```js
export default {
  default: process.env.QUEUE_CONNECTION || 'database',
  connections: {
    sync: { driver: 'sync' },
    database: { driver: 'database', queue: 'default' },
    redis: {
      driver: 'redis',
      connection: process.env.REDIS_QUEUE_CONNECTION || 'default',
    },
  },
}
```

The `database` driver stores queued jobs in the `queue_jobs` table and
failed jobs in `failed_jobs`. Both tables are auto-registered by the queue
plugin when the driver is `database` — you don't declare them yourself. The
`redis` driver stores jobs in list keys named `queues:<queueName>` and
failed jobs in `failed_queues:<queueName>`. Redis requires `foobarjs/redis`
and a running server.

## Redis Connection

When using the Redis driver, also create `config/redis.js`:

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
  },
}
```

## Creating Jobs

```bash
foobar generate job SendWelcomeEmail
```

This creates `app/jobs/send-welcome-email.job.js`:

```js
import { Job } from 'foobarjs/queue'

class SendWelcomeEmail extends Job {
  static queue = 'default'

  async handle(user) {
    // Job logic here
  }
}

export default SendWelcomeEmail
```

## Dispatching Jobs

```js
import SendWelcomeEmail from '../app/jobs/send-welcome-email.job.js'

// Default connection (from config)
await SendWelcomeEmail.dispatch(user)

// Explicit connection
await Queue.push(SendWelcomeEmail, [user], 'database')
```

With the `sync` driver, the job runs immediately and `dispatch` returns the result. With the `database` or `redis` driver, the job is persisted and processed by a worker.

## Running the Worker

```bash
# Process jobs continuously
foobar queue:work

# Process one job and exit
foobar queue:work --once

# Custom queue/connection
foobar queue:work --queue=emails --connection=database

# Tune retries and sleep
foobar queue:work --tries=3 --sleep=3 --timeout=60
```

The worker discovers job classes from `app/jobs/` automatically.

## Retrying Failed Jobs

When a job exhausts its retries it is stored in the `failed_jobs` table (database driver) or a Redis list (`failed_queues:{queue}`). Retry all failed jobs with:

```bash
foobar queue:retry --queue=default --connection=database
```

You can also retry, delete, or flush failed jobs from the admin panel at
`/admin/failed_jobs` (the URL follows the table name).

## Job Options

```js
class SendWelcomeEmail extends Job {
  static queue = 'emails'     // queue name
  static connection = null    // override connection
  static tries = 3            // override max attempts
  static timeout = 60         // override timeout (seconds)

  async handle(user) {
    // ...
  }
}
```

## Accessing Config & Environment

Jobs run outside the HTTP request cycle, but the `env()` and `config()` helpers work everywhere:

```js
import { Job } from 'foobarjs/queue'
import { env, config } from 'foobarjs/core'

class SendWelcomeEmail extends Job {
  async handle(user) {
    const apiKey = env('SENDGRID_KEY')
    const fromAddress = config('mail.from', 'noreply@example.com')
    // ...
  }
}
```

## Programmatic Failed Job Access

```js
import { Queue, FailedJob } from 'foobarjs/queue'

// Database driver
const failed = await FailedJob.where('queue', 'default').get()
await failed[0].retry()

// Redis driver
const failed = await Queue.failed('default', 'redis')
const retriedCount = await Queue.retryFailed('default', 'redis')
await Queue.flushFailed('default', 'redis')
```

## Drivers

| Driver | Description |
|--------|-------------|
| `sync` | Runs jobs immediately in the request process. |
| `database` | Stores queued jobs in `queue_jobs` and failed jobs in `failed_jobs` via the ORM. |
| `redis` | Stores jobs in Redis lists (`queues:<queue>`) for high throughput. |

## See also

- [Conventions](./conventions.md#queues)
- [Admin panel](./admin-panel.md)
- [Redis](./redis.md)
