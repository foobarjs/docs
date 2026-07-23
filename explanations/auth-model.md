[← Back to docs](../README.md)

---
title: Auth model
layout: default
nav_order: 40
---

# Auth model (Explanation)

This is an **Explanation** doc — you're here to build a mental model, not
to look up a command. For the reference docs, see
[authentication.md](../authentication.md) and
[authorization.md](../authorization.md).

The framework separates two questions that people often conflate. Getting
the vocabulary straight is the single most important thing to internalize
before you touch a route, a controller, or an ApiResource.

---

## Two concepts

**Auth** answers "**is there a user on this request?**" It runs on every
request, has no opinion about whether the user is allowed to be here, and
just resolves an identifier (a session cookie or a bearer token) into a
model instance on the context.

**Gates** answer "**can this user do this thing?**" They fire only when
there is a user, and only against a specific action + subject. Anonymous
requests do not go through gates — there's no `user` to ask about.

The two collaborate but never blur:

- **Auth without a gate** — the route is behind a login wall, and the
  identified user can do everything the route allows.
- **Gate without auth** — a public resource with a gate: anonymous
  requests bypass the gate entirely, authenticated ones go through it.
  Both correct, both intentional.
- **Auth and a gate together** — the common shape. First identify, then
  authorize.

## Three middleware layers

The auth plugin registers three named middlewares
([framework/src/auth/index.js:77-181](../../framework/src/auth/index.js:77)).
It helps to think of them as a stack.

**Layer 1 — `AuthMiddleware` (the identifier).** Always runs, on every
request that hits the session pipeline. Reads `session.userId` +
`session.userType`, falls back to a `Bearer` token in the `Authorization`
header, resolves either into a model instance via `AuthenticableRegistry`,
and puts it on the context as `c.get('user')`
([auth/index.js:82-132](../../framework/src/auth/index.js:82)). It never
rejects a request — a missing or invalid credential just leaves `user`
null.

**Layer 2 — `RequireAuthMiddleware` (the gate).** Registered as the named
middleware `'auth'`
([core/bootstrap/Foobar.js:457](../../framework/src/core/bootstrap/Foobar.js:457)).
This is what actually turns unauthenticated requests away. On an HTML
request it stashes the requested URL in `session._intended` and redirects
to `/login`; on a JSON request it returns `401`
([auth/index.js:153-168](../../framework/src/auth/index.js:153)).

**Layer 3 — `RequireTokenMiddleware` (bearer-only).** Registered as
`'tokenOnly'`
([Foobar.js:462-466](../../framework/src/core/bootstrap/Foobar.js:462)).
Rejects session-only users; requires *both* a resolved user and a
`accessToken` on the context. Rare — you reach for it when a route must
be callable from a machine, never from a browser session.

The important consequence: session cookies and bearer tokens are
interchangeable at Layer 1. Any route protected by `'auth'` accepts
either. Only `'tokenOnly'` cares which one you brought.

## What the default gives you

The framework's default is `auth.default: 'auth'`
([core/config/ConfigLoader.js:91-100](../../framework/src/core/config/ConfigLoader.js:91)),
and this flips two switches at boot:

- **Web routes** — the `'auth'` middleware is *prepended* to the global
  user-middleware chain
  ([Foobar.js:494-502](../../framework/src/core/bootstrap/Foobar.js:494)),
  so every conventionally-mounted controller route is behind login unless
  it opts out.
- **Auto-generated API routes** — the API plugin reads the same setting
  and prepends its own auth gate to every REST verb of every discovered
  model, unless the verb is marked public
  ([api/index.js:56, 246-272](../../framework/src/api/index.js:56)).

A fresh scaffold's `HomeController` opts out with
`static withoutMiddleware = ['auth']`
([cli/commands/new.js:469](../../framework/src/cli/commands/new.js:469))
so `/` renders for anonymous visitors. Everything else you add is
protected until you say otherwise.

**Why this default?** The previous shape — public-by-default combined
with model auto-discovery — meant that adding a `PasswordReset` model to
`app/models/` silently exposed a public REST surface for it. The v0.4.0
lock-down (see the deprecation warnings at
[api/index.js:42-49](../../framework/src/api/index.js:42)) chose the
safer failure mode: forgetting to lock a route down should be visible
because the browser gets a `401`, not silent because the model leaks.

## One vocabulary, three surfaces

The same four words work across web routes, controllers, and ApiResource:

| word | meaning |
| --- | --- |
| `.middleware(name, ...)` | attach middleware, in addition to whatever's already in the chain |
| `.withoutMiddleware(name, ...)` | strip a name from the inherited chain |
| `.public()` | sugar for `.withoutMiddleware('auth')` |
| `.can(action, Model)` | attach a gate check |

On a **controller**, `static withoutMiddleware = ['auth']` peels off the
default auth gate for every action
([core/router/Router.js:262-268](../../framework/src/core/router/Router.js:262)).

On an **ApiResource**, the same methods take an optional list of verbs so
you can be more surgical
([api/ApiResource.js:56-102](../../framework/src/api/ApiResource.js:56)):

```js
Api.resource(Post)
  .public('index', 'show')           // list + detail anonymous
  .middleware('throttle', 'store')   // extra rate limit on create
  .can('update', Post, 'update')     // gate check on PUT/PATCH
```

Call any of these with no verbs to apply to every REST verb (`'*'`).

## Gates and auth, composed

A subtle but important detail lives at
[api/index.js:100-103](../../framework/src/api/index.js:100) (and the
identical pattern in `show`, `store`, `update`, `destroy`):

```js
if (user && this._gateRegistry?.for(Model)) {
  const allowed = await this._gateRegistry.can(user, 'viewAny', Model)
  if (allowed === false) throw new ForbiddenError()
}
```

The auto-gate only fires **when there is a user**. On a `.public()` route
that also has a gate registered, anonymous requests slip past the gate
entirely and authenticated ones go through it. Both cases are correct —
the gate is asking "can *this specific user* do this?" and there is no
answer to give when there's no user.

If you want "must be authenticated *and* must pass a gate", don't rely on
`.public()` — leave `'auth'` in the chain and let the gate ask its
question against a guaranteed-non-null user.

## Session vs. bearer

Session cookies are signed with `app.secret` and carry `userId` and
`userType`
([auth/session.js:34-42](../../framework/src/auth/session.js:34)). The
cookie is HttpOnly, SameSite=Lax, base64-encoded `JSON.signature`, and
verified in constant time on the way in.

Bearer tokens live in `personal_access_tokens`, stored as a SHA-256 hash
of the plaintext
([auth/PersonalAccessToken.js:23-45](../../framework/src/auth/PersonalAccessToken.js:23)).
Plaintext is returned exactly once — from `POST /api/auth/token`
([auth/index.js:363-432](../../framework/src/auth/index.js:363)) — and
never persisted after that.

Both flow into the same `c.get('user')` slot. Downstream code doesn't
know or care which one identified the request unless it explicitly asks
`c.get('accessToken')`.

## Two-realm case (multi-authenticable)

Real apps often need more than one login vocabulary — an admin `User`
and a customer-facing `Attendee`, both authenticable, both sitting on
the same session pipe. The `AuthenticableRegistry`
([auth/AuthenticableRegistry.js](../../framework/src/auth/AuthenticableRegistry.js))
makes this work without duplicated middleware.

At boot, every model subclassing `AuthenticableModel` is auto-registered
by class name
([core/bootstrap/Foobar.js:287-296](../../framework/src/core/bootstrap/Foobar.js:287)).
When a session is written, `Controller.login(user)` records both the
identifier and the class name
([core/controller/Controller.js:186-202](../../framework/src/core/controller/Controller.js:186)):

```js
session.set('userId', identifier)
session.set('userType', model.constructor?.name || 'User')
```

On the next request, `AuthMiddleware` uses `userType` to look up the
right class from the registry
([auth/index.js:88-98](../../framework/src/auth/index.js:88)) — so
Admin sessions resolve to `User`, portal sessions to `Attendee`, without
either the middleware or your controllers knowing which is which.

Bearer tokens carry the same information: `tokenable_type` on the token
row selects the class at resolution time
([auth/index.js:105-113](../../framework/src/auth/index.js:105)).

## Session fixation and `_intended`

Two v0.3.3-era hardenings live in the login flow and are worth
understanding:

**Session rotation on login.** Before writing `userId` into the session,
`Controller.login()` and the plugin's `POST /login` handler both call
`session.regenerate()`
([Controller.js:198](../../framework/src/core/controller/Controller.js:198),
[auth/index.js:246-248](../../framework/src/auth/index.js:246)), which
wipes pre-auth session state and forces a fresh cookie signature. A
pre-login cookie planted by an attacker (via subdomain XSS, MITM, or a
shared kiosk) becomes worthless — the post-login cookie is a signature
they don't hold.

**Open-redirect defense on `_intended`.** When `RequireAuthMiddleware`
bounces an unauthenticated request, it stashes the URL in
`session._intended` and redirects to `/login`. After a successful login
the framework wants to send you back — but `_intended` was written by an
untrusted session, so an attacker who could plant it might store
`https://evil.com/harvest` and turn the login redirect into an open
redirect.

`_sameOriginPath`
([auth/index.js:21-33](../../framework/src/auth/index.js:21)) filters
that value: it accepts paths starting with a single `/`, rejects
protocol-relative (`//evil.com`), rejects absolute URLs unless their
origin is dropped, and rejects `\` browser-quirk variants. A rejected
`_intended` falls back to `/`.

## Common surprises

**"My public API endpoint doesn't call my gate."** By design. Gates only
fire when there's a user
([api/index.js:100-103](../../framework/src/api/index.js:100)). If you
want the check to apply to anonymous requests too, don't mark the verb
public — write the check into the handler, or use `.middleware(...)`
with a custom middleware that treats missing user as a rejection.

**"I upgraded and my old scaffold prints a deprecation warning."**
Controllers still using `static auth = false` continue to work in
v0.4.0, but the router warns once per controller
([Router.js:216-225](../../framework/src/core/router/Router.js:216)).
Rename the property to `static withoutMiddleware = ['auth']`. Same for
`config('auth.guard')` — `'required'` / `'optional'` are translated to
`'auth'` / `'public'` and a warning is logged
([Foobar.js:482-491](../../framework/src/core/bootstrap/Foobar.js:482)).
Both aliases are scheduled for removal in v0.5.0.

**"I set `config('api.auth')` and nothing happens."** As of v0.4.0
`config('api.auth')` and `config('api.models')` are no longer read
([api/index.js:42-49](../../framework/src/api/index.js:42)) — the plugin
warns you the keys are dead and points at `auth.default` +
per-resource `.public()` / `.middleware()`. Same removal timeline.

**"My session cookie doesn't stick between requests."** Two usual
causes. First, `APP_SECRET` changed between requests (or is missing —
the plugin throws at boot if unset,
[auth/index.js:187-193](../../framework/src/auth/index.js:187)); a
different secret produces a different HMAC and the cookie parser drops
the session silently
([session.js:24-31](../../framework/src/auth/session.js:24)). Second,
you configured `session.secure: true`
([auth/index.js:195](../../framework/src/auth/index.js:195)) but the app
is behind plain HTTP — the browser refuses to send it back.

**"After login I got redirected somewhere weird."** `_intended` did its
job — the URL you first tried to reach was preserved across the login
detour
([auth/index.js:245-251](../../framework/src/auth/index.js:245)). If the
value came from an untrusted source and pointed off-origin,
`_sameOriginPath` dropped it and you landed on `/` instead. Both
outcomes are working as intended.

## See also

- [Authentication reference](../authentication.md) — the how-to
- [Authorization reference](../authorization.md) — gates, policies, `authorize()`
- [API reference](../api.md) — every `ApiResource` verb and flag
- [Database workflow](../database/workflow.md) — the companion Explanation for the ORM
