# 03 — MongoDB Schema

Data shapes for v1. Mongoose 8 + TypeScript; every schema gets `{ timestamps: true }` (`createdAt`/`updatedAt`). All money = **integer paise**. All dates = UTC `Date`. IDs are `ObjectId` unless stated.

Conventions:
- Tenant-owned collections carry `vendorId` and every index on them **starts with `vendorId`**.
- `schemaVersion: number` field on orders, customer_profiles, message_logs (documents that will migrate).
- No cross-collection joins in hot paths — denormalize the few read-often fields listed here, nothing more.

## 1. `users` — all human identities

```ts
{
  role: 'customer' | 'vendor_admin' | 'vendor_staff' | 'super_admin',
  // customer identity
  phone?: string,          // E.164 ("+91..."), unique+sparse, required for customers
  name?: string,
  birthday?: { day: number, month: number },  // no year — we don't need age
  whatsappOptIn: { status: boolean, updatedAt: Date, source: 'signup'|'profile'|'stop_reply' },
  // vendor/admin identity
  email?: string,          // unique+sparse, required for vendor/admin roles
  passwordHash?: string,   // argon2id
  vendorId?: ObjectId,     // required iff vendor_admin | vendor_staff
  status: 'active' | 'disabled',
  lastLoginAt?: Date
}
// Indexes: {phone:1} unique sparse; {email:1} unique sparse; {vendorId:1, role:1}
```

One human = one document per role type. Customer and vendor identities are never merged.

## 2. `vendors` — the tenants

```ts
{
  slug: string,            // unique, immutable after creation, used in /v/{slug} QR links
  name: string, description?: string,
  logoUrl?: string, coverUrl?: string,
  city: string,            // normalized lowercase key from platform city list
  address: string,
  location: { type: 'Point', coordinates: [lng, lat] },
  contactPhone: string,
  fssaiLicense?: string,
  openingHours: [{ day: 0-6, open: 'HH:mm', close: 'HH:mm' }],  // multiple ranges/day allowed
  isOpenOverride: 'auto' | 'force_closed',     // the "temporarily closed" switch
  subscription: {
    plan: 'basic' | 'growth',
    status: 'trial' | 'active' | 'past_due' | 'suspended',
    startedAt: Date, renewsAt?: Date, notes?: string   // billing is manual in v1
  },
  quotas: { dailyMessages: number },            // resolved from plan at write time
  whatsapp: { wabaId?: string, phoneNumberId?: string },  // platform WABA values in v1; per-vendor WABA later (docs/07 §11)
  rating: { avg: number, count: number },       // denormalized, recomputed on review write/hide
  settings: {
    atRiskDays: number,        // default 30
    triggers: { birthday: boolean, winBack: boolean, postFirstOrder: boolean }
  },
  status: 'active' | 'archived'
}
// Indexes: {slug:1} unique; {city:1, status:1}; {location:'2dsphere'}
```

## 3. `categories` and `items` — catalog

```ts
// categories
{ vendorId, name: string, sortOrder: number, isActive: boolean }
// Index: {vendorId:1, sortOrder:1}

// items
{
  vendorId, categoryId: ObjectId,
  name: string, description?: string, photoUrl?: string,
  isVeg: boolean,
  basePrice: number,                       // paise; price when no variants
  variants: [{ _id, name: string, price: number, isDefault: boolean }],   // embedded, ≤20
  addonGroups: [{                           // embedded, ≤10 groups × ≤20 addons
    _id, name: string, minSelect: number, maxSelect: number,
    addons: [{ _id, name: string, price: number, isAvailable: boolean }]
  }],
  prepTimeMin: number,
  stock: {
    state: 'in_stock' | 'out_of_stock' | 'limited',
    count?: number                          // required iff state === 'limited'
  },
  isActive: boolean, sortOrder: number,
  snapshotVersion: number                   // ++ on any price/variant/addon change
}
// Indexes: {vendorId:1, categoryId:1, sortOrder:1}; {vendorId:1, isActive:1}
```

Variants/add-ons are embedded (always fetched with the item, bounded size). `snapshotVersion` lets the order flow detect "menu changed since customer loaded it".

## 4. `orders`

```ts
{
  vendorId, customerId: ObjectId,
  orderCode: string,             // human code per vendor per day, e.g. "A-042" — shown at counter
  status: 'placed'|'accepted'|'preparing'|'ready'|'completed'
        | 'rejected'|'cancelled'|'cancelled_by_vendor',
  statusHistory: [{ status, at: Date, by: 'customer'|'vendor'|'system', reason?: string }],
  fulfilmentType: 'pickup' | 'dine_in',
  tableNumber?: string,          // iff dine_in
  items: [{                      // FULL SNAPSHOT — never reference live catalog for money
    itemId, name: string, isVeg: boolean,
    variant?: { variantId, name, price },
    addons: [{ addonId, name, price }],
    unitPrice: number,           // resolved variant+addons total per unit, paise
    qty: number, lineTotal: number
  }],
  totals: { itemsTotal: number, grandTotal: number },   // taxes/charges join later
  payment: { method: 'counter', status: 'not_applicable', markedPaidByVendor: boolean },
  etaMinutes?: number,           // set at acceptance
  customerNote?: string,         // ≤ 200 chars
  idempotencyKey: string,        // client uuid — see 05 §2
  schemaVersion: 1
}
// Indexes: {vendorId:1, createdAt:-1}; {vendorId:1, status:1, createdAt:-1};
//          {customerId:1, createdAt:-1}; {vendorId:1, idempotencyKey:1} unique;
//          {customerId:1, vendorId:1, status:1}  (review-eligibility check)
```

**Order items are immutable snapshots.** Editing the catalog never changes an existing order.

## 5. `reviews`

```ts
{
  vendorId, customerId, orderId: ObjectId,   // {orderId:1} unique — one review per order
  stars: 1|2|3|4|5, text?: string,           // ≤ 1000 chars
  vendorReply?: { text: string, at: Date },
  status: 'visible' | 'hidden',
  hiddenReason?: string,                     // super_admin moderation
  editableUntil: Date                        // createdAt + 24h
}
// Indexes: {orderId:1} unique; {vendorId:1, status:1, createdAt:-1}; {customerId:1}
```

## 6. `customer_profiles` — per-vendor CRM record (the funnel's fuel)

```ts
{
  vendorId, customerId: ObjectId,        // {vendorId:1, customerId:1} unique
  name: string, phone: string,           // denormalized from user at last order
  firstOrderAt: Date, lastOrderAt: Date,
  orderCount: number, totalSpend: number,
  birthday?: { day, month },             // copied only while user keeps it set
  segment: 'new' | 'repeat' | 'loyal' | 'at_risk',
  segmentComputedAt: Date,
  lastMessagedAt?: Date,
  messagesThisWeek: number,              // reset by weekly cron; enforces per-customer cap
  serviceWindowExpiresAt?: Date,         // last inbound message + 24h; utility sends before this are free (docs/07 §6)
  schemaVersion: 1
}
// Indexes: {vendorId:1, customerId:1} unique; {vendorId:1, segment:1};
//          {vendorId:1, 'birthday.month':1, 'birthday.day':1}; {vendorId:1, lastOrderAt:-1}
```

Created/updated ONLY by the `order.completed` event handler and the segment cron — never written from HTTP controllers.

## 7. `campaigns` and `message_templates`

```ts
// message_templates (platform-managed, mirror of Meta-approved templates)
{ key: string /*unique*/, metaTemplateName: string, language: 'en',
  bodyPreview: string, variables: [{ name: string, example: string }],
  kind: 'birthday'|'win_back'|'post_first_order'|'promo'|'otp'|'order_status',
  status: 'active' | 'retired' }

// campaigns (vendor-initiated manual sends)
{
  vendorId, createdBy: ObjectId,
  templateKey: string, variableValues: Record<string,string>,
  segment: 'all'|'new'|'repeat'|'loyal'|'at_risk',
  scheduledAt: Date, status: 'scheduled'|'running'|'completed'|'cancelled'|'failed',
  counts: { targeted: number, sent: number, failed: number, skippedConsent: number, skippedQuota: number }
}
// Indexes: {vendorId:1, createdAt:-1}; {status:1, scheduledAt:1}
```

## 8. `message_logs` — every WhatsApp message ever attempted

```ts
{
  vendorId?: ObjectId,                  // absent for platform messages (OTP)
  customerId?: ObjectId, phone: string,
  templateKey: string, campaignId?: ObjectId,
  triggerType: 'birthday'|'win_back'|'post_first_order'|'campaign'|'otp'|'order_status',
  dedupeKey?: string,                   // "{customerId}:{triggerType}:{YYYY-MM-DD}" unique sparse
  status: 'queued'|'sent'|'delivered'|'read'|'failed_retryable'|'failed_permanent'|'skipped',
  skipReason?: 'no_consent'|'opted_out'|'weekly_cap'|'vendor_quota'|'suspended_vendor'|'meta_frequency_cap',  // 131049
  sentInServiceWindow?: boolean,        // true = free in-window utility send (cost analytics)
  providerMessageId?: string, error?: { code: string, detail: string },
  attempts: number, schemaVersion: 1
}
// Indexes: {dedupeKey:1} unique sparse; {vendorId:1, createdAt:-1};
//          {providerMessageId:1} sparse (webhook lookups); {status:1, createdAt:1}
```

## 9. `refresh_tokens`, `idempotency` helpers, `audit_logs`

```ts
// refresh_tokens
{ userId, tokenHash: string, familyId: string, deviceInfo?: string,
  expiresAt: Date, revokedAt?: Date, replacedBy?: ObjectId }
// Indexes: {tokenHash:1} unique; {userId:1}; TTL on expiresAt

// audit_logs (append-only: who did what — vendor suspensions, review hides, staff changes, campaign sends)
{ actorId, actorRole, action: string, targetType: string, targetId: ObjectId,
  vendorId?: ObjectId, meta?: object }
// Indexes: {vendorId:1, createdAt:-1}; {actorId:1, createdAt:-1}
```

OTPs and rate-limit counters live in **Redis only** (`otp:{phone}` hash, TTL 300s) — not in Mongo.

## 10. Transactions & atomicity rules (binding)

1. **Order placement** = one multi-document transaction: re-validate every cart line against live items → decrement `limited` stock atomically → insert order. Stock decrement uses a guarded conditional update **inside the transaction**:
   ```ts
   Item.updateOne(
     { _id, vendorId, 'stock.state': 'limited', 'stock.count': { $gte: qty } },
     { $inc: { 'stock.count': -qty } }
   )
   // modifiedCount === 0 → abort txn → 409 OUT_OF_STOCK with the offending line
   // separate follow-up: flip state to 'out_of_stock' when count hits 0
   ```
2. **Status transitions** use a guarded update (`{ _id, vendorId, status: expectedFrom }` → set new status + push history). `modifiedCount === 0` → someone else moved it first → 409 with current state. No read-then-write.
3. **Cancel while accepting** race: both sides use rule 2; exactly one wins by construction.
4. **Stock restore** on reject/cancel of an order that decremented `limited` stock: reverse `$inc` in the same transaction as the status change.
5. **`orderCode` generation:** atomic `findOneAndUpdate` with `$inc` on a per-vendor-per-day counter document (`counters` collection, upsert) — never `count()+1`.
6. **Review creation** relies on the unique `{orderId:1}` index — insert and catch `E11000` → 409 ALREADY_REVIEWED. Never check-then-insert.
7. Rating recompute (`vendors.rating`) runs post-commit from the review event; eventual consistency here is acceptable.

## 11. Migration & seed discipline

- `migrate-mongo` for schema/index migrations; indexes are created ONLY via migrations (autoIndex off in prod).
- Seed script (`npm run seed:dev`) creates: 1 super_admin, 2 vendors with full catalogs, 5 customers, orders in every status, reviews, message logs — realistic dev data from day one.
