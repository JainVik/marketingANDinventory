# 02 — System Architecture

Binding technical blueprint for v1. Product scope lives in `01-product-scope.md`; data shapes in `03-database-schema.md`; endpoints in `04-api-contract.md`.

## 1. Stack (fixed — do not substitute)

| Layer | Choice | Notes |
|---|---|---|
| Runtime | Node.js 22 LTS, TypeScript strict mode everywhere | No plain JS files |
| API framework | Express 5 | Layered MVC (below) |
| DB | MongoDB 7+ (Atlas, replica set) via Mongoose 8 | Replica set is REQUIRED — we use transactions |
| Cache / queues | Redis 7 + BullMQ | WhatsApp queue, OTP store, rate-limit counters |
| Real-time | Socket.IO (server + vendor dashboard + customer status page) | Polling fallback is built into Socket.IO |
| Frontend | React 18 + Vite + TypeScript; TanStack Query for server state; Zustand for the little client state that exists; Tailwind CSS | No Redux |
| Validation | Zod — single source of truth, shared between API and frontends | No Joi, no hand-rolled validation |
| Auth | JWT access (15 min) + rotating refresh token (30 days, httpOnly cookie) | Detail in §6 |
| WhatsApp | Meta WhatsApp Business Platform (Cloud API) via a BSP account | Behind our own `NotificationService` interface |
| Tests | Vitest + Supertest; mongodb-memory-server for unit/integration | Coverage gates in CLAUDE.md |
| Deploy v1 | Docker Compose on a single VPS (api, redis, caddy) + MongoDB Atlas + frontends on static hosting/CDN | Keep 12-factor: config only via env |

## 2. Monorepo layout (npm workspaces)

```
/
├── CLAUDE.md                  # engineering rules — AI reads this first
├── docs/                      # this documentation pack
├── package.json               # workspaces root
├── apps/
│   ├── api/                   # Express backend
│   │   └── src/
│   │       ├── config/        # env parsing (zod-validated), constants
│   │       ├── loaders/       # app bootstrap: db, redis, socket.io, express
│   │       ├── modules/       # DOMAIN MODULES — see §3
│   │       │   ├── auth/
│   │       │   ├── vendors/
│   │       │   ├── tables/      # table management, tokens, floor state
│   │       │   ├── catalog/
│   │       │   ├── orders/      # table sessions, rounds, order lifecycle
│   │       │   ├── billing/     # GST tax invoices, coupons, day-close reconciliation
│   │       │   ├── reviews/
│   │       │   ├── customers/   # per-vendor customer profiles & segments
│   │       │   ├── campaigns/   # triggers + manual campaigns
│   │       │   └── admin/       # super_admin endpoints
│   │       ├── middlewares/   # auth, tenantContext, errorHandler, rateLimit, validate
│   │       ├── jobs/          # BullMQ queues, workers, cron schedules
│   │       ├── sockets/       # socket.io namespaces & auth
│   │       └── shared/        # api-wide helpers only (logger, AppError, asyncHandler)
│   ├── customer-web/          # customer PWA
│   ├── vendor-web/            # vendor dashboard SPA
│   └── admin-web/             # minimal super_admin panel
└── packages/
    └── shared/                # zod schemas, TS types, enums, constants
                               # (order states, roles, error codes) — imported by ALL apps
```

**Rule:** anything used by ≥2 apps lives in `packages/shared`. Order-state enums, zod schemas, and error codes are NEVER redeclared per app.

## 3. Backend module pattern (layered MVC)

Every domain module has exactly this internal shape — no exceptions, no extra layers:

```
modules/orders/
├── orders.routes.ts       # route declarations + middleware chain only
├── orders.controller.ts   # HTTP in/out: parse (zod), call service, shape response. NO business logic.
├── orders.service.ts      # ALL business logic. No req/res. Throws AppError. Testable in isolation.
├── orders.model.ts        # Mongoose schema + model (data shapes in docs/03)
└── orders.events.ts       # domain events this module emits (optional)
```

- Controllers never touch models. Services never import Express types.
- Cross-module calls go **service → service** (e.g., `ordersService` calls `customersService.recordCompletedOrder()`), never service → foreign model.
- Domain events (in-process `EventEmitter` wrapper, `shared/events.ts`) decouple side effects: `order.completed` → customers module updates profile → campaigns module evaluates triggers. Event handlers must be idempotent and must never crash the emitter (wrap + log).

## 4. Multi-tenancy (the load-bearing wall)

- Single shared database. Every vendor-owned document carries an indexed `vendorId` (the tenant key). There are no cross-tenant queries outside the `admin` and customer-discovery modules.
- **`tenantContext` middleware** runs after auth on all vendor-dashboard routes: it resolves `vendorId` from the JWT (never from params/body/query) and attaches `req.tenant = { vendorId }`.
- **Repository guard:** vendor-scoped services access data only through a `scopedModel(Model, vendorId)` helper (thin wrapper that injects `{ vendorId }` into every filter and forbids `updateMany/deleteMany` without it). Direct `Model.find(...)` calls in vendor-scoped services are a lint error (custom ESLint rule, see CLAUDE.md).
- Customer-facing reads (storefront, discovery) are public but always explicitly filtered by `vendorId` or `city` — never unbounded.
- Cross-tenant access is a **test-enforced invariant**: the integration suite includes "vendor A token requests vendor B resource → 404" for every vendor resource type (checklist in `06-security-checklist.md` §4).

## 5. Request lifecycle (canonical)

```
request
 → requestId middleware (uuid, attached to logs)
 → helmet / cors (allowlist)
 → rate limiter (per route class — see 06 §6)
 → body parse (json, 100kb limit)
 → auth middleware (route-dependent)
 → tenantContext (vendor routes)
 → validate(zodSchema) — parses body/params/query; 422 on failure
 → controller → service → model
 → success: res.json(envelope)   |   throw AppError
 → central errorHandler (LAST middleware): maps AppError → envelope,
   unknown errors → 500 + logged with requestId, never leak stack/internals
```

Envelope and error codes are defined once in `04-api-contract.md` §2 and implemented once in `shared/`.

## 6. AuthN/AuthZ

- **Customers:** phone + OTP (6-digit, 5-min TTL, stored hashed in Redis, max 3 verify attempts, resend cooldown 60s, max 5 OTPs/phone/hour). OTP delivery in v1 via WhatsApp template (fallback: SMS provider interface stubbed, not implemented).
- **Vendor users & super_admin:** email + password (argon2id), optional later 2FA.
- Access JWT (15 min) carries `{ sub, role, vendorId? }`, signed RS256. Refresh token: opaque, rotating, stored hashed in DB with device info; reuse of a rotated token revokes the whole family (theft detection).
- Frontends keep access token in memory only (never localStorage); refresh via httpOnly `SameSite=Strict` cookie endpoint.
- RBAC: single `authorize(...roles)` middleware + per-service assertions for object-level checks ("this staff belongs to this vendor"). Full permission matrix in `06-security-checklist.md` §3.
- Socket.IO connections authenticate with the access JWT at handshake; vendor sockets join room `vendor:{vendorId}`, customer sockets join `order:{orderId}` (only after ownership check).

## 7. WhatsApp pipeline (never call Meta synchronously from a request)

```
trigger/campaign/OTP
  → NotificationService.send(message)        # channel-agnostic interface
  → persist message_log (status: queued)
  → BullMQ 'whatsapp' queue
  → worker: consent check → quota check → template render → BSP API call
      success  → message_log: sent (later delivered/read via webhook)
      4xx      → message_log: failed_permanent (bad template/number) — no retry
      5xx/429  → retry with exponential backoff (5 attempts) → failed_retryable → dead-letter queue
  → webhook endpoint receives delivery/read/opt-out callbacks (signature-verified)
```

- Consent + quota checks happen **in the worker at send time**, not at enqueue time (state may change in between).
- Cron jobs (BullMQ repeatable): daily birthday scan (08:00 IST), daily segment recompute (03:00 IST), win-back evaluator (03:30 IST). All cron logic must be idempotent — running twice sends nothing twice (dedupe key: `{customerId}:{triggerType}:{date}` in message_logs).
- **WABA architecture (GoKwik model)**: Cafes onboard their own WhatsApp Business Account via Meta Embedded Signup / Cloud API, displaying their own verified business name and isolating deliverability and quality ratings. Inbound webhooks route dynamically by `phoneNumberId` to the correct vendor.
- Worker tracks each customer's 24h service-window state so utility sends (e-bills, status) ride the free window (₹0) whenever open, logging window state per send for cost analytics.

## 8. Real-time strategy

- Order & Session events: after DB commit, emit `order:new`, `order:status`, and `table:service_call` to `vendor:{vendorId}` room. For diners, emit `round:status` and `table:updated` to `session:{sessionToken}` room.
- Socket emit failures never fail the HTTP request (fire-and-forget with logging).
- Both dashboards also poll every 30s as belt-and-braces reconciliation (TanStack Query refetch) — the socket is a latency optimization, never the source of truth.

## 9. Frontend architecture (all three React apps)

```
src/
├── api/          # ONE axios instance + generated per-module API functions; zod-parse responses
├── components/   # dumb/presentational, no data fetching
├── features/     # feature folders: components + hooks + local state per feature
├── hooks/        # shared hooks
├── pages/ or routes/
├── stores/       # zustand slices (cart, ui) — server data stays in TanStack Query
└── lib/          # utils, i18n, constants re-exported from packages/shared
```

- Data fetching only via TanStack Query hooks in `features/*/hooks`. Components never call axios directly.
- Cart (customer PWA) persists to localStorage with schema-versioned zod parse on load (corrupt → reset).
- PWA: vite-plugin-pwa, offline shell for storefront browsing; ordering requires connectivity (no offline order queue in v1). Bundle budget: ≤ 200 kB gzipped initial JS for customer-web.
- i18n: all strings through i18next from day one, `en` only shipped.

## 10. Observability & ops

- pino structured logs (JSON) with `requestId`, `vendorId`, `userId` on every line; PII redaction rules in 06 §8.
- `/healthz` (liveness) and `/readyz` (mongo+redis ping) endpoints.
- Global process handlers: `unhandledRejection`/`uncaughtException` → log fatal → graceful shutdown (stop accepting, drain queues, exit) — process manager restarts.
- Sentry (or self-hosted GlitchTip) on api + frontends from day one.
- Backups: Atlas continuous backup; Redis is rebuildable state only (OTPs, queues, counters) — nothing durable lives only in Redis.

## 11. Environments & config

- `local` (docker-compose incl. mongo replica-set single node), `staging`, `production`.
- All config via env vars, parsed once at boot by a zod schema in `config/env.ts` — the app refuses to start on missing/invalid config. No `process.env` access anywhere else.
- Secrets never in the repo. `.env.example` lists every variable with a comment.
