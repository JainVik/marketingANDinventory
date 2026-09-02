# 04 — API Contract

REST API for v1. Base path `/api/v1`. All request/response bodies are zod-validated with schemas from `packages/shared` — this doc names the shape; the zod schema is the executable truth.

## 1. Conventions

- JSON only. `Content-Type: application/json`. Body limit 100 kB (image uploads use a dedicated presigned-URL flow, §8).
- Auth: `Authorization: Bearer <accessJWT>`; refresh via httpOnly cookie on `/auth/refresh`.
- IDs in paths are Mongo ObjectIds; malformed ID → 422 (not 500, not 404).
- Pagination: `?page=1&limit=20` (limit ≤ 50) → response `meta: { page, limit, total, hasMore }`. Order lists use cursor style: `?before=<createdAt-cursor>`.
- Idempotency: `POST /orders` and campaign send require an `Idempotency-Key` header (client UUID). Replay with same key → return the original result with `200` + `Idempotency-Replayed: true`, never a duplicate.
- Rate limits per route class are specified in `06-security-checklist.md` §6; on limit → `429` with `Retry-After`.

## 2. Response envelope & error codes (single implementation in `shared/`)

```jsonc
// success
{ "success": true, "data": { ... }, "meta": { ... } }        // meta optional
// error
{ "success": false, "error": { "code": "OUT_OF_STOCK", "message": "human-readable, safe",
    "details": [ { "path": "items[0].qty", "issue": "..." } ],   // optional, zod issues etc.
    "requestId": "uuid" } }
```

Canonical error codes (enum in `packages/shared/errors.ts` — never invent ad-hoc strings):

| HTTP | Code | When |
|---|---|---|
| 401 | `UNAUTHENTICATED` | missing/expired/invalid token |
| 401 | `TOKEN_EXPIRED` | expired access JWT (client should refresh) |
| 403 | `FORBIDDEN` | role/ownership fails (incl. suspended vendor writes) |
| 404 | `NOT_FOUND` | truly absent OR cross-tenant (never reveal existence) |
| 409 | `CONFLICT_STATE` | invalid order-state transition (returns `currentStatus`) |
| 409 | `OUT_OF_STOCK` | stock guard failed (returns offending `itemId`s) |
| 409 | `PRICE_CHANGED` | cart snapshotVersion stale (returns fresh prices) |
| 409 | `ALREADY_REVIEWED` | duplicate review for order |
| 409 | `DUPLICATE` | other unique-index conflicts |
| 422 | `VALIDATION` | zod parse failure |
| 429 | `RATE_LIMITED` | rate limiter |
| 423 | `VENDOR_CLOSED` | ordering while store closed/suspended |
| 500 | `INTERNAL` | anything unexpected (opaque to client) |

## 3. Auth

```
POST /auth/otp/request        { phone }                    → { retryAfterSec }
POST /auth/otp/verify         { phone, code }              → { accessToken, user } + refresh cookie
POST /auth/login              { email, password }          → { accessToken, user } + refresh cookie   (vendor/admin)
POST /auth/refresh            (cookie)                     → { accessToken } + rotated cookie
POST /auth/logout             (cookie)                     → revokes token family
GET  /auth/me                                              → { user }
PATCH /me                     { name?, birthday?, whatsappOptIn? }   (customer profile)
```

## 4. Public / customer-facing (no tenant middleware; explicit filters)

```
GET  /cities                                               → platform city list
GET  /vendors?city=jaipur&openNow=1&veg=1&sort=rating      → discovery list (paginated)
GET  /vendors/{slug}                                       → vendor public profile + open/closed (computed server-side)
GET  /vendors/{slug}/menu                                  → categories + active items (+ stock state, snapshotVersion)
GET  /vendors/{slug}/reviews?page=                         → visible reviews + rating summary
```

Menu response includes `menuVersion` (max snapshotVersion) so the client can detect staleness before placing.

## 5. Customer orders & reviews (auth: customer)

```
POST /orders                                (Idempotency-Key required)
  { vendorSlug, fulfilmentType, tableNumber?, customerNote?,
    items: [{ itemId, variantId?, addonIds: [], qty }] }
  → 201 { order }            // server computes ALL prices; client prices ignored
  → 409 OUT_OF_STOCK | PRICE_CHANGED | 423 VENDOR_CLOSED

GET  /orders?before=                        → my orders (any vendor)
GET  /orders/{id}                           → my order (ownership enforced)
POST /orders/{id}/cancel                    → only from 'placed'; else 409 CONFLICT_STATE

POST /orders/{id}/review    { stars, text? }   → 201; eligibility: order mine + completed; 409 ALREADY_REVIEWED
PATCH /reviews/{id}         { stars?, text? }  → only within editableUntil; else 403
```

## 6. Vendor dashboard (auth: vendor_admin | vendor_staff; tenantContext active)

Staff role is read/act on orders + stock toggles ONLY; routes marked ⓐ are vendor_admin-only.

```
# profile & settings
GET   /vendor/profile                          PATCH /vendor/profile ⓐ
PATCH /vendor/status          { isOpenOverride }        # the open/closed switch
GET   /vendor/qr                                → SVG/PNG QR for /v/{slug} ⓐ

# catalog ⓐ (all CRUD)
GET/POST /vendor/categories        PATCH/DELETE /vendor/categories/{id}
GET/POST /vendor/items             PATCH/DELETE /vendor/items/{id}
PATCH /vendor/items/{id}/stock     { state, count? }      # staff-allowed
POST  /vendor/items/bulk-stock     { itemIds, state }     # staff-allowed
POST  /vendor/uploads/presign      { kind: 'item'|'logo'|'cover', contentType } ⓐ  → { uploadUrl, publicUrl }

# orders (staff-allowed)
GET  /vendor/orders?status=&before=            # live board polls this; socket pushes deltas
GET  /vendor/orders/{id}
POST /vendor/orders/{id}/accept    { etaMinutes }
POST /vendor/orders/{id}/status    { to: 'preparing'|'ready'|'completed' }
POST /vendor/orders/{id}/reject    { reason }
POST /vendor/orders/{id}/cancel    { reason }             # from accepted/preparing
PATCH /vendor/orders/{id}/paid     { markedPaidByVendor }

# reviews
GET  /vendor/reviews               POST /vendor/reviews/{id}/reply { text } ⓐ

# customers & funnel ⓐ
GET  /vendor/customers?segment=&page=          # masked phones
GET  /vendor/customers/{id}                    # profile + order history at THIS vendor
GET  /vendor/settings/triggers                 PATCH /vendor/settings/triggers
GET  /vendor/templates                         # active promo templates available to vendors
POST /vendor/campaigns            (Idempotency-Key)  { templateKey, variableValues, segment, scheduledAt? }
GET  /vendor/campaigns            GET /vendor/campaigns/{id}     # incl. counts
POST /vendor/campaigns/{id}/cancel             # only while 'scheduled'
GET  /vendor/quota                             # today's message quota usage

# staff ⓐ
GET/POST /vendor/staff             PATCH /vendor/staff/{id}   { status }   DELETE /vendor/staff/{id}

# stats
GET  /vendor/stats/today                       # orders, revenue, top items, repeat %
GET  /vendor/stats/ledger?month=               # attributed-revenue ledger: per-campaign/trigger revenue, orders, reads; monthly "platform made you ₹X"
PATCH /vendor/status/throttle  { busyMode: boolean }   # one-tap order throttle (staff-allowed)
```

Every `/vendor/*` handler resolves `vendorId` from the JWT via tenantContext. A vendor route that reads `vendorId` from params/body is a review-blocking defect.

## 7. Super admin (auth: super_admin)

```
GET/POST /admin/vendors            PATCH /admin/vendors/{id}          # incl. subscription status
POST /admin/vendors/{id}/users     { email, role: 'vendor_admin' }    # sends invite/set-password
GET  /admin/reviews?flagged=       POST /admin/reviews/{id}/hide { reason }  POST .../unhide
GET/POST /admin/templates          PATCH /admin/templates/{key}
GET  /admin/message-logs?vendorId=&status=
GET  /admin/stats                                                    # platform counters
GET/POST /admin/cities
```

## 8. Uploads

Direct-to-storage presigned uploads (S3-compatible). Flow: `POST /vendor/uploads/presign` → client PUTs file (≤ 2 MB, `image/jpeg|png|webp` enforced by policy AND re-checked server-side on save) → client sends back `publicUrl` in the item/profile PATCH. API never proxies file bytes.

## 9. Webhooks (no auth middleware; signature verification instead)

```
POST /webhooks/whatsapp        # Meta signature (X-Hub-Signature-256) verified against app secret
GET  /webhooks/whatsapp        # Meta verify-token handshake
```

Handler: verify signature → 200 immediately → process async (queue). Handles: delivery/read status updates (by `providerMessageId`), inbound `STOP`/opt-out messages → flip `whatsappOptIn` + log. Unknown message IDs are logged and dropped, never 500.

## 10. Socket.IO contract

```
namespace /vendor   (auth: vendor JWT)   rooms: vendor:{vendorId}
  server→client: order:new {order}, order:status {orderId, status}
namespace /customer (auth: customer JWT) rooms: order:{orderId}   (join validated by ownership)
  server→client: order:status {orderId, status, etaMinutes?}
```

Clients treat socket events as cache-invalidation hints (refetch via REST); payloads are convenience, not truth.
