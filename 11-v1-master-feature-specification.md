# 11 — V1 Master Feature Specification: Whole POS + Invoicing + QR Order + WhatsApp Retention

> **Working Product Title:** RestroSarthi (codebase: Regulars / LocalServe)  
> **Target Segment:** Independent cafes, bakeries, quick-service eateries (QSRs), and casual dining restaurants in India.  
> **Core Value Proposition:** Your cafe's complete micro-POS & GST billing + QR table ordering + automated WhatsApp retention engine that runs itself — flat monthly subscription, zero commission, own your customer data.

---

## 1. System Architecture & Foundation

### 1.1 Architecture Blueprint
* **Monorepo Layout (`npm workspaces`):**
  * `apps/api`: Node.js 22 LTS, TypeScript Strict, Express 5.
  * `apps/customer-web`: React 18, Vite, TanStack Query, Tailwind CSS (Mobile PWA).
  * `apps/vendor-web`: React 18, Vite, TanStack Query, Zustand, Tailwind CSS (Desktop/Tablet POS & Dashboard).
  * `apps/admin-web`: React 18, Vite (Internal super_admin panel).
  * `packages/shared`: Single source of truth for Zod schemas, TypeScript types, state machine enums, error codes, and currency/phone normalization utils.
* **Database & Persistence:**
  * **MongoDB 7+ Replica Set (via Mongoose 8):** Multi-document ACID transactions for atomic order placement, stock decrement, and invoice generation.
  * **Redis 7 + BullMQ:** Ephemeral storage only — BullMQ WhatsApp message queues, repeatable cron jobs, OTP verification hashes (5-min TTL), and route rate limiters. Zero durable business data lives strictly in Redis.
* **Real-time Synchronization:**
  * **Socket.IO:** Dedicated namespaces (`/vendor` and `/customer`) with room authorization (`vendor:{vendorId}` and `session:{sessionId}`). Emits on committed DB writes; client frontends maintain a 30s polling reconciliation fallback.
* **Multi-Tenancy & Security Invariant:**
  * Strict tenant context: `vendorId` is extracted exclusively from the validated JWT via `tenantContext` middleware into `req.tenant.vendorId`. 
  * Data layer uses `scopedModel(Model, vendorId)` wrapper. Direct unscoped queries in vendor routes are forbidden and enforced via automated tenant-leak matrix tests.
* **Money & Time Conventions:**
  * All currency amounts are stored as **integer paise** (₹1.00 = 100 paise). Display formatting via `formatINR()` only at UI boundaries.
  * All timestamps stored in **UTC**, displayed in **Asia/Kolkata (IST)**.

---

## 2. V1 Feature Breakdown by Domain

### Module A: Outlet Setup & Table Management
1. **Outlet Profile:**
   * Cafe name, slug (`/v/{slug}`), logo, banner cover photo, phone, address, geo-coordinates.
   * Tax & Legal: GSTIN (optional at signup, mandatory before first GST invoice), FSSAI license number (printed on bills), state/tax jurisdiction (CGST 2.5% + SGST 2.5% default for restaurant F&B).
   * Operating Hours: Multi-range opening hours per weekday, support for midnight-crossing hours (e.g. 18:00–01:00).
   * Open/Closed Overrides: 1-tap "Force Closed" switch and "Busy Mode" order throttle.
2. **Table & Area Configuration:**
   * Table list with unguessable opaque tokens (e.g. `Table 01` -> `/v/{slug}/t/{token}`).
   * Table grouping / kind: Indoor, Outdoor, Terrace, Counter.
   * **Print-Ready QR Generation:** Generates a downloadable, print-ready PDF sheet of branded table QR cards and counter takeaway stands.
3. **Practice Mode:**
   * A pre-configured "Test Table" allowing staff to place test QR orders, punch tickets, and test thermal printing without polluting financial books or firing live customer WhatsApp messages.

---

### Module B: Menu Builder & Ingestion (The Onboarding Moat)
1. **AI Photo/PDF OCR Extraction:**
   * Staff uploads a photo or PDF of their physical paper menu.
   * Backend invokes an AI vision/document parsing endpoint returning structured categories, items, prices, descriptions, and veg/non-veg flags.
   * Populates an editable staging grid for 1-click review, correction, and instant catalog creation.
2. **Spreadsheet Import / Export:**
   * Downloadable standard Excel/CSV template.
   * Bulk import & validation with error row reporting.
3. **Menu Hierarchy & Customization:**
   * Categories: Reorderable, active/inactive toggles.
   * Items: Name, description, photo URL, base price, GST rate (default 5%), veg/non-veg flag, prep time estimate, bestseller/recommended badge.
   * Variants: Nested option sets (e.g., Small, Medium, Large) with individual prices.
   * Add-on Groups: Embedded modifier groups with strict `minSelect` and `maxSelect` constraints (e.g., "Choose Cheese (Min 1, Max 1)", "Extra Toppings (Min 0, Max 4)").
4. **Availability & Stock States:**
   * Three distinct states: `in_stock`, `out_of_stock`, and `limited` (with numeric count).
   * Guarded atomic decrement on order placement; auto-flips to `out_of_stock` when count hits 0.
   * Stock restoration on order cancellation or kitchen reject.
   * 1-tap item availability toggle with optional "Auto-restore next morning" schedule.
   * Bulk-edit grid for fast multi-item price or stock updates.
5. **Live Phone Preview:**
   * Interactive mobile simulator in the vendor dashboard showing how the menu appears on a diner's smartphone, with automatic validation warnings (e.g., missing photo, truncated title).

---

### Module C: Customer QR Ordering (Mobile PWA)
1. **Zero-App Instant Access:**
   * Diner scans Table QR (`/v/{slug}/t/{token}`) -> Opens instant responsive PWA.
   * Table token auto-attaches to the browsing session.
   * Diners browse full menu, filter by Veg/Non-Veg, and search items without logging in.
2. **Cart & Customization:**
   * Add items, select variants and add-on modifiers.
   * Special cooking instructions note (up to 200 characters).
   * Cart stored in local session with version validation (`snapshotVersion`) to prevent price drift.
3. **Authentication & Identity Capture:**
   * Triggered only when the diner taps "Place Order".
   * First-time diner: Phone number + WhatsApp/SMS OTP (6 digits, 5-min TTL) + Name + WhatsApp consent checkbox.
   * Returning diner: Recognized by session; 1-tap confirmation.
4. **Table Sessions & Order Rounds:**
   * Rounds stack onto Table's shared active tab.
   * Multiple guests at Table 4 can authenticate and place rounds into Table 4's running tab.
5. **Live Order Tracking:**
   * Live status updates via Socket.IO: `Placed` -> `Accepted (with ETA)` -> `Preparing` -> `Ready / Served`.
   * Order cancellation allowed only while still in `Placed` state.
6. **In-Session Table Services:**
   * **"Call Waiter" Button:** Sends instant audio/visual alert to Order Desk ("Table 4 requests assistance").
   * **"Request Bill" Button:** Notifies cashier that Table 4 is ready to settle; displays running itemized tab with breakdown.
7. **Takeaway / Counter Ordering Mode:**
   * Customers scanning Counter QR can place takeaway orders.
   * Assigned a daily sequential token number (e.g., `#A-14`).
   * Receives automated notification when order status moves to `Ready for Pickup`.

---

### Module D: Order Desk (Dual-Mode POS Staff Interface)
1. **Dual-Mode Operational Layout:**
   * **Live Table Floor Grid:** Real-time visual cards for all tables showing current state:
     * *Empty / Available*
     * *Occupied / Eating* (shows seated duration and running tab total in ₹)
     * *Bill Requested* (pulsing visual alert indicating guest is ready to pay)
     * *Assistance Needed* (alert triggered by "Call Waiter")
   * **Live Orders Kanban Board:** Columns for incoming orders:
     * *New Orders* (with persistent audio chime & flashing visual indicator until accepted)
     * *Preparing* (shows elapsed time and kitchen ETA countdown)
     * *Ready / Served*
     * *Completed*
2. **Order Lifecycle Actions:**
   * Accept order with pre-filled or customized ETA (minutes).
   * Update status: Preparing -> Ready -> Completed.
   * Reject / Cancel order with mandatory reason dropdown (e.g., "Ingredient Out of Stock"), automatically restoring any limited inventory.
   * Reassign table number if guests switch seats.
   * One-tap "Busy Mode" order throttle to pause incoming QR orders during extreme kitchen rushes.
3. **Staff Quick-Punch Billing (Walk-ins & Phone Orders):**
   * Visual fast-punch menu dialog allowing staff to select items, add customer phone number, assign to a table or counter takeaway, and immediately send to kitchen or settle.

---

### Module E: Billing, Invoicing & Settlement
1. **GST-Compliant Tax Invoicing:**
   * Sequential tax invoice numbering per outlet per financial year (e.g., `INV-2627-0042`).
   * Computes itemized CGST, SGST, round-off paise, and voluntary tip.
   * Displays Cafe legal name, address, GSTIN, and FSSAI number.
2. **Payment Collection & Recording:**
   * **Configurable Gateway with Counter Fallback:**
     * *Counter Recording:* Staff records payment mode: Cash, UPI, or Card swiper.
     * *Direct Dynamic Cafe UPI QR:* Bill displays a dynamic UPI QR code encoded with cafe's VPA (`upi://pay?pa=...&am=...&tr=...`) for direct, zero-fee payment to cafe's bank account.
     * *Online Payment Gateway:* If cafe inputs their Razorpay API credentials in settings, diners can pay directly in the PWA.
3. **Discounts & Coupons:**
   * Owner-configured promotional codes (e.g., `WELCOME10`, flat ₹ discount, minimum bill value).
   * Staff counter discount tool: apply custom percentage or flat rupee discount, or mark individual item as complimentary with a mandatory reason note.
4. **Receipt Delivery & Printing:**
   * **WhatsApp E-Bill:** Instant automated WhatsApp message containing bill summary, amount paid, and secure link to the digital GST tax invoice.
   * **Browser Thermal Printing (80mm & 58mm):** 1-click "Print Bill" or "Print KOT" formatted via clean CSS `@media print` for standard thermal slip printers without hardware driver dependencies.
5. **Day-End Close & Reconciliation:**
   * Staff initiates "Day Close" at shift conclusion.
   * Summarizes gross sales, net sales, taxes collected, discounts, and voids.
   * Payment mode breakdown: Cash recorded vs. physical cash counted in drawer (computes variance over/short).
   * Generates printable and downloadable PDF Day-Close report with closing staff audit sign-off.

---

### Module F: WhatsApp Retention Funnel & CRM Engine
1. **WhatsApp Infrastructure (GoKwik-Style Onboarding):**
   * Supports Meta Embedded Signup: Cafes connect their own WhatsApp Business Account (WABA) or onboard via standard Meta Cloud API.
   * Displays cafe's own verified brand name and protects quality rating isolation.
   * Webhook router dynamically routes incoming message statuses and replies via `phoneNumberId`.
2. **Customer CRM Graph (Per-Vendor Isolation):**
   * Built automatically on every completed order/bill.
   * Attributes: Name, verified phone, total visits, total spend, first visit date, last visit date, birthday.
   * Rule-based dynamic segmentation:
     * `New`: 1 visit
     * `Repeat`: 2–4 visits
     * `Loyal`: 5+ visits
     * `At-Risk`: No visit in 30+ days (configurable threshold)
   * Vendor dashboard shows masked phone numbers (`98•••••210`) to safeguard customer privacy; full data export restricted.
3. **Core Automated Retention Triggers (Default-On):**
   * **Post-First-Visit Thank You:** Automated WhatsApp message sent 2 hours after first completed visit offering a welcoming next-visit perk.
   * **Birthday Offer:** Automated morning greeting at 08:00 AM IST on the customer's birthday with a special treat.
   * **30-Day Win-Back:** Automated nudge to customers entering the `At-Risk` segment (limited to max 1 win-back per customer per 45 days).
4. **Segmented Manual Campaigns:**
   * Owner selects target audience (`New`, `Repeat`, `Loyal`, `At-Risk`, or `All`).
   * Selects pre-approved Meta template and populates offer variables (discount %, validity date).
   * Enqueued into BullMQ background pipeline with strict dedupe keys and execution throttles.
5. **Safety Caps, Opt-Out & Meta Error 131049 Handling:**
   * Global per-customer weekly message cap (default max 2 marketing messages/week).
   * Immediate handling of `STOP` replies: revokes `whatsappOptIn` across all marketing triggers immediately.
   * Meta frequency cap response (Error `131049`): Logged cleanly as `skipped_meta_cap` and displayed transparently in campaign statistics.
6. **Attributed Revenue Ledger (The SaaS Retention Spine):**
   * Automatically tracks orders placed by customers who received a trigger or campaign within the attribution window.
   * Dashboard tile: "Generated ₹X from Y orders via WhatsApp this month".

---

### Module G: Verified Reviews & Smart Feedback Routing
1. **Eligibility & Fraud Guard:**
   * Gated strictly to verified diners with a `completed` order/bill at that vendor.
   * Limit: exactly 1 review per bill, editable for up to 24 hours.
2. **Smart Review Routing (The Reputation Engine):**
   * **4 or 5 Stars:** Prompts diner with a 1-tap deep link to the cafe's Google Business Maps profile, driving organic search footfall.
   * **1, 2, or 3 Stars:** Intercepted as private feedback; alerts owner dashboard immediately to resolve diner grievances before public negative reviews are posted.
3. **Public Storefront Display:**
   * Visible reviews and calculated average rating displayed on the cafe's PWA menu.
   * Owner can publish 1 official public reply per review. Super admin holds moderation hide privileges.

---

### Module H: Administration & Staff Roles
1. **Role-Based Access Control (RBAC):**
   * `customer`: Authenticated via phone OTP; access limited to own orders, bills, and reviews.
   * `vendor_staff`: Email/password login; access strictly limited to Order Desk (Table Grid, Kanban, Staff Punch, Stock Toggles).
   * `vendor_admin`: Cafe owner; full control over catalog, staff management, payment settings, campaigns, and financial reports.
   * `super_admin`: Internal platform team; manual onboarding, subscription status toggling (`trial`, `active`, `past_due`, `suspended`), template approvals, and cross-tenant telemetry.
2. **Suspension Enforcement:**
   * Suspended vendor accounts immediately hide public storefronts from customer browsing and block all staff order writes with HTTP `403 FORBIDDEN`.

---

## 3. Clear Boundaries: Roadmap Definement

```
┌──────────────────────────────────────────────────────────────────────────┐
│                               IN V1 CORE                                 │
├──────────────────────────────────────────────────────────────────────────┤
│ • Full Table Session & Running Tab management                            │
│ • Live Order Desk (Table Floor Grid + Orders Kanban + Audio Alarms)      │
│ • Staff Manual Quick-Punch for walk-in & phone orders                    │
│ • GST Invoicing (CGST/SGST/FSSAI/Sequential Numbering)                   │
│ • Counter Settlement (Cash / Card / Dynamic Cafe UPI QR / Razorpay)      │
│ • Browser Thermal Slip Printing (80mm / 58mm Bill & KOT)                 │
│ • WhatsApp E-Bills & In-Session Service ("Call Waiter", "Request Bill")  │
│ • AI Photo/PDF Menu OCR Extraction + Excel Bulk Import                   │
│ • GoKwik-style WABA Onboarding & WhatsApp Retention Pipeline             │
│ • 3 Default-On Automated Triggers + Manual Campaigns + Revenue Ledger   │
│ • Smart Review Routing (4-5★ to Google Maps, 1-3★ to Private Alert)      │
│ • Full Day-End Close & Cash Drawer Variance Reconciliation               │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                         DEFERRED TO V1.1 / V1.5                          │
├──────────────────────────────────────────────────────────────────────────┤
│ • Raw ESC/POS hardware print driver integration (WebUSB/auto-cut)        │
│ • Move / Merge table sessions & customer-side split bill payments        │
│ • Owner Morning WhatsApp Briefing & Automated Dead-Hour Campaign Filler  │
│ • Multi-outlet brand chaining & hotel room service billing               │
│ • Deep inventory / raw ingredient recipe level stock depletion           │
│ • Customer points-based loyalty wallet                                   │
│ • Petpooja / external POS bi-directional sync adapter                    │
└──────────────────────────────────────────────────────────────────────────┘
```
