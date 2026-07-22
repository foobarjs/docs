[← Back to docs](./README.md)

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

The plugin registers session middleware, authentication middleware, login rate limiting (5 attempts per minute per IP on login endpoints, configurable in `config/auth.js` — see [Security](./security.md#login-rate-limiter)), and the following routes:

| Method | URI | Description |
|--------|-----|-------------|
| GET | `/login` | Login form |
| POST | `/login` | Login handler |
| GET | `/register` | Registration form |
| POST | `/register` | Create account |
| POST | `/logout` | Log out |
| POST | `/api/auth/token` | Exchange credentials for a Bearer token |
| GET | `/api/auth/me` | Return the authenticated user's profile |

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

Protect routes by importing and applying middleware classes from `foobarjs/auth`:

| Middleware | Description |
|------------|-------------|
| `'auth'` (alias for `RequireAuthMiddleware`) | Redirects to `/login` if not authenticated (returns 401 JSON for API requests) |
| `GuestMiddleware` | Redirects to `/` if already authenticated |

### On a controller

```js
import { Controller } from 'foobarjs/core'

export default class DashboardController extends Controller {
  static middleware = ['auth']

  async index() {
    return this.render('dashboard/index', { user: this.getLoggedInUser() })
  }
}
```

Use `only` or `except` to apply auth selectively:

```js
class PostsController extends Controller {
  static middleware = {
    use: ['auth'],
    except: ['index', 'show'],
  }
}
```

### On routes

```js
// routes/web.js
import { GuestMiddleware } from 'foobarjs/auth'

export default function (router) {
  // Single route
  router.get('/dashboard', 'auth', DashboardController, 'index')

  // Group of routes
  router.group({ middleware: ['auth'] }, (router) => {
    router.get('/settings', SettingsController, 'index')
    router.post('/settings', SettingsController, 'update')
  })

  // Guest-only routes (redirect away if already logged in)
  router.group({ middleware: [GuestMiddleware] }, (router) => {
    router.get('/welcome', (c) => c.html('Welcome! Please sign up.'))
  })
}
```

See [Controllers](./controllers.md) and [Routing](./routing.md) for the full middleware API.

## API Token Authentication

For API clients, foobarjs supports Bearer token authentication via the
`PersonalAccessToken` model. Tokens are SHA-256 hashed in the database — only
the plaintext shown at creation is usable.

### API Login Endpoint

The auth plugin registers `POST /api/auth/token` which exchanges credentials
for a Bearer token:

```bash
curl -X POST http://localhost:3000/api/auth/token \
  -H "Content-Type: application/json" \
  -d '{"email": "jane@example.com", "password": "secret", "deviceName": "mobile-app"}'
```

Response:

```json
{
  "token": "42|a1b2c3d4e5f6...",
  "type": "Bearer",
  "name": "mobile-app",
  "expiresAt": "2026-08-10T00:00:00.000Z"
}
```

The `deviceName` field identifies the client (defaults to `"default"`). If a
token with the same device name already exists for that user, it is revoked
and replaced — so repeated logins from the same device don't pile up tokens.

### Token Auth Configuration

Configure token behaviour in `config/auth.js`:

```js
export default {
  tokenAuth: {
    models: {
      user: 'User',          // POST /api/auth/token with type=user (default)
      merchant: 'Merchant',  // POST /api/auth/token with type=merchant
    },
    defaultModel: 'user',
    maxTokensPerUser: 5,     // oldest tokens pruned when limit is reached
    expiry: '30d',           // default token lifetime (null = never expires)
  },
}
```

The `type` field in the request body selects which model to authenticate
against. When omitted, the `defaultModel` is used. All models in the `models`
map must extend `AuthenticableModel`.

Expiry accepts duration strings: `'30d'`, `'12h'`, `'30m'`, `'3600s'`.

### Creating Tokens Programmatically

Use the convenience methods on any `AuthenticableModel` instance:

```js
const user = await User.find(1)

// Create a token
const { plainTextToken, accessToken } = await user.createToken(
  'mobile-app',            // device name
  ['read', 'write'],       // abilities (default: ['*'])
  new Date('2026-12-31')   // expiration (default: null)
)
// plainTextToken = "42|a1b2c3d4..." — return this to the client ONCE

// List all tokens for this user
const tokens = await user.tokens()

// Revoke all tokens
await user.revokeTokens()

// Revoke only tokens with a specific device name
await user.revokeTokens('mobile-app')
```

You can also use the `PersonalAccessToken` model directly:

```js
import { PersonalAccessToken } from 'foobarjs/auth'

const { plainTextToken } = await PersonalAccessToken.createFor(user, 'cli', ['*'])
const token = await PersonalAccessToken.findToken(plainTextToken)

// Remove all expired tokens
await PersonalAccessToken.revokeExpired()
```

### Authenticating API Requests

Send the token in the `Authorization` header:

```
Authorization: Bearer 42|a1b2c3d4e5f6...
```

The auth middleware automatically resolves the token to a user. It checks
session auth first, then falls back to Bearer token.

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

### Multiple Authenticable Models

Any model that extends `AuthenticableModel` is auto-registered in
`AuthenticableRegistry` at boot. The session middleware and bearer resolver
both use this registry to load the correct model based on `session.userType`
(for cookie sessions) or `token.tokenableType` (for Bearer tokens). One
session cookie, one `this.user`, whichever authenticable type is logged in.

```js
import { AuthenticableModel } from 'foobarjs/auth'
import { Field } from 'foobarjs/orm'

export default class Merchant extends AuthenticableModel {
  static schema = {
    name: Field.string().required(),
    email: Field.string().required().unique().email(),
    password: Field.string().required().hidden(),
    apiKey: Field.string(),
  }
  // Optional. Defaults to 'email'. Used by findForAuth() for password login
  // AND as the session key so browser sessions survive Model.find() lookups.
  static authIdentifierName = 'email'
}
```

Then add `merchant: 'Merchant'` to `config/auth.js` `tokenAuth.models` and
clients can authenticate with `{"type": "merchant", "email": "...", "password": "..."}`.

Admin panel login always uses the `User` model — this is not configurable.

**Controllers know who's logged in:**

```js
async someAction() {
  // this.user is whichever AuthenticableModel is currently in session.
  if (this.user instanceof Merchant) {
    // merchant-specific flow
  } else if (this.user instanceof User) {
    // user flow
  }
}
```

### Session helpers on Controller

Any custom login flow (magic link, OAuth, SSO) should end at the same
session shape as the built-in `/login`:

```js
class MagicLinkController extends Controller {
  async verify() {
    const attendee = await Attendee.findForAuth(email)
    if (!attendee) return this.redirect('/login')
    this.login(attendee)            // writes session.userId + session.userType
    return this.redirect('/portal')
  }

  async logout() {
    this.logout()                    // clears both keys
    return this.redirect('/')
  }
}
```

`this.login(model)` writes `session.get('userId') = model.getAuthIdentifier()`
and `session.get('userType') = model.constructor.name`. On the next request,
the auth middleware looks up `AuthenticableRegistry.get(userType)` and calls
`Model.findForAuth(userId)`, producing the same `this.user` your `/login`
route would have produced.

### Gate closures with multi-model auth

Because `this.user` can be any authenticable, gate closures should defend
against the actor type when the check is model-specific:

```js
// event.gate.js — back-office actions only
export default Gate({
  model: Event,
  update(user, event) {
    if (!(user instanceof User)) return false     // reject Attendee/Merchant sessions
    return user.isAdmin || event.organizer === user.id
  },
})

// attendee.gate.js — polymorphic per actor
export default Gate({
  model: Attendee,
  view(user, ticket) {
    if (user instanceof User) return true                          // admin sees all
    if (user instanceof Attendee) return ticket?.email === user.email  // owner
    return false
  },
})
```

### Current User Endpoint

`GET /api/auth/me` returns the authenticated user's profile (excluding hidden
and password fields). If authenticated via a Bearer token, the response
includes token metadata:

```bash
curl http://localhost:3000/api/auth/me \
  -H "Authorization: Bearer 42|a1b2c3d4..."
```

```json
{
  "id": 1,
  "name": "Jane",
  "email": "jane@example.com",
  "isAdmin": true,
  "_token": {
    "name": "mobile-app",
    "abilities": ["*"]
  }
}
```

Returns `401` if unauthenticated. Works with both session and token auth.

### Admin Token Management

The admin panel shows **API Tokens** under the System group. Admins can:

- **View** all tokens with owner, device name, last used, and expiration
- **Revoke** individual tokens via an inline action
- **Bulk revoke** selected tokens
- **Delete** tokens

Tokens are created via the API login endpoint or programmatically — the admin
panel is for viewing and revoking, not creating tokens (since the plaintext
must be returned to the client at creation time).

## Accessing the Authenticated User

The authenticated user is available on the request context:

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
