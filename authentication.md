# Authentication

foobarjs provides session-based authentication out of the box via the `foobarjs/auth` plugin.

## Setup

Add `foobarjs/auth` to the plugins array in `config/app.js` and configure `app.secret` from `APP_SECRET`:

```js
export default {
  secret: process.env.APP_SECRET,
  plugins: ['foobarjs/auth'],
}
```

Set `APP_SECRET` in your `.env` file. Generate a secure secret with:

```bash
foobar key:generate
```

The auth plugin will throw a clear error at boot if `APP_SECRET` is missing.

The plugin registers session middleware, authentication middleware, and the following routes:

| Method | URI | Description |
|--------|-----|-------------|
| GET | `/login` | Login form |
| POST | `/login` | Login handler |
| GET | `/register` | Registration form |
| POST | `/register` | Create account |
| POST | `/logout` | Log out |

## User Model

The plugin expects a `User` model in `app/models/user.model.js` that extends `AuthenticableModel` and defines at minimum `email` and `password` fields.

```js
import { AuthenticableModel } from 'foobarjs/auth'
import { Field } from 'foobarjs/orm'

export default class User extends AuthenticableModel {
  static schema = {
    name: Field.string().required(),
    email: Field.string().required().unique(),
    password: Field.string().required().hidden(),
    isAdmin: Field.boolean().default(false),
  }
}
```

`AuthenticableModel` automatically hashes the `password` field whenever you assign a plaintext value. Never pre-hash passwords yourself.

```js
// Correct — the model hashes the password for you
const user = await User.create({
  name: 'Jane',
  email: 'jane@example.com',
  password: 'super-secret',
})

// Verify later
if (user.verifyPassword('super-secret')) {
  // authenticated
}
```

You can customize the identifier and password field names:

```js
class User extends AuthenticableModel {
  static authIdentifierName = 'username'
  static authPasswordName = 'secret'

  static schema = {
    username: Field.string().required().unique(),
    secret: Field.string().required().hidden(),
  }
}
```

## Finding Users for Authentication

`AuthenticableModel` provides `findForAuth(identifier)`:

```js
const user = await User.findForAuth('jane@example.com')
```

## Password Hashing

Password hashing uses `scrypt` via the framework `Crypto` utility. You generally do not need to interact with it directly, but it is available for other framework features:

```js
import { Crypto } from 'foobarjs/core'

const hash = Crypto.hashPassword('user-password')
const match = Crypto.verifyPassword('user-password', hash)  // true
```

## Auth Middleware

Protect routes by applying middleware in your controller's constructor or a base controller:

```js
import { Controller } from 'foobarjs/core'
import { RequireAuthMiddleware, GuestMiddleware } from 'foobarjs/auth'

export default class DashboardController extends Controller {
  constructor() {
    super()
    RequireAuthMiddleware  // applied globally by the plugin
  }
}
```

Available middleware:

| Middleware | Description |
|------------|-------------|
| `RequireAuthMiddleware` | Redirects to `/login` if not authenticated |
| `GuestMiddleware` | Redirects to `/` if already authenticated |

## API Token Authentication

For API clients, foobarjs supports Bearer token authentication via the `PersonalAccessToken` model.

### Creating Tokens

```js
import { PersonalAccessToken } from 'foobarjs/auth'

const user = await User.find(1)
const { plainTextToken, accessToken } = await PersonalAccessToken.createFor(
  user,
  'mobile-app',
  ['read', 'write'],
  new Date(Date.now() + 30 * 24 * 60 * 60 * 1000) // optional expiration
)

// Return plainTextToken to the client once; it cannot be retrieved again
```

### Authenticating API Requests

Send the token in the `Authorization` header:

```
Authorization: Bearer 1|abc123...
```

The auth middleware automatically resolves the token to a user. The token is hashed in the database (only the plain-text value shown at creation is usable).

### Accessing the Current Token

```js
const user = c.get('user')           // User model instance or null
const token = c.get('accessToken')   // PersonalAccessToken instance or null
const loggedIn = c.get('loggedIn')   // boolean
```

Helper methods are also available on the context:

```js
const user = c.currentUser()
const token = c.currentAccessToken()
const isLoggedIn = c.loggedIn()
```

### Token Abilities

Check abilities on the resolved token:

```js
if (c.currentAccessToken()?.can('write')) {
  // perform write operation
}
```

## Accessing the Authenticated User

The authenticated user is available on the Hono context:

```js
const user = c.get('user')      // User model instance or null
const loggedIn = c.get('loggedIn')  // boolean
```

These are also auto-injected into all views as `user` and `loggedIn`.

## Session

Sessions use HMAC-signed cookies by default. The cookie name is `foobar_session`. Session data is stored client-side (signed) with a 7-day expiry. The signature key is `APP_SECRET`.

```js
const session = c.get('session')
session.set('key', 'value')
session.get('key')
session.delete('key')
```
