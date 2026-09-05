# 10 — Backend Technical Roadmap

The ordered path for building the backend: first the **decisions** (Stage A — made on paper, cheap to change), then the **build sequence** (Stage B — each step depends on the one before it), then a **concepts refresher** so the whole team shares the same vocabulary. Where a decision is already locked in docs/02–04, this doc says so and explains *why* — so nobody relitigates it in month three without new evidence.

---

## Stage A — Decisions, in the order they must be made

### A1. Architecture style: monolith vs modular monolith vs microservices

The options, honestly:

- **Classic monolith** — one codebase, one deploy, internal structure ad hoc. Fastest week 1, painful month 6: everything touches everything, no boundaries for the AI (or humans) to respect.
- **Microservices** — separate deployable services (auth service, order service, notification service…) talking over the network. Solves organizational scale (many teams shipping independently) at the price of distributed-systems pain: network failures between your own functions, distributed transactions, service discovery, per-service CI/CD, observability sprawl. **With 3 people this is self-harm.** You'd spend the pilot debugging RPC instead of onboarding cafes.
- **Modular monolith (our choice, docs/02 §3)** — ONE deployable app, but internally split into strictly-bounded domain modules (`auth/`, `orders/`, `catalog/`, `campaigns/`…), each with its own routes/controller/service/model, talking only service→service. All microservice discipline, zero network tax.

**The escape hatch is built in**: because modules only touch each other through service interfaces and domain events, any module can be extracted into a real service later by swapping the in-process call for an HTTP/queue call. The first thing to split — long before any domain module — is the **worker process** (BullMQ consumers) from the API process: same codebase, two processes (`node api.js` / `node worker.js`). That's not microservices; that's just not letting a WhatsApp retry storm eat your order-placement CPU.

**Revisit-when**: a module needs independent scaling (worker: at launch), a different language, or a separate team. Not before.

### A2. Tech stack — chosen by criteria, not fashion

Criteria: (1) the team already knows it (we're MERN people — debugging unfamiliar tech during a pilot fire is how startups die), (2) AI codegen quality (Claude/Copilot write the most reliable code in mainstream TS/Node patterns), (3) hiring pool in India, (4) boring beats clever for a 3-person production system.

Locked (docs/02 §1): **Node 22 LTS + TypeScript strict · Express 5 · PostgreSQL 16 on Supabase via Drizzle ORM (SQL migrations are truth) · Zod · Redis 7 + BullMQ · Socket.IO · Vitest + Supertest · Docker Compose on a VPS + Caddy**. Alternatives we consciously rejected: NestJS (more framework ceremony than a 3-person team needs), Fastify (fine, but Express has the deepest AI-training-data + middleware ecosystem), **MongoDB/Mongoose (our first choice, reversed 5 Sep 2026: the money/GST/holdout/attribution model is relational, RLS gives tenant isolation in the database instead of in code, and partitions + pgvector + logical replication cover the ledger/AI/ML needs without a second service)**, Prisma (can't express partitions/RLS/pgvector; Drizzle introspects what SQL defines), GraphQL (see A4).

### A3. Data stores — one source of truth, one accelerator

- **PostgreSQL = the system of record.** Everything durable lives here — including the event ledger, message ledger, feature store and vector embeddings (schemas `app`, `ml`, `ops`). Modeling rules live in docs/03; the three habits that keep it honest: every vendor request inside `withTenant()` (RLS does the filtering), every contended write a guarded `UPDATE … WHERE` / row lock (never select-then-update), every money value integer paise (`app.paise`).
- **Redis = ephemeral accelerator, never a database.** Its four jobs here: (1) **queues** — BullMQ for WhatsApp sends, crons, retries; (2) **OTP store** — hashed codes with 5-min TTL; (3) **rate-limit counters**; (4) later, **cache** and the Socket.IO adapter for multi-instance scaling. The rule: **if Redis is wiped, the business must lose nothing durable** — only in-flight jobs (redelivered) and counters. Anything you're tempted to keep "just in Redis" belongs in Postgres.
- **Object storage (S3-compatible)** for images via presigned uploads (docs/04 §8) — the API never proxies file bytes.

### A4. API style

**Versioned REST (`/api/v1`)** with a fixed envelope, shared error-code enum, zod validation at every boundary, cursor pagination, and idempotency keys on unsafe money-adjacent POSTs (docs/04). Rejected: **GraphQL** (its win is many clients composing flexible queries; we have three first-party frontends we control — the flexibility buys nothing and costs N+1 resolvers, cache complexity, and authz-per-field), **tRPC** (lovely TS DX but couples frontend to backend deploys and locks out future third-party/API consumers). REST + shared zod schemas in `packages/shared` gives tRPC-ish type safety with none of the coupling.

### A5. Auth & identity

- **Two identity styles, one system**: customers = phone + OTP (passwordless — it's India, the phone number IS the identity, and we need the number anyway for WhatsApp); vendors/admins = email + argon2id password.
- **Stateless access JWT (15 min, RS256) + rotating refresh token** (opaque, hashed in DB, httpOnly cookie, family-revocation on reuse — docs/02 §6). Why not server sessions? Sessions need shared session storage for every request across instances; JWT verifies locally, and the 15-min expiry bounds revocation lag. Why not JWT-only with long expiry? Because you can't log out a stolen token — the refresh-rotation family is the theft alarm.
- **Authorization is two separate layers, always**: role check (`authorize('vendor_admin')` middleware) + object-level ownership check inside the service ("this order belongs to this vendor"). Role middleware alone is the classic IDOR hole.
- **Tenant context**: `vendorId` resolved from the JWT by middleware into `req.tenant`, never read from client input on vendor routes (docs/02 §4). This one rule is the multi-tenant security model.

### A6. Layering: routes → controllers → services → models (and the repository question)

- **Route** — declares the path + middleware chain. Zero logic.
- **Controller** — HTTP translator: parse (zod), call service, shape envelope. Never imports models.
- **Service** — ALL business logic, pure of HTTP. Throws typed `AppError`s. This is the layer you unit-test.
- **Data** — Drizzle tables from `db/schema.ts` + queries; services receive a tenant-bound `tx`.
- **Repository pattern?** Full repositories earn their keep in big teams/DDD codebases; for us they'd be a third name for every query. **Our decision: Drizzle IS the data layer; services take a tenant-bound `tx` from `withTenant()` and Postgres RLS enforces the tenant filter** (docs/02 §4). That's the 20 % of the repository pattern that pays (tenant safety, mockability — a `tx` is trivially faked) without the boilerplate. Revisit-when: query logic starts duplicating across services (then a `queries/` folder per module, still Drizzle).
- **Cross-module traffic**: service→service calls for commands; **domain events** (in-process emitter) for side effects — `order.completed` → customers module updates the profile → campaigns module evaluates triggers. Handlers idempotent, failures logged, never crash the emitter. This is the seam that later becomes a message bus if we ever split services.

### A7. The middleware chain (fixed order, docs/02 §5)

`requestId → helmet/CORS → rate limiter → body parse (100kb) → auth → tenantContext → validate(zod) → controller`, with the **central error handler last**. Two rules: middleware order is part of the security model (rate-limit before body-parse; auth before tenant), and no middleware swallows errors — everything funnels to the one handler that maps `AppError` → envelope and unknowns → logged 500.

---

## Stage B — Build sequence (each step stands on the previous)

| # | Step | Done means |
|---|---|---|
| B1 | **Scaffold**: monorepo workspaces, TS strict, ESLint (incl. custom rules: no `vendorId` from req in vendor modules, no `dangerouslySetInnerHTML`), Prettier, `packages/shared` with error codes + order-state machine + zod schemas | `npm run lint && npm run build` green in CI on PR #1 |
| B2 | **Config**: `config/env.ts` — zod-parses ALL env vars at boot, app refuses to start on missing config; `.env.example` | Deleting any env var makes boot fail loudly (CI-tested) |
| B3 | **Errors + logging**: `AppError`, `asyncHandler`, central error middleware, pino with requestId/vendorId/userId, PII redaction paths | Throwing anywhere returns the envelope; nothing leaks a stack trace |
| B4 | **DB layer**: `db/migrations/0001_init.sql` applied (`drizzle-kit migrate` / psql), `drizzle-kit pull` generates `db/schema.ts`, `db/client.ts` with `withTenant()` / `asWorker()`, seed script (2 outlets, menus, orders in every state, tools registry, system automations), `db/seed/smoke.sql` green | `npm run db:reset && npm run seed:dev` gives a demoable database from zero, RLS on |
| B5 | **Middleware chain** wired in canonical order + `/healthz` `/readyz` | A request travels the full chain; readyz flips when Postgres/Redis drop |
| B6 | **Auth module** (it blocks everything else): OTP request/verify (Redis TTL, attempt caps), email+password login, refresh rotation + family revocation, `authorize()`, tenantContext | docs/05 §1 cases tested — expired/garbage tokens, OTP abuse, refresh reuse |
| B7 | **Domain modules in dependency order**: vendors + wizard → menu → tables → sessions + **orders** (row lock + guarded stock + idempotency + auto-accept + outbox event — the hard one) → billing + payments (gateway adapters, refunds) → customers + consent → whatsapp (queue, worker, webhook, conversations) → marketing (automations, campaigns, holdout, ledger) → insights + briefing → agents (tools registry, owner bot read-only) → admin. Each = the vertical-slice DoD from docs/09 §P4 | Per-module: docs/05 edge cases green + tools entries seeded + leak matrix regenerated |
| B8 | **Jobs**: BullMQ queues + the separate worker process; event consumers (outbox → automations / features / sockets); repeatable crons (birthday 08:00 IST, segments 03:00, win-back 03:30, dead-hours 04:00, briefing 08:00, order auto-expiry, partitions weekly); `message_dedupe` / `automation_runs.idempotency_key`; DLQ | Kill the worker mid-job → redelivery causes no double effect |
| B9 | **Real-time**: Socket.IO namespaces, JWT handshake, room-join ownership checks, emit-after-commit, 30s polling reconciliation on clients | Vendor board updates <2s; socket death degrades to polling, invisibly |
| B10 | **Integrations behind interfaces**: `NotificationService` → WhatsApp Cloud API adapter (platform channel), `PaymentProvider` → Razorpay + Cashfree adapters (restaurant = merchant of record), `LlmProvider` → model adapter; webhook endpoints (signature-verified, dedupe table, raw stored, process-async) | Swapping a provider = one adapter file; webhook replay is idempotent |
| B11 | **Test hardening**: unit (services with a fake `tx`), integration (Supertest + real Postgres via testcontainers/docker-compose, migrations applied, RLS on), the generated **tenant-leak matrix**, load smoke (50 concurrent orders on `limited` stock, 20 concurrent settles) | docs/06 §10 release gates green |
| B12 | **Observability**: Sentry, log-based alerts (5xx spike, DLQ growth, webhook signature failures), uptime monitor | An induced staging error reaches the team group chat |
| B13 | **Security pass**: run docs/06 top to bottom — `sql.raw` audit, RLS/pooler discipline, upload magic-byte checks, CORS allowlist, vault refs for gateway/WABA secrets, secrets scan in CI, dependency audit | Checklist signed off by the founder who didn't write the code |
| B14 | **Performance sanity** (only now): `EXPLAIN (ANALYZE, BUFFERS)` on the 10 hottest queries (CI gate: no seq scan on tenant tables), partition pruning verified on events/messages, THEN caching only where measurement demands (public menu — cache-aside 60 s, invalidate on catalog write) | p95 latency known for the 5 hottest endpoints |
| B15 | **Deploy pipeline**: Dockerfiles (non-root), compose (api + worker + redis + caddy), Supabase prod project (roles, pooler, PITR, pg_cron jobs), CI: test → build → migrate (direct URL) → deploy with graceful shutdown (drain in-flight ≤ 10 s), rehearsed rollback (migrations are forward-only + PITR) | Deploy is one merged PR; rollback is one command; both rehearsed |

**Scaling path when the day comes** (in order, each step only when metrics demand): bigger VPS → API and worker on separate machines → 2+ API instances behind Caddy with the Socket.IO Redis adapter → read-heavy endpoints cached → Supabase compute up → read replica for reports/ML → detach old partitions to the parquet lake. Sharding, Kubernetes, and microservices are not on this list for years, if ever.

---

## Concepts refresher (the shared vocabulary)

- **Middleware** — a function in the request pipeline (`(req,res,next)`): sees every request before the controller. Auth, rate limiting, validation, error handling are all middleware. Order matters.
- **Controller vs Service** — controller speaks HTTP (parse, status codes); service speaks business ("place order", "compute segment"). The test: a service must be callable from a cron job or test with no `req`/`res` anywhere.
- **Repository** — an abstraction over data access so business logic doesn't know the database. We use the thin version: a tenant-bound `tx` from `withTenant()` over Drizzle, with RLS as the guard.
- **Row-Level Security (RLS)** — Postgres filters every row by a policy (`vendor_id = current tenant`) no matter what SQL the app sends. The tenant wall lives in the database.
- **Outbox** — the `events` row written in the same transaction as the business change; workers read it afterwards. Guarantees a side effect is never fired for a change that rolled back.
- **Partition** — one logical table stored as monthly physical tables (`events_2026_09`); old months detach to cheap storage.
- **Modular monolith** — one deployable, hard internal module boundaries. Microservice discipline without the network.
- **DTO / schema** — the declared shape of data crossing a boundary. Ours are zod schemas: one declaration = runtime validation + TS type.
- **Transaction** — several DB writes that succeed or fail as one (order insert + stock decrement + outbox event). Postgres: `BEGIN … COMMIT`; row locks (`FOR UPDATE`) serialise writers on the same session.
- **Guarded/atomic update** — write with the precondition inside the query (`UPDATE … WHERE status = 'placed'`); `rowCount 0` means you lost the race. The cure for select-then-update bugs.
- **Idempotency key** — client-generated UUID on a POST so a retry returns the original result instead of double-charging/double-ordering.
- **Queue / worker / DLQ** — producer enqueues a job (Redis via BullMQ), a separate worker process executes with retries + exponential backoff; permanently failing jobs land in a dead-letter queue for humans.
- **Cron (repeatable job)** — scheduled work (birthday scan). Must be idempotent: running twice sends nothing twice (dedupe keys).
- **Domain event** — "order.completed" emitted by one module, consumed by others. Decouples side effects from the command that caused them.
- **Cache-aside** — read cache → miss → read DB → write cache with TTL; invalidate on write. The only caching pattern we'll use, and only after measuring.
- **JWT vs session** — session = server remembers you (state on server); JWT = signed claim you carry (state in token). We use short JWT + DB-backed refresh = mostly stateless with a revocation lever.
- **RBAC** — role-based access control: what a `role` may do. Always paired with object-level ownership checks.
- **Tenant isolation** — every vendor sees only their rows; enforced by RLS keyed on the JWT-derived `vendorId`, proven by the generated leak-matrix tests.
- **Rate limiting** — per-key request caps (Redis counters): brute-force and cost-attack defense. Fails closed on auth routes, open elsewhere.
- **Migration** — versioned, ordered SQL files in `db/migrations/` (tables, indexes, policies, backfills) run by CI with the direct DB URL, never by hand. `drizzle-kit pull` regenerates the TypeScript schema afterwards.
- **Graceful shutdown** — on deploy: stop accepting, finish in-flight work, close connections, exit. Why deploys don't drop orders.
- **Horizontal vs vertical scaling** — more machines vs bigger machine. Vertical first; horizontal needs the Redis socket adapter + statelessness we've already designed for.

---

## Decision record (revisit only with evidence)

| Decision | Choice | Revisit when |
|---|---|---|
| Architecture | Modular monolith, worker split from API | A module needs independent scale/team |
| Stack | Node/TS/Express/Drizzle/Zod/BullMQ | Never mid-build; v2 with data |
| Database | PostgreSQL 16 on Supabase (reversed from MongoDB, 5 Sep 2026) | Never mid-build. Read replica / lake for analytics before any second store |
| Redis | Queues/OTP/rate-limit/cache — ephemeral only | Never store durable state |
| API | Versioned REST + envelope + zod | Third-party API program (add OpenAPI then) |
| Auth | OTP customers, password vendors, JWT15m + rotating refresh | 2FA for vendors post-pilot |
| Data access | Drizzle + `withTenant(tx)` + RLS (no full repositories) | Duplicated query logic → per-module `queries/` |
| Real-time | Socket.IO + polling reconciliation | >2 instances → add Redis adapter |
| Files | Presigned S3-compatible uploads | — |
| Deploy | Docker Compose on VPS + Caddy + Supabase | >~50 vendors or >1 instance needed |
