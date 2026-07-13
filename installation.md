# Installation

## Requirements

foobarjs requires Node.js 20 or higher.

## Creating a New Project

Install the CLI globally, then scaffold:

```bash
npm install -g github:foobarjs/foobarjs#v0.2.0
foobar new my-app
cd my-app
foobar serve
```

`foobar new` does everything you need to get running:

1. Prompts for your first admin user (name, email, password).
2. Writes the project skeleton (controllers, views, config, routes, etc.).
3. Generates a secure `APP_SECRET` into `.env`.
4. Runs `npm install`.
5. Creates the database schema and inserts your admin user.

After it finishes, `foobar serve` starts the dev server. Log in at
<http://localhost:3000/admin> with the credentials you provided.

### Non-interactive scaffolding

Skip the prompts by passing everything via flags — useful for CI, dotfile
setups, or automation:

```bash
foobar new my-app \
  --admin-name "Alice" \
  --admin-email "alice@example.com" \
  --admin-password "supersecret"
```

Or accept the defaults (admin@example.com / password) with `--yes`:

```bash
foobar new my-app --yes
```

### Opt-out flags

- `--skip-install` skips `npm install`. Also skips admin creation (which
  requires the framework to be installed).
- `--skip-admin` skips admin creation and prompts.

## Starting the Development Server

```bash
foobar serve
```

Runs on port 3000 by default. Override with `--port`, or enable file-watch
auto-reload with `--watch`:

```bash
foobar serve --port 8080
foobar serve --watch
```

## Next steps

Your app is running at <http://localhost:3000>. Here's where to go next:

- **Customize the home page.** Edit `app/controllers/home.controller.js`
  and `app/views/home/index.html`.
- **Add a route.** Edit `routes/web.js` — this is where explicit route
  registration lives. See [Routing](./routing.md).
- **Add a controller.**
  `foobar generate controller Products` creates
  `app/controllers/products.controller.js` (a REST-style controller
  extending `Controller` from `foobarjs/core`). It auto-mounts at
  `/products` via the filename convention, or register it in
  `routes/web.js`. See [Controllers](./controllers.md).
- **Add a model.**
  `foobar generate model Product --fields name:string,price:number,stock:number`
  creates `app/models/product.model.js`. Run `foobar db fresh` to reset the
  schema. See [ORM: getting started](./orm/getting-started.md).
- **Explore the auto-admin panel.** Visit `/admin` and log in with the
  credentials you set during `foobar new`. Configure per-model UIs in
  `app/admin/`. See [Admin panel](./admin-panel.md).
- **Explore the auto-API docs.** Visit `/api/docs` for a Scalar UI of the
  REST endpoints auto-generated for your models. See [API](./api.md).
- **Configuration.** Every subsystem is configured under `config/`. See
  [Configuration](./configuration.md).
- **Directory tour.** See [Directory structure](./directory-structure.md).

## Regenerating APP_SECRET

If you ever need a fresh key (e.g., after rotating secrets):

```bash
foobar key:generate
```

Paste the printed value into `.env` as `APP_SECRET=...`.

## Manual Setup

If you prefer to set up manually, create a project with the following
`package.json`:

```json
{
  "name": "my-app",
  "type": "module",
  "private": true,
  "dependencies": {
    "foobarjs": "github:foobarjs/foobarjs#v0.2.0"
  }
}
```

Then install and configure as described in
[Configuration](./configuration.md).

## Try the reference app first

If you want to see foobarjs in action without scaffolding, clone the demo:

```bash
git clone git@github.com:foobarjs/demo.git
cd demo
npm install
cp .env.example .env
foobar key:generate   # paste value into .env
foobar db fresh && foobar db seed
foobar serve
```

Open <http://localhost:3000>. Admin credentials: `admin@foobar.com / aaaaaaaa`.

## See also

- [Directory structure](./directory-structure.md)
- [Configuration](./configuration.md)
- [Routing](./routing.md)
- [CLI](./cli.md)
