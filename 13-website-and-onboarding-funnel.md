# 13 — Public Site & Onboarding Funnel (v1)

The public site is `apps/site` (static, SEO). Signup lands in `apps/owner-web` (the wizard). Three doors, one wizard.

## 1. Site — 10 pages

| # | Path | Job of the page | Primary CTA |
|---|---|---|---|
| 1 | `/` | The pitch in 30 s: "In 2026 you order a burger from 4 km away in 30 seconds, but inside the cafe you wave at a waiter." Trap-door thesis, headline demo trio (Regulars Board · dead-hour filler · holdout-proven offers), "month-end: this brought back ₹40,000" | Book a demo · Start free |
| 2 | `/how-it-works` | Photograph your menu → QR on tables → orders land with the table number → every order carries a phone → WhatsApp brings them back. 5 steps, one screenshot each | Start free |
| 3 | `/pricing` | Free forever (items 1–47) vs Paid (48–60, 65). ₹999–2,999 / outlet / month band; WhatsApp messages at cost; no commission; cancel anytime; export everything | Start free |
| 4 | `/qr-ordering` | Product page: order / pay-only / both, accept-gate, pay after acceptance, counter takeaway, GST bill on WhatsApp | Start free |
| 5 | `/for-cafes` | Cafe language: reorder-led, counter + tables, pay-now default, dead hours 3–5 pm | Book a demo |
| 6 | `/for-restaurants` | Dining language: pay at checkout, rounds, table sheet, kitchen speed, run alongside Petpooja for 2 weeks | Book a demo |
| 7 | `/demo` | Calendar embed + 4-field form (name, phone, outlet name, city). Creates a `vendors` row with `signup_door='demo'` and `status='onboarding'` | Book |
| 8 | `/signup` · `/login` | Self-signup (OTP on owner phone + email/password) → wizard. Login → owner app | — |
| 9 | `/contact` | WhatsApp number, email, address | — |
| 10 | `/legal` | Terms (vendor), privacy + DPDP notice (versioned: `noticeVersion` is what OTP consent records), refund policy (restaurant refunds, not us), cookie note | — |

Mega-menu: 3 columns — *Product* (QR ordering, Menu & billing, Customer book, WhatsApp campaigns, Insights & briefing, Owner bot), *For* (Cafes, Restaurants, Hotels — *coming soon* greyed), *Company* (Pricing, Demo, Contact, Legal). "Coming soon" items are greyed, not hidden: Hotels, Inventory, Table booking, Multi-outlet, POS integrations.

Rules: never mention Zomato. Never claim "AI predicts churn". Every page shows a real screenshot from the seed outlet (Brewhouse Jaipur), not mockups.

## 2. Three doors → one wizard

| Door | Who starts it | What exists when the wizard opens |
|---|---|---|
| **Self-signup** | Owner on `/signup` | `vendors` (`signup_door='self'`), owner `users`, outlet shell, `plan='free'` |
| **Demo-led** | We, after the demo call, from the admin panel (`POST /admin/vendors/{id}/onboard-manual`) | Same rows, `signup_door='demo'`, a magic-link login sent on WhatsApp |
| **Manual** | We, for a pilot outlet we set up in person | Same, `signup_door='manual'`; we may complete steps 1–4 for them |

All three land on the same `/vendor/wizard` state; `vendors.wizard_step` remembers where they are.

## 3. Wizard steps

| Step | Screen | Writes | Done when |
|---|---|---|---|
| 0 | Welcome + outlet type (cafe / dining / hotel-ready) | `outlets.type`, `pay_flow_default` (cafe → pay_now, dining → pay_at_checkout) | type chosen |
| 1 | Outlet profile: name, address, hours, open/closed | `outlets.*` | name + address + hours |
| 2 | Menu: **"Photograph your menu. Be taking orders in 20 minutes."** photo / PDF / CSV / type it | `menu_imports` → `menu_items` | ≥ 1 category with ≥ 1 item → `golive_menu` |
| 3 | Tables & QR: how many tables, kind, download the PDF sheet | `tables` | ≥ 1 table + PDF downloaded → `golive_qr` |
| 4 | QR mode: order / pay-only / both | `outlets.qr_mode` | chosen |
| 5 | Test order on the practice table (scan with own phone) | practice session/order/bill | one completed practice order → `golive_test_order` |
| 6 | GST & FSSAI (optional now; required before first real invoice) | `outlets.gstin`, `fssai_license` | skip allowed |
| 7 | Payments: connect gateway (Razorpay/Cashfree onboarding link) — skippable | `outlets.gateway_*` | skip allowed; pay-at-checkout works without it |
| 8 | Customer list import (CSV → column map) + opt-in message preview — skippable | `customer_imports` | skip allowed |
| 9 | WhatsApp: confirm owner phone for the morning briefing + bot | `users.phone`, `users.owner_bot` | phone verified |
| ✓ | **Go live** checklist: menu ✓ · one QR ✓ · one test order ✓ → `outlets.status='live'`, `vendors.live_at` | all three |

Progress bar shows steps 0–9; the go-live card shows only the three checklist items. Every step is resumable; nothing blocks except the three checklist items and GST-before-first-invoice.

## 4. After go-live (first 14 days)

- Day 0: "Your QR is live" WhatsApp to owner with the PDF link.
- Day 1: first morning briefing (even if it says "no orders yet — put the QR on 5 tables today").
- Day 3: if `identified_order_rate_30d` < 30 % → insight `identity_capture` with the pay-only suggestion.
- Day 7: first "who came back" card on the Regulars Board.
- Day 14: paid-features preview: lapsed wall count + ₹, with "start paid plan" (manual billing in v1).

## 5. Events this funnel writes

`vendor_signed_up` (door), `wizard_step_completed` (step), `menu_imported`, `qr_downloaded`, `practice_order_completed`, `went_live`, `gateway_connected`, `customers_imported` — all in `app.events` with `subject_type='vendor'`… (add these to the `app.event_type` enum in the migration that ships the wizard).

## 6. Analytics the site needs

Page → signup → wizard step → go-live funnel, per door. UTM captured into `vendors.features->>'utm'` at signup. No third-party trackers that ship customer PII.
