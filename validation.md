[← Back to docs](./README.md)

{% raw %}
# Validation

foobarjs includes a validation system integrated with the ORM. Validation rules are defined in model schemas and auto-run on every `save()`. The framework also translates database constraint violations (unique, not-null, foreign key, check) into the same `ValidationError` so form UIs can surface them uniformly.

## Defining Rules in Model Schemas

Field types and chainable methods define validation rules:

```js
import { Model, Field } from 'foobarjs/orm'

export default class User extends Model {
  static schema = {
    name: Field.string().required().maxLength(255),
    email: Field.string().required().unique().email(),
    password: Field.string().required().minLength(8).confirmed(),
    website: Field.string().url().nullable(),
    isAdmin: Field.boolean().default(false),
  }
}
```

### Available Rules

| Method | Description |
|--------|-------------|
| `.required(msg?)` | Field must not be null, undefined, or empty string |
| `.nullable()` | Field can be null |
| `.maxLength(n)` | String must not exceed n characters |
| `.minLength(n)` | String must be at least n characters |
| `.enum(...vals)` | Value must be one of the specified values |
| `.email(msg?)` | Must be a valid email address |
| `.url(msg?)` | Must be a valid URL |
| `.uuid(msg?)` | Must be a valid UUID |
| `.regex(pattern, msg?)` | Value must match the regex |
| `.min(n)` | Numeric value must be >= n |
| `.max(n)` | Numeric value must be <= n |
| `.confirmed(field?)` | Value must equal `<name>_confirmation` (or the named field) |
| `.sameAs(field)` | Value must equal another named field |
| `.unique()` | Database-level uniqueness (enforced by translated DB errors) |

Numeric type checks (`must be a number`) run automatically for `Field.number()` / `Field.float()` / `Field.decimal()`. Boolean type checks run for `Field.boolean()`.

## Auto-Validation on Save

When `instance.save()` is called, the model auto-validates all fields:

```js
try {
  const user = await User.create({ name: 'John', email: 'invalid', password: 'short' })
} catch (err) {
  if (err.name === 'ValidationError') {
    console.log(err.errors)  // { email: [...], password: [...] }
    console.log(err.status)  // 422
    console.log(err.input)   // original input, when available
  }
}
```

If validation fails, a `ValidationError` is thrown with:
- `errors` — Object mapping field names to error message arrays. Non-field ("form-wide") errors are keyed under `_form`.
- `status` — Always `422`
- `message` — `"Validation failed"`
- `input` — (optional) the raw input that failed
- `cause` — (optional) the underlying error, e.g. the database constraint exception

## Database Constraint Translation

`.unique()` is declared on the schema but is not enforced by the app-level validator (that would require a DB query per save). Instead, the framework wraps `em.flush()` and translates driver-level exceptions into `ValidationError`:

| DB error | ValidationError message | Field key |
|---|---|---|
| `UNIQUE constraint failed: users.email` | `Email has already been taken` | `email` |
| `NOT NULL constraint failed: users.name` | `Name is required` | `name` |
| `FOREIGN KEY constraint failed` (SQLite) | generic | `_form` |
| `CHECK constraint failed: ...` | `<field> is invalid` (or generic) | field or `_form` |

The translator uses your model's `.label()` for nicer messages and falls back to a humanised column name.

Because the ORM flushes multiple entities at once, the translator only attributes an error if the message references the current model's table. Unrelated errors are re-thrown so they can be handled by their true owner.

## Form Request Validation

For controller-level validation, create a `FormRequest` subclass:

```js
// app/validators/store-product.validator.js
import { FormRequest } from 'foobarjs/validation'
import { Field } from 'foobarjs/orm'

export default class StoreProductValidator extends FormRequest {
  authorize() {
    return !!this.c.get('user')
  }

  rules() {
    return {
      name: Field.string().required().maxLength(255),
      slug: Field.string().required().regex(/^[a-z0-9-]+$/, 'Slug must be kebab-case'),
      price: Field.float().required().min(0),
      stock: Field.number().default(0).min(0),
      email: Field.string().email(),
    }
  }

  messages() {
    return {
      'slug.required': 'A slug is required to publish',
      'price.min': 'Price cannot be negative',
    }
  }
}
```

Use it in a controller:

```js
import StoreProductValidator from '../validators/store-product.validator.js'

class ProductsController extends Controller {
  async store() {
    let request
    try {
      request = await this.validate(StoreProductValidator)
    } catch (err) {
      if (err.name === 'ValidationError') {
        if (this.wantsJson()) {
          return this.json({ errors: err.errors, message: err.message }, 422)
        }
        return this.back().withErrors(err).withInput(err.input)
      }
      throw err
    }
    const product = await Product.create(request.validated())
    return this.redirect('/products').with('success', 'Product created!')
  }
}
```

The framework does **not** auto-redirect on validation errors — your controller
decides how to respond. Use `this.back().withErrors(err)` for HTML forms and
`this.json()` for APIs. See [Controllers — Redirect Builder](./controllers.md#redirect-builder)
for the full builder API.

## Accessing Errors in Views

Two globals are injected on every render:

- `errors` — the flashed errors object (`{ field: ['msg', ...] }`)
- `old(key, fallback = '')` — reads the previously-submitted value for `key`

```html
<input name="email" value="{{ old('email') }}" class="{{ errors.email ? 'is-invalid' : '' }}">

@error('email')
  <div class="form-error">{{ message }}</div>
@enderror
```

The `@error('field') ... @enderror` directive only renders its body when `errors[field]` has at least one message. Inside the block, a local `message` is defined and equals the first error message. You can still access `errors['field']` for the full array.

You can also iterate all errors:

```html
@if(Object.keys(errors).length > 0)
  <ul class="alert alert-danger">
    @foreach(Object.entries(errors) as [field, messages])
      <li>{{ field }}: {{ messages.join(', ') }}</li>
    @endforeach
  </ul>
@endif
```

## Manual Validation

Use the `Validator` class directly:

```js
import { Validator, validate } from 'foobarjs/validation'
import { Field } from 'foobarjs/orm'

const v = new Validator({
  email: Field.string().required().email(),
  password: Field.string().required().minLength(8),
})
const result = await v.validate({ email: '', password: 'short' })
// result.valid → false
// result.errors → { email: [...], password: [...] }
// result.data  → the input object (useful when valid)
```

`Validator` also supports:

- `.mergeModelRules(ModelClass)` — pull rules from a model schema.
- `.only(['field1', 'field2'])` — validate a subset of rules.
- `.except(['field1'])` — skip specific rules.
- `.messages({ 'field.rule': 'msg' })` — override default messages.
- `.addRules({...})` — merge in more rules.

## Admin Panel Integration

The admin panel's create/update actions:

1. Save the model. If it throws `ValidationError` (either from app-level rules or the constraint-translation layer), the form re-renders with:
   - A summary panel listing every error at the top of the form.
   - `is-invalid` CSS class + per-field message next to each affected input.
   - The user's submitted values preserved so they don't retype the form.

2. Any other error propagates to the central `ErrorHandler` (see [Error Handling](./error-handling.md)).

This means a `Field.string().required().unique()` on a Product's slug will always surface a clean "Slug has already been taken" message, whether the failure came from the app-level `required` check or from the DB's unique constraint.

## API Plugin Integration

The API plugin returns validation failures as JSON:

```json
{
  "error": "Validation failed",
  "errors": { "email": ["Email has already been taken"] },
  "requestId": "..."
}
```

with status `422`.
{% endraw %}
