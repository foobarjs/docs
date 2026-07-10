# Events

foobarjs includes a simple event system for decoupling application logic. Events and listeners are auto-discovered from `app/events/` and `app/listeners/`.

## Defining Events

Events are plain classes placed in `app/events/`:

```js
// app/events/user-registered.event.js
export default class UserRegistered {
  constructor(user) {
    this.user = user
  }
}
```

## Defining Listeners

Listeners are classes in `app/listeners/` with a static `events` array:

```js
// app/listeners/send-welcome-email.listener.js
import UserRegistered from '../events/user-registered.event.js'

export default class SendWelcomeEmail {
  static events = [UserRegistered]

  async handle(event) {
    // event.user contains the user data
    console.log(`Sending welcome email to ${event.user.email}`)
  }
}
```

You can also use the event class name as a string if you prefer:

```js
static events = ['UserRegistered']
```

Class references are recommended — they survive renaming and give you IDE support.

### Accessing Config & Environment

Listeners run outside the HTTP request cycle, but the `env()` and `config()` helpers work everywhere:

```js
import { env, config } from 'foobarjs/core'

export default class SendWelcomeEmail {
  static events = [UserRegistered]

  async handle(event) {
    const appName = config('app.name', 'My App')
    const apiKey = env('SENDGRID_KEY')
    // ...
  }
}
```

## Dispatching Events

```js
import { Event } from 'foobarjs'
import UserRegistered from '../events/user-registered.event.js'

await Event.dispatch(new UserRegistered(user))
```

`Event.dispatch()` is async and awaits async listeners.

## Auto-Discovery

The framework automatically imports every `.js` file in `app/events/` and `app/listeners/` at boot and registers listeners based on their `static events` array. No manual registration is required.

## Manual Registration

If you need to register a listener outside of auto-discovery:

```js
import { Event, on } from 'foobarjs/events'
import UserRegistered from '../events/user-registered.event.js'
import SendWelcomeEmail from '../listeners/send-welcome-email.listener.js'

Event.on(UserRegistered, SendWelcomeEmail)

// or the shorthand
on(UserRegistered, SendWelcomeEmail)
```

## Broadcasting

Events can be broadcast to in-memory subscribers:

```js
import { Event } from 'foobarjs/events'

const unsubscribe = Event.subscribe('orders', (event) => {
  console.log('Order event:', event)
})

Event.broadcast('orders', { type: 'new_order', orderId: 123 })

unsubscribe()
```

Events that define a `static broadcastOn()` method automatically broadcast to those channels when dispatched:

```js
export default class OrderPlaced {
  static broadcastOn() {
    return ['orders']
  }

  constructor(order) {
    this.order = order
  }
}
```
