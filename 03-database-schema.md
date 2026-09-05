# 03 — MongoDB Schema (v1 Master Spec)

Data shapes for v1. Mongoose 8 + TypeScript; every schema gets `{ timestamps: true }` (`createdAt`/`updatedAt`). All money = **integer paise** (₹1 = 100 paise). All dates = UTC `Date`. IDs are `ObjectId` unless stated.

Conventions:
- Tenant-owned collections carry `vendorId` and every index on them **starts with `vendorId`**.
- `schemaVersion: number` field on orders, table_sessions, invoices, customer_profiles, message_logs (documents that will migrate).
- No cross-collection joins in hot paths — denormalize the few read-often fields listed here, nothing more.

---

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

---

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
  gstin?: string,          // optional at signup, required before first GST invoice
  fssaiLicense?: string,   // printed on bills and storefront
  googlePlaceId?: string,  // for smart 4-5★ review redirection to Google Maps
  openingHours: [{ day: 0-6, open: 'HH:mm', close: 'HH:mm' }],  // multiple ranges/day allowed
  isOpenOverride: 'auto' | 'force_closed',     // the "temporarily closed" switch
  isBusyMode: boolean,                         // the "busy mode" order throttle
  subscription: {
    plan: 'basic' | 'growth',
    status: 'trial' | 'active' | 'past_due' | 'suspended',
    startedAt: Date, renewsAt?: Date, notes?: string   // billing is manual in v1
  },
  quotas: { dailyMessages: number },            // resolved from plan at write time
  paymentSettings: {
    upiVpa?: string,                            // cafe's direct UPI ID for dynamic QR on bills
    razorpay?: {
      enabled: boolean,
      keyId?: string,
      keySecretEncrypted?: string
    }
  },
  whatsapp: {
    wabaId?: string,                            // vendor's own WABA (GoKwik style) or platform
    phoneNumberId?: string,
    businessAccountId?: string,
    accessTokenEncrypted?: string,
    status: 'unconnected' | 'connected' | 'suspended',
    qualityRating?: 'GREEN' | 'YELLOW' | 'RED' | 'UNKNOWN'
  },
  rating: { avg: number, count: number },       // denormalized, recomputed on review write/hide
  settings: {
    atRiskDays: number,                         // default 30
    defaultGSTRate: number,                     // default 5 (5% = 2.5% CGST + 2.5% SGST)
    triggers: { birthday: boolean, winBack: boolean, postFirstOrder: boolean }
  },
  status: 'active' | 'archived'
}
// Indexes: {slug:1} unique; {city:1, status:1}; {location:'2dsphere'}; {'whatsapp.phoneNumberId':1} sparse
```

---

## 3. `tables` — physical floor configuration

```ts
{
  vendorId: ObjectId,
  name: string,            // e.g. "T-01", "Table 4", "Terrace 2"
  section?: string,        // e.g. "Main Hall", "Outdoor", "Terrace"
  token: string,           // cryptographically unguessable token for QR URL /v/{slug}/t/{token}
  sortOrder: number,
  isActive: boolean
}
// Indexes: {vendorId:1, token:1} unique; {vendorId:1, isActive:1, sortOrder:1}
```

---

## 4. `categories` and `items` — catalog

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
  gstRate: number,                         // percentage (default 5)
  isBestseller?: boolean,                  // badge on customer PWA
  variants: [{ _id, name: string, price: number, isDefault: boolean }],   // embedded, ≤20
  addonGroups: [{                           // embedded, ≤10 groups × ≤20 addons
    _id, name: string, minSelect: number, maxSelect: number,
    addons: [{ _id, name: string, price: number, isAvailable: boolean }]
  }],
  prepTimeMin: number,
  stock: {
    state: 'in_stock' | 'out_of_stock' | 'limited',
    count?: number,                         // required iff state === 'limited'
    autoRestoreNextDay?: boolean            // auto-restores to in_stock on morning cron
  },
  isActive: boolean, sortOrder: number,
  snapshotVersion: number                   // ++ on any price/variant/addon change
}
// Indexes: {vendorId:1, categoryId:1, sortOrder:1}; {vendorId:1, isActive:1}
```

---

## 5. `table_sessions` — running tabs & table state

```ts
{
  vendorId: ObjectId,
  tableId?: ObjectId,                       // null for direct counter takeaway sessions
  tableNumber?: string,                     // e.g. "T-04" or "Takeaway #12"
  type: 'dine_in' | 'takeaway',
  sessionToken: string,                     // unique session reference
  status: 'active' | 'bill_requested' | 'settled' | 'voided',
  orderIds: [ObjectId],                     // all rounds placed in this session
  openedAt: Date,
  closedAt?: Date,
  billRequestedAt?: Date,
  serviceCalls: [{                          // "Call Waiter" / "Request Bill" alerts
    at: Date,
    type: 'call_waiter' | 'request_bill',
    status: 'pending' | 'acknowledged',
    acknowledgedBy?: ObjectId
  }],
  runningTotal: number,                     // paise; sum of accepted item lines
  activeCustomers: [{                       // diners who submitted rounds to this tab
    customerId: ObjectId,
    phone: string,
    name?: string
  }],
  schemaVersion: 1
}
// Indexes: {vendorId:1, status:1}; {vendorId:1, sessionToken:1} unique;
//          {vendorId:1, tableId:1, status:1}
```

---

## 6. `orders` — individual order rounds

```ts
{
  vendorId, customerId?: ObjectId,          // customerId absent for anonymous staff walk-ins
  sessionId: ObjectId,                      // references table_sessions
  roundNumber: number,                      // 1, 2, 3...
  placedBy: 'customer' | 'staff',
  staffId?: ObjectId,                       // iff placedBy === 'staff'
  orderCode: string,                        // human daily code per vendor, e.g. "A-042"
  status: 'placed'|'accepted'|'preparing'|'ready'|'completed'
        | 'rejected'|'cancelled'|'cancelled_by_vendor',
  statusHistory: [{ status, at: Date, by: 'customer'|'vendor'|'system', reason?: string }],
  fulfilmentType: 'pickup' | 'dine_in',
  tableNumber?: string,
  items: [{                                 // FULL SNAPSHOT — never reference live catalog for money
    itemId, name: string, isVeg: boolean,
    variant?: { variantId, name, price },
    addons: [{ addonId, name, price }],
    unitPrice: number,                      // resolved variant+addons total per unit, paise
    qty: number, lineTotal: number,
    gstRate: number
  }],
  totals: { itemsTotal: number, grandTotal: number },
  etaMinutes?: number,                      // set at acceptance
  customerNote?: string,                    // ≤ 200 chars
  idempotencyKey: string,                   // client uuid — see 05 §2
  schemaVersion: 1
}
// Indexes: {vendorId:1, createdAt:-1}; {vendorId:1, status:1, createdAt:-1};
//          {sessionId:1}; {vendorId:1, idempotencyKey:1} unique;
//          {customerId:1, vendorId:1, status:1}
```

---

## 7. `invoices` — GST compliant tax bills

```ts
{
  vendorId: ObjectId,
  sessionId: ObjectId,
  invoiceNumber: string,                    // e.g. "INV-2627-0042" (sequential per FY)
  financialYear: string,                    // e.g. "2026-2027"
  date: Date,
  customer?: {
    customerId?: ObjectId,
    phone: string,
    name?: string
  },
  tableNumber?: string,
  items: [{
    itemId: ObjectId, name: string, isVeg: boolean,
    qty: number, unitPrice: number, lineTotal: number,
    gstRate: number
  }],
  totals: {
    itemsTotal: number,                     // paise; sum of line items before tax & discount
    discountTotal: number,                  // paise
    subTotal: number,                       // itemsTotal - discountTotal
    cgst: number,                           // paise (typically 2.5%)
    sgst: number,                           // paise (typically 2.5%)
    roundOff: number,                       // paise (+/-)
    grandTotal: number                      // paise (payable integer)
  },
  discount?: {
    code?: string,
    type: 'coupon' | 'staff_percent' | 'staff_flat' | 'complimentary',
    percent?: number,
    amount: number,
    reason?: string                         // mandatory for complimentary/manual discounts
  },
  payment: {
    method: 'cash' | 'upi' | 'card' | 'razorpay',
    status: 'paid',
    paidAt: Date,
    transactionRef?: string,
    collectedBy: ObjectId                   // staff/admin user who marked paid
  },
  ebillWhatsApp: {
    status: 'queued' | 'sent' | 'delivered' | 'failed' | 'skipped',
    messageLogId?: ObjectId
  },
  schemaVersion: 1
}
// Indexes: {vendorId:1, financialYear:1, invoiceNumber:1} unique;
//          {vendorId:1, date:-1}; {sessionId:1} unique
```

---

## 8. `coupons` — promotional discount rules

```ts
{
  vendorId: ObjectId,
  code: string,                             // uppercase, e.g. "WELCOME10"
  discountType: 'percentage' | 'flat',
  value: number,                            // percentage (e.g. 10 for 10%) or paise (e.g. 5000 for ₹50)
  minOrderValue: number,                    // minimum itemsTotal in paise
  maxDiscountAmount?: number,               // cap in paise for percentage discounts
  startsAt: Date,
  expiresAt: Date,
  maxUses?: number,
  usedCount: number,
  isActive: boolean
}
// Indexes: {vendorId:1, code:1} unique; {vendorId:1, isActive:1}
```

---

## 9. `day_closes` — shift & register cash balancing

```ts
{
  vendorId: ObjectId,
  date: string,                             // "YYYY-MM-DD"
  closedAt: Date,
  closedBy: ObjectId,                       // staff/admin user ID
  orderCount: number,
  invoiceCount: number,
  grossSales: number,                       // paise
  discounts: number,                        // paise
  netSales: number,                         // paise
  taxes: { cgst: number, sgst: number },    // paise
  payments: {
    cash: number,                           // system recorded cash
    upi: number,                            // system recorded UPI
    card: number,                           // system recorded card
    razorpay: number                        // online gateway
  },
  cashReconciliation: {
    systemCash: number,                     // paise expected in drawer
    physicalCashEntered: number,            // paise counted by cashier
    variance: number,                       // physicalCashEntered - systemCash (paise)
    notes?: string
  },
  status: 'closed'
}
// Indexes: {vendorId:1, date:1} unique; {vendorId:1, closedAt:-1}
```

---

## 10. `reviews`

```ts
{
  vendorId, customerId, orderId: ObjectId, invoiceId?: ObjectId,
  stars: 1|2|3|4|5, text?: string,           // ≤ 1000 chars
  routing: 'google_prompted' | 'private_feedback', // 4-5★ -> Google; 1-3★ -> private
  vendorReply?: { text: string, at: Date },
  status: 'visible' | 'hidden',
  hiddenReason?: string,                     // super_admin moderation
  editableUntil: Date                        // createdAt + 24h
}
// Indexes: {orderId:1} unique; {vendorId:1, status:1, createdAt:-1}; {customerId:1}
```

---

## 11. `customer_profiles` — per-vendor CRM record

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
  serviceWindowExpiresAt?: Date,         // last inbound message + 24h; utility sends before this are free
  schemaVersion: 1
}
// Indexes: {vendorId:1, customerId:1} unique; {vendorId:1, segment:1};
//          {vendorId:1, 'birthday.month':1, 'birthday.day':1}; {vendorId:1, lastOrderAt:-1}
```

---

## 12. `campaigns`, `message_templates`, `message_logs`

```ts
// message_templates
{ key: string /*unique*/, metaTemplateName: string, language: 'en',
  bodyPreview: string, variables: [{ name: string, example: string }],
  kind: 'birthday'|'win_back'|'post_first_order'|'promo'|'otp'|'ebill'|'order_status',
  status: 'active' | 'retired' }

// campaigns
{
  vendorId, createdBy: ObjectId,
  templateKey: string, variableValues: Record<string,string>,
  segment: 'all'|'new'|'repeat'|'loyal'|'at_risk',
  scheduledAt: Date, status: 'scheduled'|'running'|'completed'|'cancelled'|'failed',
  counts: { targeted: number, sent: number, failed: number, skippedConsent: number, skippedQuota: number },
  attributedStats: { orders: number, revenue: number }
}
// Indexes: {vendorId:1, createdAt:-1}; {status:1, scheduledAt:1}

// message_logs
{
  vendorId?: ObjectId,
  customerId?: ObjectId, phone: string,
  templateKey: string, campaignId?: ObjectId, invoiceId?: ObjectId,
  triggerType: 'birthday'|'win_back'|'post_first_order'|'campaign'|'otp'|'ebill'|'order_status',
  dedupeKey?: string,                   // "{customerId}:{triggerType}:{YYYY-MM-DD}" unique sparse
  status: 'queued'|'sent'|'delivered'|'read'|'failed_retryable'|'failed_permanent'|'skipped',
  skipReason?: 'no_consent'|'opted_out'|'weekly_cap'|'vendor_quota'|'suspended_vendor'|'skipped_meta_cap',
  sentInServiceWindow?: boolean,
  providerMessageId?: string, error?: { code: string, detail: string },
  attempts: number, schemaVersion: 1
}
// Indexes: {dedupeKey:1} unique sparse; {vendorId:1, createdAt:-1};
//          {providerMessageId:1} sparse; {status:1, createdAt:1}
```

---

## 13. Transactions & Atomicity Rules (Binding)

1. **Order Placement inside a Table Session:**
   One multi-document MongoDB transaction:
   - Verify `table_sessions` is `active` (not settled).
   - Re-validate every cart line against live items.
   - Decrement `limited` stock atomically (`$inc: -qty` with `$gte: qty`).
   - Insert `orders` document.
   - `$push: { orderIds: order._id }` and `$inc: { runningTotal: order.totals.itemsTotal }` into `table_sessions`.
2. **Invoice Generation & Bill Settlement:**
   One multi-document transaction:
   - Guarded transition on `table_sessions`: `{ _id, vendorId, status: { $in: ['active', 'bill_requested'] } }` -> set `status: 'settled'`.
   - Atomic `$inc` on `counters` collection with `{ vendorId, key: "invoice:" + financialYear }` to generate the next integer invoice number (e.g. 42 -> `"INV-2627-0042"`). Never `count()+1`.
   - Insert immutable `invoices` document.
   - Emit `table.settled` and `invoice.created` domain events (which enqueue the WhatsApp E-Bill job).
3. **Status Transitions & Concurrency Guards:**
   Guarded update: `{ _id, vendorId, status: expectedFrom }` -> set new status. `modifiedCount === 0` -> 409 `CONFLICT_STATE`.
4. **Stock Restoration:**
   On order rejection or cancellation: reverse `$inc` stock in the same transaction as the status update.
5. **Day-End Close:**
   Upsert unique `{ vendorId, date }` in `day_closes` to forbid duplicate conflicting day closures.
