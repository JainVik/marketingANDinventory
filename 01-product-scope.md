# 01 — Product Scope (v1 Master Spec)

> **Working name:** RestroSarthi (codebase: Regulars / LocalServe)  
> **One-liner:** A multi-tenant micro-POS, GST invoicing, QR table ordering, and automated WhatsApp retention platform for independent Indian cafes, bakeries, and QSRs — without them building an app or getting squeezed by aggregators.  
> **Launch strategy:** Single city pilot. Vendors pay a flat subscription; customers use it free.

This document defines WHAT v1 does. It deliberately excludes HOW (see `02-architecture.md`). Anything not listed under "In scope" is OUT of v1.

---

## 1. Actors & Roles

| Role | Description | Access surface |
|---|---|---|
| `customer` | End user who scans QR, browses menu, places rounds, and tracks bills | Customer PWA (Mobile) |
| `vendor_admin` | Cafe owner (the tenant). Full control over catalog, staff, payment settings, campaigns, and reports | Vendor dashboard & Order Desk |
| `vendor_staff` | Employee added by vendor_admin. Can operate Order Desk (Table Grid, Orders Kanban, Walk-in Punch, Stock toggles) and perform Day Close. No catalog/campaign/settings access | Vendor dashboard (restricted) |
| `super_admin` | Platform operator (us). Onboards/suspends vendors, manages subscriptions, moderates reviews, views platform metrics | Admin panel (minimal internal UI) |

A single human may hold `customer` and `vendor_admin` accounts, but they are **separate identities** (different login flows).

---

## 2. In Scope — v1 Modules

### 2.1 Vendor Onboarding & Outlet Profile
- Outlet profile: name, slug (`/v/{slug}`), logo, cover image, address, city, geo-coordinates, operating hours per weekday (supports midnight-crossing hours), GSTIN, and FSSAI license number.
- Operational switches: 1-tap "Force Closed" switch and "Busy Mode" order throttle.
- Practice Mode: Pre-configured test table to test order flows and thermal receipts without polluting live sales or firing live customer WhatsApp messages.
- Super admin sets subscription status (`trial`, `active`, `past_due`, `suspended`). Billing collection is manual in v1. Suspended vendor: storefront hidden, ordering blocked, dashboard writes 403.

### 2.2 Table Management & QR Generation
- Table list with cryptographically unguessable tokens (e.g. `Table 04` -> `/v/{slug}/t/{token}`).
- Table grouping: Indoor, Outdoor, Terrace, Counter.
- **Print-Ready QR Generation:** 1-click generation of a downloadable/printable PDF sheet of branded table cards with crisp QR codes and counter takeaway stands.

### 2.3 Catalog & AI Menu Builder (The Onboarding Moat)
- **AI Photo/PDF OCR Extraction:** Upload paper menu photo or PDF -> AI vision extracts categories, items, prices, veg/non-veg tags into an editable staging grid for 1-click review and commit.
- **Spreadsheet Import / Bulk Grid:** Standard Excel/CSV template import + bulk-edit grid (select many -> change price / tax / stock).
- Catalog structure: Categories -> items -> embedded variants (Small/Large) -> add-on groups (with min/max rules).
- Item fields: name, description, photo URL, base price (paise), GST rate (default 5%), veg/non-veg flag, prep time, bestseller badge, sort order, `isActive`.
- Stock states: `in_stock`, `out_of_stock`, and `limited` (with atomic decrement and auto-flip to `out_of_stock` at 0). 1-tap availability toggle with optional "Auto-restore next morning".
- **Live Phone Preview:** Interactive mobile simulator in the dashboard showing real-time diner view with validation warnings.

### 2.4 Table Sessions & Customer Ordering (Mobile PWA)
- Instant access: scan table QR -> menu opens with table auto-attached (no login to browse).
- Cart per phone, special cooking instructions (≤ 200 chars).
- Phone-OTP on order placement (first-timer: phone + WhatsApp/SMS OTP + name + WhatsApp consent checkbox; returning diner: 1-tap).
- **Table Sessions & Rounds:** Diners place rounds that stack onto the table's shared active tab. Multiple guests at the same table can submit rounds.
- In-Session Table Services:
  - **"Call Waiter" Button:** Triggers instant assistance alert on the Order Desk.
  - **"Request Bill" Button:** Notifies cashier table is ready to settle; displays running itemized bill.
- Takeaway / Counter Mode: Counter QR scan issues daily sequential token (e.g., `#A-14`) with live ready notifications.

### 2.5 Order Desk (Dual-Mode Staff POS)
- **Live Table Floor Grid:** Real-time visual floor cards showing table states: *Empty*, *Occupied & Eating* (shows running tab ₹ total and seated duration), *Bill Requested*, and *Assistance Needed*.
- **Live Orders Kanban:** Columns for *New Orders* (with loud persistent audio chime until accepted), *Preparing*, *Ready*, and *Completed*.
- Order lifecycle actions: Accept with ETA, prepare, ready, served, reject/cancel with mandatory reason (restores limited stock).
- **Staff Quick-Punch Billing:** Visual menu popup to punch walk-in counter orders or manual table rounds.
- Table reassign tool & one-tap "Busy Mode" order throttle.

### 2.6 Billing, Invoicing & Settlement
- **GST-Compliant Tax Invoices:** Sequential numbering per FY (e.g. `INV-2627-0042`), itemized CGST (2.5%), SGST (2.5%), round-off paise, voluntary tip, cafe GSTIN and FSSAI.
- **Payment Collection & Recording:**
  - *Counter Recording:* Staff records payment mode: Cash, UPI, or Card.
  - *Direct Cafe Dynamic UPI QR:* Bill displays a dynamic UPI QR with cafe's VPA (`upi://pay?pa=...`) for instant, zero-commission payment.
  - *Configurable Gateway:* Optional Razorpay keys allow automated in-app payment.
- **Discounts & Coupons:** Owner-created coupon codes (e.g. `WELCOME10`, flat ₹ off) + staff counter discount tool (% or flat ₹) + complimentary items with reason.
- **Receipts & Printing:**
  - *WhatsApp E-Bill:* Instant digital bill sent via WhatsApp with link to tax invoice.
  - *Browser Thermal Printing:* 1-click "Print Bill" or "Print KOT" via standard browser print formatted for 80mm/58mm thermal rolls.
- **Day-End Close & Reconciliation:** Shift closure summary: gross/net sales, GST collected, payment mode totals, and physical cash drawer variance balancing (over/short).

### 2.7 WhatsApp Retention Funnel (GoKwik-Style WABA)
- **WABA Architecture:** Cafes connect their own WhatsApp Business Account via Meta Embedded Signup / Cloud API, displaying their own verified brand name and isolating quality ratings.
- **Per-Vendor Customer Graph:** Auto-built on every completed order/bill: name, masked phone, visit count, total spend, birthday, dynamic segments (`new`, `repeat`, `loyal`, `at_risk`).
- **Core Automated Triggers (Default-On):**
  1. *Post-first-visit thank you* with next-visit offer (sent 2h post completion).
  2. *Birthday treat* (sent 08:00 AM IST morning of birthday).
  3. *30-day Win-back* for at-risk regulars (max 1 per 45 days).
- **Manual Segmented Campaigns:** Broadcast targeted offers to segments with pre-approved Meta templates.
- **Frequency Caps & Opt-Out:** Max 2 marketing messages/week/customer. `STOP` reply revokes consent platform-wide immediately. Meta error 131049 handled cleanly as `skipped_meta_cap`.
- **Attributed Revenue Ledger:** Dashboard tile proving exact rupees generated and orders driven by WhatsApp marketing.

### 2.8 Verified Reviews & Smart Feedback Routing
- Gated strictly to verified diners with a `completed` order/bill (1 review per bill, editable 24h).
- **Smart Routing:**
  - *4 or 5 Stars:* 1-tap deep link to cafe's Google Business Maps profile to drive SEO and organic footfall.
  - *1, 2, or 3 Stars:* Captured as private feedback alerting owner dashboard to resolve grievances before public negative reviews.

---

## 3. Explicitly OUT of v1 (Do Not Build)

- Native raw ESC/POS hardware print driver / auto-cut integration (browser thermal print used instead).
- Move / merge tables & customer-side split bill payments (deferred to v1.1).
- White-label delivery logistics & rider tracking.
- Table reservations / advance bookings.
- Multi-outlet brand chaining (1 vendor = 1 outlet in v1).
- Customer-side native iOS/Android apps (PWA only).
- Deep raw ingredient recipe-level inventory depletion (item-level stock states used instead).
- Customer points-based loyalty wallet (segmented rule-based retention used instead).
- Petpooja / external POS bi-directional sync adapters.
