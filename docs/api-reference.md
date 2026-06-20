# REST API Reference

**Base URL:** `http://localhost:3000` (or your server origin)  
**Content-Type:** `application/json` for all requests and responses.  
**Errors** return `{ "error": "message" }` unless noted otherwise.

---

## Authentication

### Session (browser / dashboard)

1. `POST /api/login` — server sets `connect.sid` cookie.
2. Include the cookie on all subsequent requests. Same-origin: automatic. Cross-origin: `credentials: 'include'` + CORS origin allowed.

### Instance API key (server-to-server)

Send either:
- `X-API-Key: <key>`
- `Authorization: Bearer <key>`

Keys are per-instance — a key for `instanceA` cannot access `instanceB`'s routes. Retrieve or regenerate keys via the dashboard or the API-key endpoints below.

---

## HTTP Status Summary

| Code | Meaning |
|---|---|
| 200 | Success |
| 201 | Created |
| 400 | Validation failure or business-rule violation |
| 401 | Not authenticated / invalid API key |
| 403 | Authenticated but forbidden (wrong role, no instance access) |
| 404 | Resource not found |
| 503 | Instance not ready |
| 500 | Unexpected server error |

---

## Auth & Session

### `POST /api/login`

No auth required.

**Request:**
```json
{ "username": "admin", "password": "admin123" }
```

**200:**
```json
{ "success": true, "user": { "id": 1, "username": "admin", "role": "admin" } }
```

**401:** Invalid credentials.

---

### `POST /api/logout`

No auth required (destroys the current session if one exists).

**200:** `{ "success": true }`

---

### `GET /api/check-auth`

No auth required — check whether the current session is valid.

**200:**
```json
{ "authenticated": true,  "user": { "id": 1, "username": "admin", "role": "admin" } }
{ "authenticated": false }
```

---

### `GET /api/me`

**Auth:** Session.

**200:**
```json
{ "id": 1, "username": "admin", "role": "admin" }
```

`role` is `"admin"` or `"user"`.

**401:** Unauthorized.

---

## Instances

### `GET /api/instances`

**Auth:** Session. Admins see all instances; users see only assigned instances.

**200:**
```json
[
  {
    "id": "client1",
    "status": "ready",
    "linkedAccount": { "jid": "...", "number": "919876543210", "name": "My Name" },
    "linkedAccountChanged": false,
    "previousLinkedAccount": null
  }
]
```

**`status` values:** `initializing` | `qr_ready` | `authenticated` | `ready` | `disconnected`

**`linkedAccount`:** `null` until the instance authenticates at least once. Fields: `jid`, `number`, `name`.

---

### `POST /api/instances`

**Auth:** Admin.

**Request:**
```json
{ "instanceId": "client1" }
```

- `instanceId`: alphanumeric + hyphens/underscores, must be unique.

**200:**
```json
{ "success": true, "instanceId": "client1" }
```

**400:** Instance ID required / already exists.  
**500:** Server error during initialization.

After creation: connect via WebSocket to receive the QR code, then scan with WhatsApp.

---

### `DELETE /api/instances/:instanceId`

**Auth:** Admin.

Destroys the WhatsApp client session, removes all DB records for the instance (API key, linked account, messages), and removes the `LocalAuth` directory.

**200:** `{ "success": true }`

**404:** Instance not found.  
**500:** Cleanup error.

---

### `GET /api/instances/:instanceId/api-key`

**Auth:** Session (with access) or instance API key.

Returns the instance's current API key. Creates one if none exists.

**200:** `{ "apiKey": "64-char-hex" }`

**401:** Not allowed.  
**404:** Instance not found.

---

### `POST /api/instances/:instanceId/api-key/regenerate`

**Auth:** Session (with access) or instance API key.

Generates a new key. The previous key stops working immediately.

**200:** `{ "apiKey": "64-char-hex" }`

---

### `GET /api/instances/:instanceId/users`

**Auth:** Admin.

**200:**
```json
[{ "id": 2, "username": "agent1", "role": "user" }]
```

**404:** Instance not found.

---

## Messaging

### `POST /api/instances/:instanceId/send-message`

**Auth:** Session (with access) or instance API key.

**Request:**
```json
{
  "to": "919876543210",
  "message": "Hello from API"
}
```

- **`to`** (required): Phone number with country code, digits only — e.g. `919876543210`. No `+` or spaces. OR pass a full JID: `919876543210@c.us` (user) / `120363xxxxxx@g.us` (group).
- **`message`** (required): Text content.

The server calls `client.getNumberId()` to verify the recipient exists on WhatsApp before sending. Group JIDs (`@g.us`) skip the number check.

**200:**
```json
{ "success": true, "messageId": "true_1234567890@c.us_3EB0XXXXX" }
```

**400:** `to` and `message` required.  
**404:** Instance not found / recipient not registered on WhatsApp.  
**503:** Instance not ready.  
**500:** Send failed / recipient lookup failed.

**curl example:**
```bash
curl -X POST http://localhost:3000/api/instances/client1/send-message \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_KEY" \
  -d '{"to":"919876543210","message":"Hello"}'
```

---

### `POST /api/instances/:instanceId/send-file`

**Auth:** Session (with access) or instance API key.

Send a file with an optional text caption. File provided as base64.

**Request:**
```json
{
  "to": "919876543210",
  "filename": "document.pdf",
  "fileBase64": "JVBERi0xLjQK...",
  "caption": "See attached",
  "mimetype": "application/pdf"
}
```

- **`to`** (required): Phone number or JID (same rules as send-message).
- **`filename`** (required): Original filename; used to infer MIME type if `mimetype` omitted.
- **`fileBase64`** (required): Base64-encoded file bytes. Data-URL prefix (`data:application/pdf;base64,`) is stripped automatically.
- **`caption`** (optional): Text caption sent alongside the file.
- **`mimetype`** (optional): Override MIME type. Inferred from extension if omitted.

**Supported extensions:** `pdf`, `doc`, `docx`, `xls`, `xlsx`, `png`, `jpg`, `jpeg`, `gif`, `webp`, `mp4`, `mp3`, `ogg`, `wav`, `txt`, `csv`, `json`. Unknown extensions use `application/octet-stream`.

**Body size limit:** Default 10 MB (configurable via `JSON_BODY_LIMIT`). Base64 is ~1.33× the raw file size; keep source files under ~7.5 MB at the default limit.

**200:**
```json
{ "success": true, "messageId": "true_1234567890@c.us_3EB0XXXXX" }
```

**400:** Missing required fields / invalid base64.  
**404:** Instance not found / recipient not registered on WhatsApp.  
**503:** Instance not ready.  
**413:** Body too large.

---

### `GET /api/instances/:instanceId/messages`

**Auth:** Session (with access) or instance API key.

Returns the persisted message log. Messages are ordered newest first.

**Query params:**
- `limit` — integer, default `500`, max `2000`.

**200:**
```json
[
  {
    "id": 42,
    "messageId": "true_1234567890@c.us_3EB0XXXXX",
    "from": "1234567890@c.us",
    "senderDisplay": "John Doe",
    "senderNumber": "1234567890",
    "groupName": null,
    "body": "Hello!",
    "source": "native",
    "messageKind": "individual",
    "timestamp": 1234567890,
    "createdAt": 1234567891
  }
]
```

**Field notes:**
- `id` — DB row ID.
- `messageId` — WhatsApp message ID (unique per instance).
- `from` — sender JID (`@c.us` for users, `@g.us` for groups).
- `senderDisplay` — push name, contact name, or number.
- `senderNumber` — digits-only number (no JID suffix); `null` for group JIDs.
- `groupName` — group chat name; `null` for individual messages.
- `source` — `"native"` (received directly) or `"webhook"` (via webhook POST).
- `messageKind` — `"individual"` | `"group"` | `"status"`.
- `timestamp` — Unix seconds from WhatsApp; falls back to `createdAt` for webhook messages.
- `createdAt` — Unix seconds when stored in DB.

---

### `DELETE /api/instances/:instanceId/messages`

**Auth:** Session (with access) or instance API key.

Deletes all messages for the instance from the DB and broadcasts `{ type: 'messages_cleared' }` to WS subscribers.

**200:** `{ "success": true, "cleared": 42 }`

---

## User Management (Admin Only)

### `GET /api/users`

**200:**
```json
[
  { "id": 1, "username": "admin", "role": "admin" },
  { "id": 2, "username": "agent1", "role": "user" }
]
```

---

### `POST /api/users`

**Request:**
```json
{ "username": "agent1", "password": "secret", "role": "user" }
```

- `role` optional; `"admin"` or `"user"`, defaults to `"user"`.

**201:**
```json
{ "id": 2, "username": "agent1", "role": "user" }
```

**400:** Username/password required / username already taken.

---

### `DELETE /api/users/:userId`

**200:** `{ "success": true, "deleted": true }`

**400:** Cannot delete yourself / cannot delete the last admin.  
**404:** User not found.

---

### `PATCH /api/users/:userId/password`

**Request:** `{ "password": "newPassword" }`

**200:** `{ "success": true }`

**400:** Password required.  
**404:** User not found.

---

### `GET /api/users/:userId/instances`

**200:** `["client1", "client2"]`

**404:** User not found.

---

### `POST /api/users/:userId/instances`

Assign an instance to a user (no-op if already assigned).

**Request:** `{ "instanceId": "client1" }`

**200:** `{ "success": true }`

**400:** Valid instanceId required.  
**404:** User not found.

---

### `DELETE /api/users/:userId/instances/:instanceId`

Remove an instance assignment from a user.

**200:** `{ "success": true }`

---

## Webhook

### `POST /webhook/wahub/:instanceId`

**Auth:** Instance API key only (no session). Optional HMAC-SHA256 signature — see [webhooks.md](webhooks.md).

Accepts incoming message events from external services (WaHub or custom). Stores messages in DB and broadcasts to WebSocket subscribers.

**200:** `{ "success": true, "received": true }`

Full integration guide: [webhooks.md](webhooks.md).
