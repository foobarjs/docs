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

### Fake Data

The built-in `Fake` utility generates realistic-looking data for tests and seeders — no external dependencies required.

```js
import { Fake } from 'foobarjs/support'

Fake.name()          // "Alice Johnson"
Fake.email()         // "bob.garcia42@example.com"
Fake.phone()         // "(415) 555-1234"
Fake.sentence()      // "Crisp amber flame gleam haven ivory."
Fake.paragraph()     // Multiple sentences...
Fake.number(1, 100)  // 42
Fake.float(0, 1, 2)  // 0.73
Fake.boolean()       // true
Fake.price(10, 50)   // "$34.99"
Fake.date()          // Date within the past year
Fake.uuid()          // "550e8400-e29b-41d4-a716-446655440000"
Fake.slug()          // "crisp-amber-flame"
Fake.url()           // "https://example.com/delta-haven"
Fake.pick(['a','b']) // "b"
Fake.pickMany([1,2,3,4,5], 3)  // [2, 5, 1]
Fake.city()          // "Portland"
Fake.country()       // "Canada"
Fake.company()       // "Globex"
Fake.address()       // "4521 Johnson Ave"
Fake.image(400, 300) // "https://placehold.co/400x300"
```

`Fake` is also re-exported from `foobarjs/test` for convenience.

> For comprehensive fake data generation with locales and advanced types, consider [`@faker-js/faker`](https://fakerjs.dev) (~960KB) or [`chance.js`](https://chancejs.com) (~20KB). Use them directly in your seeders or pass their output to the factory.

### Model Factory

```js
import { factory, Fake } from 'foobarjs/test'

const user = await factory(User, { name: 'John' })  // 1 record with override
const users = await factory(User, 5)                 // 5 records, auto-generated
const admins = await factory(User, { role: 'admin' }, 10)  // 10 admins
```

The factory auto-generates realistic data based on field names — an `email` field gets a fake email, a `name` field gets a fake name, a `slug` gets a fake slug, etc. Override any field by passing explicit values.

For unique values per record, pass a function — it's called once per record with the index:

```js
await factory(User, {
  email: (i) => Fake.email(),       // unique email each time
  role: 'admin',                     // same for all 10
}, 10)
```

### Request Helper

```js
import { createRequestHelper } from 'foobarjs/test'

const request = createRequestHelper(app)
const response = await request.get('/products')
response.assertOk()
```

The request helper automatically preserves cookies across requests within the same test, so session-based features like flash messages and authentication work end-to-end.

For state-changing requests (`POST`, `PUT`, `PATCH`, `DELETE`), the helper automatically adds `Origin` and `Referer` headers so CSRF protection is satisfied:

```js
const res = await request
  .post('/cart')
  .set('Accept', 'text/html')
  .form({ product_id: '1', quantity: '1' })
```

### Authentication in Tests

Use `actingAs()` to set a default user for all requests, or `as()` for a single request:

```js
test('staff can view orders', async ({ request }) => {
  const staff = await factory(User, { roles: JSON.stringify(['staff']) })

  // All requests from this helper will be authenticated as staff
  request.actingAs(staff)

  const res = await request.get('/api/orders')
  res.assertOk()

  // Switch to guest
  request.actingAsGuest()
  const res2 = await request.get('/api/orders')
  res2.assertUnauthorized()
})

test('per-request auth', async ({ request }) => {
  const admin = await factory(User, { isAdmin: true })
  const res = await request.get('/admin/users').as(admin)
  res.assertOk()
})
```

Use `withToken()` for token-based authentication:

```js
request.withToken('my-api-token')
const res = await request.get('/api/orders')
```

### Response Assertions

Every response from the request helper includes assertion methods:

```js
const res = await request.get('/api/orders')

// Status assertions
res.assertOk()           // 200
res.assertCreated()      // 201
res.assertNoContent()    // 204
res.assertNotFound()     // 404
res.assertForbidden()    // 403
res.assertUnauthorized() // 401
res.assertUnprocessable()// 422
res.assertStatus(418)    // any status code
res.assertRedirect('/login')

// JSON assertions
await res.assertJson({ name: 'John' })        // body contains subset
await res.assertJsonPath('data.0.name', 'John')  // dot-path check
await res.assertJsonCount(5, 'data')           // array length at path

// HTML assertions
await res.assertSee('Welcome')        // body contains text
await res.assertDontSee('Error')      // body does not contain text
await res.assertSeeText('Welcome')    // text content (HTML tags stripped)
```

All assertions are chainable and throw clear error messages on failure.

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

### Database Assertions

Assert database state directly:

```js
test('creates an order', async ({ request, assertDatabaseHas, assertDatabaseCount }) => {
  request.actingAs(user)
  await request.post('/api/orders').send({ product_id: 1, quantity: 2 })

  await assertDatabaseHas(Order, { product_id: 1, quantity: 2 })
  await assertDatabaseCount(Order, 1)
})

test('deletes an order', async ({ request, assertDatabaseMissing }) => {
  const order = await factory(Order)
  request.actingAs(admin)
  await request.delete(`/api/orders/${order.id}`)

  await assertDatabaseMissing(Order, { id: order.id })
})
```

### Gate Assertions

Test gate authorization directly without making HTTP requests:

```js
test('staff can view orders', async ({ assertCan, assertCannot }) => {
  const staff = await factory(User, { roles: JSON.stringify(['staff']) })
  const order = await factory(Order, { user_id: staff.id })

  await assertCan(staff, 'view', order)
  await assertCannot(staff, 'delete', order)
})
```

`assertCan` and `assertCannot` accept a user, action name, and either a Model class or a model instance (the constructor is used to find the gate).

## Test Database

By default, `foobar new` generates a `.env.test` file that isolates the test database:

```env
NODE_ENV=test
DB_DATABASE=test.db
LOG_FILE=
```

Since tests run with `NODE_ENV=test`, the framework automatically loads `.env.test` and overrides `.env` values. This means tests use `test.db` instead of your development `foobar.db`, keeping test data out of your dev environment.

If you don't have a `.env.test`, create one — otherwise tests will share the dev database and pollute it with test data.

## Test Setup

The default project skeleton includes `test/example.test.js` as a starting point.

## Assertions

The `assert` object is re-exported from Node.js `node:assert`, providing all standard assertion methods.
