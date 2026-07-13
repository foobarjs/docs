---
title: Helpers
layout: default
nav_order: 18
---

# Helpers

foobarjs ships a set of zero-dependency utility classes for common web development tasks. All helpers are available from `foobarjs/support`:

```js
import { Str, Arr, Num, Obj, dayjs } from 'foobarjs/support'
```

Or import them directly from `foobarjs`:

```js
import { Str, Arr, Num, Obj, dayjs } from 'foobarjs'
```

Every method is **static** and **null-safe** — passing `null` or `undefined` returns a sensible default (empty string, empty array, etc.) instead of throwing.

---

## Str (String Helpers)

### Case Conversion

```js
Str.camel('user_name')       // 'userName'
Str.pascal('user_name')      // 'UserName'
Str.snake('userName')        // 'user_name'
Str.kebab('userName')        // 'user-name'
Str.title('hello world')     // 'Hello World'
```

### Inflection

```js
Str.pluralize('post')        // 'posts'
Str.pluralize('person')      // 'people'
Str.singularize('posts')     // 'post'
Str.singularize('people')    // 'person'
Str.humanize('created_at')   // 'Created at'
Str.humanize('user_id')      // 'User'  (strips _id)
Str.labelize('userStatus')   // 'User Status'
```

### Slug & HTML

```js
Str.slug('Hello World!')           // 'hello-world'
Str.slug('Hello World!', '_')      // 'hello_world'
Str.escapeHtml('<script>')         // '&lt;script&gt;'
Str.unescapeHtml('&lt;b&gt;')     // '<b>'
```

### Truncation

```js
Str.limit('Hello World', 5)              // 'Hello...'
Str.limit('Hello World', 5, ' →')        // 'Hello →'
Str.words('The quick brown fox', 2)       // 'The quick...'
```

### Inspection

```js
Str.startsWith('foobar', 'foo')   // true
Str.endsWith('foobar', 'bar')     // true
Str.contains('foobar', 'oba')     // true
Str.isEmpty('')                    // true
Str.isEmpty(null)                  // true
Str.length('hello')                // 5
```

### Manipulation

```js
Str.before('hello@world', '@')          // 'hello'
Str.after('hello@world', '@')           // 'world'
Str.between('[hello]', '[', ']')        // 'hello'
Str.replace('foo bar', 'bar', 'baz')    // 'foo baz'
Str.replaceAll('aabaa', 'a', 'x')       // 'xxbxx'
Str.trim('  hello  ')                    // 'hello'
Str.squish('  hello   world  ')          // 'hello world'
Str.reverse('hello')                     // 'olleh'
Str.mask('1234567890', '*', 4)           // '1234******'
Str.padLeft('42', 5, '0')               // '00042'
Str.padRight('hi', 5)                    // 'hi   '
```

### Generation

```js
Str.random(16)     // 'a3f8b2c1d4e5f6a7' (cryptographically random hex)
Str.uuid()         // 'f47ac10b-58cc-4372-a567-0e02b2c3d479'
```

---

## Arr (Array Helpers)

```js
Arr.wrap(null)                  // []
Arr.wrap('hello')               // ['hello']
Arr.wrap([1, 2])                // [1, 2]

Arr.flatten([1, [2, [3]]])      // [1, 2, 3]
Arr.flatten([1, [2, [3]]], 1)   // [1, 2, [3]]

Arr.first([1, 2, 3])                   // 1
Arr.first([1, 2, 3], n => n > 1)       // 2
Arr.last([1, 2, 3])                    // 3

Arr.pluck(users, 'name')               // ['Alice', 'Bob']
Arr.keyBy(users, 'id')                 // { 1: {...}, 2: {...} }
Arr.groupBy(users, 'role')             // { admin: [...], user: [...] }
Arr.groupBy([1,2,3,4], n => n % 2 === 0 ? 'even' : 'odd')

Arr.unique([1, 2, 2, 3])               // [1, 2, 3]
Arr.unique(users, 'email')             // dedupe by email

Arr.chunk([1, 2, 3, 4, 5], 2)          // [[1,2], [3,4], [5]]
Arr.shuffle([1, 2, 3, 4, 5])           // [3, 1, 5, 2, 4] (Fisher-Yates)
Arr.sortBy(users, 'age')               // sorted ascending
Arr.sortBy(users, 'age', 'desc')       // sorted descending

Arr.random([1, 2, 3, 4, 5])            // 3 (single random element)
Arr.random([1, 2, 3, 4, 5], 2)         // [1, 4] (multiple)

Arr.only(items, ['name', 'email'])      // pick keys from each object
Arr.except(items, ['password'])         // omit keys from each object
Arr.compact([1, null, 2, undefined])    // [1, 2]

Arr.sum([1, 2, 3])                      // 6
Arr.sum(orders, 'total')                // sum by key
Arr.sum(orders, o => o.qty * o.price)   // sum with function

Arr.range(1, 5)                         // [1, 2, 3, 4, 5]
Arr.range(0, 10, 3)                     // [0, 3, 6, 9]
```

---

## Num (Number Helpers)

### Formatting

```js
Num.format(1234.5, 2)                   // '1,234.50'
Num.currency(1234.5)                    // '$1,234.50'
Num.currency(1234.5, 'EUR', 'en-US')   // '€1,234.50'
Num.percentage(85.5, 1)                 // '85.5%'
Num.abbreviate(1500000)                 // '1.5M'
```

### Human-Readable Units

```js
Num.bytes(1048576)          // '1.00 MB'
Num.bytes(1073741824)       // '1.00 GB'
Num.duration(1500)          // '1.5s'
Num.duration(90000)         // '1.5m'
Num.duration(2, { unit: 's' })  // '2.0s'
```

### Other

```js
Num.ordinal(1)              // '1st'
Num.ordinal(22)             // '22nd'
Num.clamp(15, 0, 10)        // 10
Num.isBetween(5, 1, 10)     // true
```

---

## Obj (Object Helpers)

### Dot-Path Access

```js
const config = { db: { host: 'localhost', port: 5432 } }

Obj.get(config, 'db.host')                 // 'localhost'
Obj.get(config, 'db.name', 'mydb')         // 'mydb' (default)
Obj.has(config, 'db.port')                 // true
Obj.set(config, 'db.name', 'mydb')         // mutates config
```

### Pick & Omit

```js
Obj.pick({ a: 1, b: 2, c: 3 }, ['a', 'c'])   // { a: 1, c: 3 }
Obj.omit({ a: 1, b: 2, c: 3 }, ['b'])         // { a: 1, c: 3 }
```

### Deep Merge

```js
Obj.deepMerge(
  { a: 1, nested: { x: 1, y: 2 } },
  { b: 2, nested: { y: 3, z: 4 } }
)
// { a: 1, b: 2, nested: { x: 1, y: 3, z: 4 } }
```

Arrays are **replaced**, not merged — matching the framework's config semantics.

### Flatten & Unflatten

```js
Obj.flatten({ a: { b: 1, c: { d: 2 } } })
// { 'a.b': 1, 'a.c.d': 2 }

Obj.unflatten({ 'a.b': 1, 'a.c.d': 2 })
// { a: { b: 1, c: { d: 2 } } }
```

### Other

```js
Obj.isEmpty({})          // true
Obj.isEmpty(null)        // true
Obj.wrap(null)           // {}
Obj.wrap('hello')        // { value: 'hello' }
```

---

## Dates (dayjs)

foobarjs includes [dayjs](https://day.js.org/) as a first-class date library, pre-configured with `relativeTime` and `utc` plugins.

### Import

```js
import { dayjs } from 'foobarjs/support'
// or
import { dayjs } from 'foobarjs'
```

### Model Date Fields

Date and datetime fields on models automatically return dayjs instances:

```js
const user = await User.find(1)

user.createdAt.format('YYYY-MM-DD')      // '2026-07-14'
user.createdAt.format('MMM D, YYYY')     // 'Jul 14, 2026'
user.createdAt.fromNow()                  // '2 hours ago'
user.createdAt.add(7, 'day')              // new dayjs (immutable)
user.createdAt.isBefore(dayjs())          // true/false
```

Timestamps (`createdAt`, `updatedAt`, `deletedAt`) are also dayjs instances.

### Serialization

`toJSON()` automatically converts dayjs instances to ISO strings for API responses:

```js
const json = user.toJSON()
json.createdAt  // '2026-07-14T10:30:00.000Z'
```

### Saving

When saving a model, dayjs instances are automatically converted back to native `Date` objects for the database:

```js
const post = new Post({ title: 'Hello', publishedAt: dayjs('2026-08-01') })
await post.save()  // publishedAt stored as Date in DB
```

### Query Builder

The query builder accepts dayjs instances in where clauses:

```js
const recentUsers = await User
  .where('createdAt', '>', dayjs().subtract(30, 'day'))
  .get()

const todaysPosts = await Post
  .whereDate('publishedAt', dayjs())
  .get()
```

### Admin Panel

Date fields in the admin panel display with a clean `MMM D, YYYY h:mm A` format (e.g., "Jul 14, 2026 10:30 AM"). Edit forms use standard HTML date/datetime-local inputs.

### Common dayjs Operations

```js
dayjs()                              // current time
dayjs('2026-07-14')                  // parse string
dayjs(someDate)                      // wrap native Date

d.format('YYYY-MM-DD')              // '2026-07-14'
d.format('h:mm A')                  // '10:30 AM'
d.format('MMM D, YYYY')             // 'Jul 14, 2026'

d.add(1, 'month')                   // add time
d.subtract(2, 'week')               // subtract time
d.startOf('day')                     // midnight
d.endOf('month')                     // end of month

d.fromNow()                         // '2 hours ago'
d.from(otherDate)                    // 'in 3 days'
d.toNow()                           // 'in 2 hours'

d.isBefore(other)                   // comparison
d.isAfter(other)
d.isSame(other, 'day')              // same day?

d.toDate()                          // native Date
d.toISOString()                     // ISO 8601 string
d.unix()                            // Unix timestamp (seconds)
d.valueOf()                         // Unix timestamp (milliseconds)

dayjs.utc('2026-07-14T12:00:00Z')   // UTC mode
```

See the [dayjs documentation](https://day.js.org/docs/en/display/format) for the full API.
