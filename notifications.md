# Notifications

foobarjs provides a notification system for sending messages through multiple channels: database, mail, and broadcast.

## Setup

Add `foobarjs/notifications` to the plugins array in `config/app.js`:

```js
export default {
  plugins: ['foobarjs/notifications'],
}
```

Create `config/notifications.js`:

```js
export default {
  default: ['database', 'mail'],
}
```

The `database` channel stores notifications in a `notifications` table. The `mail` channel sends email. The `broadcast` channel publishes events over Redis (requires `foobarjs/redis` and `foobarjs/broadcast`).

## Creating Notifications

```bash
foobar generate notification OrderShipped
```

This creates `app/notifications/order-shipped.notification.js`:

```js
import { Notification } from 'foobarjs/notifications'

class OrderShipped extends Notification {
  via(_notifiable) {
    return ['database', 'mail', 'broadcast']
  }

  toDatabase(_notifiable) {
    return {
      title: 'Order Shipped',
      message: 'Your order has been shipped.',
    }
  }

  toMail(_notifiable) {
    return {
      subject: 'Your order has shipped',
      text: 'Your order has been shipped.',
    }
  }

  toBroadcast(_notifiable) {
    return this.toDatabase(_notifiable)
  }

  broadcastOn(_notifiable) {
    return 'notifications'
  }

  broadcastAs(_notifiable) {
    return 'OrderShipped'
  }
}

export default OrderShipped
```

## Sending Notifications

```js
import OrderShipped from '../app/notifications/order-shipped.notification.js'
import { Notification } from 'foobarjs/notifications'

const user = await User.find(1)
await Notification.send(user, new OrderShipped({ id: 123 }))
```

## Channels

### Database

Stores a JSON payload in the `notifications` table. Retrieve notifications for a user:

```js
const notifications = await NotificationModel.where('notifiable_id', user.id).get()
```

Mark a notification as read:

```js
notification.markAsRead()
await notification.save()
```

### Mail

Sends an email using the configured mail driver. Return an object with `subject` and `text`/`html` from `toMail()`.

### Broadcast

Publishes a JSON event to a Redis channel. Useful for real-time updates when combined with `foobarjs/realtime`.

## Customizing Broadcasts

```js
class OrderShipped extends Notification {
  broadcastOn(_notifiable) {
    return `users.${_notifiable.id}`
  }

  broadcastAs(_notifiable) {
    return 'order.shipped'
  }
}
```
