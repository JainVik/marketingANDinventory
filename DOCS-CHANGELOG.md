# Docs changelog — 5 Sep 2026 ("make the docs")

Folds HANDOVER.md, the v1 Final Cut (items 1–66) and the PostgreSQL decision into the doc pack. Paste each file over the one in the repo; new files are marked.

| File | Action | What changed |
|---|---|---|
| `CLAUDE.md` | **replace** | Rules 5, 7 rewritten for `withTenant()` + RLS + guarded UPDATEs; new rules: state = timestamps (8), outbox event in same txn (9), every owner action is a `tools` entry (10), practice mode is `is_practice` (21); tests against real Postgres (13); DoD adds migration + `drizzle-kit pull` + tools seed |
| `README.md` | **replace** | Product name Regulars; pack layout adds `db/`, `03-schema-atlas.html`, `13-…`; DB-first setup commands; new build order; locked decisions updated (Postgres, pay after acceptance, merchant of record) |
| `docs/01-product-scope.md` | **replace** | Now the numbered final cut 1–66 (the numbers are referenced from 04/05); roles (single owner login, bot); free/paid split; business model; version ledger v1.1/v1.5/v2; dropped list; open items |
| `docs/02-architecture.md` | **replace** | Stack → Postgres 16 on Supabase + Drizzle; monorepo adds `db/`, `worker`, `site`, new modules (sessions, payments, marketing, insights, whatsapp, agents, events); module pattern adds `*.tools.ts` + `*.events.ts`; §4 tenancy = RLS + `withTenant`; new §7 payments, §8 pipeline with 24 h window/crons, §9 AI layer |
| `docs/03-database-schema.md` | **replace** | Entire Postgres blueprint: conventions, table map, ERD, 11 migration sections verbatim, Drizzle layer, JSON payload standards, index/partition/vector/CDC/DPDP plan, coverage map 1–66 |
| `docs/03-schema-atlas.html` | **new** | Interactive ERD (65 tables, every column), generated from the live catalog |
| `docs/04-api-contract.md` | **replace** | UUIDs + cursor pagination; new error codes; signup/wizard, QR mode, auto-accept, cancellation request, coupons, payments + refunds (3 modes), settle with multiple modes, takeaway, imports, segments, dead hours, automations, insights, briefings, owner bot, admin tools/agents; every owner route carries its `tools` key; webhooks for Razorpay/Cashfree; new socket events |
| `docs/05-edge-cases-and-failures.md` | **replace** | All cases restated in Postgres terms (row lock, guarded UPDATE, 23505 replay); new sections/cases: auto-accept, cancellation request vs accept, pay-after-accept refunds, multi-mode bills, gateway webhook ordering, own-items vs whole-table, consent/STOP/imports, holdout + attribution, dead-hour cap, owner bot guards, pooler discipline, partitions |
| `docs/06-security-checklist.md` | **edit** | §1 NoSQL → SQL injection (`sql.raw` audit, DSL whitelist); §3 roles renamed; §4 RLS as the wall, pooler discipline, global customers policy; §5 gateway signatures, vault refs, bot tool guard; §7 Postgres roles; §8 purpose-level consent, erasure path, lake PII allowlist, Mumbai region; §9 audit_logs partitioned; §10 gates add `sql.raw` audit, invoice-number concurrency, `smoke.sql` |
| `docs/09-execution-roadmap.md` | **edit** | Two environment lines: docker-compose postgres, Supabase PITR restore rehearsal |
| `docs/10-backend-technical-roadmap.md` | **edit** | A2 stack + the MongoDB reversal rationale; A3 data stores; A6 data access = `withTenant(tx)` + RLS; B4/B7/B8/B10/B11/B13/B14/B15 rows; scaling path; glossary (transaction, guarded update, repository, RLS, outbox, partition, migration); decision record |
| `docs/11-v1-master-feature-specification.md` | **edit** | Title → Regulars with "01 wins where they differ"; §1.1 DB and tenancy bullets |
| `docs/13-website-and-onboarding-funnel.md` | **new** | 10 pages with CTAs, mega-menu, three doors → one wizard (10 steps, what each writes), go-live checklist, first-14-days sequence, funnel events |
| `db/migrations/0001_init.sql` | **new** | The schema (65 tables, 65 enums, RLS, partitions, triggers). Includes the funnel event types from doc 13 |
| `db/seed/smoke.sql` | **new** | Proves RLS isolation, idempotency, guarded transitions, counters, partitions, append-only |
| `docs/07`, `08`, `12`, `Regulars — v1 Final Cut.txt`, `v1-dataMaking.txt` | untouched | |

Numbering note: `13-…` because `11` and `12` already exist. `HANDOVER.md` is now folded in; keep it only as history.
