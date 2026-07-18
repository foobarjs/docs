{% raw %}
# Session

foobarjs provides cookie-based sessions out of the box via the `foobarjs/auth` plugin.

## Session Middleware

The auth plugin registers session middleware globally. Sessions are stored in HMAC-signed cookies:

- Cookie name: `foobar_session`
- Data: JSON-encoded, base64-encoded, HMAC-SHA256 signed
- Expiry: 7 days
- Flags: `HttpOnly`, `SameSite=Lax`

## Using Sessions

Access the session from the request context:

```js
const session = c.get('session')

// Set a value
session.set('key', 'value')

// Get a value
const value = session.get('key')

// Delete a value
session.delete('key')

// Cart example
const cart = session.get('cart') || []
cart.push({ id: 1, quantity: 1 })
session.set('cart', cart)
```

## Session in Views

The session key `cart` is used by the cart system. The total cart count is auto-injected into all views as `cartCount`.

## Flash Messages

Flash messages are stored in the session for one request only — perfect for post-redirect-get feedback.

### Setting Flash Messages

```js
async store(c) {
  const session = c.get('session')
  // ... create resource ...
  session.flash('success', 'User created successfully')
  return c.redirect('/users')
}
```

### Reading Flash Messages

```js
const session = c.get('session')
const success = session.getFlash('success')
```

You can also check for flash messages:

```js
if (session.hasFlash('error')) {
  // ...
}
```

### Keeping Flash Messages

Flash messages are consumed after one request. If you need to carry them over an extra redirect, keep them:

```js
// Keep everything for one more request
session.reflash()

// Keep specific keys
session.keep('success', 'error')
```

### Flash Messages in Views

All views automatically receive a `flash` object containing any flashed messages:

```html
@if(flash.success)
  <div class="alert alert-success">{{ flash.success }}</div>
@endif

@if(flash.error)
  <div class="alert alert-danger">{{ flash.error }}</div>
@endif
```

Flashed messages are removed from the session after they are read once, so a reload of the target page will not show them again.

### Admin Alert Rendering

The admin plugin auto-renders alerts from these flash keys only: `success`, `danger`, `warning`, `info`, `error` (aliased to `danger`), `notice` (aliased to `info`). Other flash keys (`old`, `errors`, `_intended`, custom bags) are ignored by the alert component so form-repopulation payloads never leak into the UI.

## Configuration

Session settings are in `config/session.js`:

```js
export default {
  driver: process.env.SESSION_DRIVER || 'cookie',
  lifetime: 60 * 24 * 7,  // 7 days in minutes
  secure: false,           // true in production with HTTPS
}
```

## Secret Key

The session HMAC secret comes from `app.secret` in config, which should be loaded from the `APP_SECRET` environment variable. Generate a secure secret with:

```bash
foobar key:generate
```

Set the printed value in your `.env` file:

```env
APP_SECRET=...
```

The auth plugin requires `APP_SECRET` and will throw a clear error at boot if it is missing.
{% endraw %}
