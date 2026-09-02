# CLAUDE.md — Engineering Rules (binding for all AI-generated code)

You are writing production code for a multi-tenant restaurant ordering + retention platform. Before implementing ANY feature, read the relevant sections of `docs/`:

| Doc | Read when |
|---|---|
| `docs/01-product-scope.md` | Always — defines what exists and what does NOT |
| `docs/02-architecture.md` | Creating any file — layout, layers, stack are fixed |
| `docs/03-database-schema.md` | Touching any model or query |
| `docs/04-api-contract.md` | Adding/changing any endpoint |
| `docs/05-edge-cases-and-failures.md` | **Before implementing a module — its section is a requirements list** |
| `docs/06-security-checklist.md` | Auth, input handling, anything vendor-scoped |

If a request conflicts with these docs, STOP and say so instead of silently diverging. If something is ambiguous, choose the option consistent with these docs and note the assumption in the PR description.

## Hard rules (violations are review-blocking)

1. **TypeScript strict everywhere.** No `any` (use `unknown` + narrowing), no `@ts-ignore` (use `@ts-expect-error` with a reason comment, sparingly), no non-null `!` on values that can actually be null.
2. **Layering:** routes → controller → service → model. Controllers contain zero business logic and never import models. Services never import Express types. Cross-module calls are service→service only.
3. **DRY with judgment:** shared logic lives in `packages/shared` (cross-app) or `apps/api/src/shared` (api-wide). Before writing a helper, search for an existing one. But do NOT abstract two merely-similar things into one config-flag function — duplication is cheaper than the wrong abstraction; three real occurrences earn an abstraction.
4. **Single sources of truth:** order-state machine, roles, error codes, zod schemas, money/phone/date utils exist ONCE in `packages/shared`. Redeclaring any of them locally is a defect.
5. **Multi-tenancy:** on `/vendor/*` code paths, `vendorId` comes only from `req.tenant` (JWT-derived). Data access via `scopedModel()`. Never read `vendorId` from body/params/query in vendor modules.
6. **Money is integer paise. Timestamps are UTC.** Formatting only at the display edge via shared utils.
7. **No read-then-write for contended state.** Stock, order status, counters: guarded atomic updates / transactions as specified in `docs/03 §10`. If you find yourself writing `find` → `if` → `save` on shared mutable state, stop and use the guarded pattern.
8. **Errors:** services throw `AppError(code, httpStatus, message, details?)` with codes from the shared enum. Controllers are wrapped in `asyncHandler`; the central error middleware is the only place that shapes error responses. No `try/catch` that swallows an error or `console.log`s it — pino logger only, always with context.
9. **Validation at the boundary:** every route has a zod schema for body/params/query, via `validate()` middleware. Inside services, trust the parsed types — do not re-validate ad hoc.
10. **Every endpoint ships with tests in the same PR:** happy path + its cases from `docs/05` (marked 🧪) + a tenant-isolation case if vendor-scoped. Vitest + Supertest + mongodb-memory-server (replica-set mode for transaction paths).
11. **No new dependencies without justification.** Prefer stdlib/existing deps. Anything with postinstall scripts, < 6 months of maintenance, or overlapping an existing dep needs explicit sign-off in the PR description.
12. **Secrets & config:** only via `config/env.ts`. `process.env` anywhere else is a defect. Update `.env.example` in the same PR that adds a variable.
13. **Naming:** files `kebab-case`, classes/types `PascalCase`, functions/vars `camelCase`, constants `SCREAMING_SNAKE`, module files `orders.service.ts` style. API JSON is `camelCase`. DB fields `camelCase`.
14. **Functions small and single-purpose.** Max nesting depth 3 — use early returns. A function that needs a comment to separate its phases wants to be two functions.
15. **Comments & JSDoc:** JSDoc on every exported service function (what + why, not how) and on any non-obvious business rule, referencing the doc section (e.g., `// docs/05 §2.4 PRICE_CHANGED flow`). No commented-out code, no TODO without an issue link.
16. **Frontend:** components presentational; data fetching only in TanStack Query hooks under `features/*/hooks`; no `axios` in components; no `dangerouslySetInnerHTML`; all strings through i18n; loading/error/empty states are part of "done" for every screen.
17. **Idempotency & retries are not optional** where the docs specify them (order placement, campaign send, all job handlers, webhooks).
18. **Never weaken a security control to make a test or type check pass.**

## Definition of done (per PR)

- [ ] Types compile, ESLint clean (`no-restricted-syntax` rules included), Prettier applied
- [ ] Tests written and green (incl. relevant `docs/05` 🧪 cases)
- [ ] Tenant-leak matrix updated if a new vendor resource exists (`docs/06 §4`)
- [ ] `.env.example` / seed script / migration updated if needed
- [ ] No duplicated logic introduced (searched shared/ first)
- [ ] PR description: what, why, assumptions made, doc sections implemented

## Workflow expectations for AI sessions

- Work module-by-module in the order the human directs; do not scaffold future modules "while you're at it".
- Small, reviewable increments — one endpoint or one flow per commit, conventional commit messages (`feat(orders): guarded status transitions`).
- After implementing, self-review against this file and the module's `docs/05` section, and list which edge cases you handled and which remain.
- When you touch a file, leave it fully consistent with these rules even if surrounding code predates them — but do not refactor unrelated files in the same PR.
