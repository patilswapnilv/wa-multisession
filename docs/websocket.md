# WebSocket Reference

**URL:** same origin as the HTTP server — `ws://localhost:3000` (dev) / `wss://yourdomain.com` (prod).

The WebSocket server and the HTTP server share the same port. Session middleware runs during the HTTP upgrade, so a browser with a valid session cookie is authenticated immediately on connect.

---

## Authentication

Two methods — pick one per connection.

### Session (browser / dashboard)

The session cookie is sent automatically during the HTTP upgrade. No extra auth message is needed. You can optionally send `{ "type": "auth" }` to get an explicit `auth_success` acknowledgement.

```
connect  →  (cookie sent on upgrade)
send:  { "type": "auth" }
recv:  { "type": "auth_success" }
send:  { "type": "subscribe", "instanceId": "client1" }
recv:  { "type": "status", ... }   ← current state snapshot
recv:  { "type": "message", ... }  ← live events
```

### API key (scripts / server-to-server)

No cookie is required. Send the `auth` message first with the instance API key.

```
connect
send:  { "type": "auth", "apiKey": "64-char-hex" }
recv:  { "type": "auth_success", "instanceId": "client1" }
send:  { "type": "subscribe", "instanceId": "client1" }
recv:  { "type": "status", ... }
recv:  { "type": "message", ... }
```

An API key authenticates exactly one instance. The `subscribe` instanceId must match.

---

## Messages You Send

| `type` | Extra fields | Description |
|---|---|---|
| `auth` | `apiKey?` | Authenticate. Omit `apiKey` for session auth. |
| `subscribe` | `instanceId` | Subscribe to events for an instance. |

---

## Events You Receive

| `type` | Description |
|---|---|
| `auth_success` | Auth accepted. Includes `instanceId` when API-key auth was used. |
| `status` | Snapshot sent immediately after `subscribe`. Includes current status, QR (if `qr_ready`), and `linkedAccount`. |
| `qr` | New QR code generated. Scan with WhatsApp. |
| `authenticated` | WhatsApp QR accepted. Session being established. |
| `ready` | Instance connected and ready to send/receive. |
| `disconnected` | Instance disconnected. Includes `reason` string. |
| `message` | Incoming WhatsApp message. Full payload below. |
| `messages_cleared` | All messages for the instance were deleted via `DELETE /messages`. |
| `error` | Protocol or access error. Includes `message` string. |

---

## Event Payloads

### `status`

```json
{
  "type": "status",
  "instanceId": "client1",
  "status": "ready",
  "qr": null,
  "linkedAccount": { "jid": "919876543210@c.us", "number": "919876543210", "name": "My Name" },
  "linkedAccountChanged": false,
  "previousLinkedAccount": null
}
```

`qr` is a `data:image/png;base64,...` data URL when `status` is `qr_ready`, otherwise `null`.

---

### `qr`

```json
{
  "type": "qr",
  "instanceId": "client1",
  "qr": "data:image/png;base64,...",
  "linkedAccount": null,
  "linkedAccountChanged": false,
  "previousLinkedAccount": null
}
```

Render the `qr` data URL as an `<img>` or write it to a PNG file. QR codes expire; the client rotates them automatically — handle each `qr` event by re-rendering.

---

### `ready`

```json
{
  "type": "ready",
  "instanceId": "client1",
  "linkedAccount": { "jid": "...", "number": "919876543210", "name": "My Name" },
  "linkedAccountChanged": false,
  "previousLinkedAccount": null
}
```

`linkedAccountChanged: true` means the instance was re-authenticated with a different WhatsApp number. Older messages in the log belong to the previous account.

---

### `disconnected`

```json
{
  "type": "disconnected",
  "instanceId": "client1",
  "reason": "LOGOUT"
}
```

Common reasons: `LOGOUT` (signed out from phone), `CONFLICT` (another session opened), `UNPAIRED` (phone removed the linked device), `NAVIGATION` (browser-level navigation).

---

### `message`

```json
{
  "type": "message",
  "instanceId": "client1",
  "message": {
    "messageId": "true_919876543210@c.us_3EB0ABCDEF12",
    "from": "919876543210@c.us",
    "to": "910000000001@c.us",
    "body": "Hello!",
    "timestamp": 1718000000,
    "fromMe": false,
    "hasMedia": false,
    "type": "chat",
    "author": null,
    "isStatus": false,
    "isForwarded": false,
    "hasQuotedMsg": false,
    "senderDisplay": "John Doe",
    "senderNumber": "919876543210",
    "groupName": null,
    "source": "native",
    "messageKind": "individual"
  }
}
```

**Field reference:**

| Field | Type | Notes |
|---|---|---|
| `messageId` | string | Unique WhatsApp message ID for this instance. |
| `from` | string | Sender JID. `@c.us` = user, `@g.us` = group. |
| `to` | string\|null | Recipient JID (your linked number). |
| `body` | string | Text content. Empty string for media-only messages. |
| `timestamp` | number | Unix seconds (from WhatsApp). |
| `fromMe` | boolean | Always `false` — only incoming messages are broadcast. |
| `hasMedia` | boolean | `true` if the message includes an image, file, etc. |
| `type` | string\|null | WhatsApp message type: `chat`, `image`, `audio`, `document`, `video`, `sticker`, etc. |
| `author` | string\|null | Group participant JID. `null` for private messages. |
| `isStatus` | boolean | `true` for WhatsApp Status (story) updates. |
| `isForwarded` | boolean | Whether the message was forwarded. |
| `hasQuotedMsg` | boolean | `true` if the message replies to another. |
| `senderDisplay` | string | Push name, contact name, or number — best human-readable label. |
| `senderNumber` | string\|null | Digits-only sender number (no `@` suffix). `null` if unresolvable. |
| `groupName` | string\|null | Group chat name. `null` for individual messages. |
| `source` | `"native"` \| `"webhook"` | How the message reached this server. |
| `messageKind` | `"individual"` \| `"group"` \| `"status"` | Derived from `from` JID and `isStatus`. |

---

## Code Examples

### Node.js

```javascript
import WebSocket from 'ws';

const API_KEY = 'YOUR_64_CHAR_API_KEY';
const INSTANCE_ID = 'client1';

const ws = new WebSocket('ws://localhost:3000');

ws.on('open', () => {
  ws.send(JSON.stringify({ type: 'auth', apiKey: API_KEY }));
});

ws.on('message', (raw) => {
  const event = JSON.parse(raw);

  switch (event.type) {
    case 'auth_success':
      ws.send(JSON.stringify({ type: 'subscribe', instanceId: INSTANCE_ID }));
      break;

    case 'status':
      console.log('status:', event.status, '/ qr:', !!event.qr);
      break;

    case 'qr':
      console.log('Scan QR:', event.qr); // render as <img> or write to file
      break;

    case 'ready':
      console.log('Instance ready. Linked:', event.linkedAccount?.number);
      break;

    case 'message':
      console.log(`[${event.message.messageKind}] ${event.message.senderDisplay}: ${event.message.body}`);
      break;

    case 'disconnected':
      console.warn('Disconnected:', event.reason);
      break;

    case 'error':
      console.error('WS error:', event.message);
      break;
  }
});

ws.on('close', () => console.log('WS closed'));
ws.on('error', (err) => console.error('WS error:', err.message));
```

### Browser (session auth)

```javascript
const ws = new WebSocket('ws://localhost:3000'); // cookie sent on upgrade

ws.addEventListener('open', () => {
  // auth message is optional; skip straight to subscribe
  ws.send(JSON.stringify({ type: 'subscribe', instanceId: 'client1' }));
});

ws.addEventListener('message', ({ data }) => {
  const event = JSON.parse(data);
  if (event.type === 'message') {
    console.log(event.message.senderDisplay, ':', event.message.body);
  }
});
```

### Python

```python
import json
import websocket  # pip install websocket-client

API_KEY = 'YOUR_64_CHAR_API_KEY'
INSTANCE_ID = 'client1'

def on_open(ws):
    ws.send(json.dumps({'type': 'auth', 'apiKey': API_KEY}))

def on_message(ws, raw):
    event = json.loads(raw)
    if event['type'] == 'auth_success':
        ws.send(json.dumps({'type': 'subscribe', 'instanceId': INSTANCE_ID}))
    elif event['type'] == 'message':
        msg = event['message']
        print(f"{msg['senderDisplay']}: {msg['body']}")

ws = websocket.WebSocketApp(
    'ws://localhost:3000',
    on_open=on_open,
    on_message=on_message,
)
ws.run_forever()
```

---

## Reconnection Pattern

The server does not reconnect on your behalf. Implement exponential backoff:

```javascript
function connect(delay = 1000) {
  const ws = new WebSocket('ws://localhost:3000');

  ws.on('open', () => {
    delay = 1000; // reset on successful connect
    ws.send(JSON.stringify({ type: 'auth', apiKey: API_KEY }));
  });

  ws.on('close', () => {
    console.log(`Reconnecting in ${delay}ms…`);
    setTimeout(() => connect(Math.min(delay * 2, 30000)), delay);
  });

  ws.on('message', /* … */);
}

connect();
```

---

## Notes

- One WebSocket connection can subscribe to **one instance** when using API-key auth. Session auth connections can subscribe to **any accessible instance**.
- Dead connections (closed tabs, crashed scripts) are pruned lazily when the next broadcast runs — no explicit ping/pong.
- Message events are not replayed on reconnect. Call `GET /api/instances/:id/messages` to load history after reconnecting.
