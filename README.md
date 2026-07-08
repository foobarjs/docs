# foobarjs documentation

Documentation for the [foobarjs](https://github.com/foobarjs/foobarjs)
framework — a batteries-included Node.js MVC framework inspired by Laravel.

## Philosophy

foobarjs is opinionated the way Laravel is opinionated.

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
- **Escape hatches everywhere.** `foobar.app` is raw Hono.
  `Model.query().getQueryBuilder()` is raw MikroORM. `this.c` is raw
  Hono context. When the framework's opinion doesn't fit, drop down one
  layer.
- **Laravel-inspired, not Laravel-clone.** Ergonomics translated to
  idiomatic ESM Node — not PHP-in-JavaScript.

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

## Auth

- [Authentication](./authentication.md)

## Admin & API

- [Admin panel](./admin-panel.md)
- [API](./api.md)

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

## Related

- Framework source: [foobarjs/foobarjs](https://github.com/foobarjs/foobarjs)
- Reference application: [foobarjs/demo](https://github.com/foobarjs/demo)

## License

Documentation is licensed under [CC BY 4.0](./LICENSE).
