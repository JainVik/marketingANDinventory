# CLAUDE.md — Engineering Rules (binding for all AI-generated code)

You are writing production code for **Regulars**: a multi-tenant QR ordering + WhatsApp retention platform for independent Indian cafes and restaurants. Before implementing ANY feature, read the relevant sections of `docs/`:

| Doc | Read when |
|---|---|
| `docs/01-product-scope.md` | Always — defines what exists in v1 and what does NOT (the numbered final cut) |
| `docs/02-architecture.md` | Creating any file — layout, layers, stack are fixed |
| `docs/03-database-schema.md` | Touching any table or query. The SQL in `db/migrations/` is the truth; `db/schema.ts` is generated from it |
| `docs/04-api-contract.md` | Adding/changing any endpoint |
| `docs/05-edge-cases-and-failures.md` | **Before implementing a module — its section is a requirements list** |
| `docs/06-security-checklist.md` | Auth, input handling, anything vendor-scoped |
| `docs/13-website-and-onboarding-funnel.md` | Public site, signup, wizard |
| `docs/14-screen-specification.md` | Building any screen — IDs, layouts, overlays, states |
| `docs/15-system-architecture-blueprint.md` | Adding a process, queue, adapter or module; anything about fault isolation or deployment |
| `docs/16-build-playbook.md` | Start of every session — which module, which docs, the folder/naming rules; update `docs/00-coverage.md` at the end |

If a request conflicts with these docs, STOP and say so instead of silently diverging. If something is ambiguous, choose the option consistent with these docs and note the assumption in the PR description.

## Hard rules (violations are review-blocking)

1. **TypeScript strict everywhere.** No `any` (use `unknown` + narrowing), no `@ts-ignore` (use `@ts-expect-error` with a reason comment, sparingly), no non-null `!` on values that can actually be null.
2. **Layering:** routes → controller → service → db. Controllers contain zero business logic and never import `db`. Services never import Express types. Cross-module calls are service→service only.
3. **DRY with judgment:** shared logic lives in `packages/shared` (cross-app) or `apps/api/src/shared` (api-wide). Search before writing a helper. Do NOT abstract two merely-similar things into one config-flag function — three real occurrences earn an abstraction.
4. **Single sources of truth:** order-state machine, roles, error codes, zod schemas, money/phone/date utils exist ONCE in `packages/shared`. Enums come from the Postgres enums via `drizzle-zod`. Redeclaring any of them locally is a defect.
5. **Multi-tenancy:** on `/vendor/*` code paths, the tenant comes only from `req.tenant` (JWT-derived) and every query runs inside `withTenant(req.tenant, tx => …)`. Services receive `tx`; they never import `db`. Row-Level Security in Postgres is the wall; the app never appends `WHERE vendor_id` itself. Workers use `asWorker()` and pass `vendor_id` explicitly — allowed only under `jobs/`.
6. **Money is `app.paise` (bigint integer paise). Timestamps are `timestamptz` UTC.** IST `local_date` / `day_part` are set by DB triggers, never computed in app code. Formatting only at the display edge via shared utils.
7. **No read-then-write for contended state.** Stock, order status, counters, coupon uses: guarded `UPDATE … WHERE <expected state>` and check `rowCount`; sequences via `ops.next_seq()`; row locks (`FOR UPDATE`) on the session row during placement — as specified in `docs/03` §2b and `docs/05`. If you find yourself writing `select` → `if` → `update` on shared mutable state, stop and use the guarded pattern.
8. **Every state change is a timestamp column** (`accepted_at`, `ready_at`, `settled_at`…); `status` is derived and changed only in the same guarded UPDATE. Never mutate history: `order_lines`, `bills`, `payments`, `events`, `audit_logs` are append-only (DB triggers enforce it — do not work around them).
9. **Every business moment writes an `app.events` row inside the same transaction** (the outbox). Event types and payload shapes are in `docs/03` Step 3.3. Side effects (WhatsApp, features, automations) consume events; request handlers never call Meta or the gateway directly.
10. **Every owner action is a `tools` registry entry** (`key`, input schema from the route's zod, `side_effect`, `requires_confirmation`) and one service function. UI, owner bot, automations and API all call the same function. Adding an endpoint without its `tools` row is a defect.
11. **Errors:** services throw `AppError(code, httpStatus, message, details?)` with codes from the shared enum. Controllers are wrapped in `asyncHandler`; the central error middleware is the only place that shapes error responses. Postgres `23505` (unique) on an idempotency key → replay, never a 500.
12. **Validation at the boundary:** every route has a zod schema for body/params/query via `validate()`. Inside services, trust the parsed types.
13. **Every endpoint ships with tests in the same PR:** happy path + its cases from `docs/05` (marked 🧪) + a tenant-isolation case if vendor-scoped. Vitest + Supertest against a real Postgres (testcontainers or the local docker-compose DB), migrations applied, RLS on.
14. **No new dependencies without justification.** Prefer stdlib/existing deps. Anything with postinstall scripts, < 6 months of maintenance, or overlapping an existing dep needs explicit sign-off in the PR description.
15. **Secrets & config:** only via `config/env.ts`. `process.env` anywhere else is a defect. Gateway/WABA credentials are vault references (`*_ref` columns), never plaintext in the DB. Update `.env.example` in the same PR that adds a variable.
16. **Naming:** files `kebab-case`, classes/types `PascalCase`, functions/vars `camelCase`, constants `SCREAMING_SNAKE`, module files `orders.service.ts` style. API JSON is `camelCase`. DB tables/columns `snake_case` (Drizzle maps them).
17. **Functions small and single-purpose.** Max nesting depth 3 — early returns. A function that needs a comment to separate its phases wants to be two functions.
18. **Comments & JSDoc:** JSDoc on every exported service function (what + why, not how) and on any non-obvious business rule, referencing the doc section (e.g., `// docs/05 §2.5 PRICE_CHANGED flow`). No commented-out code, no TODO without an issue link.
19. **Frontend:** components presentational; data fetching only in TanStack Query hooks under `features/*/hooks`; no `axios` in components; no `dangerouslySetInnerHTML`; all strings through i18n; loading/error/empty states are part of "done" for every screen. Owner side is desktop-first; customer side is a mobile PWA.
20. **Idempotency & retries are not optional** where the docs specify them (order placement, payments, campaign send, all job handlers, webhooks).
21. **Practice mode is real data with `is_practice = true`** — never a separate code path. Reports, briefings, campaigns and the ML export filter it out.
22. **Never weaken a security control to make a test or type check pass.**

## Definition of done (per PR)

- [ ] Types compile, ESLint clean (`no-restricted-syntax` rules included), Prettier applied
- [ ] Tests written and green (incl. relevant `docs/05` 🧪 cases)
- [ ] New table → migration file added, `drizzle-kit pull` re-run, tenant-leak matrix regenerated (`docs/06 §4`)
- [ ] New owner action → `tools` seed entry + `events` type if it is a business moment
- [ ] `.env.example` / seed script updated if needed
- [ ] No duplicated logic introduced (searched shared/ first)
- [ ] PR description: what, why, assumptions made, doc sections implemented

## Workflow expectations for AI sessions

- Work module-by-module in the order the human directs; do not scaffold future modules "while you're at it". v1.1 / v1.5 / v2 items in `docs/01` §4 are **not** to be built, only their columns exist.
- Small, reviewable increments — one endpoint or one flow per commit, conventional commit messages (`feat(orders): guarded status transitions`).
- After implementing, self-review against this file and the module's `docs/05` section, and list which edge cases you handled and which remain.
- When you touch a file, leave it fully consistent with these rules even if surrounding code predates them — but do not refactor unrelated files in the same PR.
