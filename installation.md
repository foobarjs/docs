# Installation

## Requirements

foobarjs requires Node.js 20 or higher.

## Creating a New Project

Install the CLI globally, then scaffold:

```bash
npm install -g github:foobarjs/foobarjs#v0.1.0
foobar new my-app
cd my-app
npm install
```

This generates a complete project skeleton with the default directory
structure, configuration files, a home controller, a welcome view, and a
`routes/web.js` starter.

## Configure APP_SECRET

The default skeleton includes `foobarjs/auth`, which requires an `APP_SECRET`
environment variable to sign sessions. Generate one and add it to your `.env`
file before starting the server:

```bash
foobar key:generate
```

Copy the printed value into `.env`:

```env
APP_SECRET=...
```

## Starting the Development Server

```bash
foobar serve
```

By default the server runs on port 3000. You can specify a custom port:

```bash
foobar serve --port 8080
```

For development with auto-restart on file changes:

```bash
foobar serve --watch
```

## Next steps

Your app is running at http://localhost:3000. Here's where to go next:

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
- **Explore the auto-admin panel.** Visit `/admin` (after creating a user
  with `isAdmin: true`). Configure per-model UIs in `app/admin/`. See
  [Admin panel](./admin-panel.md).
- **Explore the auto-API docs.** Visit `/api/docs` for a Scalar UI of the
  REST endpoints auto-generated for your models. See [API](./api.md).
- **Configuration.** Every subsystem is configured under `config/`. See
  [Configuration](./configuration.md).
- **Directory tour.** See [Directory structure](./directory-structure.md).

## Manual Setup

If you prefer to set up manually, create a project with the following
`package.json`:

```json
{
  "name": "my-app",
  "type": "module",
  "private": true,
  "dependencies": {
    "foobarjs": "github:foobarjs/foobarjs#v0.1.0"
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

Open http://localhost:3000. Admin credentials: `admin@foobar.com / aaaaaaaa`.
