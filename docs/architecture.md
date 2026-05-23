# 04 — Architecture Overview

High-level view of the Quelora platform: the monorepo layout, runtime services, how
data flows between them, and the cross-cutting concepts (multi-tenancy, editions,
encryption) that every package shares.

> Sources: `.ai/architecture-overview.md` + synthesized from all package overviews.

---

## 1. The monorepo

Quelora is an npm-workspace monorepo. All packages live under `apps/packages/`.

```
quelora-dx-env/
├── apps/
│   ├── build.sh                        # Builds & tags production Docker images
│   ├── node_modules/                   # Hoisted workspace deps (e.g. ws lives here)
│   └── packages/
│       ├── quelora-common/             # @quelora/common — shared library
│       ├── quelora-public-api/         # Community-facing API           → image: mrger/quelora-api
│       ├── quelora-dashboard-api/      # Admin API + Sentinel broker     → image: mrger/quelora-dashboard-api
│       ├── quelora-dashboard/          # Admin React SPA                 → image: mrger/quelora-dashboard
│       ├── quelora-widget-community/   # Embeddable widget (CDN-served)
│       ├── quelora-worker/             # BullMQ worker                   → image: mrger/quelora-worker
│       └── quelora-jobs/               # BullMQ scheduled jobs           → image: mrger/quelora-jobs
├── openai/delegate.js                  # Task-delegation CLI helper
├── mongo-seed/                         # MongoDB demo seed data
└── docs/                               # Unified documentation (this folder)
```

There is also an optional `@quelora/enterprise` package, loaded dynamically by
`featureLoader`. When absent, every API degrades cleanly to **Community Edition**.

A WordPress plugin (`quelora-wp-plugin`) consumes the platform externally — its
integration contract is documented from the API side (see
[quelora-dashboard-api → WordPress Sync Contract](./packages/quelora-dashboard-api.md)).

---

## 2. Runtime components

| Component | Tech | Port | Responsibility |
|-----------|------|------|----------------|
| `quelora-public-api` | Express 4 | 3000 | All end-user interactions: auth, SSO, posts, comments, profiles, follows, notifications, GIF proxy |
| `quelora-dashboard-api` | Express 4 + `ws` | 3010 | Multi-tenant admin backend, RBAC, WordPress sync, Sentinel Debug Broker |
| `quelora-dashboard` | React 18 SPA | — | Admin UI consuming the dashboard API |
| `quelora-widget-community` | ES6+ widget | — | Embeddable community engagement widget (Main Thread + Web Worker + Service Worker) |
| `quelora-worker` | BullMQ worker | — | Consumes emails, notifications, activity, aggregation queues |
| `quelora-jobs` | BullMQ worker | — | Consumes reputation, suggestion, system, gravity-decay queues |
| `quelora-common` | Library | — | Shared models, services, middlewares, providers — no process of its own |
| MongoDB | `mongo:4.4.0` | 27017 | Primary datastore |
| Redis | `redis:alpine` | 6379 | Cache, rate-limit counters, BullMQ broker, presence, active CID |
| nginx | `nginx:mainline-alpine` | 80/443 | Reverse proxy, TLS termination, rate limiting |
| coturn | `coturn/coturn` | host | TURN server for P2P/WebRTC |
| docker-mailserver + Snappymail | — | 25/465/587/993/8888 | SMTP + webmail |

See [deployment/production-deployment](./deployment/production-deployment.md) for the
full Docker Compose stack.

---

## 3. Request & data flow

```
Browser / Widget ──HTTPS──▶ nginx ──▶ quelora-public-api  ──▶ MongoDB
                                  │                       └─▶ Redis (cache + counters)
                                  │                       └─▶ BullMQ enqueue ──▶ quelora-worker / quelora-jobs
                                  │
Dashboard SPA ────HTTPS──▶ nginx ──▶ quelora-dashboard-api ──▶ MongoDB / Redis
                                  │                          └─▶ Sentinel Debug Broker (WS /ws/debug-broker)
                                  │
WordPress plugin ─HTTPS──▶ nginx ──▶ quelora-dashboard-api  (sync + integration-config endpoints)
```

- **Multi-tenancy**: every model carries a `cid` (Client ID, format `QU-XXXXXXXX-XXXXX`).
  Every query MUST filter by `cid`. Tenant configuration lives in the `Client` document.
- **Background work**: APIs never block on side effects. User events are dispatched
  fire-and-forget via `Promise.allSettled` and, where durable work is needed, enqueued
  to BullMQ for the worker/jobs processes to consume.
- **Caching**: Redis fronts MongoDB for hot reads (post threads, stats, client config).
  The `cacheInvalidator` middleware auto-purges related keys on any mutation request.

---

## 4. Community vs. Enterprise edition

The platform ships as **Community Edition** by default. Optional features come from the
`@quelora/enterprise` package, loaded at runtime via `featureLoader('@quelora/enterprise')`.

- If the package is **present** and modules are enabled per client, extra capabilities
  activate: surveys, gamification, advertising, network (SSE + chat), resilience/P2P,
  push, live mode.
- If **absent**, every enterprise call site is guarded with optional chaining and the
  API runs as plain Community Edition.

Activation is **per client**, not per user:
- `Client.enterpriseModules` — `string[]` of enabled enterprise modules
  (`surveys`, `gamification`, `advertising`, `network`, `resilience`, `push`, `liveMode`).
  Managed by **god** users.
- `Client.communityPlugins` — `string[]` of enabled community plugins
  (`sentinel`, `placer`). Managed by **admin+** users.

`@quelora/common/utils/pluginRegistry.js` turns these arrays into the `{ features, plugins }`
manifest delivered to the widget by `GET /config`.

---

## 5. Role hierarchy (RBAC)

Shared by `quelora-dashboard-api` and `quelora-dashboard`:

```
god (100)          — Master; manages all tenants via active_cid
  admin (50)       — Full client management
    editor (40)    — Content editing
    moderator (30) — Moderation actions
    advertiser (20)— Ad management
  analyst (15)     — Analytics read-only
  user (10)        — Base
```

Higher roles always include lower permissions (numeric-level comparison).

---

## 6. Encryption model

| Layer | Algorithm | Key | Used for |
|-------|-----------|-----|----------|
| Field-level at rest | AES-256-CBC | `ENCRYPTION_KEY` env var | Secrets in MongoDB: VAPID, email, TURN, Nostr, 2FA, ed25519 private key |
| Transport (client config) | AES-CBC | `SHA-256(cid)` derived | Client config delivered to dashboard/widget — non-secret-at-rest, deterministic per CID |
| JWT (dashboard) | HMAC-SHA256 | `JWT_SECRET` / `JWT_ADMIN_SECRET` | Dashboard sessions |
| JWT (WP integration) | HS256 | `client.login.jwtSecret` | WordPress setup wizard, 60s TTL |

`@quelora/common/utils/cipher.js` is the single encryption module. Private keys
(resilience ed25519, TURN credentials) are encrypted server-side and **never** returned
to any client.

---

## 7. Where to go next

- Working rules → [01 — Getting Started](./01-getting-started.md), [02 — Coding Standards](./02-coding-standards.md)
- Shared building blocks → [quelora-common](./packages/quelora-common.md)
- Per-service detail → the [package docs](./README.md#packages)
- Running it in production → [deployment/production-deployment](./deployment/production-deployment.md)
