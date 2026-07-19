# Realtime

foobarjs provides WebSocket-based realtime features through the `foobarjs/realtime` package. It integrates with the foobarjs server and can use Redis pub/sub for broadcasting across multiple server processes.

## Setup

Add `foobarjs/realtime` to your project dependencies and create `config/realtime.js`:

```js
export default {
  enabled: process.env.REALTIME_ENABLED !== 'false',
  path: process.env.REALTIME_PATH || '/ws',
  redisBroadcast: process.env.REALTIME_REDIS_BROADCAST === 'true',
}
```

When `redisBroadcast` is `true`, realtime events are fanned out across server processes via Redis (requires `foobarjs/redis`).

## Server-Side Broadcasting

```js
import { Realtime } from 'foobarjs/realtime'

const realtime = Realtime.getInstance()

// Broadcast to everyone subscribed to a channel
realtime.channel('orders').emit('order.created', { id: 1 })

// Broadcast to a specific user
realtime.channel('orders').to(user.id).emit('order.updated', { id: 1 })

// Broadcast to all connected clients
realtime.broadcast('server.message', { text: 'Hello' })
```

## Client-Side Usage

Connect to the WebSocket endpoint and subscribe to channels:

```js
const ws = new WebSocket('ws://localhost:3000/ws')

ws.onopen = () => {
  ws.send(JSON.stringify({ event: 'subscribe', payload: { channel: 'orders' } }))
}

ws.onmessage = (event) => {
  const { event: name, payload } = JSON.parse(event.data)
  console.log(name, payload)
}
```

## Supported Client Messages

| Event | Payload | Description |
|-------|---------|-------------|
| `subscribe` | `{ channel: 'orders' }` | Subscribe to a channel |
| `unsubscribe` | `{ channel: 'orders' }` | Unsubscribe from a channel |
| `auth` | `{ token: '42\|abc...' }` | Authenticate the connection with a Bearer token |
| `broadcast` | `{ channel, event, data }` | Relay a broadcast through the server (disabled by default) |

## Channels

Clients subscribe to named channels. The server emits events only to clients subscribed to that channel (or the wildcard channel `*`). When Redis broadcast is enabled, events published on one server are delivered to subscribers on all servers.

### Private channels

Channels prefixed with `private-` or `presence-` require the client to authenticate first (via the `auth` event) and pass the `authorizeChannel` check. Public channels (no prefix) are open to all connected clients.

## Security

By default, client-side `broadcast` is disabled and private channels require server-side authentication. See [Security — WebSocket security](./security.md#websocket-security) for configuration details including `authCallback`, `authorizeChannel`, and `allowClientBroadcast`.
