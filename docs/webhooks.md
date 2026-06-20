# Webhooks

This server can receive incoming WhatsApp message events via HTTP POST. Any service that can POST JSON to a URL can act as a webhook source — WaHub is the primary integration, but the endpoint is generic.

---

## Endpoint

```
POST /webhook/wahub/:instanceId
```

**Auth:** Instance API key only. Send as `X-API-Key: <key>` or `Authorization: Bearer <key>`.

**Content-Type:** `application/json`

**Body size limit:** 1 MB (hardcoded, independent of `JSON_BODY_LIMIT`).

---

## What happens when a webhook fires

1. API key is verified against the `instanceId` in the URL.
2. Optional HMAC-SHA256 signature is verified (if `WEBHOOK_SECRET` is set).
3. The payload is normalized — multiple incoming shapes are supported (see below).
4. Each extracted message is inserted into `instance_messages` (deduped on `instance_id + message_id`).
5. Each message is broadcast to all WebSocket subscribers for the instance.
6. The server responds `200 { "success": true, "received": true }` immediately, before side effects.

---

## Supported Payload Shapes

The server is intentionally flexible. All of the following work:

### Event wrapper (recommended)

```json
{
  "event": "message",
  "data": {
    "from": "919876543210",
    "body": "Hello",
    "id": "msg-uuid-123"
  }
}
```

`event` can also be `type` or `eventType`. `data` can be an object or array.

### Multiple messages

```json
{
  "event": "messages",
  "data": [
    { "from": "919876543210", "body": "Hi",  "id": "id1" },
    { "from": "919876543210", "body": "Bye", "id": "id2" }
  ]
}
```

### Flat message wrapper

```json
{
  "message": { "from": "919876543210", "body": "Hello" }
}
```

### messages array

```json
{
  "messages": [
    { "from": "919876543210", "body": "Hello" }
  ]
}
```

### Flat payload (fallback)

```json
{
  "from": "919876543210",
  "body": "Hello"
}
```

---

## Message Object Fields

| Field | Required | Description |
|---|---|---|
| `from` | Yes | Sender phone number (digits; country code optional) or JID (`number@c.us`). Aliases: `sender`, `remoteJid` (JID suffix stripped). |
| `body` / `text` / `content` | Yes | Message text. Also accepts `message.conversation` (Baileys-style). |
| `id` / `messageId` / `key.id` | No | Unique message ID for deduplication. Auto-generated if absent. |

At minimum, `from` and one of `body`/`text`/`content` must be present. Anything else is logged and ignored.

---

## Signature Verification (Optional but Recommended)

Set `WEBHOOK_SECRET` in `.env` to enable HMAC verification. When set, every webhook POST must include:

```
X-Webhook-Signature: sha256=<hex>
```

or

```
X-Wahub-Signature: sha256=<hex>
```

**Computing the signature (sender side):**

```
hex = HMAC-SHA256(key=WEBHOOK_SECRET, message=<raw UTF-8 body bytes>)
header = "sha256=" + hex
```

Use the **exact bytes** of the body as transmitted — no re-serialization, no extra spaces.

**Node.js example (sender):**
```javascript
import crypto from 'crypto';

const signature = 'sha256=' + crypto
  .createHmac('sha256', process.env.WEBHOOK_SECRET)
  .update(rawBodyBuffer)
  .digest('hex');

// Include in request:
// X-Webhook-Signature: <signature>
```

**Python example (sender):**
```python
import hmac, hashlib

sig = 'sha256=' + hmac.new(
    WEBHOOK_SECRET.encode(),
    raw_body_bytes,
    hashlib.sha256
).hexdigest()
```

When `WEBHOOK_SECRET` is not set, any POST with a valid API key is accepted.

---

## WaHub-Specific Setup

1. In the WaHub dashboard, register a webhook URL per instance:
   ```
   https://YOUR_SERVER/webhook/wahub/<instanceId>
   ```
2. Add `X-API-Key: <instance-api-key>` to the webhook request headers.
3. Configure WaHub to send `message` (or `messages`) events.
4. Optionally set a shared secret and configure WaHub to send `X-Webhook-Signature`.

**Local development with ngrok:**
```bash
ngrok http 3000
# Use the generated https URL, e.g.:
# https://abc123.ngrok.io/webhook/wahub/client1
```

---

## Response

The server always responds immediately before processing:

```json
{ "success": true, "received": true }
```

Return codes:
- `200` — Accepted (even if no messages were extracted).
- `400` — Invalid JSON body.
- `401` — Missing/invalid API key or invalid signature.

Your sender should treat any `2xx` as delivered. Retry on `5xx` or network errors (exponential backoff recommended).

---

## Troubleshooting

**`[webhook] ignored non-message payload`**  
The body contains no message data (e.g. a webhook registration ping). The server responds 200 but nothing is stored. No action needed; ensure the event type is `message` or `messages`.

**`[webhook] no messages extracted from payload; keys: … full body: …`**  
The payload shape is not recognized. Check the logged `keys` and `full body` in the server console. The fallback (treating the whole body as one message) also failed — meaning `from` or `body` couldn't be found. Post the actual payload to add support for the new shape.

**`401` on every POST**  
- Verify `X-API-Key` matches the instance in the URL path.
- If `WEBHOOK_SECRET` is set, verify the signature header is present and correct.

**Messages stored but not appearing in WebSocket**  
- The instance has no active WebSocket subscribers. Connect a client and subscribe.
- Check server logs: `[webhook] instance=X 0 listeners`.

**Duplicate messages**  
- The `UNIQUE(instance_id, message_id)` constraint in `instance_messages` prevents exact duplicates if a stable `id` is sent. If no `id` is provided, each POST creates a new row. Include a stable `id` in each message object.
