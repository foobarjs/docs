# Database: Seeding

Seeders populate your database with initial data for development and testing.

## Creating a Seeder

Create `database/seeders/DatabaseSeeder.js`. It should export a default async function:

```js
// database/seeders/DatabaseSeeder.js
import User from '../../app/models/user.model.js'

export default async function seed() {
  const existing = await User.where('email', 'admin@example.com').first()
  if (!existing) {
    await User.create({
      name: 'Admin',
      email: 'admin@example.com',
      password: 'secret',
    })
  }
}
```

## Running Seeders

```bash
foobar db seed
```

This discovers and runs the seed file. The ORM is booted before the seeder runs, so model classes are available.

### Via Fresh

```bash
foobar db fresh
```

This drops all tables, recreates them, and then runs the seeders.

## Using Factory and Fake

The `factory` function and `Fake` utility make it easy to generate realistic seed data without manually writing every field:

```js
// database/seeders/DatabaseSeeder.js
import { Fake } from 'foobarjs/support'
import { factory } from 'foobarjs/test'
import User from '../../app/models/user.model.js'
import Product from '../../app/models/product.model.js'

export default async function seed() {
  // Create an admin user with explicit data
  await User.create({
    name: 'Admin',
    email: 'admin@example.com',
    password: 'secret',
  })

  // Create 20 users with realistic fake data
  await factory(User, {
    email: () => Fake.email(),
    password: 'password',
  }, 20)

  // Create 50 products with overrides
  await factory(Product, {
    name: () => Fake.words(2),
    slug: () => Fake.slug(),
    price: () => Fake.float(5, 200, 2),
    stock: () => Fake.number(0, 500),
    published: true,
  }, 50)
}
```

### Factory Signatures

```js
await factory(User, 5)                            // 5 records, fully auto-generated
await factory(User, { role: 'admin' })             // 1 record with override
await factory(User, { role: 'admin' }, 10)         // 10 records with override
await factory(User, { email: () => Fake.email() }, 10)  // unique email per record
```

When a value is a function, it's called once per record with the index — use this for fields that need to be unique across records. Static values apply to all records.

The factory auto-generates data based on field names: an `email` field gets a fake email, a `name` field gets a fake name, a `slug` gets a fake slug, etc. Only provide overrides for fields where you need specific values.

### Fake Methods

| Category | Methods |
|----------|---------|
| Identity | `name()`, `firstName()`, `lastName()`, `email()`, `phone()`, `company()` |
| Text | `word()`, `words(n)`, `sentence()`, `paragraph()` |
| Numbers | `number(min, max)`, `float(min, max, decimals)`, `boolean()`, `price(min, max)` |
| Date | `date(from?, to?)` |
| Collections | `pick(array)`, `pickMany(array, n)` |
| Identifiers | `uuid()`, `slug()` |
| Web/Location | `url()`, `image(w, h)`, `address()`, `city()`, `country()` |

> For comprehensive fake data with locales and advanced types, consider [`@faker-js/faker`](https://fakerjs.dev) (~960KB) or [`chance.js`](https://chancejs.com) (~20KB). Use them directly in your seeders or pass their output to the factory.

## Best Practices

- Use `slug` or `email` uniqueness checks to make seeders idempotent
- Seed essential data (admin users, default categories) with explicit values
- Use `factory` + `Fake` for bulk test data instead of hand-writing every record
- Use `foobar db fresh` during development to reset data to a known state
