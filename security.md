{% raw %}
# Security

foobarjs ships with security defaults that protect against the most common web vulnerabilities. This page covers what's enabled out of the box and what you can configure.

## Secure by default

These protections are always on and require no configuration:

| Protection | What it does |
|------------|--------------|
| **HMAC-signed sessions** | Session cookies are signed with `APP_SECRET` using SHA-256 HMAC. Timing-safe comparison prevents signature oracle attacks. |
| **XSS-safe views** | JSX and Blade templates auto-escape all interpolated values. Never use `raw()` on user-supplied content. |
| **Path traversal protection** | Storage operations reject paths that escape the disk root (`../../etc/passwd` throws). |
| **Filename sanitization** | Admin file uploads strip directory components and special characters from filenames. |
| **MIME magic byte validation** | File upload validation checks actual file bytes (JPEG, PNG, GIF, PDF, WebP, SVG, ZIP) — not just the client-reported Content-Type. |
| **CSRF protection** | State-changing requests require a valid CSRF token (see [Middleware](./middleware.md)). |
| **Template expression safety** | The Blade template engine blocks dangerous expressions (`process`, `require`, `eval`, `Function`, `globalThis`, `__proto__`) in template interpolation. |

## Content Security Policy (CSP)

foobarjs enables a Content Security Policy by default via `config/security.js`:

```js
export default {
  helmet: {
    contentSecurityPolicy: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:"],
      fontSrc: ["'self'"],
      connectSrc: ["'self'", "ws:", "wss:"],
    },
  },
}
```

### Adding a CDN or external origin

If your app loads assets from a CDN or external service, add the origin to the relevant directive:

```js
export default {
  helmet: {
    contentSecurityPolicy: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'unsafe-inline'", "https://cdn.example.com"],
      styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
      imgSrc: ["'self'", "data:", "https://images.example.com"],
      fontSrc: ["'self'", "https://fonts.gstatic.com"],
      connectSrc: ["'self'", "ws:", "wss:", "https://api.example.com"],
    },
  },
}
```

### Disabling CSP

For development or if your app is incompatible with CSP, set it to `false`:

```js
export default {
  helmet: {
    contentSecurityPolicy: false,
  },
}
```

## Rate limiting

### Global rate limiter

All routes are rate-limited by default — 100 requests per minute per IP. Only static admin assets (`/admin-assets/*`) are exempt.

Configure in `config/security.js`:

```js
export default {
  rateLimit: {
    max: 100,          // requests per window
    windowMs: 60000,   // window size in milliseconds
    skip: ['/admin-assets/*', '/health'],  // paths to exempt
  },
}
```

Rate limit headers are added to every response:

| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Maximum requests allowed |
| `X-RateLimit-Remaining` | Requests remaining in current window |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |

When the limit is exceeded, the server responds with `429 Too Many Requests`.

### Login rate limiter

Login endpoints (`POST /login` and `POST /api/auth/token`) have a stricter rate limit: **5 attempts per minute per IP**, separate from the global limiter. This is not configurable — it's a security baseline.

### Pluggable store

The default rate limiter uses an in-memory `Map`, which means limits are per-process and reset on restart. For production deployments with multiple processes, pass a custom store:

```js
export default {
  rateLimit: {
    max: 100,
    windowMs: 60000,
    store: redisStore,  // any Map-compatible object (get, set, delete, entries)
  },
}
```

The store must implement the `Map` interface (`get(key)`, `set(key, value)`, `delete(key)`, iterable with `for...of`).

## Session security

Sessions are HMAC-signed cookies with these flags:

| Flag | Default | Description |
|------|---------|-------------|
| `HttpOnly` | Always on | Not accessible from JavaScript |
| `SameSite` | `Lax` | Sent on same-site requests and top-level navigations |
| `Secure` | On in production | Cookie only sent over HTTPS |

The `Secure` flag is controlled by `config/session.js`:

```js
export default {
  driver: 'cookie',
  lifetime: 120,
  secure: process.env.NODE_ENV === 'production',
}
```

Set `secure: true` whenever your app is behind HTTPS. The default enables it automatically in production.

### Known limitation: client-side sessions

Session data is stored in the cookie itself (signed, not encrypted). This means:

- **Sessions cannot be revoked server-side.** Logging out deletes the `userId` from the cookie, but a captured cookie remains valid until it expires.
- **Session data is visible to the client** (though it cannot be tampered with thanks to the HMAC signature).

For apps that need server-side session revocation (e.g. "log out all devices"), use database-backed sessions with a custom session driver.

## WebSocket security

By default, WebSocket connections have these protections:

| Protection | Default | Description |
|------------|---------|-------------|
| Client broadcast | **Disabled** | Clients cannot relay messages through the server |
| Private channels | **Require auth** | Channels prefixed `private-` or `presence-` require authentication |
| Auth verification | **Server-side** | Client-supplied `userId` is verified via a callback, not trusted |

### Configuring WebSocket auth

In `config/realtime.js`, provide callbacks to verify tokens and authorize channel access:

```js
import { PersonalAccessToken } from 'foobarjs/auth'

export default {
  enabled: true,
  path: '/ws',

  // Verify the token sent on the 'auth' event — return a userId or null
  async authCallback(token) {
    const accessToken = await PersonalAccessToken.findToken(token)
    if (!accessToken) return null
    return accessToken.tokenable_id
  },

  // Authorize access to private/presence channels — return true to allow
  async authorizeChannel(userId, channel) {
    if (channel === `private-user-${userId}`) return true
    return false
  },

  // Set to true only if your app needs client-to-client broadcast
  allowClientBroadcast: false,
}
```

Without an `authCallback`, all attempts to authenticate a WebSocket connection are rejected. Public channels (no `private-` or `presence-` prefix) remain open to all connected clients.

## File upload security

### MIME validation

When using the `.accepted()` validation rule on file fields, foobarjs performs two checks:

1. **Content-Type header** — the MIME type reported by the client
2. **Magic byte sniffing** — the first bytes of the file are compared against known signatures

Supported magic byte signatures: JPEG, PNG, GIF, PDF, WebP, SVG, ZIP. Files with other MIME types pass if the Content-Type matches the accepted list.

### Filename sanitization

Admin file uploads automatically sanitize filenames:

- Directory components are stripped (`../../evil.txt` → `evil.txt`)
- Special characters are replaced with underscores (only `a-z`, `0-9`, `.`, `-`, `_` are kept)

### Path traversal

All `Storage` operations (`put`, `get`, `delete`, `exists`, `stat`, `list`) validate that the resolved path stays within the disk's root directory. Attempting to access `../../etc/passwd` throws an error.

## Secure headers

foobarjs sets security headers via [Hono's `secureHeaders` middleware](https://hono.dev/docs/middleware/builtin/secure-headers), which includes:

- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: SAMEORIGIN`
- `X-XSS-Protection: 0` (modern browsers use CSP instead)
- `Strict-Transport-Security` (when applicable)
- Content Security Policy (see [above](#content-security-policy-csp))

## Production checklist

Before deploying to production:

- [ ] Set `APP_SECRET` to a strong random value (`foobar key:generate`)
- [ ] Set `NODE_ENV=production` (enables `Secure` cookie flag)
- [ ] Review your CSP policy — add CDN origins, remove `'unsafe-inline'` if possible
- [ ] Use HTTPS (required for `Secure` cookies and HSTS)
- [ ] Consider a Redis-backed rate limiter store for multi-process deployments
- [ ] Configure WebSocket `authCallback` if using private channels
- [ ] Review file upload size limits and accepted MIME types

## Next steps

- Session details: [Session](./session.md)
- Auth setup: [Authentication](./authentication.md)
- WebSocket setup: [Realtime](./realtime.md)
- File validation: [Validation](./validation.md)
- Storage: [Storage](./storage.md)
{% endraw %}
