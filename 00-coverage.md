# 00 — Coverage tracker (v1 Final Cut items 1–68 → module → session)

Update this file after **every** Claude Code session. Status values: `todo` · `partial (what is missing)` · `done` · `moved v1.1`. At the end of session 14 every row must be `done` or `moved`. Rules for using it are in `docs/16-build-playbook.md` §6.

Session numbers are from the playbook (S0 scaffold … S14 polish + site). "Also" = a second session finishes the item. Screen IDs are from `docs/14`. `05 §` = the edge-case section that is the requirements list for the item.

## Section 1 — Data-making (free)

| # | Item | Module | Session | Also | Screens | 05 § | Status |
|---|---|---|---|---|---|---|---|
| 1 | Three signup doors → one wizard | auth | S1 | S14 (site door) | A01, A02, S10 | §1 | todo |
| 2 | Outlet profile (name, address, type, hours, open/closed) | vendors | S2 | | A03.0, A03.1, O38 | §1 | todo |
| 3 | GST/FSSAI optional at signup, required before first invoice | vendors | S2 | S6 (gate) | A03.6 | §4 | todo |
| 4 | QR mode per outlet: order / pay-only / both | vendors | S2 | S5 (behaviour), C13 | A03.4, O39 | §2 | todo |
| 5 | Wizard with progress + go-live checklist | vendors | S2 | S5 (test order completes it) | A03, A03.5, O01 | §1 | todo |
| 6 | Practice mode (`is_practice`) | vendors + tables | S2 | S4 (table), S5 (filter) | O42, C18 | §2 | todo |
| 7 | Customer CSV import + opt-in ask; no marketing before opt-in | customers | S8 | S9 (opt-in message) | A03.8, O27 | §5 | todo |
| 8 | Gateway onboarding (merchant of record), skippable | payments | S7 | | A03.7, O40 | §4 | todo |
| 9 | Export everything; monthly plan; cancel anytime | vendors | S2 (plan) | S13 (export job) | O43 | §11 | todo |
| 10 | Categories, items, veg flag, photo, GST rate, bestseller | menu | S3 | | O12, O13 | §8 | todo |
| 11 | Variants + add-on groups (min/max) | menu | S3 | | O13, C02 | §2 | todo |
| 12 | Availability toggle + auto-restore | menu | S3 | S9 (restore cron) | O12 | §2 | todo |
| 13 | Bulk-edit grid | menu | S3 | | O14 | — | todo |
| 14 | Import: photo (AI) / PDF / Excel-CSV → review grid | menu | S3 (CSV/Excel + grid) | S12 (photo/PDF via OCR + LLM adapters) | A03.2, O15, O16 | §8 | todo |
| 15 | Re-photograph = diff update | menu | S12 | | O16 | §8 | todo |
| 16 | Live phone preview with warnings | menu | S3 | | O12 | — | todo |
| 17 | Offers banner | menu | S3 | ⚠ open edge cases (01 §6) | O17, C01 | — | todo |
| 18 | Table list, kind, opaque token per table | tables | S4 | | A03.3, O10 | §9 | todo |
| 19 | QR PDF sheet + reprint/regenerate | tables | S4 | | O10, C17 | §9 | todo |
| 20 | Counter QR → takeaway token, ready notification, collected | tables + orders | S4 (counter kind) | S5 (flow), S7 (pay first) | O05, C12 | §3 | todo |
| 21 | Scan → menu, table auto-attached, browse without login | tables (public storefront) | S4 | | C01 | §2 | todo |
| 22 | Item sheet, cart, instructions (cart per phone) | customer-web | S4 | | C02, C03 | §2 | todo |
| 23 | Place order → OTP → accept-gate → pay prompt after acceptance | sessions + orders | S5 | S7 (pay prompt) | C04, C05 | §2, §3 | todo |
| 24 | Counter / takeaway: pay first | orders | S5 | S7 | C12 | §3, §4 | todo |
| 25 | Live status, order more, call waiter, request bill | sessions + orders | S5 | | C06 | §2 | todo |
| 26 | My bill: rounds, per-person, pay whole / pay own items | billing | S6 | | C09 | §4 | todo |
| 27 | Cancellation request (if enabled) → owner decides | orders | S5 | | C14, O03 | §3 | todo |
| 28 | Coupon code at checkout | billing | S6 | | C07 | §4 | todo |
| 29 | First visit: e-bill on WhatsApp (consent moment); repeat: in-app | whatsapp | S9 | | C10 | §5, §6 | todo |
| 30 | Customer history across outlets + add-to-home-screen | customers + identity | S8 (history) | S14 (PWA prompt) | C15, §12.1 | §5 | todo |
| 31 | Order desk: New / Preparing / Ready + customer badge | orders | S5 | | O02, O03 | §3 | todo |
| 32 | Loud persistent alert until accepted | orders (owner-web) | S5 | | O02 | §3 | todo |
| 33 | Accept + ETA / preparing / ready / complete / reject presets | orders | S5 | | O02, O03 | §3 | todo |
| 34 | Staff order entry (table / no-table) | orders | S5 | | O04 | §3 | todo |
| 35 | Reassign (move) table, busy mode | sessions | S5 | | O07, O02 | §2 | todo |
| 36 | Owner cancel with reason; handle cancellation requests | orders | S5 | | O03 | §3 | todo |
| 37 | Table grid → table sheet, running total, settle | sessions (grid) + billing (settle) | S5 (grid) | S6 (settle) | O06, O07, O08 | §2, §4 | todo |
| 38 | Regulars Board (seated now, visits, usual order) | customers | S8 | | O11 | §5 | todo |
| 39 | Slow-order alert | orders | S5 | S9 (cron) | O02, O47 | §3 | todo |
| 40 | Pay now (gateway) / pay at checkout, default by outlet type | billing + payments | S6 | S7 | C07, O39 | §4 | todo |
| 41 | GST invoice, sequential per outlet per FY, round-off, tip | billing | S6 | | O22, C10 | §4 | todo |
| 42 | Coupons: owner-generated, limits, before payment only | billing | S6 | | O24 | §4 | todo |
| 43 | Complimentary item with reason | billing | S6 | | O07 | §4 | todo |
| 44 | Payment-mode capture; multi-mode settle | billing | S6 | | O08 | §4 | todo |
| 45 | Refunds: back-to-source / cash / adjust-replace; default = original mode | billing (cash, adjust) + payments (gateway) | S6 | S7 (gateway refund + webhook) | O09 | §4.9 | todo |
| 46 | Day-close report + cash reconciliation | billing | S6 | | O23 | §4 | todo |
| 47 | Discount and void-rate report | insights | S11 | | O20 | §4 | todo |

## Section 2 — Marketing & analysis (paid)

| # | Item | Module | Session | Also | Screens | 05 § | Status |
|---|---|---|---|---|---|---|---|
| 48 | Customer book: rule segments, block customer | customers | S8 | | O25, O26, O28 | §5 | todo |
| 49 | Identity capture rate tile | insights | S11 | | O01, O25 | — | todo |
| 50 | Lapsed wall (count + ₹ value) | insights | S11 | | O01, O25 | §5 | todo |
| 51 | Automatic triggers: birthday, win-back, thank-you | marketing | S10 | | O29 | §6 | todo |
| 52 | Manual segmented campaigns + templates | marketing | S10 | | O29, O30 | §6 | todo |
| 53 | Dead-hour filler (detect → capped send → measure) | insights (detect) + marketing (send) | S11 | S10 (send) | O32 | §6 | todo |
| 54 | Holdout on every campaign (20 %) | marketing | S10 | | O31 | §6 | todo |
| 55 | Revenue ledger | marketing | S10 | | O33 | §6 | todo |
| 56 | Morning WhatsApp briefing | insights + whatsapp | S11 | | O35, A03.9 | §7 | todo |
| 57 | Menu conclusions + viewed-but-never-ordered + never-viewed | insights | S11 | S4 (item 68 supplies the data) | O34 | §12.3 | todo |
| 58 | Market-basket upsell suggestions | insights | S11 | | O34 | — | todo |
| 59 | Kitchen speed by hour/day | insights | S11 | | O19 | — | todo |
| 60 | Smart review routing (happy → Google v1.1, unhappy → private) | reviews | S8 | | C11, O36 | §5 | todo |

## Foundations

| # | Item | Module | Session | Also | Screens | 05 § | Status |
|---|---|---|---|---|---|---|---|
| 61 | WhatsApp pipeline: queue, retries, DLQ, STOP, caps, dedupe, 24 h, 131049 | whatsapp + jobs | S9 | | O21, O41 | §6 | todo |
| 62 | Admin panel + foundations: auth, RLS, **PII wall**, sockets, events log, CI | events + admin + jobs | S0 | S13 (admin UI) | X01–X09 | §9, §11 | todo |
| 63 | Public site: 10 pages + mega-menu | site | S14 | | S01–S10 | — | todo |
| 64 | Every owner action = `tools` registry entry | agents (`runTool`) + every module's `*.tools.ts` | S0 (skeleton) | every session; S12 verifies the seed is complete | — | §7 | todo |
| 65 | Read-only owner bot on WhatsApp | agents | S12 | | O41, X05 | §7 | todo |
| 66 | Auto-accept setting, conditional | orders (logic) + vendors (setting) | S5 | S2 (setting UI) | O39 | §3 | todo |
| 67 | One identity everywhere; sign in once; reveal to vendor on first order | identity + auth (login) + customers (reveal) | S1 | S8 (reveal + book), S5a (reveal txn) | A01, C04, C15, O25, O26 | §12.1, §12.2 | todo |
| 68 | Browse & intent tracking (scan → view → scroll → no order) | telemetry | S4 | S11 (funnel + item attention) | C01, C02, O01, O34 (+ funnel frame, reserved) | §12.3 | todo |

## Not items, but must exist (tick when done)

| What | Session | Status |
|---|---|---|
| `withTenant` / `asWorker` + RLS leak test matrix (`docs/06 §4`) | S0 | todo |
| Schema `pii` + `regulars_identity` role + second pool + PII-wall gate (`docs/06 §4b`) | S0 | todo |
| `pii.customer_public` view + `pii.reveal_identity()` + reveal audit | S1 | todo |
| Column-level grants excluding `phone`/`raw` on `conversations` + `messages` | S0 (grants), S9 (verify) | todo |
| Browse partitions + 90/400-day retention job + browse-session sweeper | S4 (tables), S9 (crons) | todo |
| Lake publication excludes `pii.*` (CI check) | S0 | todo |
| Outbox: `app.events` writer + consumer skeleton | S0 | todo |
| 10 BullMQ queues + worker process (`docs/15`) | S9 | todo |
| Socket.IO namespaces (order desk live) | S5 | todo |
| Plan flags + upgrade sheet paywall (`vendors.plan`, `features`) | S2 (flags), S10 (paywall on paid routes) | todo |
| Security pass `docs/06` top to bottom | after S13 | todo |
| Deploy pipeline (compose → Mumbai VPS, Supabase prod, PITR rehearsal) | after S13 | todo |

## Misses log (things we noticed we forgot — sorted every Sunday)

| Date | What | Found in | Decision (fix-now / v1.1 / no) | Session assigned |
|---|---|---|---|---|
| | | | | |
