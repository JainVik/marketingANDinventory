# 04 — API Contract (v1)

REST API for v1. Base path `/api/v1`. All request/response bodies are zod-validated with schemas from `packages/shared` — this doc names the shape; the zod schema is the executable truth. Every owner action carries a **tool key** (right column) — the same function is callable by the UI, the owner bot (read tools in v1) and automations.

---

## 1. Conventions

- JSON only, camelCase. Body limit 100 kB (uploads use presigned URLs, §10).
- Auth: `Authorization: Bearer <accessJWT>`; refresh via httpOnly cookie on `/auth/refresh`.
- IDs are UUIDs (or opaque tokens for tables/sessions); malformed → 422.
- Pagination: cursor style `?limit=20&cursor=<opaque>` (limit ≤ 50) → `meta: { nextCursor }`.
- Idempotency: every unsafe money-adjacent POST (`rounds`, `pos/orders`, `payments`, `refunds`, `settle`, `campaigns`) requires `Idempotency-Key` (client UUID). Replay → `200` + `Idempotency-Replayed: true` with the original result; never a duplicate.
- Rate limits per route class in `06-security-checklist.md` §6; on limit → `429` + `Retry-After`.
- Money in paise (integer). Times ISO-8601 UTC.

---

## 2. Response envelope & error codes (single implementation in `packages/shared`)

```jsonc
{ "success": true,  "data": { … }, "meta": { … } }
{ "success": false, "error": { "code": "OUT_OF_STOCK", "message": "safe, human-readable", "details": [ … ], "requestId": "uuid" } }
```

| HTTP | Code | When |
|---|---|---|
| 401 | `UNAUTHENTICATED` / `TOKEN_EXPIRED` | missing/invalid / expired access JWT |
| 403 | `FORBIDDEN` | role fails, suspended vendor writes, feature not in plan (`details.feature`) |
| 404 | `NOT_FOUND` | truly absent OR cross-tenant (RLS returns no row; never reveal existence) |
| 409 | `CONFLICT_STATE` | invalid transition (returns `currentStatus`) |
| 409 | `OUT_OF_STOCK` | guarded stock decrement failed (returns `itemIds`) |
| 409 | `PRICE_CHANGED` | cart `priceVersion` stale (returns fresh prices) |
| 409 | `SESSION_CLOSED` | round on a settled/voided session |
| 409 | `TABLE_OCCUPIED` | opening a second active session on a table |
| 409 | `ALREADY_PAID` / `PAYMENT_PENDING` | duplicate pay attempt |
| 409 | `ALREADY_REVIEWED` / `DAY_ALREADY_CLOSED` / `DUPLICATE` | unique violations |
| 409 | `CANCEL_NOT_ALLOWED` | cancellation request disabled or order past `placed` |
| 422 | `VALIDATION` / `INVALID_COUPON` / `GST_REQUIRED` | zod failure / coupon rule / first invoice without GSTIN |
| 423 | `VENDOR_CLOSED` | ordering while closed / busy-throttled / suspended |
| 429 | `RATE_LIMITED` | |
| 500 | `INTERNAL` | opaque |

---

## 3. Auth

```
POST /auth/otp/request        { phone }                              → { retryAfterSec }
POST /auth/otp/verify         { phone, code, name?, consent?: { marketing: boolean, noticeVersion } }
                                                                     → { accessToken, customer } + refresh cookie
POST /auth/login              { email, password }                    → { accessToken, user } + refresh cookie (owner/admin)
POST /auth/signup             { name, email, password, phone, door: 'self' }   → creates vendor + owner + outlet shell, returns wizard state   (docs/13)
POST /auth/refresh            (cookie)                               → { accessToken } + rotated cookie
POST /auth/logout             (cookie)                               → revokes token family
GET  /auth/me                                                        → { user | customer }
PATCH /me                     { name?, birthday?: {day,month}, consent?: {...}, language? }   (customer)
GET  /me/history?cursor=                                             → my visits across all Regulars outlets (#30)
```

---

## 4. Public storefront (no tenant middleware; explicit filters)

```
GET  /o/{slug}                                        → outlet public profile, open/busy status, qrMode, banner
GET  /o/{slug}/menu                                   → categories + active items (+ availability, priceVersion, variants, addons)
GET  /v/{slug}/t/{qrToken}                            → resolves table; returns/creates the active session for that table (dine_in) or a counter session (takeaway)
                                                        410 GONE if the token was regenerated
```

---

## 5. Customer: sessions, orders, bill, payment (auth: customer for writes)

```
GET  /sessions/{token}                                → session, rounds, runningTotal, diners (masked), status, myLines
POST /sessions/{token}/rounds        (Idempotency-Key) { lines: [{ itemId, priceVersion, variantId?, addonIds: [], qty, instructions? }], couponCode? }
                                                      → 201 { order, session }  | 409 OUT_OF_STOCK | PRICE_CHANGED | SESSION_CLOSED | 423 VENDOR_CLOSED
POST /sessions/{token}/call-waiter                    → 200 { status: 'pending' }  (debounced 60 s)
POST /sessions/{token}/request-bill                   → 200 { session, bill preview }
GET  /sessions/{token}/bill                           → rounds, per-person attribution, totals, coupon, payable (mine / whole table)
POST /sessions/{token}/coupon        { code }         → 200 { discount } | 422 INVALID_COUPON        (before payment only)

# payment (pay-now outlets: after acceptance for table rounds; before acceptance for takeaway)
POST /sessions/{token}/payments      (Idempotency-Key) { scope: 'whole' | 'own_items', orderId? }
                                                      → 201 { paymentId, gateway: { provider, orderId, checkoutPayload } }
GET  /payments/{id}                                   → status (created|pending|captured|failed)

# order lifecycle
GET  /orders/{id}                                     → my order detail + timeline
POST /orders/{id}/cancel-request     { reason? }      → 202 (only if outlet enables; only while placed/accepted) | 409 CANCEL_NOT_ALLOWED
GET  /bills/{id}                                      → my bill (e-bill link target)
POST /bills/{id}/send-whatsapp                        → queue e-bill to me (repeat visits)
POST /bills/{id}/review              { stars, text? } → 201 { review, routing: 'google_prompted' | 'private', googleUrl? }
```

---

## 6. Owner: onboarding, outlet, settings (auth: owner; tenantContext)

| Route | Body → Result | Tool key |
|---|---|---|
| `GET /vendor/wizard` | wizard step, go-live checklist (menu / qr / testOrder) | `onboarding.status` |
| `PATCH /vendor/wizard` | `{ step }` | `onboarding.setStep` |
| `GET /vendor/outlet` · `PATCH /vendor/outlet` | profile, address, hours, type, branding, gstin/fssai | `outlet.get` / `outlet.update` |
| `PATCH /vendor/outlet/status` | `{ openOverride }` | `outlet.setOpen` |
| `PATCH /vendor/outlet/busy` | `{ on, extraEtaMin? }` | `outlet.setBusy` |
| `PATCH /vendor/outlet/qr-mode` | `{ qrMode: 'order'\|'pay_only'\|'both' }` | `outlet.setQrMode` |
| `PATCH /vendor/settings/ordering` | `{ cancellationRequests, slowOrderAlertMin, placedAutoExpireMin }` | `settings.ordering.set` |
| `GET/PUT /vendor/settings/auto-accept` | `{ enabled, rules: [{ kind, value?, active }] }` (#66) | `settings.autoAccept.get` / `.set` |
| `PATCH /vendor/settings/payment` | `{ payFlowDefault, refundDefault, upiVpa }` | `settings.payment.set` |
| `POST /vendor/settings/gateway/connect` | `{ provider }` → onboarding URL; `GET …/gateway/status` | `gateway.connect` / `gateway.status` |
| `PATCH /vendor/settings/triggers` | `{ birthday, winBack, postFirstOrder, deadHour, reviewAsk }` | `settings.triggers.set` |
| `PATCH /vendor/settings/practice` | `{ enabled }` | `settings.practice.set` |
| `POST /vendor/export` | → job; `GET /vendor/export/latest` → presigned zip (#9) | `vendor.export` |
| `GET /vendor/subscription` · `POST /vendor/subscription/cancel` | | `subscription.get` / `.cancel` |

## 7. Owner: menu

| Route | Body → Result | Tool key |
|---|---|---|
| `GET/POST /vendor/menu/categories` · `PATCH/DELETE …/{id}` | | `menu.category.*` |
| `GET/POST /vendor/menu/items` · `PATCH/DELETE …/{id}` | variants + addonGroups nested in body | `menu.item.*` |
| `PATCH /vendor/menu/items/{id}/availability` | `{ state, count?, autoRestore, restoreAt? }` (#12) | `menu.setAvailability` |
| `POST /vendor/menu/items/bulk` | `{ updates: [{ id, price?, gstRate?, isActive?, categoryId? }] }` (#13) | `menu.bulkUpdate` |
| `POST /vendor/menu/imports` | `{ kind: 'photo'\|'pdf'\|'csv'\|'rephoto', fileKeys[] }` → 202 `{ importId }` (#14–15) | `menu.import.start` |
| `GET /vendor/menu/imports/{id}` | rows sorted by confidence asc, diff summary | `menu.import.get` |
| `PATCH /vendor/menu/imports/{id}/rows/{rowId}` | `{ final, action }` | `menu.import.editRow` |
| `POST /vendor/menu/imports/{id}/apply` | → creates/updates items, writes revisions | `menu.import.apply` |
| `GET/POST /vendor/menu/banners` · `PATCH/DELETE …/{id}` | (#17) | `menu.banner.*` |
| `GET /vendor/menu/preview` | phone-preview payload + warnings (#16) | `menu.preview` |
| `GET /vendor/menu/items/{id}/revisions` | price history | `menu.item.history` |

## 8. Owner: tables & QR

| Route | Result | Tool key |
|---|---|---|
| `GET/POST /vendor/tables` · `PATCH/DELETE …/{id}` | kind table/room/counter | `tables.*` |
| `POST /vendor/tables/{id}/regenerate-qr` | new token; old → 410 | `tables.regenerateQr` |
| `GET /vendor/tables/qr-sheet.pdf` · `GET /vendor/tables/{id}/qr.pdf` | (#19) | `tables.qrSheet` |
| `POST /vendor/tables/standees/order` | paid standees request (manual fulfilment) | `tables.orderStandees` |

## 9. Owner: order desk, sessions, billing

| Route | Body → Result | Tool key |
|---|---|---|
| `GET /vendor/desk` | New / Preparing / Ready columns with table + customer badge (#31) | `desk.board` |
| `GET /vendor/floor` | table grid: empty / eating / needs-you, running totals (#37) | `desk.floor` |
| `GET /vendor/regulars-board` | seated now: name, visit count, usual order (#38) | `desk.regularsBoard` |
| `POST /vendor/orders/{id}/accept` | `{ etaMin }` | `orders.accept` |
| `POST /vendor/orders/{id}/status` | `{ to: 'preparing'\|'ready'\|'completed' }` | `orders.setStatus` |
| `POST /vendor/orders/{id}/reject` | `{ reason: preset }` | `orders.reject` |
| `POST /vendor/orders/{id}/cancel` | `{ reason }` (#36) | `orders.cancel` |
| `POST /vendor/orders/{id}/cancel-request/decide` | `{ decision: 'approved'\|'denied' }` | `orders.decideCancelRequest` |
| `POST /vendor/pos/orders` (Idempotency-Key) | `{ tableId?, type, customerPhone?, lines[] }` staff entry (#34) | `orders.staffEntry` |
| `POST /vendor/sessions/{id}/move` | `{ toTableId }` (#35) | `sessions.moveTable` |
| `POST /vendor/sessions/{id}/service-calls/{callId}/ack` | | `sessions.ackServiceCall` |
| `GET /vendor/sessions/{id}` | table sheet: rounds, diners, running total | `sessions.get` |
| `POST /vendor/sessions/{id}/discount` | `{ kind: 'coupon'\|'manual', code?, amount?, reason? }` | `billing.discount` |
| `POST /vendor/sessions/{id}/lines/{lineId}/complimentary` | `{ reason }` (#43) | `billing.complimentary` |
| `POST /vendor/sessions/{id}/settle` (Idempotency-Key) | `{ payments: [{ mode, amount }], tip? }` → bill issued (#40, #44) · 422 `GST_REQUIRED` if no GSTIN | `billing.settle` |
| `POST /vendor/bills/{id}/payments` (Idempotency-Key) | `{ mode, amount, recordedBy }` partial/extra payment | `billing.recordPayment` |
| `POST /vendor/bills/{id}/refunds` (Idempotency-Key) | `{ ofPaymentId, amount, mode: 'back_to_source'\|'cash'\|'adjust_replace', reason, replacementOrderId? }` (#45) | `billing.refund` (irreversible) |
| `POST /vendor/bills/{id}/void` | `{ reason }` | `billing.void` (irreversible) |
| `GET /vendor/bills/{id}` · `GET …/{id}/print` | invoice JSON / 80 mm HTML | `billing.get` |
| `POST /vendor/takeaway/{sessionId}/collected` | mark collected (#20) | `takeaway.collected` |
| `GET /vendor/day-close/preview?date=` · `POST /vendor/day-close` `{ cashCounted, note? }` · `GET /vendor/day-close/history` | (#46) | `dayclose.preview` / `dayclose.close` (irreversible) |
| `GET/POST /vendor/coupons` · `PATCH/DELETE …/{id}` | (#42) | `coupons.*` |

## 10. Uploads

`POST /vendor/uploads/presign { kind: 'menu_photo'|'menu_pdf'|'menu_csv'|'item_photo'|'customer_csv'|'logo', contentType, size }` → `{ url, key }`. Client PUTs, then passes `key`. Limits: images ≤ 2 MB, PDF ≤ 5 MB, CSV ≤ 2 MB. API never proxies bytes.

## 11. Owner: customer book & marketing (paid; 403 `FORBIDDEN` with `details.feature` on free plan)

| Route | Body → Result | Tool key |
|---|---|---|
| `GET /vendor/customers?segment=&tag=&q=&cursor=` | masked phone, segment, visits, spend (#48) | `customers.list` |
| `GET /vendor/customers/{id}` | profile + visits at this vendor + messages | `customers.get` |
| `PATCH /vendor/customers/{id}` | `{ tags?, customAttributes?, name? }` | `customers.update` |
| `POST /vendor/customers/{id}/block` · `/unblock` | `{ reason }` | `customers.block` |
| `POST /vendor/customers/imports` (presigned key) · `PATCH …/{id}/map` · `POST …/{id}/run` · `GET …/{id}` | CSV → column map → import → opt-in campaign (#7) | `customers.import.*` |
| `GET /vendor/segments` · `POST` · `PATCH/DELETE …/{id}` · `GET …/{id}/preview` | rule DSL, count | `segments.*` |
| `GET /vendor/stats/identity-capture?range=` | tile (#49) | `reports.identityCapture` |
| `GET /vendor/stats/lapsed-wall` | at-risk count + ₹ (#50) | `reports.lapsedWall` |
| `GET /vendor/templates` | approved templates for this channel | `templates.list` |
| `GET/POST /vendor/campaigns` (Idempotency-Key) · `GET …/{id}` · `POST …/{id}/schedule` · `/cancel` | `{ type, name, segmentId, templateId, variableValues, couponId?, sendAt, holdoutPercent? }` (#52, #54) | `campaigns.*` |
| `GET /vendor/campaigns/{id}/results` | treatment vs control, lift, incremental ₹ | `campaigns.results` |
| `GET /vendor/dead-hours` · `POST /vendor/dead-hours/{slot}/fill` | detect → capped campaign (#53) | `deadHours.list` / `deadHours.fill` |
| `GET /vendor/automations` · `PATCH …/{id}` | enable/disable, limits (#51) | `automations.*` |
| `GET /vendor/stats/ledger?month=` | revenue ledger (#55) | `reports.ledger` |
| `GET /vendor/insights?status=` · `POST …/{id}/act` · `/dismiss` | menu conclusions, upsell pairs, kitchen speed (#57–59) | `insights.*` |
| `GET /vendor/reports/revenue?range=&groupBy=` · `/kitchen-speed` · `/discounts-voids` · `/messaging-costs` | | `reports.*` |
| `GET /vendor/briefings?cursor=` · `PATCH /vendor/briefings/settings` `{ enabled, at }` | (#56) | `briefings.*` |
| `GET /vendor/reviews` · `POST …/{id}/reply` | (#60) | `reviews.*` |

## 12. Owner bot (v1 read-only)

No REST surface for owners — it lives on WhatsApp. Internals:

```
inbound WhatsApp from users.phone → conversations(kind owner) → agents.owner_bot → agent_runs
  tools allowed in v1: every key above with side_effect = 'read' (reports.*, desk.*, customers.get/list, insights.*, campaigns.results, menu.preview …)
  write tools → refused with "I can show you, but changing it is coming in the next version" (logged as tool_executions.status = denied)
GET  /vendor/bot/sessions?cursor=      → transcript (owner can review what the bot answered)
POST /vendor/bot/feedback              { runId, value: 'up'|'down', comment? }    → ml.ai_feedback
```

## 13. Super admin (auth: super_admin)

```
GET/POST /admin/vendors · PATCH /admin/vendors/{id}         # plan, sub_status, features flags, suspend
POST /admin/vendors/{id}/onboard-manual                     # door 'manual' / 'demo' → wizard link
GET/POST /admin/templates · PATCH /admin/templates/{id}     # Meta template sync
GET /admin/messages?vendorId=&status=&cursor=               # cross-tenant ledger
GET /admin/webhooks?source=&status=                         # raw webhook events + reprocess
POST /admin/webhooks/{id}/reprocess
GET /admin/tools · PATCH /admin/tools/{key}                 # registry: enable bot surface, deprecate
GET /admin/agents · PATCH /admin/agents/{key}               # prompt version, allowed tools
GET /admin/stats                                            # platform counters
GET /admin/audit?vendorId=&action=&cursor=
```

## 14. Webhooks (signature verification; no auth middleware)

```
GET/POST /webhooks/whatsapp             # Meta X-Hub-Signature-256; challenge handshake; routed by metadata.phone_number_id
POST /webhooks/payments/razorpay        # X-Razorpay-Signature
POST /webhooks/payments/cashfree        # x-webhook-signature
```

Flow: verify → `webhook_dedupe` insert (dup → 202) → `webhook_events` raw row → 202 → job → normalise (`docs/03` Step 3.1) → `messages` / `payments` / `conversations` → events.

## 15. Socket.IO contract

```
namespace /vendor   (auth: owner JWT)     room: vendor:{vendorId}
  order:new { orderId, tableName, badge }   order:status { orderId, status, etaMin? }   order:cancel_request { orderId }
  table:service_call { sessionId, type }    table:updated { sessionId, runningTotal, status }
  board:updated { sessionId }               payment:captured { sessionId, amount, mode }   slow:order { orderId, minutes }

namespace /customer (auth: customer JWT or session token)   room: session:{token}
  round:status { orderId, status, etaMin? }   table:updated { runningTotal, status }
  payment:status { paymentId, status }        bill:issued { billId }   takeaway:ready { token }
```
