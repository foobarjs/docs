[← Back to docs](./README.md)

# Troubleshooting

Common errors and how to fix them.

---

## `APP_SECRET is not set`

The framework requires an application secret for session encryption, CSRF tokens, and signed URLs.

```bash
foobar key:generate
```

This writes a random `APP_SECRET` to your `.env` file.

---

## `401 Unauthenticated` on API routes

The API plugin is **public by default** (`auth: false`). If you added `.middleware('auth')` to an `Api.resource()`, make sure:

1. The `foobarjs/auth` plugin is registered **before** `foobarjs/api` in `config/app.js`:

```js
plugins: [
  'foobarjs/auth',   // must come first
  'foobarjs/admin',
  'foobarjs/api',
]
```

2. You are sending a valid session cookie or bearer token with the request.

---

## `403 Forbidden` on authenticated API requests

When an API model uses session or token auth, every authenticated request is checked against the model's gate. If no gate file exists for that model, the request passes through. If a gate exists but the user isn't authorized, you get a 403.

**Fix:** Create a gate file in `app/gates/`:

```bash
foobar generate gate <model-name>
```

Then implement the authorization logic in the generated gate.

---

## `CSRF token mismatch` (419)

Forms that submit via POST/PUT/PATCH/DELETE need a CSRF token.

**In JSX views:**

```jsx
<form method="POST" action="/checkout">
  {csrfField()}
  {/* form fields */}
</form>
```

---

## `Body stream already consumed`

This happens when you call `c.req.json()` or `c.req.text()` directly in a controller. The framework already parses the body for you.

**Wrong:**

```js
async store(c) {
  const data = await c.req.json()  // stream already consumed
}
```

**Right:**

```js
async store() {
  const data = this.body  // already parsed
  // or
  const name = this.input('name')  // single field
}
```

---

## `Cannot find module`

1. Check that the file has a `.js` extension (foobarjs uses strict ESM)
2. Verify `"type": "module"` is set in your `package.json`
3. Use full file paths in imports: `import User from './user.model.js'` (not `./user.model`)

---

## Migration errors

**Generate a migration:**

```bash
foobar db:make
```

This compares your model schemas to the current database and generates migration files.

**Run pending migrations:**

```bash
foobar db:migrate
```

**Common issues:**
- `SQLITE_ERROR: table already exists` — The migration was partially applied. Delete the database file and re-run `foobar db:migrate` followed by `foobar db:seed`.
- `column not found` — Your model schema changed but migrations haven't been generated yet. Run `foobar db:make` first.

---

## `Rate limit exceeded` (429)

Rate limiting is configured in `config/security.js`:

```js
export default {
  rateLimit: {
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 100,                   // requests per window
  },
}
```

Increase `max` or `windowMs` for development. To disable rate limiting entirely:

```js
export default {
  rateLimit: false,
}
```

---

## JSX views not rendering

JSX views require the JSX loader to be registered before the app starts. Make sure your start script includes the `--import` flag:

```json
{
  "scripts": {
    "dev": "node --import foobarjs/jsx-loader app.js"
  }
}
```

If you're running tests, the test runner needs the same flag:

```json
{
  "scripts": {
    "test": "foobar test"
  }
}
```

The `foobar test` command automatically registers the JSX loader.

---

## `foobar` command not found

The CLI is available via npx or as a local script:

```bash
npx foobar <command>

# or add to package.json scripts:
{
  "scripts": {
    "foobar": "foobar"
  }
}
```

## `Starts at must be a valid date` (or `Trying to set X of type 'Date' to '...' of type 'string'`)

This came from MikroORM rejecting string values on native `Date` columns. The
framework now auto-coerces any parseable string on save for `Field.date()`
and `Field.datetime()` — pass ISO strings, HTML `datetime-local` values, or
space-separated timestamps directly:

```js
await Event.create({
  startsAt: '2026-07-14T09:00:00Z',        // ISO
  endsAt:   '2026-07-14 12:00:00',         // space-separated
  salesStart: request.body.salesStart,     // 'YYYY-MM-DDTHH:MM' from a form input
})
```

If you still see this error, upgrade the framework or convert the value
yourself with `new Date(v)` before assigning.

## `handler is not a function`

Your controller/route references a middleware name that isn't registered.
For built-in aliases like `'auth'`, this only happens if the auth plugin
failed to load — check `console` output at boot. For custom aliases
(filename-based, in `app/middlewares/`), verify the file exports a middleware
class or function and that the filename stem matches the string alias you
used.

## `NODE_MODULE_VERSION` mismatch when starting the app

The native `better-sqlite3` binary was compiled against a different Node
version than the one running your app. Rebuild against your active Node:

```bash
cd node_modules/.pnpm/better-sqlite3@*/node_modules/better-sqlite3
npm run build-release
```

If you switch between Node versions (Volta, nvm, Herd), keep the terminal
where you rebuild on the same version as the terminal where you run
`foobar serve`.

## Cannot GET a custom controller action (404)

Convention routing only mounts the seven REST verbs
(`index`/`new`/`store`/`show`/`edit`/`update`/`destroy`). Custom methods
must be registered explicitly in `routes/web.js` — see
[Routing → Custom controller actions](./routing.md#custom-controller-actions).
The 404 response will tell you the exact line to add.
