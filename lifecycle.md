# Request Lifecycle

## Boot Sequence

When `foobar serve` runs, the framework goes through this boot sequence:

1. **Load Configuration** — `.env` files and `config/*.js` are loaded via `ConfigLoader`
2. **Initialize Auto-Discovery** — Scans `app/` directories for components
3. **Apply Global Middleware** — Method override, CORS, CSRF, secure headers, compression
4. **Serve Static Files** — `public/` directory served at root
5. **Discover Models** — Scans `app/models/`, imports each file, collects model classes
6. **Discover Middleware** — Scans `app/middlewares/`, `app/middlewares/web/`, `app/middlewares/api/`; async functions and classes are auto-applied, sync functions (factories) are available for explicit use
7. **Discover Events & Listeners** — Scans `app/events/` and `app/listeners/`
8. **Create Router** — Initializes the router
9. **Initialize Views** — Sets up the template engine with per-request context (user, loggedIn, flash, errors, old, shared data from `c.share()`)
10. **Boot Plugins** — Loads plugins from `config/app.js#plugins`, calls `plugin.register(this)`
11. **Initialize Database** — Calls `Db.boot()` with model classes, creating/updating tables
12. **Mount Routes** — Discovers controllers, builds convention routes, mounts on the app
13. **Register 404 Handler** — Sets up the not-found error handler

## Request Lifecycle

Each request flows through:

1. **The server receives the request**
2. **Global middleware** — compression, CORS, CSRF, security headers, method override
3. **Static file check** — if a matching file exists in `public/`, it's served directly
4. **Plugin middleware** — auth session middleware, etc.
5. **User middleware** — auto-discovered from `app/middlewares/` (global, web, api), filtered by opt-outs
6. **View middleware** — injects `user`, `loggedIn`, flash, errors, and `c.share()` data into all rendered views
7. **Route matching** — The router matches the request to a controller method
8. **Controller execution** — your controller handles the request
9. **Response** — JSON, rendered HTML, or redirect returned to the client

## Model Lifecycle Hooks

Models provide lifecycle hooks — instance methods that fire at specific points during a model's lifecycle.

| Hook | Fires | Use Case |
|------|-------|----------|
| `beforeValidate` | Before validation on save | Set default values, normalize data |
| `afterValidate` | After validation passes | Check validation results |
| `beforeSave` | Before any save (create or update) | General pre-save logic |
| `afterSave` | After any save completes | Post-save side effects |
| `beforeCreate` | Before inserting a new record | Set read-only defaults |
| `afterCreate` | After a new record is inserted | Log creation, notify |
| `beforeUpdate` | Before updating an existing record | Track changes |
| `afterUpdate` | After an existing record is updated | Invalidate caches |
| `beforeDelete` | Before deleting (or soft-deleting) | Check delete permissions |
| `afterDelete` | After delete completes | Cleanup related data |
| `afterFetch` | After a model is loaded from the DB | Transform loaded data |

### Example

```js
import { Model, Field } from 'foobarjs/orm'

class Post extends Model {
  static schema = {
    title: Field.string().required(),
    slug: Field.string().required(),
    body: Field.text().required(),
    published: Field.boolean().default(false),
  }

  beforeValidate() {
    if (!this.slug) {
      this.slug = this.title
        .toLowerCase().replace(/\s+/g, '-')
        .replace(/[^a-z0-9-]/g, '')
    }
  }

  beforeCreate() {
    this.published = false      // force unpublished on creation
  }

  afterCreate() {
    console.log(`Post #${this.id} created: ${this.title}`)
  }

  beforeDelete() {
    if (this.published) {
      throw new Error('Cannot delete published posts')
    }
  }

  afterFetch() {
    this._loaded = true
    // Access model values via $attributes or property shorthand:
    // this.$attributes.title or this.title
  }
}
```

### Hook execution order

**Create:** `beforeValidate` → `afterValidate` → `beforeSave` → `beforeCreate` → `afterCreate` → `afterSave`

**Update:** `beforeValidate` → `afterValidate` → `beforeSave` → `beforeUpdate` → `afterUpdate` → `afterSave`

**Delete:** `beforeDelete` → `afterDelete`

**Fetch:** `afterFetch`

### Throwing from hooks

If a hook throws, the operation is aborted and the error propagates:

```js
class Post extends Model {
  beforeSave() {
    if (!this.title) throw new Error('Title is required')
  }
}

await Post.create({})  // throws: Title is required
```
