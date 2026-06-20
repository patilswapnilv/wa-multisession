# Configuration

All configuration is via environment variables. Create a `.env` file in the project root (loaded automatically by `dotenv`).

```
# .env
SESSION_SECRET=change-this-to-a-random-64-char-string
PORT=3000
```

---

## Environment Variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `SESSION_SECRET` | string | `'your-secret-key-change-this'` | Signs session cookies. **Change in production.** Use `openssl rand -hex 32`. |
| `PORT` | number | `3000` | HTTP + WebSocket listen port. |
| `SESSION_TTL_SECONDS` | number | `86400` (24 h) | Session lifetime in seconds. Rolling: reset on each request. |
| `JSON_BODY_LIMIT` | string | `'10mb'` | Max request body size for JSON endpoints. Accepts Express size strings (`'10mb'`, `'50mb'`). Webhook endpoint always uses `'1mb'` regardless. |
| `NODE_ENV` | string | unset | Set to `'production'` to enable `secure` and `sameSite: strict` on session cookies. Required when running behind HTTPS. |
| `WEBHOOK_SECRET` | string | unset | When set, all POST requests to `/webhook/wahub/:id` must include a valid `X-Webhook-Signature: sha256=<hex>` HMAC-SHA256 header. See [webhooks.md](webhooks.md). |

---

## Session Configuration Detail

```
secret     → SESSION_SECRET (sign cookie)
resave     → false
saveUninitialized → false       (no cookie until login)
rolling    → true               (TTL resets on every request)
cookie.maxAge → SESSION_TTL_SECONDS * 1000
cookie.secure → true  when NODE_ENV=production
cookie.sameSite → 'strict' when NODE_ENV=production, 'lax' otherwise
```

`app.set('trust proxy', 1)` is enabled — required when running behind a reverse proxy (nginx, Caddy, Render, etc.) so `secure` cookies work over HTTPS.

---

## Default Admin Credentials

On first start, if the `users` table is empty, a seed admin is created:

| Username | Password |
|---|---|
| `admin` | `admin123` |

**Change this immediately.** Use the dashboard → User Management → change password, or call `PATCH /api/users/1/password`.

The seed password is hardcoded in `db.js:seedAdminIfNeeded`. Move it to an env var if scripted seeding is needed.

---

## Body Size Limit

`JSON_BODY_LIMIT` applies to all JSON and URL-encoded routes **except** `/webhook/*` (which has its own `1mb` limit hardcoded). Base64 inflates file size by ~33%; for a default `10mb` limit, the source file must be under ~7.5 MB.

To send larger files, raise the limit:
```
JSON_BODY_LIMIT=50mb
```

---

## Production Checklist

- [ ] Set `SESSION_SECRET` to a cryptographically random string (`openssl rand -hex 32`).
- [ ] Set `NODE_ENV=production` (enables `secure` and `sameSite: strict` cookies).
- [ ] Terminate TLS at a reverse proxy (nginx/Caddy) pointing to this app; set `proxy_set_header X-Forwarded-Proto https`.
- [ ] Change the default `admin123` password.
- [ ] Set `WEBHOOK_SECRET` if accepting webhooks from external services.
- [ ] Set `SESSION_TTL_SECONDS` to an appropriate value for your use case.
- [ ] Confirm `JSON_BODY_LIMIT` matches your largest expected file upload.
- [ ] Mount `data.db` and `.wwebjs_auth/` on a persistent volume (Docker / cloud deploys).

---

## `.env` Template

```bash
# Required
SESSION_SECRET=

# Optional — shown with defaults
PORT=3000
SESSION_TTL_SECONDS=86400
JSON_BODY_LIMIT=10mb
NODE_ENV=development

# Webhook HMAC verification (leave empty to disable)
WEBHOOK_SECRET=
```
