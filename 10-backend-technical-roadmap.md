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

Locked (docs/02 §1): **Node 22 LTS + TypeScript strict · Express 5 · MongoDB 7 (Atlas, replica set) via Mongoose 8 · Zod · Redis 7 + BullMQ · Socket.IO · Vitest + Supertest · Docker Compose on a VPS + Caddy**. Alternatives we consciously rejected: NestJS (more framework ceremony than a 3-person team needs; our layering gives the same structure with less magic), Fastify (fine, but Express has the deepest AI-training-data + middleware ecosystem), Prisma+Postgres (we chose Mongo for team skill — the discipline cost is documented in docs/03), GraphQL (see A4).

### A3. Data stores — one source of truth, one accelerator

- **MongoDB = the system of record.** Everything durable lives here. Non-negotiable requirement: **replica set** (Atlas default) because order placement uses multi-document transactions (docs/03 §10). Modeling rules live in docs/03; the three habits that keep Mongo honest: every vendor-scoped query filtered by `vendorId`, every contended write is a guarded atomic update (never read-then-write), every money value an integer.
- **Redis = ephemeral accelerator, never a database.** Its four jobs here: (1) **queues** — BullMQ for WhatsApp sends, crons, retries; (2) **OTP store** — hashed codes with 5-min TTL; (3) **rate-limit counters**; (4) later, **cache** and the Socket.IO adapter for multi-instance scaling. The rule: **if Redis is wiped, the business must lose nothing durable** — only in-flight jobs (redelivered) and counters. Anything you're tempted to keep "just in Redis" belongs in Mongo.
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
- **Model** — Mongoose schema + data access.
- **Repository pattern?** A repository is an interface between services and the DB (`orderRepo.findByVendor(...)`) so you could swap databases or mock storage. Full repositories earn their keep in big teams/DDD codebases; for us they'd be a third name for every query. **Our decision: Mongoose models ARE the data layer, wrapped by one thin guard — `scopedModel(Model, vendorId)` — which force-injects the tenant filter** (docs/02 §4). That's the 20% of the repository pattern that pays (tenant safety, mockability) without the boilerplate. Revisit-when: a second data store appears, or query logic starts duplicating across services.
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
| B4 | **DB layer**: Mongoose connection with retry, `migrate-mongo` set up (indexes ONLY via migrations, autoIndex off in prod), seed script (2 cafes, menus, orders in every state) | `npm run seed:dev` gives a demoable database from zero |
| B5 | **Middleware chain** wired in canonical order + `/healthz` `/readyz` | A request travels the full chain; readyz flips when Mongo/Redis drop |
| B6 | **Auth module** (it blocks everything else): OTP request/verify (Redis TTL, attempt caps), email+password login, refresh rotation + family revocation, `authorize()`, tenantContext | docs/05 §1 cases tested — expired/garbage tokens, OTP abuse, refresh reuse |
| B7 | **Domain modules in dependency order**: vendors → catalog → public storefront reads → **orders** (transactions + guarded stock + idempotency — the hard one) → reviews → customers/segments → campaigns → admin. Each = the vertical-slice DoD from docs/09 §P4 | Per-module: docs/05 edge cases green + tenant-leak test added |
| B8 | **Jobs**: BullMQ queues + the separate worker process, repeatable crons (birthday 08:00 IST, segments 03:00, win-back 03:30, order auto-expiry), dedupe keys, DLQ | Kill the worker mid-job → redelivery causes no double effect |
| B9 | **Real-time**: Socket.IO namespaces, JWT handshake, room-join ownership checks, emit-after-commit, 30s polling reconciliation on clients | Vendor board updates <2s; socket death degrades to polling, invisibly |
| B10 | **Integrations behind interfaces**: `NotificationService` → WhatsApp BSP adapter (wholesale rail first), webhook endpoint (signature-verified, 200-fast, process-async), payment interface stubbed for v1.1 | Swapping BSP = one adapter file; webhook replay is idempotent |
| B11 | **Test hardening**: unit (services), integration (Supertest + mongodb-memory-server replica-set mode for transactions), the generated **tenant-leak matrix**, load smoke (50 concurrent orders on `limited` stock) | docs/06 §10 release gates green |
| B12 | **Observability**: Sentry, log-based alerts (5xx spike, DLQ growth, webhook signature failures), uptime monitor | An induced staging error reaches the team group chat |
| B13 | **Security pass**: run docs/06 top to bottom — mongo-sanitize, regex escaping, upload magic-byte checks, CORS allowlist, secrets scan in CI, dependency audit | Checklist signed off by the founder who didn't write the code |
| B14 | **Performance sanity** (only now): verify every hot query hits an index (`explain()`), projection whitelists on public reads, THEN add caching only where measurement demands (menu reads are the likely first candidate — cache-aside with 60s TTL, invalidate on catalog write) | p95 latency known for the 5 hottest endpoints |
| B15 | **Deploy pipeline**: Dockerfiles (non-root), compose (api + worker + redis + caddy), Atlas prod cluster (IP allowlist, least-privilege users), CI: test → build → migrate → deploy with graceful shutdown (drain in-flight ≤10s), rehearsed rollback | Deploy is one merged PR; rollback is one command; both rehearsed |

**Scaling path when the day comes** (in order, each step only when metrics demand): bigger VPS → API and worker on separate machines → 2+ API instances behind Caddy with the Socket.IO Redis adapter → read-heavy endpoints cached → Atlas tier up. Sharding, Kubernetes, and microservices are not on this list for years, if ever.

---

## Concepts refresher (the shared vocabulary)

- **Middleware** — a function in the request pipeline (`(req,res,next)`): sees every request before the controller. Auth, rate limiting, validation, error handling are all middleware. Order matters.
- **Controller vs Service** — controller speaks HTTP (parse, status codes); service speaks business ("place order", "compute segment"). The test: a service must be callable from a cron job or test with no `req`/`res` anywhere.
- **Repository** — an abstraction over data access so business logic doesn't know the database. We use the thin version: `scopedModel` (tenant-guard) over raw Mongoose.
- **Modular monolith** — one deployable, hard internal module boundaries. Microservice discipline without the network.
- **DTO / schema** — the declared shape of data crossing a boundary. Ours are zod schemas: one declaration = runtime validation + TS type.
- **Transaction** — several DB writes that succeed or fail as one (order insert + stock decrement). Mongo needs a replica set for this.
- **Guarded/atomic update** — write with the precondition inside the query (`{status:'placed'} → set 'accepted'`); `modifiedCount 0` means you lost the race. The cure for read-then-write bugs.
- **Idempotency key** — client-generated UUID on a POST so a retry returns the original result instead of double-charging/double-ordering.
- **Queue / worker / DLQ** — producer enqueues a job (Redis via BullMQ), a separate worker process executes with retries + exponential backoff; permanently failing jobs land in a dead-letter queue for humans.
- **Cron (repeatable job)** — scheduled work (birthday scan). Must be idempotent: running twice sends nothing twice (dedupe keys).
- **Domain event** — "order.completed" emitted by one module, consumed by others. Decouples side effects from the command that caused them.
- **Cache-aside** — read cache → miss → read DB → write cache with TTL; invalidate on write. The only caching pattern we'll use, and only after measuring.
- **JWT vs session** — session = server remembers you (state on server); JWT = signed claim you carry (state in token). We use short JWT + DB-backed refresh = mostly stateless with a revocation lever.
- **RBAC** — role-based access control: what a `role` may do. Always paired with object-level ownership checks.
- **Tenant isolation** — every vendor sees only their rows; enforced by JWT-derived `vendorId` in every query, proven by the leak-matrix tests.
- **Rate limiting** — per-key request caps (Redis counters): brute-force and cost-attack defense. Fails closed on auth routes, open elsewhere.
- **Migration** — versioned, ordered DB change scripts (indexes, backfills) run by CI, never by hand.
- **Graceful shutdown** — on deploy: stop accepting, finish in-flight work, close connections, exit. Why deploys don't drop orders.
- **Horizontal vs vertical scaling** — more machines vs bigger machine. Vertical first; horizontal needs the Redis socket adapter + statelessness we've already designed for.

---

## Decision record (revisit only with evidence)

| Decision | Choice | Revisit when |
|---|---|---|
| Architecture | Modular monolith, worker split from API | A module needs independent scale/team |
| Stack | Node/TS/Express/Mongoose/Zod/BullMQ | Never mid-build; v2 with data |
| Database | MongoDB Atlas (replica set) | Relational reporting pain dominates (then add a read store, don't migrate) |
| Redis | Queues/OTP/rate-limit/cache — ephemeral only | Never store durable state |
| API | Versioned REST + envelope + zod | Third-party API program (add OpenAPI then) |
| Auth | OTP customers, password vendors, JWT15m + rotating refresh | 2FA for vendors post-pilot |
| Data access | Mongoose + `scopedModel` guard (no full repositories) | Second data store or duplicated query logic |
| Real-time | Socket.IO + polling reconciliation | >2 instances → add Redis adapter |
| Files | Presigned S3-compatible uploads | — |
| Deploy | Docker Compose on VPS + Caddy + Atlas | >~50 vendors or >1 instance needed |
