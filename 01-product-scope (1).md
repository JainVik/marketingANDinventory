# 01 — Product Scope (v1 Final Cut, 5 Sep 2026)

> **Name:** Regulars. **Line:** "We make customers come back."
> **What it is:** QR dine-in ordering + WhatsApp retention SaaS for independent Indian cafes, restaurants and (v2) hotels.
> **Thesis:** ordering is the trap door. Every QR order or payment carries a verified phone number, so the owner finally knows who eats there. Then rules + AI + WhatsApp bring them back. Flat subscription, zero commission, the customer list is the owner's.
> **Launch:** single city (Jaipur), 3–5 pilot outlets; run-alongside play — keep Petpooja, put our QR on 5 tables for 2 weeks.

This document defines WHAT v1 does. HOW is in `02-architecture.md`. Anything not in §2 is OUT of v1 (§4 says which version it belongs to).

---

## 1. Actors & roles

| Role | Description | Surface |
|---|---|---|
| `customer` | Scans a QR, browses without login, verifies phone by OTP **once — ever** (item 67), then is recognised at every Regulars outlet; tracks status and bill | Customer PWA (mobile, no app) |
| `owner` | The tenant. One login in v1. Full control: menu, tables, order desk, billing, customer book, campaigns, settings | Owner desktop: left sidebar Orders / Tables / Menu / Reports / Settings |
| `staff` | **v1.1.** Role and permission columns exist; no staff logins in v1 | — |
| `super_admin` | Platform operator: onboards/suspends vendors, subscriptions, templates, moderation, cross-tenant logs | Admin panel (minimal) |
| Owner bot | Read-only assistant on WhatsApp: reply to the morning briefing, ask questions in plain language, answers from the owner's own data | WhatsApp (owner's number) |

A vendor (business) has one or more outlets (v1: exactly one; multi-outlet v2). All data is scoped by `vendor_id` + `outlet_id` from day 1.

**Two halves of the product:** Section 1 *Data-making* (items 1–47, free forever) and Section 2 *Marketing & analysis* (48–60 + 65, paid). 64, 66, 67 and 68 are foundations/settings.

**Who owns what.** The *identity* — the phone number, the name, the cross-outlet history — belongs to the platform and to the customer. The *relationship* belongs to the outlet: an owner sees a human being only after that human has ordered from them (item 67). Before that they see counts. This is not a setting; it is enforced in the database (`docs/03` §0, the PII boundary).

---

## 2. In scope — v1 (numbered; these numbers are referenced across the docs)

### Section 1 — Data-making (free forever; customer *count* visible)

**A. Onboarding**
1. Three signup doors — self-signup (OTP), demo-led, manual — all land in one wizard.
2. Outlet profile: name, address, type (cafe / dining / hotel-ready), hours, open–closed switch.
3. GST/FSSAI optional at signup, required before the first invoice.
4. QR mode per outlet: **order / pay-only / both**. Pay-only = zero staff disruption on-ramp; all modes capture the phone.
5. First-run wizard with progress + "go live" checklist: menu + one QR + one test order = live.
6. Practice mode: a test table; its orders never hit the kitchen, the books, or customer WhatsApp.
7. Customer-list import (CSV → column map) + WhatsApp opt-in ask to imported numbers. **No marketing to imported numbers before opt-in.**
8. Payment-gateway onboarding (restaurant is merchant of record, Razorpay/Cashfree) — skippable; pay-at-checkout works without it.
9. Export everything anytime; monthly plan, cancel anytime.

**B. Menu**
10. Categories, items (name / description / price / veg–non-veg / photo optional / GST rate), bestseller tag.
11. Variants + add-on groups (min/max).
12. Availability toggle on the row, with auto-restore (next open / at time).
13. Bulk-edit grid.
14. Import: photo (AI extract → review grid, low-confidence rows first) / PDF / Excel-CSV.
15. Re-photograph = diff update ("12 prices changed, 3 new, 2 gone — apply?").
16. Live phone preview with warnings.
17. Offers banner (owner-written / day-wise / festive) atop the customer menu.

**C. Tables & QR**
18. Table list, kind = table / room / counter, one opaque token per table (`/v/{slug}/t/{token}`, never `?table=4`; rotating QRs = never).
19. QR PDF sheet (free) + per-table reprint/regenerate; delivered standees as a paid option.
20. Counter QR for takeaway → token number, ready notification, mark collected.

**D. Customer (mobile PWA)**
21. Scan → menu, table auto-attached, browse without login.
22. Item sheet, cart, instructions — cart is per phone (client side; no shared cart).
23. Place order → phone OTP (first-timer: name + consent checkbox) → **accept-gate** (nothing is made until staff accepts) → **pay prompt after acceptance**.
24. Counter/takeaway orders: pay first.
25. Live status, order more (rounds), call waiter, request bill.
26. My bill: rounds, per-person attribution, pay whole table OR pay own items.
27. Cancellation *request* (shown only if the outlet enables it) → owner decides.
28. Coupon code field at checkout.
29. First visit: e-bill on WhatsApp (the consent moment — "get your bill on WhatsApp"). Repeat: in-app bill + optional send.
30. Customer history across all Regulars outlets + add-to-home-screen prompt after first order.

**E. Order desk (desktop)**
31. Live New / Preparing / Ready columns with table + customer badge (new / repeat / visit count).
32. Loud persistent alert until accepted.
33. Accept + ETA / preparing / ready / complete / reject presets.
34. Staff order entry (table and no-table) — required for pay-only mode and walk-ins.
35. Reassign table, busy mode.
36. Owner-initiated cancel with reason; handle cancellation requests.
37. Table grid (empty / eating / needs-you) → table sheet with running total + settle.
38. **Regulars Board** panel: who's seated now, name, visit count, usual order.
39. Slow-order alert.

**F. Billing**
40. Pay now (gateway, direct to the restaurant's account) / pay at checkout (running tab) — default by outlet type (cafe = pay now, dining = at checkout).
41. GST invoice: CGST/SGST, GSTIN, FSSAI, sequential number per outlet per FY, round-off, voluntary tip line. One invoice per bill. Generating is mandatory, printing optional. No auto service charge.
42. Coupons: owner-generated, flat / %, validity, total-uses + per-customer limits, applied before payment only.
43. Complimentary item with reason.
44. Payment-mode capture (cash / UPI / card); a bill may be settled in more than one mode.
45. Refunds: back-to-source (gateway) / cash at counter (recorded) / adjust-replace — reason + who on every one. Default = the way it was paid (gateway → back to source, counter → cash); adjust-replace always offered. We never refund; the restaurant does.
46. Day-close report + cash reconciliation.
47. Discount and void-rate report.

### Section 2 — Marketing & analysis (paid)

48. Customer book: rule segments (new / repeat / loyal / at-risk), block customer.
49. **Identity capture rate** tile.
50. **Lapsed wall**: at-risk count + rupee value.
51. Automatic triggers: birthday (opt-in at OTP + WhatsApp follow-up), win-back, post-first-order thank-you.
52. Manual segmented campaigns with pre-written templates.
53. **Dead-hour filler**: detect empty hours → capped send to customers who visit then → measure.
54. **Holdout on every campaign** (20 % control) → real lift shown.
55. Revenue ledger ("this brought back ₹X").
56. **Morning WhatsApp briefing** to the owner.
57. Menu conclusions: move up / kill / slowing the kitchen; **viewed-but-never-ordered** list and never-viewed list (both from item 68).
58. Market-basket upsell suggestions.
59. Kitchen speed by hour/day.
60. Smart review routing: happy → Google ask (routing switch ships v1.1), unhappy → private.

### Foundations (both sections)

61. WhatsApp pipeline: queue, retries, DLQ, opt-in/STOP, quotas, per-customer caps, dedupe, 24 h window, error 131049 handling.
62. Admin panel + foundations: auth, tenant isolation (RLS), **PII isolation (schema `pii` + `regulars_identity` role)**, sockets, events log, CI.
63. Public site: 10 pages + mega-menu with "coming soon" greyed (`13-website-and-onboarding-funnel.md`).
64. **Every owner action is exposed as a callable API function** (the `tools` registry) — design rule; enables the bot and all future automation.
65. **Read-only owner bot on WhatsApp**: reply to the morning briefing, ask in plain language, answers from own data only. Reads free; writes v1.5 with confirmation; irreversible actions never automatic.
66. **Auto-accept setting, conditional** (repeat customers / under ₹X / after first 2 orders) — owner toggles in v1; bot toggles it in v1.5.
67. **One identity, everywhere — revealed on first order.** A customer verifies their phone **once**; the device holds a 180-day rotating session, so any later scan at any Regulars outlet in any city resumes the same identity with no OTP. The identity record is global and lives in its own database schema that the main API cannot read. A vendor's view of that person — name, phone, history — is created **in the same transaction as their first order at that outlet**, never before: scan-and-leave at a new cafe leaves the cafe with a number, not a person.
68. **Browse & intent tracking.** Every visit records what actually happened, not just what was bought: scanned, which categories and items were looked at and for how long, how far they scrolled, what went into the cart, and whether it ended in an order. Anonymous by device before login, stitched to the person at OTP. Feeds items 53, 57, 58 and the v1.1 abandoned-cart nudge. Aggregate-only to the owner until that customer has ordered with them (item 67).

**AI positioning (honest):** v1 AI = things that work on day one with zero data — morning briefing, RFM + rule-based lapse alerts, market-basket upsell, menu-photo extraction, owner bot Q&A. **Do not market "AI predicts churn" in v1.**

---

## 3. Business model

- Flat subscription only, ₹999–2,999 / outlet / month band; WhatsApp message costs pass-through. No commission, ever.
- Free forever: 1–47. Paid: 48–60, 65.
- Money never touches us (RBI PA rules). Each restaurant is merchant of record with a licensed gateway; we create the payment request and listen for the webhook. "Your money is in your account today."
- Verify before launch: Meta India service-message rate card (1 Oct 2026 change), gateway/merchant-of-record structure with a lawyer, CA on e-invoice position, DPDP notice text.

---

## 4. Not in v1 — version ledger (columns may exist; features do not)

| Version | Items |
|---|---|
| **v1.1** | abandoned-cart nudge now buildable on item 68 · staff PIN on sensitive actions · staff logins/permissions · merge table · order modification after placing · split payment-mode entry at settlement UI · GSTR-1 export · happy-hour pricing · dine-in vs takeaway price lists · KOT routing + thermal printing (Android print-agent) · combos · credit notes · competitor-POS menu import · waiter-assist · abandoned-cart nudge · Google review routing (4–5★) · 2nd-visit nudge |
| **v1.5** | owner bot write actions (sold out / busy / price / item via WhatsApp with confirm; campaigns + bulk + refunds hand off to in-app preview) · Hinglish AI copy (human approval mandatory) · AI-suggested campaigns · churn ML replaces rules · pooled cross-tenant forecasting · smart send times · Meta Tech Provider → per-vendor WABA · points loyalty (replayable from events) |
| **v2** | hotels (room QR + room tabs + during/post-stay funnel) · inventory · table booking · multi-outlet · group "pool" cart · full discovery · POS integrations |

## 5. Dropped — do not re-open

Shared server-side cart · "bill vanishes if not sent on WhatsApp" · customer-side split payment · money through us / commission · AI-generated dish photos · Zomato listing scraper · "AI predicts churn" as a v1 claim · forced staff logins in v1 · rotating QRs.

## 6. Still open (decide before the module is built)

Offers-banner edge cases · group-ordering validation (sit in 10 restaurants) · DPDP notice wording for browse tracking before consent (item 68 — lawyer). *Decided 5 Sep: refund default = original payment mode; move table is v1 (item 35), merge is v1.1.* *Decided 9 Sep: identity stays in one Postgres behind a `pii` schema and a locked-down role rather than a second physical database (see `docs/03` §0 and `docs/06` §8); items 67 and 68 added.*
