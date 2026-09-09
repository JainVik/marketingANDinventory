# Regulars — AI Build Documentation Pack

Feeding documents for building Regulars with AI coding tools (Claude Code, Copilot, Cursor). The docs are **binding specs**, not inspiration: the AI reads them, then codes against them. Current as of the v1 Final Cut (5 Sep 2026) and the PostgreSQL decision.

## What's in the pack

```
CLAUDE.md                          ← REPO ROOT — Claude Code auto-reads it every session
db/
├── migrations/0001_init.sql       ← the database (70 tables across `pii`/`app`/`ml`/`ops`, RLS, the PII wall, partitions, triggers) — run this   [feed to AI]
└── schema.ts                      ← GENERATED: `drizzle-kit pull` after migrating; do not hand-edit
docs/
├── 00-coverage.md                 ← the 68-row checklist: item → module → session → status (update every session)
├── 01-product-scope.md            ← WHAT v1 is: the numbered final cut 1–68, version ledger, dropped     [feed to AI]
├── 02-architecture.md             ← stack (Postgres/Supabase/Drizzle), monorepo, layers, RLS, rails      [feed to AI]
├── 03-database-schema.md          ← every table, index, policy, payload schema, scaling plan             [feed to AI]
├── 03-schema-atlas.html           ← the ERD (open in a browser; full map + one sheet per domain)
├── 04-api-contract.md             ← endpoints with tool keys, envelope, error codes, sockets             [feed to AI]
├── 05-edge-cases-and-failures.md  ← the failure catalog — requirements with tests                        [feed to AI]
├── 06-security-checklist.md       ← injection, RBAC, RLS leak gate, PII wall, DPDP, releases                       [feed to AI]
├── 06-pii-wall-smoke.sql          ← runnable proof of the §4b PII wall — run in CI after migrations
├── 07-market-research.md          ← India POS/WhatsApp/AI market — FOUNDERS
├── 08-owner-com-teardown.md       ← Owner.com teardown — FOUNDERS
├── 09-execution-roadmap.md        ← day one → production — FOUNDERS
├── 10-backend-technical-roadmap.md← backend decisions + build order — BOTH
├── 11-v1-master-feature-specification.md ← long-form feature spec (01 wins where they differ)             [feed to AI]
├── 12-ux-wireframe-flow-map.md    ← screen flows & Mermaid maps                                          [feed to AI]
├── 13-website-and-onboarding-funnel.md ← 10-page site, three signup doors, the wizard, go-live          [feed to AI]
├── 14-screen-specification.md     ← every screen, overlay and state; Figma planning                       [feed to AI]
├── 15-system-architecture-blueprint.md ← repo, processes, queues, fault domains, deploy, extraction        [feed to AI]
├── 15-system-design.png / .svg    ← the system diagram
└── 16-build-playbook.md           ← HOW we build: session order S0–S14, per-session prompts, folder/naming rules, misses [feed to AI]
```

Docs 01–06, 11–15 + CLAUDE.md + `db/migrations` are the AI's binding contract. 07–09 are founder references (don't paste them into coding sessions). Doc 10 bridges both. Where 01 and 11 disagree, **01 (the final cut) wins**.

## How to use with Claude Code

1. Create the repo, copy `CLAUDE.md` to the root, `docs/` and `db/` alongside it, commit.
2. Database first:
   ```
   psql "$DATABASE_DIRECT_URL" -f db/migrations/0001_init.sql
   npx drizzle-kit pull            # generates db/schema.ts + relations.ts
   psql "$DATABASE_DIRECT_URL" -f db/seed/smoke.sql   # optional: proves RLS, counters, partitions
   ```
3. Scaffold: > "Read CLAUDE.md and docs/02. Scaffold the monorepo exactly as specified — workspaces, apps/api bootstrap (loaders, middlewares, error handler, env config, db/client.ts with withTenant/asWorker), packages/shared with enums from db/schema.ts, error codes, zod schemas. No feature modules yet."
4. Then **one module per session**, in the order and with the prompts in `docs/16-build-playbook.md` §2–3. Example: > "Implement the orders module. Requirements: docs/01 §2 items 23–39, docs/03 §2b, docs/04 §5 + §9, docs/05 §2–3, docs/06 §4. Include the 🧪 tests and the tools registry entries."
5. Build order (each step shippable): auth + identity → vendors + wizard → menu → tables + telemetry → sessions + orders (the hard one) → billing + payments → customers + consent → WhatsApp pipeline (queue, worker, webhook, conversations) → marketing (automations, campaigns, holdout, ledger) → insights + briefing → agents (tools registry, owner bot read-only) → admin → PWA polish → public site.
6. After each module, run the self-review step at the bottom of CLAUDE.md, the test suite, and update `docs/00-coverage.md` (playbook §7) before moving on.

## Keeping the docs honest

- These docs are the contract. When a real decision changes, **update the doc in the same PR** — stale specs are worse than none. `DOCS-CHANGELOG.md` lists what changed and when.
- Anything marked 🚦 in doc 06 is a launch blocker. Anything marked 🧪 must exist as an automated test.
- Decisions locked: PostgreSQL 16 on Supabase + Drizzle (SQL migrations are truth), RLS tenant isolation, **customer identity in schema `pii` behind its own role — one database, not two** (9 Sep), one global customer login revealed to a vendor on first order, customer side as PWA (QR-first), pay after acceptance for tables / pay first for counter, restaurant is merchant of record (Razorpay/Cashfree adapters), platform WABA in v1, flat subscription, single city, one owner login in v1.

## Open items for the founders (not for the AI)

- Meta India service-message rate card (1 Oct 2026 change) before pricing the free tier.
- Lawyer: gateway / merchant-of-record structure; DPDP notice text. CA: e-invoice position.
- Offers-banner edge cases.
- Sit in 10 restaurants and watch group ordering.
- Pilot outlets (their menus become seed data).
