# foobarjs documentation

> **Experimental — not production ready.** foobarjs is under active development (v0.2.14). APIs, conventions, and database schemas may change between releases without a migration path. Use it for prototyping, learning, and side projects. Do not deploy to production yet.

Documentation for the [foobarjs](https://github.com/foobarjs/foobarjs)
framework — a batteries-included Node.js MVC framework.

## Philosophy

foobarjs is opinionated so you can focus on building.

- **Batteries included.** ORM, auth, admin panel, auto-generated REST API,
  queues, cache, mail, storage, realtime, notifications, validation, views
  — all in one npm package. No plugin archaeology.
- **Conventions over configuration.** File in the right folder, named the
  right way, gets picked up. Controllers, models, admin resources,
  listeners, jobs, and validators all auto-discover.
- **Zero build step.** Strict ESM, Node 20+, no transpiler, no bundler,
  no `dist/` directory. Source is what runs.
- **One package on npm.** Install `foobarjs`. Sub-paths (`foobarjs/orm`,
  `foobarjs/auth`, ...) are subpath exports of the same package.
- **Escape hatches everywhere.** `foobar.app` exposes the underlying HTTP
  router. `Model.query().getQueryBuilder()` gives you the raw query
  builder. `this.c` is the request context. When the framework's opinion
  doesn't fit, drop down one layer.

![Define a model — foobarjs auto-generates an admin panel and REST API endpoints](./assets/model-to-admin-api.svg)

## Getting started

- [Installation](./installation.md)
- [Directory structure](./directory-structure.md)
- [Conventions](./conventions.md)
- [Configuration](./configuration.md)
- [Lifecycle](./lifecycle.md)
- [CLI](./cli.md)

## HTTP

- [Routing](./routing.md)
- [Controllers](./controllers.md)
- [Views](./views.md)
- [Session](./session.md)
- [Error handling](./error-handling.md)
- [Logging](./logging.md)

## Data

- [ORM: getting started](./orm/getting-started.md)
- [ORM: relationships](./orm/relationships.md)
- [Database migrations](./database/migrations.md)
- [Database seeding](./database/seeding.md)
- [Validation](./validation.md)
- [Serialization](./serialization.md)

## Auth & Security

- [Authentication](./authentication.md)
- [Authorization](./authorization.md)
- [Middleware](./middleware.md)
- [Security](./security.md)

## Admin & API

- [Admin panel](./admin-panel.md)
- [API](./api.md)
- [Helpers](./helpers.md)

## Async

- [Queues](./queues.md)
- [Cache](./cache.md)
- [Redis](./redis.md)
- [Events](./events.md)
- [Realtime](./realtime.md)
- [Mail](./mail.md)
- [Notifications](./notifications.md)
- [Storage](./storage.md)

## Testing

- [Testing](./testing.md)

## Recipes & Troubleshooting

- [Recipes](./recipes.md)
- [Troubleshooting](./troubleshooting.md)

## Related

- Framework source: [foobarjs/foobarjs](https://github.com/foobarjs/foobarjs)
- Reference application: [foobarjs/demo](https://github.com/foobarjs/demo)

## License

Documentation is licensed under [CC BY 4.0](./LICENSE).
