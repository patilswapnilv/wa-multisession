# Development Guide

## Prerequisites

- Node.js 18+ (ES modules required)
- pnpm (`npm i -g pnpm`)
- Chromium dependencies — `whatsapp-web.js` drives a headless browser via Puppeteer

**Linux only:**
```bash
apt-get install -y chromium-browser libgbm-dev
```

---

## Local Setup

```bash
# 1. Install all deps (root + ui workspace)
pnpm install

# 2. Copy and fill in env vars
cp .env.example .env   # or create manually — see docs/configuration.md

# 3. Start server + Svelte dev server concurrently
pnpm run full:dev
```

| Script | What it does |
|---|---|
| `pnpm dev` | Server only (`node server.js`) |
| `pnpm run dev:ui` | Svelte Vite dev server (HMR on port 5173) |
| `pnpm run full:dev` | Both concurrently |
| `pnpm run build:ui` | Build Svelte to `ui/dist/` |
| `pnpm run build` | Build UI + copy to `public/` |

The server serves `ui/dist/` statically when it exists. During development, point your browser at `http://localhost:5173` (Vite, with HMR) or `http://localhost:3000` (server-served build).

---

## Project Structure

```
server.js                 Main process: Express + WebSocket + WhatsApp lifecycle (~1140 lines)
db.js                     All SQLite access: schema, migrations, queries (~334 lines)
package.json              Root scripts + server dependencies
pnpm-workspace.yaml       Workspace config (root + ui/)
instances.json            Persisted list of instance IDs (auto-managed, do not edit manually)
data.db                   SQLite database (created on first run)
.wwebjs_auth/             LocalAuth session dirs (one per instance, auto-managed)

ui/
  src/
    main.js               Svelte entry point
    App.svelte            Auth gate: Login | Dashboard | Docs
    app.css               Global styles (CSS custom properties)
    lib/api.js            Typed fetch wrappers for all API endpoints
    components/
      Login.svelte        Login form
      Dashboard.svelte    Instance grid + user management (admin)
      InstanceCard.svelte QR display, linked account, API key, send form
      MessageLog.svelte   Message history + live WebSocket feed + filters
      Docs.svelte         In-app API documentation page
      Disclaimer.svelte   Footer disclaimer

docs/                     Developer documentation (you are here)
scripts/                  Manual test scripts and utilities
```

---

## Adding a REST Endpoint

Pattern: pick the appropriate auth middleware, add the route to `server.js`, add any DB helpers to `db.js`.

```javascript
// server.js — example: GET /api/instances/:instanceId/stats

app.get('/api/instances/:instanceId/stats', requireInstanceAccess, (req, res) => {
  const instanceId = req.instanceId; // set by requireInstanceAccess
  const instance   = whatsappInstances.get(instanceId); // live in-memory state

  const stats = db.getMessageStats(instanceId); // add to db.js
  res.json({
    status:       instance.status,
    messageCount: stats.total,
    lastMessage:  stats.lastAt,
  });
});
```

**Auth middleware options:**

| Middleware | Use when |
|---|---|
| `requireAuth` | Any logged-in session user |
| `requireAdmin` | Admin users only |
| `requireInstanceAccess` | Session user with instance access OR instance API key; sets `req.instanceId` |
| `requireWebhookAuth` | API key only (webhook endpoints) |

---

## Adding a DB Column

1. Add `ensureColumn()` call in `initDb()`:

```javascript
// db.js → initDb()
ensureColumn(database, 'instance_messages', 'my_new_field', 'TEXT');
```

2. If existing rows need a computed default, add a backfill function called from `initDb()`:

```javascript
function backfillMyNewField(database) {
  const rows = database.prepare(
    "SELECT id FROM instance_messages WHERE my_new_field IS NULL LIMIT 10000"
  ).all();
  const stmt = database.prepare("UPDATE instance_messages SET my_new_field = ? WHERE id = ?");
  for (const row of rows) {
    stmt.run(computeValue(row), row.id);
  }
}
```

3. Update `insertMessage()` signature and SQL, and any query functions that return the column.

**Rules:**
- Never add a `NOT NULL` column without a `DEFAULT` — SQLite rejects `ALTER TABLE ADD COLUMN NOT NULL` with no default.
- Backfills run on every startup until all rows are filled; keep them idiomatic and fast.

---

## Adding a WebSocket Event

Broadcast from anywhere by calling `broadcastToInstance(instanceId, payload, logOrigin?)`:

```javascript
// Notify all subscribers for an instance
broadcastToInstance('client1', {
  type: 'my_event',
  instanceId: 'client1',
  data: { /* … */ }
}, 'my_event'); // logOrigin used in console output
```

`broadcastToInstance` prunes dead connections, serializes the payload once, and sends to all open subscribers. It returns `{ sent, failed, errors }`.

---

## Auth Middleware Internals

### `requireInstanceAccess`

Checks `X-API-Key` header first (no session needed). If present, looks up the key in `instance_api_keys` and verifies it matches `req.params.instanceId`. Falls through to session check if no key.

```
X-API-Key present?
  → getInstanceIdByApiKey(key)
  → key.instanceId === req.params.instanceId && whatsappInstances.has(instanceId)
  → req.instanceId = instanceId; next()

Otherwise:
  → requireAuth (session check)
  → db.userCanAccessInstance(user.id, user.role, instanceId)
  → req.instanceId = instanceId; next()
```

### `userCanAccessInstance`

Admin role → always true. User role → checks `user_instances` table for a row.

---

## UI Development

The Svelte UI talks to the backend via `ui/src/lib/api.js`. All fetch wrappers live there. When adding a new endpoint:

1. Add a typed wrapper in `api.js` following the existing pattern:

```javascript
export async function getInstanceStats(instanceId) {
  const res = await apiFetch(`${API_BASE}/api/instances/${instanceId}/stats`);
  if (!res.ok) throw new Error((await res.json()).error || 'Failed');
  return res.json();
}
```

2. Call it from the appropriate component.
3. Update `Dashboard.svelte` or add a new component if the feature needs its own view.

**Dev URL:** `http://localhost:5173` (Vite, with HMR). The Vite config proxies API calls to `http://localhost:3000`.

---

## Code Conventions

- **ES modules** (`import`/`export`), no CommonJS `require`.
- **Async/await** everywhere; no raw Promise chains.
- **Logging:** Use `console.log`/`console.error` with a `[tag]` prefix, e.g. `[send-message]`, `[webhook]`, `[message]`. Include `instance=`, `to=`, status codes, and IDs so logs are greppable.
- **Error responses:** `res.status(N).json({ error: 'human-readable message' })` — never expose stack traces.
- **No ORM:** All SQL is inline in `db.js` with `better-sqlite3` prepared statements.
- **Mutations are synchronous** in better-sqlite3 (`run()`, `get()`, `all()`) — no `await` needed for DB calls.
- **One file per concern:** HTTP routing in `server.js`, DB in `db.js`, no third file needed yet.

---

## Testing

No automated test suite currently. Manual smoke tests:

```bash
# Start server
pnpm dev

# Login and get a cookie
curl -c cookies.txt -X POST http://localhost:3000/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'

# List instances
curl -b cookies.txt http://localhost:3000/api/instances

# Get API key for an instance
curl -b cookies.txt http://localhost:3000/api/instances/client1/api-key

# Send a message
curl -X POST http://localhost:3000/api/instances/client1/send-message \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_KEY" \
  -d '{"to":"919876543210","message":"Hello"}'
```

Scripts in `scripts/` have more comprehensive examples for API, webhooks, and WebSocket testing.

---

## Troubleshooting

**QR code not appearing:**
- Check server console for `client.on('qr')` log lines.
- Check browser console for WebSocket errors.
- Confirm the instance status is `qr_ready` in `GET /api/instances`.

**Instance stuck at `initializing`:**
- Puppeteer / Chromium may be missing system dependencies (Linux).
- Check server logs for `Failed to launch the browser process`.

**Session expired / 401 on every request:**
- `SESSION_SECRET` changed between restarts — all existing sessions invalidate.
- Confirm cookie is being sent (`credentials: 'include'` for cross-origin).

**WebSocket closes immediately:**
- Confirm the session cookie is valid (browser) or the API key is correct (script).
- Check for `error: 'Access denied to this instance'` in the WS message log.

**`data.db` locked:**
- Only one process should write to `data.db`. If running multiple instances of the app, each needs its own DB path.
