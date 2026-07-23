[← Back to docs](./README.md)

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

## Redis driver (v0.6.0+)

Use Redis for multi-process workers, background daemons, or when jobs
must survive a restart. Batteries-included pick: **in-memory `sync`
driver is fine for dev and single-process apps; `database` works for
low-throughput jobs; use `redis` for real background workers.**

### Prerequisites

```bash
# macOS
brew services start redis
# One-off npm install (ioredis is an optional dependency)
npm install ioredis
```

### Config

Pick `redis` as the default driver:

```js
// config/queue.js
export default {
  default: 'redis',
  connections: {
    redis: {
      driver: 'redis',
      connection: 'default',   // key in config/redis.js
      visibilityTimeout: 60,   // seconds a claimed job may sit before rescue
    },
  },
}
```

### Semantics

- **Reliable pop.** Workers pop with `BRPOPLPUSH` from `queues:<name>`
  into `queues:<name>:processing`. A killed worker doesn't drop the
  job — it stays in the processing list.
- **Visibility timeout / stuck-job sweep.** On each pop, entries in
  `:processing` older than `visibilityTimeout` seconds are moved back
  to the main queue for re-processing. Set higher than your longest
  expected job runtime.
- **Delayed dispatch.** `Queue.later(delayMs, JobClass, args)` writes
  to `queues:<name>:delayed` (a ZSET). Any worker that polls promotes
  due entries into the main list — no separate scheduler process
  required.
- **Retries + failures.** On exception, `attempts` is incremented and
  the job is released back onto the queue with delay = `--sleep`. When
  `attempts >= tries` the job moves to `failed_queues:<name>` (a
  bounded LIST, last 1000). Inspect with `Queue.failed('default')`,
  retry with `Queue.retryFailed('default')`, wipe with
  `Queue.flushFailed('default')`.
- **Graceful shutdown.** The worker installs SIGTERM/SIGINT handlers
  that let the in-flight job finish, ack it, then exit.

### Worker

```bash
foobar queue:work --queue=default --tries=3
```

Run several worker processes for parallelism — Redis guarantees each
job is claimed by exactly one worker (BRPOPLPUSH is atomic).

### When to use which driver

| Driver     | Persists across restart | Multi-process workers | Extra infra |
|------------|-------------------------|-----------------------|-------------|
| `sync`     | n/a (runs inline)       | n/a                   | none        |
| `database` | yes                     | yes (rows locked)     | your DB     |
| `redis`    | yes                     | yes                   | Redis 5+    |


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
# Process jobs continuously (all discovered queues)
foobar queue:work

# Process one job and exit
foobar queue:work --once

# Specific queue(s) — comma-separated
foobar queue:work --queue=emails
foobar queue:work --queue=default,emails,exports

# Custom connection, retries, sleep
foobar queue:work --connection=database --tries=3 --sleep=3 --timeout=60
```

By default the worker processes **all** queues discovered from your job classes (each job's `static queue` property) plus the `default` queue. Pass `--queue` to restrict to specific queues.

The worker discovers job classes from `app/jobs/` automatically, plus any framework-internal jobs (e.g. admin exports).

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

---

## Task Scheduling

The scheduler lets you define recurring tasks in code instead of managing individual cron entries. Define your schedule in `config/schedule.js`, then run a single worker.

### Defining the Schedule

Create `config/schedule.js`:

```js
import CleanupJob from '../app/jobs/cleanup.job.js'
import ReportJob from '../app/jobs/report.job.js'

export default function (scheduler) {
  // Dispatch a job on a schedule
  scheduler.job(CleanupJob).daily().name('nightly-cleanup')

  // Run a job with arguments
  scheduler.job(ReportJob, { type: 'weekly' }).weeklyOn(1, '9:00').name('weekly-report')

  // Run an inline closure
  scheduler
    .call(async () => {
      console.log('Health check at', new Date().toISOString())
    })
    .everyFiveMinutes()
    .name('health-check')
}
```

### Running the Scheduler

**Long-running worker** (recommended for pm2, systemd, Docker):

```bash
foobar schedule:work
```

The worker stays alive and checks for due tasks every minute — no cron entry needed. Run it under a process manager:

```bash
# pm2
pm2 start "npx foobar schedule:work" --name scheduler

# systemd, Docker, etc.
foobar schedule:work
```

**One-shot** (for OS cron):

```bash
foobar schedule:run
```

Add a single cron entry to invoke it every minute:

```
* * * * * cd /path/to/app && npx foobar schedule:run >> /dev/null 2>&1
```

**List registered tasks:**

```bash
foobar schedule:list
```

### Schedule Frequency Options

| Method | Cron Equivalent |
|--------|-----------------|
| `everyMinute()` | `* * * * *` |
| `everyTwoMinutes()` | `*/2 * * * *` |
| `everyFiveMinutes()` | `*/5 * * * *` |
| `everyTenMinutes()` | `*/10 * * * *` |
| `everyFifteenMinutes()` | `*/15 * * * *` |
| `everyThirtyMinutes()` | `*/30 * * * *` |
| `hourly()` | `0 * * * *` |
| `hourlyAt(15)` | `15 * * * *` |
| `daily()` | `0 0 * * *` |
| `dailyAt('13:30')` | `30 13 * * *` |
| `weekly()` | `0 0 * * 0` |
| `weeklyOn(1, '8:00')` | `0 8 * * 1` |
| `monthly()` | `0 0 1 * *` |
| `monthlyOn(15, '9:00')` | `0 9 15 * *` |

### Day Constraints

Chain a day constraint after any frequency:

```js
scheduler.job(SyncJob).hourly().weekdays()    // Mon–Fri only
scheduler.call(fn).daily().mondays()          // Mondays only
```

Available: `weekdays()`, `sundays()`, `mondays()`, `tuesdays()`, `wednesdays()`, `thursdays()`, `fridays()`, `saturdays()`.

### Preventing Overlap

By default, a task will not run if its previous invocation is still running:

```js
scheduler.call(longRunningFn).everyMinute().withoutOverlapping()
```

To allow concurrent runs (e.g. if each invocation is independent):

```js
scheduler.call(fn).everyMinute().allowOverlapping()
```

### Task Types

| Method | Description |
|--------|-------------|
| `scheduler.job(JobClass, ...args)` | Dispatch a queued job via `JobClass.dispatch()` |
| `scheduler.call(fn)` | Run an inline async function directly |
| `scheduler.command(name, runner)` | Run a CLI command |

### Production Setup

Run both `queue:work` and `schedule:work` under your process manager:

```bash
# pm2 ecosystem.config.js
module.exports = {
  apps: [
    { name: 'web',       script: 'npx foobar serve' },
    { name: 'worker',    script: 'npx foobar queue:work' },
    { name: 'scheduler', script: 'npx foobar schedule:work' },
  ],
}
```

The scheduler dispatches jobs to the queue; the worker processes them. This keeps scheduled work non-blocking and retryable.

## See also

- [Conventions](./conventions.md#queues)
- [Admin panel](./admin-panel.md)
- [Redis](./redis.md)
