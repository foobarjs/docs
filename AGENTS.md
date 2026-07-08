# AGENTS.md

Guidance for AI coding agents working on the **foobarjs docs** — the
Markdown documentation site for [foobarjs](https://github.com/foobarjs/foobarjs).

## Repository role

This repo is one of three that make up foobarjs:

- [`foobarjs/foobarjs`](https://github.com/foobarjs/foobarjs) — framework source
- [`foobarjs/demo`](https://github.com/foobarjs/demo) — reference app
- [`foobarjs/docs`](https://github.com/foobarjs/docs) — this repo

Pure Markdown. No build step. Published to
<https://foobarjs.github.io/docs> via GitHub Pages (Jekyll default theme,
serving from the `develop` branch root).

## Package management

No `package.json`, no `node_modules`. Nothing to install. Editing is
plain-text Markdown.

## Editing workflow

- Write valid Markdown. GitHub-flavored is fine; kramdown (Jekyll's default
  processor) accepts a superset.
- Relative links between pages use `./name.md`. Keep the `.md` extension —
  Jekyll's `jekyll-relative-links` plugin rewrites them to `.html` at
  build time. Sub-folder links: `./orm/getting-started.md`,
  `../routing.md`.
- Anchors: `[section](./page.md#heading)` — Jekyll auto-generates anchor
  ids from headings.
- No images currently. If you add one, put it in `assets/` and reference
  it with a relative link.
- No custom CSS or JavaScript. If you want a real docs site with search,
  dark mode, etc., migrate to VitePress or similar — but that's a
  separate decision.

## Docs conventions

- **One canonical example per topic.** If two pages show the same API,
  they must agree. Never leave the reader unsure which is "the right way."
- **Match the framework.** Every code sample must compile against the
  current framework source. When the framework changes, the docs change
  in the same release.
- **File naming.** kebab-case, `.md` extension.
  Sub-folders group related pages (`orm/`, `database/`).
- **Heading structure.**
  - H1 (`#`) — page title, one per file.
  - H2 (`##`) — top-level sections.
  - H3 (`###`) — subsections.
  - Don't skip levels.
- **Every page ends with a "Next steps" or "See also" section**
  that cross-links to 2-4 related pages. Prevents dead ends.
- **Code fences use language tags** (```js`, ```bash`, ```html`).
- **No emojis** unless the user asks for them.
- **No filler comments** in code samples ("// import the model" above
  `import Model from ...`).

## Import style in code samples

Userland imports use the bare `foobarjs` package name and its subpath
exports. Never use `@foobarjs/*` (a legacy scheme, removed in 0.1.0).

Good:

```js
import { Controller } from 'foobarjs/core'
import { Model, Field } from 'foobarjs/orm'
import { AuthenticableModel } from 'foobarjs/auth'
import { Foobar, Controller, Model } from 'foobarjs'
```

Bad:

```js
import { Model } from '@foobarjs/orm'    // don't
```

## Framework at a glance

foobarjs is a single npm package (`foobarjs`) providing a batteries-included
MVC framework:

- **Controllers** extend `Controller` from `foobarjs/core`.
- **Routes** come from either `routes/web.js` (explicit) or the filename
  convention (`app/controllers/foo.controller.js` → `/foo` REST resource).
  Only methods that exist get routes.
- **Models** extend `Model` (or `AuthenticableModel` for users) and declare
  `static schema = { field: Field.string()... }`.
- **Views** are Edge-like HTML templates rendered via `this.render()` or
  the convention-based fallback (return data → view lookup → JSON fallback).
- **Admin panel** auto-generates CRUD UI from registered models.
- **API** auto-generates REST endpoints and OpenAPI docs.

## Cross-repo coordination

When the framework ships a change that affects docs:

1. The framework PR includes matching doc updates in this repo.
2. Broken examples are fixed in the same release.
3. Add cross-links from any new page to at least one existing page.

When adding a new subsystem doc:

1. Add the page in the right group in `README.md`'s TOC.
2. Cross-link from adjacent pages (e.g. `cache.md` links to `redis.md`
   if cache uses redis as a driver).
3. Verify the code samples work by pasting them into a scratch project.

## Verification

Before pushing:

- Read the diff. Every code sample changed? Every link changed?
- Spot-check that each code sample matches current framework primitives
  (grep the framework source or the demo).
- Verify all relative links resolve on GitHub (click through in the PR
  preview) and will resolve on GitHub Pages (they will, because Jekyll
  rewrites `.md` → `.html`).

## GitHub Pages setup

The site is published from `develop` at `/` (repo root).

- Config: `_config.yml` sets the theme (`jekyll-theme-cayman`) and
  excludes `.github`, `LICENSE`, `AGENTS.md`.
- Enabling Pages is a one-time manual step in the repo Settings:
  Settings → Pages → Deploy from a branch → `develop` / `/` (root).
- Every push to `develop` triggers a rebuild.

## Never do

- Never commit or push automatically. Wait for explicit user instruction.
- Never leave a broken link.
- Never write two pages that contradict each other.
- Never use `@foobarjs/*` scoped imports in code samples.
- Never add tracking scripts, analytics, or third-party embeds without
  explicit user request.
- Never introduce build tooling (npm/webpack/vite/etc.) without a
  discussion. This is intentionally pure Markdown.
