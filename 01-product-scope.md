# 01 — Product Scope (MVP v1)

> **Working name:** LocalServe (placeholder — rename globally when decided)
> **One-liner:** A subscription-based, multi-tenant platform that gives independent cafes and small restaurants their own digital ordering storefront, live order management, and a WhatsApp-based customer-retention funnel — without them building an app.
> **Launch strategy:** Single city at launch. Vendors pay a subscription; customers use it free.

This document defines WHAT v1 does. It deliberately excludes HOW (see `02-architecture.md`). Anything not listed under "In scope" is OUT of v1 — do not build it, do not scaffold for it beyond what section 6 allows.

---

## 1. Actors & Roles

| Role | Description | Access surface |
|---|---|---|
| `customer` | End user who discovers vendors and places orders | Customer PWA |
| `vendor_admin` | Owner of a cafe/restaurant (the tenant). Full control of their vendor account | Vendor dashboard |
| `vendor_staff` | Employee added by vendor_admin. Can manage orders and stock toggles only — no catalog edits, no campaigns, no staff management, no settings | Vendor dashboard (restricted) |
| `super_admin` | Platform operator (us). Onboards/suspends vendors, manages subscriptions, moderates reviews, views platform metrics | Admin panel (minimal internal UI) |

A single human may hold `customer` and `vendor_admin` accounts, but they are **separate identities** (different login flows). Do not merge them in v1.

## 2. In Scope — v1 Modules

### 2.1 Vendor onboarding & subscription
- Super admin creates the vendor (tenant), sets city, plan, and subscription status (`trial`, `active`, `past_due`, `suspended`). **Billing collection itself is offline/manual in v1** — no payment gateway. The system only enforces the status.
- `past_due`: dashboard shows warning banner, everything still works. `suspended`: vendor storefront hidden from customers, ordering disabled, dashboard read-only.
- Vendor admin completes profile: name, logo, cover image, address, city, geo-coordinates, opening hours per weekday, FSSAI license number (optional field), contact phone.
- Vendor can toggle the whole store `open` / `temporarily_closed` with one switch, independent of opening hours.

### 2.2 Catalog & inventory (vendor dashboard)
- CRUD: categories → items → variants (e.g., Small/Large) → add-ons (e.g., extra shot, toppings). Add-ons can be grouped with min/max selection rules.
- Item fields: name, description, photo, base price, veg/non-veg flag, variants, add-on groups, prep-time estimate, sort order, `is_active`.
- Stock state per item: `in_stock` | `out_of_stock` | `limited` (with a numeric `stock_count` that auto-decrements on order acceptance and auto-flips to `out_of_stock` at 0). See edge cases doc §3.
- Bulk actions: toggle stock for many items; reorder items within a category.

### 2.3 Order management (vendor dashboard)
- Live order board with new-order sound alert. States and allowed transitions (this is THE canonical state machine — every doc uses it):

```
placed → accepted → preparing → ready → completed
placed → rejected            (vendor, with reason)
placed → cancelled           (customer, only while still 'placed')
accepted/preparing → cancelled_by_vendor (vendor, with reason — e.g., item unavailable)
```

- No other transitions exist. Terminal states: `completed`, `rejected`, `cancelled`, `cancelled_by_vendor`.
- Vendor sets an ETA when accepting (prefilled from item prep times).
- Order types in v1: **pickup** and **dine-in** (table number entered by customer). ~~Delivery~~ is OUT of v1.
- Payment in v1: **pay at counter** only. The order shows the total; settlement is offline. No payment states in the system beyond an optional vendor-side "mark as paid" checkbox for their own bookkeeping.

### 2.4 Customer ordering (PWA)
- Phone-number + OTP login (no passwords for customers). Profile: name, phone, birthday (optional, for the birthday funnel), WhatsApp opt-in consent (explicit checkbox, default OFF).
- Discovery: list of vendors in the selected city; filter by open-now, veg; sort by rating. City selection is manual in v1 (with browser-geolocation as a convenience to preselect the nearest vendor / city — no full geo-search).
- **QR deep-link is the primary entry point:** each vendor gets a QR (printed on tables/counter) that opens their storefront directly at `/v/{vendorSlug}`. Discovery browsing is secondary.
- Storefront: menu with categories, item detail with variants/add-ons, cart (single-vendor cart — adding from a second vendor prompts to clear cart), order placement, live order status screen, order history, re-order button.
- Cart price is re-validated server-side at placement — client-sent prices are never trusted.

### 2.5 Verified reviews
- Only a customer with a `completed` order at that vendor may review it: 1–5 stars + optional text, **one review per order**, editable for 24h after submission.
- Vendor sees reviews and may post one public reply per review. Vendor cannot delete reviews; super_admin can hide a review (moderation) with a reason.
- Vendor rating = mean of visible review stars, shown with count ("4.3 ★ · 128").

### 2.6 Customer retention & WhatsApp funnel
- On every completed order the platform builds/updates a **per-vendor customer profile**: name, phone, order count, last order date, total spend, birthday (if shared), computed segment (`new` = 1 order, `repeat` = 2–4, `loyal` = 5+, `at_risk` = no order in 30 days, configurable per vendor).
- **A vendor only ever sees profiles of customers who ordered from THEM.** Phone numbers are shown masked (`98•••••210`) in the dashboard; full numbers are never exportable in v1.
- Messaging is **WhatsApp template messages via the official WhatsApp Business Platform (Meta Cloud API through a BSP)** — see architecture doc §7. No SMS in v1.
- Automated triggers (vendor can enable/disable each, with a platform-set message template):
  1. Birthday offer (sent morning of birthday, only if opted in).
  2. Win-back (customer crosses `at_risk` threshold; max 1 win-back per customer per 45 days).
  3. Post-first-order thank-you with a next-visit offer (sent a few hours after first `completed` order).
- Manual campaigns: vendor picks a segment, picks an approved template, fills variables (offer text, validity date), schedules or sends. **Hard caps:** per-vendor daily message quota (plan-based), and a customer can be messaged max N times per week across all triggers+campaigns (platform config, default 2).
- Every message requires prior customer opt-in; every template includes opt-out wording; opt-out (customer replies STOP or toggles in profile) is enforced platform-wide immediately.

### 2.7 Funnel additions from market research (docs/07 §11, docs/08 §7)

In v1: **attributed-revenue ledger** on the vendor dashboard ("this platform made you ₹X this month" + per-campaign revenue/orders/reads — the SaaS-retention spine); **frequency-cap handling** (Meta error 131049 → `skipped` with reason `meta_frequency_cap`, shown honestly in campaign counts); **service-window-aware sending** (order-status templates sent inside the free 24h utility window when open); **wa.me chat-first QR variant** behind an experiment flag (opens the 72h free CTWA-style window and captures opt-in in one scan); vendor dashboard gets **order throttle ("busy mode")** alongside the existing ETA control.

Queued for v1.1 (build-ready, not in v1): abandoned-cart/session recovery with the compressed ladder (~10 min → same-meal → next-mealtime; never multi-day discount ladders), one-tap reorder journey ("repeat your last order? → UPI link"), 2nd-visit nudge after first order, slow-day boost campaigns, points-based loyalty with fixed mechanics, Razorpay/UPI incl. WhatsApp in-chat payments, Meta Tech Provider migration to per-vendor WABAs.

## 3. Core Flows (happy paths)

1. **Order:** customer scans QR → storefront → adds items → login via OTP (if not already) → places order → vendor gets alert → accepts with ETA → preparing → ready → customer shows order code at counter, pays → vendor marks completed → customer nudged (in-app) to review.
2. **Win-back:** cron marks customer `at_risk` → trigger enqueues WhatsApp template → customer taps link → lands on vendor storefront with offer banner → orders.
3. **Onboarding:** super_admin creates vendor + admin user → vendor_admin logs in, builds catalog, prints QR from dashboard → goes live.

## 4. Non-Functional Requirements (v1)

- Production-grade from day one: this is not a throwaway prototype. All engineering rules in `CLAUDE.md` are binding.
- Scale target v1: ~200 vendors, ~50k customers, ~5k orders/day, peak ~10 orders/sec platform-wide. Design for 10× that without re-architecture.
- Vendor order board reflects new orders within 2s (real-time channel), customer status screen within 5s.
- Mobile-first UI; PWA installable; storefront usable on low-end Android over 3G (bundle budget in architecture doc).
- All money values in INR paise (integers). All timestamps stored UTC, displayed Asia/Kolkata.
- Languages: English UI in v1; all user-facing strings go through an i18n layer from day one so Hindi can be added without refactoring.

## 5. Explicitly OUT of v1 (do not build)

- Online payments / Razorpay, refunds, settlements
- Delivery logistics, rider tracking
- Table booking / reservations
- Multi-outlet vendors (1 vendor = 1 outlet in v1)
- Customer-side native apps
- Vendor self-signup (super_admin onboards manually)
- Coupons/promo-code engine (offers in v1 are just message text; staff honors them at counter)
- Loyalty points/wallet
- SMS/email channels
- Analytics dashboards beyond basic counters (orders today, revenue today, top items, repeat rate)
- Multi-city per vendor; platform runs in multiple cities, each vendor belongs to exactly one

## 6. Forward-compatibility rules (build for, don't build)

- Every order carries `payment: { method: 'counter', status: 'not_applicable' }` so online payments can be added as new enum values, not a schema change.
- Vendor schema has `outlets` designed as its own collection with `vendorId` even though v1 enforces exactly one.
- Order has `fulfilmentType: 'pickup' | 'dine_in'` enum — `delivery` joins later.
- Message-sending goes through a channel-agnostic `NotificationService` interface even though WhatsApp is the only v1 implementation.
