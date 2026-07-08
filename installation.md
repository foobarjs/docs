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

This generates a complete project skeleton with the default directory structure, configuration files, and a home controller and view ready to go.

## Configure APP_SECRET

The default skeleton includes `foobarjs/auth`, which requires an `APP_SECRET` environment variable to sign sessions. Generate one and add it to your `.env` file before starting the server:

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

## Manual Setup

If you prefer to set up manually, create a project with the following `package.json`:

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

Then install and configure as described in the [Configuration](configuration.md) guide.

## Try the reference app first

If you want to see foobarjs in action without scaffolding:

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
