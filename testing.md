# Testing

foobarjs uses Node.js built-in test runner for testing.

## Test Files

Tests are placed in the `test/` directory:

```js
// test/example.test.js
import { test, describe, assert } from 'foobarjs/test'

describe('Math', () => {
  test('addition', () => {
    assert.strictEqual(1 + 1, 2)
  })
})
```

## Running Tests

```bash
foobar test
```

Tests run in parallel via Node.js built-in test runner. The `foobarjs/test` helper caches the booted application within a single process so multiple test suites share one boot.

## Test Helpers

### Model Factory

```js
import { factory } from 'foobarjs/test'

const user = await factory(User, { name: 'John' })
const users = await factory(User, 5)
```

### Request Helper

```js
import { createRequestHelper } from 'foobarjs/test'

const request = createRequestHelper(app)
const response = await request.get('/products')
assert.strictEqual(response.status, 200)
```

The request helper automatically preserves cookies across requests within the same test, so session-based features like flash messages and authentication work end-to-end.

For state-changing requests (`POST`, `PUT`, `PATCH`, `DELETE`), the helper automatically adds `Origin` and `Referer` headers so CSRF protection is satisfied:

```js
const res = await request
  .post('/cart')
  .set('Accept', 'text/html')
  .form({ product_id: '1', quantity: '1' })
```

### Mock Helper

```js
import { createMockHelper } from 'foobarjs/test'

const mock = createMockHelper()
mock.queue() // Intercept queued jobs
```

#### Mail Mock

When using the `array` mail driver, inspect sent emails via the mock helper:

```js
test('sends welcome email', async ({ mock }) => {
  const mail = new Mailer({ driver: 'array' })
  await mail.to('user@test.com').subject('Welcome').html('<h1>Hi</h1>').send()

  const emails = await mock.mail.sent()
  assert.strictEqual(emails.length, 1)
  assert.strictEqual(emails[0].subject, 'Welcome')

  // Filter by recipient
  const toUser = await mock.mail.sent({ to: 'user@test.com' })
  assert.strictEqual(toUser.length, 1)

  // Clear between tests
  await mock.mail.clear()
})
```

## Test Setup

The default project skeleton includes `test/example.test.js` as a starting point.

## Assertions

The `assert` object is re-exported from Node.js `node:assert`, providing all standard assertion methods.
