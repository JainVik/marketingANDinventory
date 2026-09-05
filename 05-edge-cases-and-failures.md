# 05 — Edge-Case & Failure Catalog (v1 Master Spec)

This is the binding failure catalog: the things AI-written code silently gets wrong. Every case below is **binding**: each one must have explicit handling in code AND (where marked 🧪) an automated test. When implementing a module, re-read that module's section here first.

Format: **Case → Required behavior.**

---

## 1. Auth & Sessions

1. Expired access JWT → 401 `TOKEN_EXPIRED`; frontends auto-refresh once and retry the request transparently; refresh also failed → hard logout to login screen. 🧪
2. Malformed/absent/garbage JWT (`null`, `undefined`, truncated) → 401 `UNAUTHENTICATED`, never a 500 stack trace. 🧪
3. Refresh-token reuse (old rotated token replayed — possible theft) → revoke entire token family, all devices logged out. 🧪
4. OTP: wrong code 3× → invalidate that OTP, require new request. Resend inside 60s cooldown → 429. >5 OTP requests/phone/hour → 429. Codes are single-use and hashed in Redis. 🧪
5. OTP requested for a phone that is a vendor-staff/admin identity → still fine (separate identity), creates/logs into a *customer* account.
6. Vendor user disabled (`status: 'disabled'`) or vendor suspended while their JWT is still valid → write vendor routes check `subscription.status` via tenantContext (cached 60s). Suspended vendor write → 403. 🧪
7. Same user logged in on two devices — allowed; refresh families are per-device; logout only kills that device's family.
8. Deleted/archived vendor's staff tries to log in → 403 with clear message, not 500 on a null vendor.

---

## 2. Table Sessions & Order Rounds

1. **Two guests at the same table submit rounds concurrently:** Both rounds execute within MongoDB transactions; both check `table_sessions.status === 'active'`; both atomically decrement stock for their respective lines, insert their `order`, and append to `session.orderIds` with `$inc: { runningTotal: order.totals.itemsTotal }`. Zero lost rounds. 🧪
2. **Order submitted while cashier is settling the table:** If the session is already transitioning to `settled`, the round transaction aborts and returns 409 `SESSION_CLOSED`. 🧪
3. **Double-tap on order round submission:** Same `Idempotency-Key` arrives twice. First creates the round; second catches unique index `{vendorId, idempotencyKey}` → returns the existing round with `200` + `Idempotency-Replayed: true`. 🧪
4. **Stock race on limited items:** Two tables order the last cappuccino simultaneously. Guarded `$inc` with `$gte: qty` inside the transaction; loser gets 409 `OUT_OF_STOCK` naming the item; diner cart prompts adjustment. NEVER read-check-then-decrement. 🧪
5. **Menu price changed while diner had items in cart (`snapshotVersion` mismatch):** Returns 409 `PRICE_CHANGED` with fresh item prices; client cart shows old vs. new diff and asks diner to re-confirm. 🧪
6. **Store closed or "Busy Mode" toggled after menu load:** Placement returns 423 `VENDOR_CLOSED`. Open and busy states are computed server-side at placement time only. 🧪
7. **Rapid "Call Waiter" / "Request Bill" taps:** Debounced to max 1 pending request per 60 seconds per table; subsequent taps return 200 with existing pending status. 🧪
8. **Table Reassignment:** Staff moves diner from Table 2 to Table 5: updates `tableId` and `tableNumber` on the active `table_sessions` document; existing rounds and running totals remain fully preserved. 🧪

---

## 3. Order Desk & Kitchen Operations

1. **Two staff members accept the same order simultaneously:** Guarded transition `{ _id, vendorId, status: 'placed' }` → set `accepted`. Loser receives 409 `CONFLICT_STATE` with current state; UI refreshes without error toast. 🧪
2. **Diner cancels at the exact moment staff accepts:** Both use guarded transitions; exactly one wins atomically by construction. 🧪
3. **Staff cancels after accepting (out of stock):** Allowed from `accepted` or `preparing` with mandatory reason; restores `limited` inventory atomically; notifies diner via Socket.IO and WhatsApp. 🧪
4. **Order stuck in `placed` (staff overwhelmed/asleep):** Background auto-expiry job cancels `placed` orders older than 15 minutes with reason `vendor_no_response`, restores stock, and notifies diner. 🧪
5. **Dine-in tab left open overnight (forgot to settle):** Nightly job flags unclosed active sessions older than 12 hours for manual cashier audit / auto-settlement so revenue metrics and table availability aren't blocked.

---

## 4. Invoicing, GST & Day-End Close

1. **Concurrent bill settlements generating sequential invoice numbers:** Sequential invoice numbers (`INV-2627-0042`) generated via atomic `findOneAndUpdate` with `$inc: { seq: 1 }` on `{ vendorId, key: "invoice:" + financialYear }`. Zero duplicate invoice numbers even under concurrent settlements. 🧪
2. **Fiscal Year rollover (April 1st in India):** Financial year key shifts dynamically (e.g. `2026-2027` → `2027-2028`), resetting sequence to `1` automatically.
3. **Rounding paise for GST compliance:** CGST and SGST calculated at 2.5% each on discounted subtotal. Net total rounded to nearest rupee with `roundOff` integer paise recorded explicitly (`totals.roundOff`).
4. **Coupon code abuse & race condition:** Coupon with `maxUses: 100`: Guarded update `updateOne({ _id, usedCount: { $lt: maxUses } }, { $inc: { usedCount: 1 } })`. If `modifiedCount === 0`, coupon is exhausted → 422 `INVALID_COUPON`. 🧪
5. **Day-End Close race condition:** Two cashiers click "Day Close" simultaneously on different tabs: Unique index `{ vendorId, date }` ensures only the first succeeds; second receives 409 `DAY_ALREADY_CLOSED`. 🧪
6. **Negative Cash Variance:** Physical cash entered in drawer is less than system recorded cash: system records negative variance integer paise cleanly and flags the day-close log for owner review.

---

## 5. WhatsApp Retention Pipeline & Meta Integration

1. **Meta API outage or 5xx/429:** Background BullMQ job retries with exponential backoff (5 attempts with jitter), then moves to `failed_retryable` in Dead Letter Queue. NEVER blocks the billing or ordering HTTP request. 🧪
2. **Meta Frequency Cap (Error 131049):** When Meta blocks marketing template delivery due to user fatigue, message status updates to `skipped` with `skipReason: 'skipped_meta_cap'`. Campaign stats reflect this transparently. 🧪
3. **Inbound `STOP` / Opt-Out:** Inbound webhook receiving "STOP" or "UNSUBSCRIBE" flips `whatsappOptIn: false` across all marketing triggers platform-wide immediately. Subsequent marketing jobs skip with `opted_out`. 🧪
4. **Weekly Per-Customer Cap:** Worker checks `customer_profiles.messagesThisWeek` at send time. If `>= 2`, send is skipped with reason `weekly_cap`. 🧪
5. **Birthday deduplication on worker crash:** Cron generates deterministic `dedupeKey`: `"{customerId}:birthday:{YYYY-MM-DD}"`. Unique sparse index prevents duplicate birthday messages even if cron re-runs. 🧪
6. **Leap year birthday (Feb 29):** In non-leap years, cron sends on Feb 28.
7. **Vendor revokes Meta WABA permissions:** Meta Cloud API returns 401/403: system marks vendor `whatsapp.status: 'suspended'` and alerts vendor dashboard with a "Reconnect WhatsApp" banner.

---

## 6. AI Menu OCR Extraction

1. **Blurry / unreadable menu photo:** AI parser returns low confidence scores on specific items: UI flags those rows in yellow/red on the staging grid for mandatory staff review before commit.
2. **Price parsing anomalies (e.g. "150/-", "Rs 150.00"):** Parser normalizes strictly to integer paise (`15000`). If price is missing or unparseable, field is marked required in staging.
3. **Duplicate category names in OCR output:** Staging importer merges items under the existing category rather than creating duplicate categories.

---

## 7. Multi-Tenancy (Leaks are P0)

1. **Vendor A token + Vendor B resource ID:** Returns **404 NOT_FOUND** (never 403, to avoid confirming existence). Enforced by `vendorId` in every query filter from JWT only. 🧪
2. **Tenant isolation in Socket.IO:** Socket joins `vendor:{vendorId}` and `session:{sessionId}` only after strict JWT or sessionToken ownership verification.
3. **Cross-tenant customer profile leakage:** A vendor ONLY ever queries `customer_profiles` where `vendorId === req.tenant.vendorId`. Phone numbers are masked (`98•••••210`) in UI responses.

---

## 8. Money, Time & Text

1. **Integer paise everywhere:** Any floating point number in a price, tax, or total calculation is a review-blocking defect.
2. **UTC storage, IST display:** Database stores UTC `Date`. Cron schedules evaluate at Indian Standard Time (UTC + 05:30). Display formatted as `Asia/Kolkata`.
3. **Phone normalization:** Normalized strictly to E.164 (`+91...`) across all intake points (QR ordering, staff walk-in punch, customer profile).

---

## 9. Infrastructure & Fallbacks

1. **Redis outage:** Ordering and billing continue uninterrupted (ordering does not depend on Redis). WhatsApp message jobs queue in MongoDB or reject with 503 until Redis recovers.
2. **Thermal printer offline / jammed:** Browser native print dialog (`window.print()`) allows cashier to re-click "Print Bill" or "Print KOT" at any time. WhatsApp E-bill is already dispatched digitally.
3. **Customer offline at placement:** Mobile PWA retains cart in localStorage; shows retry button. Idempotency key guarantees that a delayed retry never double-orders.
