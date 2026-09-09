# 16 — Build Playbook (module-wise, one session at a time)

How we go from an empty repo to 5 pilot outlets without losing track of what lives where. Binding for the humans and for Claude Code. Companion: `docs/00-coverage.md` (the 66-row checklist this playbook fills in).

---

## 1. The five rules

1. **One module per session.** Never "build the backend". Always "build the `menu` module".
2. **A module is done before the next starts.** Done = tests green + `tools` entries seeded + coverage rows updated + you clicked the screen.
3. **Backend and its screens ship together.** Session 3 = menu API **and** the owner Menu screens. Every week ends with something clickable.
4. **No scaffolding ahead.** Claude Code never creates folders for future modules (CLAUDE.md already says so — repeat it in every prompt).
5. **Misses go to a doc first, then to a session.** Never "just add it" while another module is open (§6).

---

## 2. Session sequence

| Session | Week | Module(s) | Feeds (items from docs/01) | You can see at the end |
|---|---|---|---|---|
| S0 | 1 | Scaffold + foundations (incl. schema `pii`, the second pool, the PII-wall gate) | 62, 64 skeleton | `GET /healthz`; tests run against real Postgres; RLS leak test AND PII-wall gate pass |
| S1 | 1 | `auth` + `identity` | 1, 67 | Owner signs up with OTP, gets JWT; a customer OTPs once and resumes on a second scan with no OTP |
| S2 | 1 | `vendors` (outlet, wizard, settings, plan flags) | 2, 3, 4, 5, 6, 9, 66 setting | Wizard saves each step; settings screens work |
| S3 | 2 | `menu` (CRUD, variants, availability, bulk, CSV import, preview, banner) | 10–13, 14 (CSV/Excel), 16, 17 | Menu editor + live phone preview |
| S4 | 2 | `tables` + public storefront + customer browse + `telemetry` | 18, 19, 20 (kind), 21, 22, 6 (practice table), 68 | Scan a real QR → menu opens on a phone, cart works; the scan-browse-leave trail lands in `browse_sessions` |
| S5a | 3 | `sessions` + `orders` — customer side | 23, 25, 27, 24 (flow) | Place order → waiting screen → status updates |
| S5b | 4 | `orders` — owner side + auto-accept + sockets | 31–36, 39, 66, 37 (grid) | Order desk with loud alert; accept → phone updates live |
| S6 | 4 | `billing` (bills, invoice, coupons, comp, settle, day-close) | 26, 28, 37 (settle), 40–46 (non-gateway), 3 (gate) | Bill + GST invoice number; settle in cash/UPI |
| S7 | 5 | `payments` (Razorpay + Cashfree adapters, webhooks, refunds) | 8, 23/24 (pay prompt), 40, 45 (gateway) | Pay via gateway test mode; webhook marks bill paid; refund back to source |
| S8 | 5 | `customers` + `reviews` (incl. reveal-on-first-order) | 7 (import), 30 (history), 38, 48, 60, 67 (reveal) | Customer book fills from orders; a customer who scanned but never ordered shows as a count, not a name |
| S9 | 6 | `whatsapp` + `jobs` (queues, worker process, templates, STOP, crons) | 29, 61, 7 (opt-in ask), 12/39 (crons) | E-bill reaches a real phone; STOP works; DLQ visible |
| S10 | 7 | `marketing` (automations, campaigns, holdout, ledger) + paywall | 51–55, 53 (send) | First campaign with holdout and lift; free plan hits upgrade sheet |
| S11 | 7 | `insights` (briefing, lapsed wall, capture rate, dead hours, menu, kitchen speed, funnel, reports) | 47, 49, 50, 53 (detect), 56–59, 68 (funnel + viewed-never-ordered) | Morning briefing arrives on WhatsApp; the owner can see scans → orders and which items get looked at and skipped |
| S12 | 8 | `agents` (tools seed check, owner bot, RAG) + AI menu import | 64 (verify), 65, 14 (photo/PDF), 15 | Ask the bot "kal ka sale?" and get an answer; photo → menu |
| S13 | 8 | `admin` + exports + security pass | 62 (UI), 9 (export), docs/06 | Suspend a vendor; export a vendor's data |
| S14 | 9 | PWA polish, `site`, deploy | 30 (A2HS), 63, docs/09 | Pilots can scan, order, pay, get e-bill on production |

Checkpoints (no coding, one day each, §5): **after S5b, after S9, after S12.**

Timeline is a guide, not a promise. If S5 takes three sessions, it takes three. Do not skip S5's tests to make the week.

---

## 3. Session cards

Every card has the same four lines. Copy the prompt, change nothing but the module.

### S0 — Scaffold

- **Reads:** CLAUDE.md, docs/02 §1–5, docs/15 §5 (tree), docs/03 §2b.1, docs/06 §4
- **Builds:** monorepo + workspaces, `apps/api` bootstrap (`config/env.ts`, `loaders/`, `middlewares/`, `shared/`), `db/client.ts` (`withTenant`, `asWorker`), `packages/shared` (enums from `db/schema.ts`, error codes, money/phone/date utils), `modules/events` (outbox writer), `modules/agents/run-tool.ts` skeleton (`runTool` + registry type only), test harness with real Postgres, CI
- **Not:** any feature module, any screen
- **Prompt:** "Read CLAUDE.md and docs/02. Scaffold the monorepo exactly as docs/15 §5. Include db/client.ts with withTenant/asWorker per docs/03 §2b.1, packages/shared per docs/02 §2, the events outbox writer, the runTool skeleton, the Vitest+Supertest harness against the docker-compose Postgres with migrations applied, and the tenant-leak test from docs/06 §4. No feature modules, no frontend apps beyond empty Vite shells."

### S1 — `auth` + `identity`

- **Reads:** docs/01 #1, #67, docs/02 §4b, docs/03 §0 (PII boundary) + §2.10, docs/04 §3, docs/05 §1 + §12.1–12.2, docs/06 §2–3 + §4b, docs/14 SCR-A01, A02, O46 (shell only)
- **Builds:** OTP request/verify, email+password login, refresh, `tenantContext`, owner-web login screens, **the `identity` module and its `regulars_identity` pool**, customer device sessions + 180-day rotating refresh, `/auth/customer/resume`, `/me/devices`
- **Do not:** grant `regulars_app` anything in `pii`, or import the identity pool anywhere but `modules/identity/`. The gate in docs/06 §4b must be green at the end of this session.
- **Prompt:** "Implement the `auth` module only. Requirements: docs/01 item 1, docs/04 §3, docs/05 §1, docs/06 §2–3, screens SCR-A01, A02 in docs/14. Follow the module shape in docs/02 §3. Tests, tools entries, events included. Before writing any helper, search packages/shared and apps/api/src/shared and tell me what you found. Do not touch other modules. End with the CLAUDE.md self-review and list which docs/05 cases you handled and which remain."

### S2 — `vendors`

- **Reads:** docs/01 #2–6, 9, 66; docs/04 §6; docs/05 §1; docs/13 (wizard steps); docs/14 SCR-A03.x, O37–O43, O46
- **Builds:** vendor/outlet CRUD, wizard state, settings (QR mode, auto-accept rules, cancellation toggle, practice flag), plan + feature flags, upgrade sheet component
- **Prompt:** same shape as S1 with these references.

### S3 — `menu`

- **Reads:** docs/01 #10–17; docs/03 §2.2; docs/04 §7; docs/05 §8 (CSV part only); docs/14 SCR-O12–O17, A03.2
- **Builds:** categories/items/variants/add-ons, availability + restore schedule (column only; cron in S9), bulk grid, CSV/Excel import + review grid, phone preview, offers banner
- **Not:** photo/PDF extraction (S12)

### S4 — `tables` + storefront

- **Reads:** docs/01 #18–22, #68; docs/03 §2.3, §2.10, §3.4; docs/04 §4, §8; docs/05 §9, §12.3; docs/14 SCR-O10, A03.3, C01–C03, C17, C18
- **Builds:** tables, tokens, QR PDF, regenerate, public storefront routes (explicit filters, no tenant middleware), customer-web menu/item/cart, **the `telemetry` module**: visitor issue, beacon ingest, browse session sweeper, client batching
- **Do not:** put a browse write inside an order transaction or through `app.events`. Prove it: an induced telemetry failure must leave placement green (docs/05 §12.3 case 14).

### S5a — `sessions` + `orders` (customer side)

- **Reads:** docs/01 #23–25, 27; docs/03 §2.3, §2b.2; docs/04 §5; docs/05 §2 (all of it); docs/14 SCR-C04–C06, C12, C14
- **Builds:** session open/join, row lock on placement, rounds, idempotent place, OTP at first order, accept-gate wait screen, live status, call waiter, request bill, cancellation request
- **Prompt add-on:** "This is the hardest module. Implement docs/05 §2 cases one by one, each with its test, before writing any screen."

### S5b — `orders` (owner side)

- **Reads:** docs/01 #31–37, 39, 66; docs/04 §9, §15; docs/05 §3; docs/14 SCR-O02–O07
- **Builds:** guarded status transitions, presets, staff entry, move table, busy mode, owner cancel, auto-accept evaluation, floor grid, Socket.IO order desk namespace, loud alert

### S6 — `billing`

- **Reads:** docs/01 #26, 28, 37, 40–46 (non-gateway), 3; docs/03 §2.4; docs/04 §9; docs/05 §4; docs/14 SCR-O08, O09 (cash/adjust), O22–O24, C09, C10 (in-app)
- **Builds:** bills, `ops.next_seq` invoice numbers, GST lines, coupons, complimentary, multi-mode settle, cash/adjust refunds, day-close

### S7 — `payments`

- **Reads:** docs/01 #8, 40, 45; docs/02 §7; docs/04 §5 (pay), §14; docs/05 §4 (gateway cases); docs/06 §5; docs/14 SCR-C07, C08, O40, A03.7
- **Builds:** `PaymentProvider` port, Razorpay + Cashfree adapters, webhook ingress (verify → dedupe → raw → 202), pay-after-accept prompt, pay-first for counter, back-to-source refunds

### S8 — `customers` + `reviews`

- **Reads:** docs/01 #7, 30, 38, 48, 60; docs/03 §2.5; docs/04 §11 (book part); docs/05 §5; docs/06 §8; docs/14 SCR-O11, O25–O28, O36, C11, C15
- **Builds:** profiles from orders, consent ledger, CSV import + column map, segments, block, Regulars Board, cross-outlet history, review capture + routing (private only in v1)

### S9 — `whatsapp` + `jobs`

- **Reads:** docs/01 #29, 61; docs/02 §8; docs/15 §3–4 (queues); docs/03 §2.7, Step 3.1; docs/04 §14; docs/05 §6; docs/14 SCR-O21, O41
- **Builds:** `NotificationService` port + Meta adapter, all 10 queues + worker process, outbox consumers, templates, e-bill, opt-in ask, STOP, caps, dedupe, 24 h window, 131049, DLQ, crons (availability restore, slow-order)

### S10 — `marketing`

- **Reads:** docs/01 #51–55; docs/03 §2.6; docs/04 §11; docs/05 §6 (holdout, attribution); docs/14 SCR-O29–O33
- **Builds:** automations engine, campaigns, holdout split, attribution ledger, dead-hour send, paid-route paywall (`403 FORBIDDEN` + feature)

### S11 — `insights`

- **Reads:** docs/01 #47, 49, 50, 53, 56–59; docs/03 §2.9 (features); docs/04 §11; docs/05 §7 (briefing); docs/14 SCR-O01, O18–O20, O32, O34, O35, A03.9
- **Builds:** RFM/rule segments refresh, lapsed wall, capture rate, dead-hour detection, menu conclusions, market basket, kitchen speed, reports, briefing builder + send

### S12 — `agents` + AI import

- **Reads:** docs/01 #14, 15, 64, 65; docs/02 §9; docs/03 §2.8, Step 3.2; docs/04 §12; docs/05 §7, §8; docs/14 SCR-O41, O15, O16
- **Builds:** tools seed audit (every owner route has a row — fail the test if not), `LlmProvider` + `OcrProvider` ports, owner bot read-only loop, RAG documents, photo/PDF menu extraction + diff

### S13 — `admin` + exports + security

- **Reads:** docs/01 #9, 62; docs/04 §13; docs/06 (all); docs/14 SCR-X01–X09, O43
- **Builds:** admin endpoints + panel, vendor export job, then run docs/06 top to bottom and fix

### S14 — polish + `site` + deploy

- **Reads:** docs/01 #30, 63; docs/13; docs/14 §3–4 (nav, patterns), SCR-S01–S10; docs/15 §6 (deploy); docs/09
- **Builds:** PWA manifest + A2HS prompt, loading/empty/error states audit on every screen, the public site, Docker Compose to the Mumbai VPS, Supabase prod, PITR rehearsal

---

## 4. Folder and naming rules (so a stranger finds things in three hops)

### 4.1 Backend module — same eight files, every module

```
apps/api/src/modules/menu/
  menu.routes.ts       URLs + middleware chain. Nothing else.
  menu.controller.ts   parse (zod) → withTenant → call service → respond. No logic.
  menu.service.ts      ALL business logic. Takes tx. Throws AppError.
  menu.tools.ts        owner actions for the registry (key, schema, side effect, handler)
  menu.events.ts       event types this module writes + payload schemas
  menu.owns.ts         one line per table this module is allowed to write
  menu.test.ts         happy path + docs/05 cases + tenant isolation
  index.ts             the ONLY file other modules import from
```

A module that needs a ninth file is usually two modules. Ask before adding one. Sub-folders inside a module are allowed only for adapters (`payments/adapters/razorpay.ts`).

### 4.2 Frontend feature — mirrors the backend name

```
apps/owner-web/src/features/menu/
  screens/     one file per SCR-ID: menu-editor.screen.tsx (SCR-O12)
  components/  presentational only
  hooks/       TanStack Query hooks: use-menu-items.ts
  api.ts       the typed client calls for this feature
```

Same names on both sides: backend `modules/menu` ↔ frontend `features/menu`. If you cannot find the frontend for a module, it has the same name.

### 4.3 Names

| Thing | Rule | Yes | No |
|---|---|---|---|
| Module folder | the noun the DB table uses, plural | `orders`, `bills`, `customers` | `orderManagement`, `OrderModule`, `core` |
| File | `<module>.<layer>.ts` | `orders.service.ts` | `service.ts`, `ordersSvc.ts` |
| Function | verb + noun, one job | `acceptOrder`, `settleBill`, `sendEbill` | `handle`, `process`, `doStuff` |
| Screen file | `<screen-name>.screen.tsx` with the SCR-ID in a comment on line 1 | `order-desk.screen.tsx` | `Page2.tsx` |
| Shared helper | by topic | `money.ts`, `phone.ts`, `dates.ts` | `utils.ts`, `helpers.ts`, `common.ts` |
| Event | `<noun>.<past-tense-verb>` | `order.accepted` | `ORDER_EVENT_2` |
| Tool key | `<module>.<verb-noun>` | `menu.set-availability` | `toggleItem` |

Banned folder names anywhere: `common/`, `misc/`, `helpers/`, `utils/`, `lib/`, `core/`, `v2/`, `new/`, `old/`, `temp/`. Shared code has exactly two homes: `packages/shared` (used by ≥ 2 apps) and `apps/api/src/shared` (api only).

### 4.4 Finding a bug years later

1. Bug says "coupon applied twice" → `grep -ri coupon apps/api/src/modules/` → `billing/billing.service.ts`.
2. The function's JSDoc points to `docs/05 §4.x`.
3. `billing.test.ts` next to it shows the expected behaviour.

If any hop fails, the code is wrong, not the reader.

---

## 5. No-duplicate rules

- Before any new helper Claude Code runs `grep -rn "function <name>\|export const <name>" packages/ apps/` and reports the result in the session. No report = no merge.
- Enums, state machines, money, phone, dates exist once in `packages/shared`; DB enums come from `db/schema.ts`. Retyping any of them is a defect.
- Modules import each other only through `modules/<name>/index.ts`. A circular import means the boundary is wrong — stop and split.
- Seen twice → leave it. Seen three times → move to shared, in its own small PR.
- A "similar but with a flag" function is duplication with extra steps. Two functions.

---

## 6. Handling misses (there will be misses)

**Where misses get caught**

| Net | When | Catches |
|---|---|---|
| Coverage file | end of every session | items never started, items half done |
| docs/05 handled/remaining list | end of every session | edge cases skipped |
| Checkpoint 1 — full fake dinner: scan → order → accept → pay → bill → day-close, on the practice table | after S5b (and again after S6) | gaps between menu / tables / sessions / orders / billing |
| Checkpoint 2 — same dinner on two real phones, e-bill, STOP, opt-in | after S9 | consent, phone, message gaps |
| Checkpoint 3 — ask the bot the 10 questions the briefing answers | after S12 | data gaps in events / insights |
| Pilots — 5 outlets, 5 tables each | after S14 | everything we never thought of |

**What to do with a miss**

1. Write it in the Misses log at the bottom of `docs/00-coverage.md` the moment you notice it. Include which screen/flow you were in.
2. Do not fix it in the currently open session unless it blocks that session's own module.
3. Every Sunday sort the log: **fix-now** (breaks ordering, billing, or money) → gets the next session; **v1.1** → move to docs/01 §4; **no** → docs/01 §5 (dropped) with one line why.
4. A fix-now that touches an existing module is a mini-session with the same prompt shape: "In the `billing` module only, add … per docs/05 §4.x. Do not touch other modules."
5. If a miss changes a decision, the doc changes in the same PR (README rule). Stale docs are how the next miss happens.

**Why misses are cheap here**

- Every v1–v2 column already exists in `0001_init.sql`. A missed feature is code, not a migration.
- Fixed module shape → a missed action = one service function + one route + one tools row + one test. Half a day.
- Modules are isolated → a fix in `billing` cannot break `orders`.

---

## 7. Your 30 minutes after every session

1. Read Claude Code's self-review and its handled/remaining list.
2. Open the module folder. ≤ 8 files? Same names as §4.1? If not, ask why before merging.
3. Run the tests. Click the screen. Do the thing a pilot would do.
4. Update `docs/00-coverage.md` rows for this session.
5. Commit: `feat(menu): categories, items, variants (S3)`. One module per PR; docs changes in the same PR.
6. Write any "hmm, we should…" in the Misses log. Close the laptop.

---

## 8. Git

- `main` is always deployable. One branch per session: `s03-menu`. Squash-merge with the session number in the message.
- Tag after each checkpoint: `checkpoint-1`, `checkpoint-2`, `checkpoint-3`, `pilot-1`.
- Never commit `db/schema.ts` by hand-edit; it is regenerated by `drizzle-kit pull` in CI and diffed — a diff fails the build.
