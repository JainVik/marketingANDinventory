# Docs changelog

## 9 Sep 2026 — identity separation + browse tracking (items 67, 68)

Two decisions, taken before any code exists.

**1. Identity is separated from transactions — inside one Postgres, not across two.** The team weighed two physical databases against a locked-down schema and chose the schema: the same protection (a stolen `regulars_app` credential cannot read a phone number, because Postgres refuses the grant) without losing foreign keys, single transactions, one-query joins or a single PITR timeline. The physical split remains available later — schema `pii` is exactly the seam to cut along.

**2. The customer signs in once, and the vendor meets them on first order.** `pii.customers` is global (one row per human, every outlet), a device holds a 180-day rotating session, and `customer_profiles.revealed_at` — written in the same transaction as that customer's first `order_placed` — is what turns a count into a person for that vendor.

**3. Browsing is recorded, not just buying.** Anonymous visitor id at scan, stitched to the person at OTP; a summary row per visit plus a capped detail trail; deliberately outside the outbox so it can never fail an order.

| File | Action | What changed |
|---|---|---|
| `docs/01-product-scope.md` | **edit** | New items **67** (one identity everywhere, revealed on first order) and **68** (browse & intent tracking); item 57 now "viewed-but-never-ordered"; item 62 adds PII isolation; new "Who owns what" paragraph in §1; abandoned-cart nudge noted as buildable on 68; §6 records the 9 Sep decisions and adds the DPDP notice question |
| `docs/02-architecture.md` | **edit** | New **§4b Identity isolation** (the second wall); stack rows for the `pii` schema and the 180-day customer refresh; monorepo adds `modules/identity/` (owns the second pool) and `modules/telemetry/` |
| `docs/03-database-schema.md` | **edit** | Schema `pii` added; `customers` + `customer_consent_events` moved into it; new `pii.customer_devices`, `pii.customer_refresh_tokens`, `app.visitors`, `app.browse_sessions` ⧉, `app.browse_events` ⧉ (65 → **70 tables**); `customer_profiles` loses `phone`/`name`, gains `phone_hash` + `display_name` + `revealed_at`; `phone_hash` generated column on `pii.customers`; new role `regulars_identity`; `pii.customer_public` view + `pii.reveal_identity()`; column-level grants excluding `phone`/`raw` on `conversations`/`messages`; new enums `browse_outcome`, `browse_event_kind`; 4 new event types; new **§3.4** telemetry beacon contract; partition/retention/volume rows; §4.6 lake PII allowlist; §4.7 reveal + purge + erasure paths; Appendix B rows 67–68 |
| `docs/04-api-contract.md` | **edit** | New error code `CUSTOMER_NOT_REVEALED`; `/auth/customer/resume` + `/me/devices*`; storefront issues `visitorId`/`browseSessionId` and gains the telemetry beacon; new **§11b** identity reveal + browse insight endpoints with their tool keys |
| `docs/05-edge-cases-and-failures.md` | **edit** | New **§12** (22 cases, 18 marked 🧪): one identity everywhere, device/token reuse, shared phones, `pii` unreachable, reveal-on-first-order, imports, erasure while revealed, telemetry failure isolation, beacon replay, mid-visit OTP stitching, practice traffic, aggregate-only owner view, volume caps, visitor retention |
| `docs/06-security-checklist.md` | **edit** | New **§4b PII wall** as a release gate 🚦 (zero `regulars_app` privileges in `pii`, enumerated doors, reveal-requires-order tested per vendor, reveal auditing, lake publication check); §7 now four Postgres roles; §8 adds identity separation, the reveal promise, and browse-tracking notice/retention with a lawyer flag; §10 gains gates 7 and 8 |
| `CLAUDE.md` | **edit** | New binding rules **5b** (the PII wall — never grant `regulars_app` on `pii`, match on `phone_hash`, reveal only through the function), **5c** (reveal is transactional) and **9b** (browse telemetry is not the outbox and may never fail an order); DoD adds the PII-wall gate |
| `docs/16-build-playbook.md` | **edit** | S0 builds the `pii` schema + gate; S1 becomes `auth` + `identity` (items 1, 67) with a "do not" list; S4 adds `telemetry` (item 68); S8 adds reveal-on-first-order; S11 adds the funnel and viewed-never-ordered |
| `docs/00-coverage.md` | **edit** | Rows 67 and 68 added; 30, 57, 62 amended; six new foundation rows (pii schema + gate, reveal function + audit, column grants, browse partitions/retention/sweeper, lake publication check) |
| `README.md` | **edit** | Table count, locked decisions, the new smoke file |
| `06-pii-wall-smoke.sql` | **new** | Runnable proof of the §4b wall: zero `regulars_app` privileges on `pii`, direct read denied, reveal refused before the first order and granted after, a second vendor still refused, reveal audited, `phone`/`raw` column-revoked on the WhatsApp ledger, `regulars_ml` locked out, plus browse-telemetry IST stamping, append-only and replay safety |

**Verified, not just written.** The Step 2 DDL was extracted and run against a real PostgreSQL 16 (pgvector stubbed, unavailable locally), then the smoke test above was run against the result. Five defects were caught and fixed before this entry was written:

1. `phone_hash` used `sha256(convert_to(...))` — `convert_to` is STABLE, and a generated column requires IMMUTABLE. Postgres rejected the column and 380 downstream statements failed with it. Now `encode(public.digest(phone::text,'sha256'),'hex')`.
2. `cp_phone_ix` and `cp_name_trgm` still indexed the `phone` and `name` columns removed from `customer_profiles`. Now `phone_hash` and `display_name`.
3. The browse `DEFAULT` partitions and `ensure_month_partitions` calls sat in §2.9, before the tables were created in §2.10. Moved.
4. `GRANT USAGE ON SCHEMA pii TO regulars_app` was missing, and the column-level `REVOKE` on `conversations`/`messages` sat **before** the blanket `GRANT … ON ALL TABLES IN SCHEMA app`, which silently re-granted the phone column. The whole PII-grant block now runs last, and the wall was re-tested.
5. `browse_events` omitted `weekday`, which the shared `set_local_time()` trigger sets — every insert failed. And its dedupe index could not stop a beacon replay while `occurred_at` defaulted to `now()`: the default is removed, the ingest must copy the client's `at`, and a replay now collides with 23505 as intended.

**Still open after this change:** DPDP notice wording for tracking a visitor before they identify themselves (lawyer, `docs/01` §6); whether the funnel tile gets its own screen frame in `docs/14` or lives on O01/O34.

---

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
| `docs/14-screen-specification.md` | **new** | 112 screens (97 v1 + 15 reserved), 88 overlays, nav architecture, design-system inputs; companion `regulars-screen-inventory.xlsx` (not in repo) |
| `docs/15-system-architecture-blueprint.md` + `15-system-design.png/.svg` | **new** | Monorepo (Turborepo), api + worker processes, 10 BullMQ queues by fault domain, outbox, adapters/ports, Docker Compose → Fly/ECS, 2-hour extraction recipe, directory tree |
| `db/migrations/0001_init.sql` | **new** | The schema (65 tables, 65 enums, RLS, partitions, triggers). Includes the funnel event types from doc 13 |
| `db/seed/smoke.sql` | **new** | Proves RLS isolation, idempotency, guarded transitions, counters, partitions, append-only |
| `docs/00-coverage.md` | **new** (6 Sep) | 66 rows: item → module → session → screens → docs/05 section → status; foundations checklist; misses log |
| `docs/16-build-playbook.md` | **new** (6 Sep) | Session sequence S0–S14, session cards with doc refs + prompts, folder/naming rules, no-duplicate rules, miss handling, checkpoints, git |
| `README.md`, `CLAUDE.md` | **edit** (6 Sep) | Point at 00 and 16 |
| `docs/07`, `08`, `12`, `Regulars — v1 Final Cut.txt`, `v1-dataMaking.txt` | untouched | |

Decisions folded 5 Sep (late): refund default = original payment mode (01 #45, 05 §4.9); move table confirmed v1, merge v1.1 (01 §4, §6).

Numbering note: `13-…` because `11` and `12` already exist. `HANDOVER.md` is now folded in; keep it only as history.
