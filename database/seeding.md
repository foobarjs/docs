# Database: Seeding

Seeders populate your database with initial data for development and testing.

## Creating a Seeder

Create a seed file at `database/seeders/DatabaseSeeder.js` or `seed.js` in your project root. It should export a default async function:

```js
// database/seeders/DatabaseSeeder.js
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
foobar db:seed
```

This discovers and runs the seed file. The ORM is booted before the seeder runs, so model classes are available.

### Via Fresh

```bash
foobar db:fresh
```

This drops all tables, recreates them, and then runs the seeders.

## Direct Execution

Seed files can also be run directly with Node.js for debugging:

```bash
node seed.js
```

When run directly, the file handles ORM bootstrapping itself (see `process.argv[1]` pattern in seed files).

## Best Practices

- Use `slug` or `email` uniqueness checks to make seeders idempotent
- Seed essential data (admin users, default categories) in your primary seeder
- Use `db:fresh` during development to reset data to a known state
