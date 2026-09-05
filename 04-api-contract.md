# 04 — API Contract (v1 Master Spec)

REST API for v1. Base path `/api/v1`. All request/response bodies are zod-validated with schemas from `packages/shared` — this doc names the shape; the zod schema is the executable truth.

---

## 1. Conventions

- JSON only. `Content-Type: application/json`. Body limit 100 kB (image uploads use a dedicated presigned-URL flow, §9; AI menu OCR uses dedicated multipart/presigned route, §8).
- Auth: `Authorization: Bearer <accessJWT>`; refresh via httpOnly cookie on `/auth/refresh`.
- IDs in paths are Mongo ObjectIds (or unguessable table/session tokens); malformed ID → 422.
- Pagination: `?page=1&limit=20` (limit ≤ 50) → response `meta: { page, limit, total, hasMore }`. Order lists use cursor style: `?before=<createdAt-cursor>`.
- Idempotency: `POST /orders`, `POST /sessions/{token}/rounds`, and campaign send require an `Idempotency-Key` header (client UUID). Replay with same key → return the original result with `200` + `Idempotency-Replayed: true`, never a duplicate.
- Rate limits per route class are specified in `06-security-checklist.md` §6; on limit → `429` with `Retry-After`.

---

## 2. Response Envelope & Error Codes (Single implementation in `shared/`)

```jsonc
// success
{ "success": true, "data": { ... }, "meta": { ... } }        // meta optional
// error
{ "success": false, "error": { "code": "OUT_OF_STOCK", "message": "human-readable, safe",
    "details": [ { "path": "items[0].qty", "issue": "..." } ],   // optional, zod issues etc.
    "requestId": "uuid" } }
```

Canonical error codes (enum in `packages/shared/errors.ts`):

| HTTP | Code | When |
|---|---|---|
| 401 | `UNAUTHENTICATED` | missing/expired/invalid token |
| 401 | `TOKEN_EXPIRED` | expired access JWT (client should refresh) |
| 403 | `FORBIDDEN` | role/ownership fails (incl. suspended vendor writes) |
| 404 | `NOT_FOUND` | truly absent OR cross-tenant (never reveal existence) |
| 409 | `CONFLICT_STATE` | invalid order-state transition (returns `currentStatus`) |
| 409 | `OUT_OF_STOCK` | stock guard failed (returns offending `itemId`s) |
| 409 | `PRICE_CHANGED` | cart snapshotVersion stale (returns fresh prices) |
| 409 | `SESSION_CLOSED` | trying to add rounds to an already-settled table tab |
| 409 | `ALREADY_REVIEWED` | duplicate review for order |
| 409 | `DAY_ALREADY_CLOSED` | day-close already performed for this date |
| 409 | `DUPLICATE` | other unique-index conflicts |
| 422 | `VALIDATION` | zod parse failure |
| 422 | `INVALID_COUPON` | coupon code expired, min spend not met, or inactive |
| 429 | `RATE_LIMITED` | rate limiter |
| 423 | `VENDOR_CLOSED` | ordering while store closed/suspended/busy-throttled |
| 500 | `INTERNAL` | anything unexpected (opaque to client) |

---

## 3. Auth

```
POST /auth/otp/request        { phone }                              → { retryAfterSec }
POST /auth/otp/verify         { phone, code }                        → { accessToken, user } + refresh cookie
POST /auth/login              { email, password }                    → { accessToken, user } + refresh cookie (vendor/admin)
POST /auth/refresh            (cookie)                               → { accessToken } + rotated cookie
POST /auth/logout             (cookie)                               → revokes token family
GET  /auth/me                                                        → { user }
PATCH /me                     { name?, birthday?, whatsappOptIn? }   (customer profile)
```

---

## 4. Public / Discovery & Storefront (No tenant middleware; explicit filters)

```
GET  /cities                                                         → platform city list
GET  /vendors?city=jaipur&openNow=1&veg=1&sort=rating                → discovery list (paginated)
GET  /vendors/{slug}                                                 → vendor public profile + open/busy status
GET  /vendors/{slug}/menu                                            → categories + active items (+ stock, snapshotVersion)
GET  /vendors/{slug}/reviews?page=                                   → visible reviews + rating summary
GET  /v/{slug}/t/{tableToken}                                        → resolves table & initiates/fetches active session
```

---

## 5. Customer Table Sessions & Orders (Auth: Customer for write actions)

```
# table tab access (session token from QR)
GET  /sessions/{sessionToken}                                        → active table tab, rounds, runningTotal, status
POST /sessions/{sessionToken}/rounds          (Idempotency-Key)      # auth: customer
  { items: [{ itemId, variantId?, addonIds: [], qty }], customerNote? }
  → 201 { order, session }
  → 409 OUT_OF_STOCK | PRICE_CHANGED | SESSION_CLOSED | 423 VENDOR_CLOSED

# in-session table assistance
POST /sessions/{sessionToken}/call-waiter                            → 200 { status: 'alerted' }
POST /sessions/{sessionToken}/request-bill                           → 200 { session, upiQrString, totals }

# customer order lifecycle & reviews
GET  /orders?before=                                                 → my order history (across visits)
GET  /orders/{id}                                                    → my order detail
POST /orders/{id}/cancel                                             → cancel only while 'placed'; else 409

POST /invoices/{id}/review                    { stars, text? }
  → 201 { review, routing: 'google_prompted' | 'private_feedback', googleUrl? }
PATCH /reviews/{id}                           { stars?, text? }      → only within 24h window
```

---

## 6. Vendor Dashboard & Order Desk (Auth: vendor_admin | vendor_staff; tenantContext active)

Routes marked ⓐ are `vendor_admin`-only. Staff access is permitted on Order Desk, Table Floor, POS Punch, and Stock toggles.

```
# profile, settings & operational switches
GET   /vendor/profile                          PATCH /vendor/profile ⓐ
PATCH /vendor/status          { isOpenOverride }                     # the open/closed switch
PATCH /vendor/status/busy     { isBusyMode: boolean }                # one-tap order throttle (staff-allowed)
PATCH /vendor/settings/payment { upiVpa?, razorpay?: { enabled, keyId, keySecret } } ⓐ
GET   /vendor/qr                                                     → SVG/PNG master vendor QR ⓐ

# table management ⓐ
GET/POST /vendor/tables                                              # CRUD tables
PATCH/DELETE /vendor/tables/{id}
GET   /vendor/tables/qr-sheet                                        → downloadable/printable PDF of table stand cards ⓐ

# order desk & floor view (staff-allowed)
GET   /vendor/floor                                                  → live table cards (empty, seated, bill_requested, alert)
GET   /vendor/orders?status=&before=                                 → live order kanban (polls + socket pushes)
GET   /vendor/orders/{id}
POST  /vendor/orders/{id}/accept               { etaMinutes }
POST  /vendor/orders/{id}/status               { to: 'preparing'|'ready'|'completed' }
POST  /vendor/orders/{id}/reject               { reason }
POST  /vendor/orders/{id}/cancel               { reason }            # restores limited inventory

# staff quick-punch billing & table settlement (staff-allowed)
POST  /vendor/pos/punch                        (Idempotency-Key)
  { tableId?, type: 'dine_in'|'takeaway', customerPhone?, customerName?,
    items: [{ itemId, variantId?, addonIds: [], qty }] }             → 201 { order, session }
POST  /vendor/sessions/{id}/discount           { code?, type: 'coupon'|'staff_percent'|'staff_flat', value?, reason? }
POST  /vendor/sessions/{id}/settle
  { paymentMethod: 'cash'|'upi'|'card'|'razorpay', transactionRef? }  → 200 { invoice, session }
GET   /vendor/invoices/{id}                                          → GST tax invoice data
GET   /vendor/invoices/{id}/print                                    → HTML 80mm/58mm formatted thermal receipt

# day-end close & cash reconciliation (staff-allowed)
POST  /vendor/day-close                        { physicalCashEntered, notes? } → 201 { dayClose }
GET   /vendor/day-close/summary?date=                                → preview current day shift numbers
GET   /vendor/day-close/history                                      → historical day-close records

# catalog management ⓐ
GET/POST /vendor/categories                    PATCH/DELETE /vendor/categories/{id}
GET/POST /vendor/items                         PATCH/DELETE /vendor/items/{id}
PATCH /vendor/items/{id}/stock                 { state, count? }     # staff-allowed
POST  /vendor/items/bulk-stock                 { itemIds, state }    # staff-allowed
POST  /vendor/menu/bulk-update                 { items: [...] } ⓐ    # bulk price/tax grid update

# AI menu OCR extraction & bulk import ⓐ
POST  /vendor/menu/ocr-extract                 (presigned URL or image upload) → { stagingCategories: [...] }
POST  /vendor/menu/ocr-confirm                 { categories: [...] } → commits extracted items to catalog

# coupons ⓐ
GET/POST /vendor/coupons                       PATCH/DELETE /vendor/coupons/{id}

# reviews & reputation
GET   /vendor/reviews                          POST /vendor/reviews/{id}/reply { text } ⓐ

# customers & retention funnel ⓐ
GET   /vendor/customers?segment=&page=                               # masked phone CRM list
GET   /vendor/customers/{id}                                         # customer profile & visit history at THIS vendor
GET   /vendor/settings/triggers                PATCH /vendor/settings/triggers
GET   /vendor/templates                                              # pre-approved Meta marketing templates
POST  /vendor/campaigns                        (Idempotency-Key)     { templateKey, variableValues, segment, scheduledAt? }
GET   /vendor/campaigns                        GET /vendor/campaigns/{id}
POST  /vendor/campaigns/{id}/cancel
GET   /vendor/stats/ledger?month=                                    # attributed revenue ledger: revenue, orders, reads
GET   /vendor/stats/today                                            # real-time summary: gross, net, orders, active tables

# WhatsApp WABA management (GoKwik style) ⓐ
POST  /vendor/whatsapp/connect                 { wabaId, phoneNumberId, accessToken } # Embedded Signup callback
GET   /vendor/whatsapp/status                                        → connection status & quality rating

# staff user management ⓐ
GET/POST /vendor/staff                         PATCH/DELETE /vendor/staff/{id}
```

---

## 7. Super Admin (Auth: super_admin)

```
GET/POST /admin/vendors                        PATCH /admin/vendors/{id} # subscription status, plan
POST  /admin/vendors/{id}/users                { email, role: 'vendor_admin' }
GET   /admin/reviews?flagged=                  POST /admin/reviews/{id}/hide { reason }
GET/POST /admin/templates                      PATCH /admin/templates/{key}
GET   /admin/message-logs?vendorId=&status=
GET   /admin/stats                                                    # platform-wide counters
GET/POST /admin/cities
```

---

## 8. Uploads

Direct-to-storage presigned uploads (S3-compatible). Flow: `POST /vendor/uploads/presign` → client PUTs file (≤ 2 MB for images, ≤ 5 MB for menu PDF OCR) → client passes `publicUrl` to catalog or OCR parser. API never proxies file bytes.

---

## 9. Webhooks (Signature Verification; No Auth Middleware)

```
POST /webhooks/whatsapp        # Meta signature (X-Hub-Signature-256) verified against app secret
GET  /webhooks/whatsapp        # Meta webhook challenge handshake
POST /webhooks/razorpay        # Razorpay signature verified against secret (for online payment callbacks)
```

WhatsApp webhook routing:
- Dynamic multi-tenant dispatch: reads `metadata.phone_number_id` -> finds matching `vendor` -> updates message log / opt-out state.
- Inbound `STOP` / unsubscribe replies: immediately flips customer's `whatsappOptIn` to false globally.

---

## 10. Socket.IO Contract

```
namespace /vendor   (auth: vendor JWT)
  room: vendor:{vendorId}
    server→client:
      order:new              { order, tableNumber }
      order:status           { orderId, status }
      table:service_call     { tableNumber, type: 'call_waiter' | 'request_bill', at }
      table:settled          { tableNumber, sessionId }

namespace /customer (auth: customer JWT or sessionToken handshake)
  room: session:{sessionToken}
    server→client:
      round:status           { orderId, status, etaMinutes? }
      table:updated          { runningTotal, status }
```
