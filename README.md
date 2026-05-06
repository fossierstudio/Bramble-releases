# Bramble

> A multi-tenant, real-time chat platform
<img width="500" height="250" alt="d184bbc3-4440-4660-8c0c-ff904a73c8eb" src="https://github.com/user-attachments/assets/eb7e4a8a-518e-4a44-b3e7-ee8ec144c026" />

---

## What it is

Bramble is a production-oriented chat backend with an **Expo** (React Native) cross-platform client. Users join **servers** (tenant boundaries), chat in **channels**, and see each other's messages appear live over WebSocket. Profile pictures, presence (online/idle/dnd/offline), typing indicators, reconnect/resume, and per-user rate limiting are all implemented.

It is intentionally a **modular monolith** — a single Node.js process today with a clear, documented path to a horizontally-scaled Redis-backed cluster in Phase 2.

The web client is an Expo web build served by Nginx; the Node.js backend is purely API + WebSocket.

This is the releases repo! Source code will be open source before too long

---

## Tech Stack

| Concern | Choice | Notes |
|---|---|---|
| Runtime | Node.js ≥ 22 | ESM, native `async/await` |
| Language | TypeScript (strict) | Branded types, `noUncheckedIndexedAccess` |
| HTTP | Fastify v5 | Schema validation, plugin model |
| WebSocket | `ws` | Raw WebSocket, custom gateway protocol |
| Database | PostgreSQL 16 + Kysely | Type-safe SQL, no ORM |
| Auth | `jose` (JWT) + Argon2id | Access + refresh token rotation |
| Validation | Zod | Runtime schemas, env-var config |
| Logging | Pino | Structured JSON, context-bound |
| File uploads | `@fastify/multipart` | Avatars streamed to disk |
| Static files | `@fastify/static` | Serves `/uploads/avatars/*` |
| Web client | Expo (React Native Web) | Built with `npx expo export --platform web`, served by Nginx |
| Rate limiting | `@fastify/rate-limit` | Per-user/IP sliding window |
| IDs | UUIDv7 | Time-ordered, decentralised generation |
| Testing | Vitest | Integration tests against a real Postgres DB |

---

## Features

### Auth
- Register, login (username **or** email), JWT access token + rotating refresh token
- `PATCH /api/users/me` — update display name
- `POST /api/users/me/avatar` — upload avatar (jpeg, png, gif, webp — max 2 MB)
- `DELETE /api/users/me/avatar` — remove avatar

### Servers & Channels
- Create servers, invite others via one-time codes, leave, delete (owner-only)
- Every new user is automatically added to the **Global** server
- Channels with position ordering; default channel is protected from deletion
- Full tenant isolation — every query is scoped to `server_id`

### Messages
- Send, edit (author-only), soft-delete (author **or** moderator/admin/owner)
- Cursor-based pagination via UUIDv7 PKs (`?before=<id>&limit=50`)
- Idempotency-Key header prevents duplicate sends on retry

### Real-time Gateway (`/gateway`)
- Custom WebSocket protocol over `wss://`
- HELLO → IDENTIFY (JWT) → READY (servers, channels, members)
- Heartbeat / ACK with zombie session detection
- RESUME with 2-minute session window and 500-event ring buffer
- Server-initiated RECONNECT for graceful deploys (staggered delays)


### Presence & Typing
- Per-user status: `online`, `idle`, `dnd`, `offline` (invisible)
- Auto-idle sweep after 5 minutes of inactivity
- Typing indicators — ephemeral, channel-scoped, not buffered in RESUME

### Security
- CORS with configurable `ALLOWED_ORIGINS`
- HTTP rate limiting: 100 req/min global (per user/IP); stricter on auth routes
- WebSocket rate limiting: 60 msg/min per connection (close `4007` on violation)
- Control-character stripping on message content
- Argon2id password hashing — locked system accounts can never authenticate

### Profile Avatars in UI
- Extensible `ContextMenu` singleton — click your name to edit profile or log out
- Avatars rendered in: user panel, member sidebar, and every message row
- Fallback to coloured initial when no avatar is set
