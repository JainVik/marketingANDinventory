# 14 — Screen Specification & Architecture (v1 + reserved frames)

> Every screen, drawer, sheet and state Regulars needs, ready for Figma sprint planning. Numbers in **items** refer to the final cut in `01-product-scope.md`. Data shapes in `03`, routes in `04`, failure states in `05`, funnel copy in `13`. Design language from HANDOVER §4.2: **screens for reading, sheets for doing, one primary button per screen, status by colour + position; the table badge is the visual spine on both sides.** Structure inspired by Owner.com (structure, not theme).

Versions: **v1** = build now · **v1.1 / v1.5 / v2** = reserved frames so Figma has the slot; nothing behind them is built.


---

## 1. Executive screen directory

**97 v1 screens** (+ **15 reserved frames**) across **88 shared overlays** (drawers, sheets, modals). Four surfaces:

| Surface | Who | Form factor | v1 screens |
|---|---|---|---|
| Public site | prospective owners | responsive web, SEO | 10 |
| Owner desktop app | owner (single login in v1) | desktop-first (≥ 1280), usable on a phone for alerts | 60 |
| Customer mobile PWA | diners | mobile-only PWA (desktop = centered 420 px) | 18 |
| Super-admin panel | us | desktop only | 9 |

### 1.1 Modules and counts

| Module | Screens | v1 | Reserved | IDs |
|---|---|---|---|---|
| Public site | 10 | 10 | 0 | SCR-S01 – SCR-S10 |
| Auth & onboarding | 14 | 14 | 0 | SCR-A01 – SCR-A04 |
| Owner · Home | 1 | 1 | 0 | SCR-O01 – SCR-O01 |
| Owner · Orders | 4 | 4 | 0 | SCR-O02 – SCR-O05 |
| Owner · Tables | 5 | 5 | 0 | SCR-O06 – SCR-O10 |
| Owner · Regulars Board | 1 | 1 | 0 | SCR-O11 – SCR-O11 |
| Owner · Menu | 6 | 6 | 0 | SCR-O12 – SCR-O17 |
| Owner · Reports | 7 | 7 | 0 | SCR-O18 – SCR-O24 |
| Owner · Customer book | 4 | 4 | 0 | SCR-O25 – SCR-O28 |
| Owner · Marketing | 8 | 8 | 0 | SCR-O29 – SCR-O36 |
| Owner · Settings | 9 | 7 | 2 | SCR-O37 – SCR-O45 |
| Owner · Global | 3 | 3 | 0 | SCR-O46 – SCR-O48 |
| Customer · Order | 5 | 5 | 0 | SCR-C01 – SCR-C05 |
| Customer · Session | 2 | 2 | 0 | SCR-C06 – SCR-C14 |
| Customer · Pay | 2 | 2 | 0 | SCR-C07 – SCR-C08 |
| Customer · Bill | 3 | 3 | 0 | SCR-C09 – SCR-C11 |
| Customer · Takeaway | 1 | 1 | 0 | SCR-C12 – SCR-C12 |
| Customer · Pay-only | 1 | 1 | 0 | SCR-C13 – SCR-C13 |
| Customer · Account | 1 | 1 | 0 | SCR-C15 – SCR-C15 |
| Customer · States | 3 | 3 | 0 | SCR-C16 – SCR-C18 |
| Admin | 9 | 9 | 0 | SCR-X01 – SCR-X09 |
| Future · v1.1 | 5 | 0 | 5 | SCR-F01 – SCR-F05 |
| Future · v1.5 | 4 | 0 | 4 | SCR-F06 – SCR-F09 |
| Future · v2 | 4 | 0 | 4 | SCR-F10 – SCR-F13 |
| **Total** | **112** | **97** | **15** | |

### 1.2 Density grouping (for theme tokens)

- **High-density (tables, grids, boards — 13 px base, 4 px rhythm, tabular numbers)** — 30: SCR-S09, SCR-O02, SCR-O04, SCR-O05, SCR-O06, SCR-O10, SCR-O12, SCR-O14, SCR-O16, SCR-O18, SCR-O19, SCR-O20, SCR-O21, SCR-O22, SCR-O24, SCR-O25, SCR-O29, SCR-O31, SCR-O33, SCR-O36, SCR-C01, SCR-X01, SCR-X03, SCR-X04, SCR-X05, SCR-X06, SCR-X08, SCR-X09, SCR-F04, SCR-F12
- **Mixed (cards + a table or a timeline — 14 px base)** — 26: SCR-A03.2, SCR-A03.8, SCR-O01, SCR-O03, SCR-O07, SCR-O08, SCR-O11, SCR-O13, SCR-O15, SCR-O23, SCR-O26, SCR-O27, SCR-O28, SCR-O32, SCR-O34, SCR-O47, SCR-C03, SCR-C06, SCR-C09, SCR-C12, SCR-C15, SCR-X02, SCR-X07, SCR-F03, SCR-F07, SCR-F10
- **Low-density (focused forms, wizards, customer flow — 16 px base, one primary button)** — 56: SCR-S01, SCR-S02, SCR-S03, SCR-S04, SCR-S05, SCR-S06, SCR-S07, SCR-S08, SCR-S10, SCR-A01, SCR-A02, SCR-A03, SCR-A03.0, SCR-A03.1, SCR-A03.3, SCR-A03.4, SCR-A03.5, SCR-A03.6, SCR-A03.7, SCR-A03.9, SCR-A04, SCR-O09, SCR-O17, SCR-O30, SCR-O35, SCR-O37, SCR-O38, SCR-O39, SCR-O40, SCR-O41, SCR-O42, SCR-O43, SCR-O44, SCR-O45, SCR-O46, SCR-O48, SCR-C02, SCR-C04, SCR-C05, SCR-C07, SCR-C08, SCR-C10, SCR-C11, SCR-C13, SCR-C14, SCR-C16, SCR-C17, SCR-C18, SCR-F01, SCR-F02, SCR-F05, SCR-F06, SCR-F08, SCR-F09, SCR-F11, SCR-F13

---

## 2. Detailed screen specifications

Format per screen: goal · layout · overlays (ID = shared across screens) · Owner.com principles · states (standard / empty / mobile) · final-cut items.


### Public site

#### `SCR-S01` Home

**Surface:** Public site · **Route:** `/` · **Density:** low · **Items:** 63

**Primary user goal:** Make an owner believe the trap-door thesis in 30 seconds and click Book a demo or Start free.

**Layout & key components:**

- Hero: one sentence ('In 2026 you can order a burger from 4 km away in 30 seconds, but inside the cafe you wave at a waiter'), two CTAs (Book a demo · Start free), phone-in-hand product shot of the customer menu with a table badge
- Thesis strip: 3 numbers (7 of 10 first-timers never return · every order carries a phone · ₹40,000 brought back) — real seed-outlet figures, tabular-nums
- Headline demo trio as 3 alternating rows (screenshot left/right): Regulars Board (feel) · Dead-hour filler (money this week) · Holdout-proven offers (believe)
- 'How it lands on the table' 5-step horizontal stepper (photo → QR → order → phone → WhatsApp)
- Free vs Paid two-column card (items 1–47 free forever / 48–60 + bot paid) with the ₹999–2,999 band
- Run-alongside band: 'Keep Petpooja. Put our QR on 5 tables for 2 weeks.'
- Footer: mega-menu columns, legal, WhatsApp contact

**Modals, drawers & sheets:**

- `OV-S01a` **Demo booking sheet** — Slide-over with 4 fields (name, phone, outlet, city) + calendar embed; submits without leaving the page
- `OV-S01b` **Video/product tour modal** — 90-second screen recording; closes on Esc

**Owner.com UX principles applied:**

- Owner.com pattern: one promise per viewport, screenshot always adjacent to the claim
- Sticky top bar with a single primary CTA that persists after the hero scrolls away
- No feature grid — three proofs, each with a real screenshot and a number
- Never mention Zomato; never claim 'AI predicts churn'

**States:** standard — Full page · empty — n/a · mobile — Hero stacks; CTAs full-width sticky at bottom; demo trio becomes a swipeable carousel

#### `SCR-S02` How it works

**Surface:** Public site · **Route:** `/how-it-works` · **Density:** low · **Items:** 63

**Primary user goal:** Show the 5 steps from photographed menu to WhatsApp win-back with one screenshot each.

**Layout & key components:**

- Sticky left step rail (5 steps) with the active step highlighted on scroll
- Right column: one screenshot + 3-line caption per step (photograph menu → QR on tables → order lands with table number → phone captured → WhatsApp brings them back)
- Time-to-live callout: 'Photograph your menu. Be taking orders in 20 minutes.'
- CTA band: Start free

**Modals, drawers & sheets:**

- `OV-S02a` **Screenshot lightbox** — Click any screenshot to enlarge

**Owner.com UX principles applied:**

- Scroll-linked step rail (Owner.com 'how it works')
- One idea per step; captions ≤ 3 lines

**States:** standard — Full page · empty — n/a · mobile — Rail becomes a horizontal progress bar pinned under the top bar

#### `SCR-S03` Pricing

**Surface:** Public site · **Route:** `/pricing` · **Density:** low · **Items:** 63

**Primary user goal:** Answer 'what does it cost and what do I get' without a sales call.

**Layout & key components:**

- Two plan cards side by side: Free forever (list of the 7 free groups) · Paid (customer book, campaigns, triggers, ledger, briefing, insights, owner bot) with ₹999–2,999 band
- Toggle: 'What counts as a message cost?' expands the WhatsApp pass-through explainer
- Comparison table (collapsed by default) mapping the numbered final cut to Free/Paid
- FAQ accordion: cancel anytime · export everything · no commission · money goes to your account · GST invoice
- CTA: Start free (primary), Book a demo (secondary)

**Modals, drawers & sheets:**

- `OV-S03a` **Message-cost calculator** — Small inline calculator: customers × messages/month → ₹ estimate

**Owner.com UX principles applied:**

- Owner.com pricing: one flat number, month-to-month, no setup fee, stated three times
- Free plan is a real plan, not a trial — say 'forever'

**States:** standard — Full page · empty — n/a · mobile — Cards stack; comparison table scrolls horizontally inside its container

#### `SCR-S04` QR ordering

**Surface:** Public site · **Route:** `/qr-ordering` · **Density:** low · **Items:** 63

**Primary user goal:** Product page for the free half: order / pay-only / both, accept-gate, pay after acceptance, counter takeaway, GST bill on WhatsApp.

**Layout & key components:**

- Hero with the three QR modes as a segmented illustration
- Feature rows with screenshots: accept-gate, pay after acceptance, rounds, my bill, e-bill
- 'Pay-only' callout for owners who don't want staff disruption
- CTA band

**Owner.com UX principles applied:**

- Show the customer phone and the owner desk side by side in every row — the product is the pair

**States:** standard — Full page · empty — n/a · mobile — Stacked rows

#### `SCR-S05` For cafes

**Surface:** Public site · **Route:** `/for-cafes` · **Density:** low · **Items:** 63

**Primary user goal:** Cafe-specific pitch: reorder-led, counter + tables, pay-now default, dead hours 3–5 pm.

**Layout & key components:**

- Hero in cafe language
- Dead-hour filler screenshot with a real weekday heat strip
- Regulars Board screenshot
- Testimonial slot (real pilot quote or none)
- CTA: Book a demo

**Modals, drawers & sheets:**

- `OV-S01a` **Demo booking sheet** — shared

**Owner.com UX principles applied:**

- Segment pages differ in words and screenshots, not layout

**States:** standard — Full page · empty — Testimonial slot hidden until a real quote exists · mobile — Stacked

#### `SCR-S06` For restaurants

**Surface:** Public site · **Route:** `/for-restaurants` · **Density:** low · **Items:** 63

**Primary user goal:** Dining pitch: pay at checkout, rounds, table sheet, kitchen speed, run alongside Petpooja.

**Layout & key components:**

- Hero
- Table sheet + settle screenshot
- Kitchen speed by hour screenshot
- Run-alongside band
- CTA: Book a demo

**Modals, drawers & sheets:**

- `OV-S01a` **Demo booking sheet** — shared

**Owner.com UX principles applied:**

- Same as S05

**States:** standard — Full page · empty — n/a · mobile — Stacked

#### `SCR-S07` Book a demo

**Surface:** Public site · **Route:** `/demo` · **Density:** low · **Items:** 1, 63

**Primary user goal:** Get name, phone, outlet, city and a slot in under 40 seconds.

**Layout & key components:**

- Left: 4-field form (name, phone, outlet name, city) with WhatsApp-number hint
- Right: calendar embed
- Below: 'what happens next' 3 steps (call → we build your menu with you → live in 20 minutes)

**Modals, drawers & sheets:**

- `OV-S07a` **Booked confirmation** — Inline replace of the form: date/time, add-to-calendar, WhatsApp confirmation note

**Owner.com UX principles applied:**

- Zero-friction input: phone field auto-formats +91; no email required
- Confirmation replaces the form in place — no redirect

**States:** standard — Form · empty — n/a · mobile — Form first, calendar below

#### `SCR-S08` Contact

**Surface:** Public site · **Route:** `/contact` · **Density:** low · **Items:** 63

**Primary user goal:** Reach us on WhatsApp in one tap.

**Layout & key components:**

- WhatsApp deep-link button (primary)
- Email, address
- Support hours in IST

**Owner.com UX principles applied:**

- Primary action is the channel owners already use

**States:** standard — Page · empty — n/a · mobile — Single column

#### `SCR-S09` Legal

**Surface:** Public site · **Route:** `/legal` · **Density:** high · **Items:** 63

**Primary user goal:** Terms, privacy + DPDP notice (versioned), refund policy, cookie note — readable, anchored.

**Layout & key components:**

- Left anchor rail (Terms · Privacy & DPDP notice vX · Refunds · Cookies)
- Long-form text column at 65 ch
- Notice version + effective date visible at the top of the DPDP section (this is the `noticeVersion` OTP consent records)

**Owner.com UX principles applied:**

- Anchor rail sticky; version stamp visible

**States:** standard — Page · empty — n/a · mobile — Rail collapses to a dropdown

#### `SCR-S10` Mega-menu (global)

**Surface:** Public site · **Route:** `(overlay)` · **Density:** low · **Items:** 63

**Primary user goal:** Show the whole product map, including what is coming, without lying.

**Layout & key components:**

- 3 columns: Product (QR ordering · Menu & billing · Customer book · WhatsApp campaigns · Insights & briefing · Owner bot) · For (Cafes · Restaurants · Hotels — coming soon) · Company (Pricing · Demo · Contact · Legal)
- 'Coming soon' items greyed with a small tag, not hidden: Hotels, Inventory, Table booking, Multi-outlet, POS integrations

**Owner.com UX principles applied:**

- Grey, don't hide — honesty is the brand

**States:** standard — Desktop overlay · empty — n/a · mobile — Full-screen drawer with accordion columns


### Auth & onboarding

#### `SCR-A01` Owner login

**Surface:** Owner desktop app · **Route:** `/login` · **Density:** low · **Items:** 1

**Primary user goal:** Get the owner into the desk in one screen.

**Layout & key components:**

- Centered card: email, password, 'Log in'
- Links: forgot password · new here? Start free
- Magic-link notice for demo/manual doors ('we sent a link on WhatsApp')

**Modals, drawers & sheets:**

- `OV-A01a` **Forgot password sheet** — Email → reset link; success replaces the form

**Owner.com UX principles applied:**

- One primary button; error inline under the field, never a toast

**States:** standard — Card · empty — n/a · mobile — Full-width card

#### `SCR-A02` Owner signup (self door)

**Surface:** Owner desktop app · **Route:** `/signup` · **Density:** low · **Items:** 1

**Primary user goal:** Create the vendor + owner + outlet shell in under a minute, then land in the wizard.

**Layout & key components:**

- Step 1: name, WhatsApp phone (OTP), email, password — one column
- Step 2: OTP 6-box input with 60 s resend
- Progress dots (2)
- Below: 'What you get free forever' 3 bullets

**Modals, drawers & sheets:**

- `OV-A02a` **Existing account modal** — Email already used → 'log in instead' with the email prefilled

**Owner.com UX principles applied:**

- Zero-friction: phone first because the owner bot and briefing need it; no outlet questions yet
- Auto-advance on the 6th OTP digit

**States:** standard — Form · empty — n/a · mobile — Same, sticky CTA

#### `SCR-A03` Setup wizard (shell)

**Surface:** Owner desktop app · **Route:** `/setup` · **Density:** low · **Items:** 1, 5

**Primary user goal:** Take any of the three doors to 'live' with the least typing; resumable at any step.

**Layout & key components:**

- Left rail: 10 steps with done/current/todo marks and the separate Go-live card (menu ✓ · one QR ✓ · one test order ✓)
- Center: one step at a time, one primary button ('Continue'), secondary 'Skip for now' where allowed
- Right (desktop ≥ 1280): live phone preview of the customer menu, updating as steps complete
- Top bar: outlet name, 'Save & exit'

**Modals, drawers & sheets:**

- `OV-A03a` **Skip confirmation** — Only for GST (step 6): 'You can skip now; you will need it before your first invoice'

**Owner.com UX principles applied:**

- Owner.com onboarding: progress always visible, every step resumable, done-for-you option ('we can do this with you on a call') on the hard steps
- The preview pane is the reward for every step

**States:** standard — Shell · empty — First run shows step 0 · mobile — Rail becomes a top progress bar; preview hidden behind a 'Preview' button

*Note: Steps below are sub-frames of the same shell.*

#### `SCR-A03.0` Wizard · Outlet type

**Surface:** Owner desktop app · **Route:** `/setup/0` · **Density:** low · **Items:** 2

**Primary user goal:** Pick cafe / dining / hotel-ready; sets pay-flow default.

**Layout & key components:**

- 3 large selectable cards with an icon and one line of consequence ('Cafes usually pay before food' / 'Restaurants settle at the end')

**Owner.com UX principles applied:**

- Selection cards, not radio buttons; consequence text under each

**States:** standard — 3 cards · empty — n/a · mobile — Stacked cards

#### `SCR-A03.1` Wizard · Outlet profile

**Surface:** Owner desktop app · **Route:** `/setup/1` · **Density:** low · **Items:** 2

**Primary user goal:** Name, address, hours, open/closed.

**Layout & key components:**

- Name, address (pincode → city/state autofill), hours grid (7 rows, multiple ranges, 'copy Mon to all')
- Open/closed switch preview

**Modals, drawers & sheets:**

- `OV-A03.1a` **Hours range editor** — Add a second range for split shifts (e.g. 08–11, 17–23)

**Owner.com UX principles applied:**

- Defaults filled (10:00–22:00 all days); editing is the exception

**States:** standard — Form · empty — n/a · mobile — Hours as an accordion per day

#### `SCR-A03.2` Wizard · Menu import

**Surface:** Owner desktop app · **Route:** `/setup/2` · **Density:** mixed · **Items:** 14

**Primary user goal:** 'Photograph your menu. Be taking orders in 20 minutes.'

**Layout & key components:**

- 4 entry cards: Photograph (camera/upload) · PDF · Excel/CSV (download template) · Type it
- After upload: processing state ('reading 3 pages… ~20 s') then the review grid (see SCR-O17)
- Right: phone preview fills as rows are accepted

**Modals, drawers & sheets:**

- `OV-O16` **Import review grid** — Confidence-sorted staging grid with page-crop thumbnail and phone preview (SCR-O16)

**Owner.com UX principles applied:**

- Low-confidence rows first; apply is one button with a count ('Add 54 items')

**States:** standard — Entry cards · empty — No file yet → cards · mobile — Camera capture is the primary card

#### `SCR-A03.3` Wizard · Tables & QR

**Surface:** Owner desktop app · **Route:** `/setup/3` · **Density:** low · **Items:** 18, 19

**Primary user goal:** How many tables, what kinds, download the PDF sheet.

**Layout & key components:**

- Stepper 'How many tables?' with quick chips (6 · 10 · 16 · 24) and a kind toggle per row (table / room / counter)
- 'Add a counter QR for takeaway' toggle
- Download QR sheet (PDF) button — marks go-live QR ✓

**Modals, drawers & sheets:**

- `OV-A03.3a` **QR sheet preview** — PDF preview in a modal with print

**Owner.com UX principles applied:**

- Generate names automatically (T-01…); rename later

**States:** standard — Stepper · empty — n/a · mobile — Same

#### `SCR-A03.4` Wizard · QR mode

**Surface:** Owner desktop app · **Route:** `/setup/4` · **Density:** low · **Items:** 4

**Primary user goal:** Order / pay-only / both.

**Layout & key components:**

- 3 selectable cards with a 2-line story each and a small phone illustration of what the customer sees

**Owner.com UX principles applied:**

- Recommend one based on outlet type (badge 'Recommended for cafes')

**States:** standard — Cards · empty — n/a · mobile — Stacked

#### `SCR-A03.5` Wizard · Test order

**Surface:** Owner desktop app · **Route:** `/setup/5` · **Density:** low · **Items:** 5, 6

**Primary user goal:** Scan the practice table with your own phone, place an order, accept it on the desk — the go-live moment.

**Layout & key components:**

- Big QR of the practice table on screen + 'scan with your phone'
- Live mini order desk on the right that lights up when the order lands
- Success state: confetti-free, a green 'Test order completed' with the go-live card ticking

**Owner.com UX principles applied:**

- Immediate feedback loop across two devices — the aha moment

**States:** standard — Waiting for scan · empty — n/a · mobile — Shows the QR full-screen with 'open on another phone'

#### `SCR-A03.6` Wizard · GST & FSSAI

**Surface:** Owner desktop app · **Route:** `/setup/6` · **Density:** low · **Items:** 3

**Primary user goal:** Optional now, required before the first invoice.

**Layout & key components:**

- GSTIN (15-char mask, live validation), FSSAI (14-digit), invoice prefix
- 'Skip for now' secondary

**Modals, drawers & sheets:**

- `OV-A03a` **Skip confirmation** — shared

**Owner.com UX principles applied:**

- Explain the consequence of skipping once, in the button's helper text

**States:** standard — Form · empty — n/a · mobile — Same

#### `SCR-A03.7` Wizard · Payments

**Surface:** Owner desktop app · **Route:** `/setup/7` · **Density:** low · **Items:** 8, 40

**Primary user goal:** Connect Razorpay/Cashfree (skippable); confirm pay-flow default; UPI ID for the bill.

**Layout & key components:**

- Provider cards (Razorpay · Cashfree) → external onboarding, status pill on return (Not started / Pending KYC / Live)
- Pay-flow default toggle (pay now / at checkout) prefilled from outlet type
- UPI ID field

**Modals, drawers & sheets:**

- `OV-A03.7a` **Gateway onboarding return modal** — Status + 'what's pending' checklist from the provider

**Owner.com UX principles applied:**

- 'Pay-at-checkout works without this' stated above the skip button

**States:** standard — Cards · empty — Not started · mobile — Same

#### `SCR-A03.8` Wizard · Import customers

**Surface:** Owner desktop app · **Route:** `/setup/8` · **Density:** mixed · **Items:** 7

**Primary user goal:** CSV → column map → preview the opt-in message that will be sent.

**Layout & key components:**

- Upload → column mapper (name/phone/birthday columns as dropdowns over the first 5 rows) → counts (valid / duplicate / invalid) → opt-in message preview (locked template) → 'Import & send opt-in'

**Modals, drawers & sheets:**

- `OV-O27a` **Column mapper** — Dropdown per CSV column (name / phone / birthday / tags) over a 5-row preview; shows valid / duplicate / invalid counts live

**Owner.com UX principles applied:**

- Consent is visible: the owner sees exactly the message their imported customers get
- Skip allowed; never sends marketing without opt-in

**States:** standard — Upload · empty — No file · mobile — Upload only; mapping on desktop ('finish this on a laptop')

#### `SCR-A03.9` Wizard · WhatsApp & briefing

**Surface:** Owner desktop app · **Route:** `/setup/9` · **Density:** low · **Items:** 56, 65

**Primary user goal:** Confirm the owner's WhatsApp number, pick the briefing time, meet the bot.

**Layout & key components:**

- Phone (prefilled from signup) with 'send test message'
- Briefing time picker (default 08:00)
- Preview card of what tomorrow's briefing looks like + 'reply to it to ask the bot anything'

**Owner.com UX principles applied:**

- Show the artifact (a real briefing) instead of describing the feature

**States:** standard — Form · empty — n/a · mobile — Same

#### `SCR-A04` Go-live

**Surface:** Owner desktop app · **Route:** `/setup/live` · **Density:** low · **Items:** 5

**Primary user goal:** Flip the outlet to live and hand off to the desk.

**Layout & key components:**

- Checklist card with three ticks
- 'Go live' primary; on success → SCR-O01 with a 'Your QR is live' banner and the PDF link

**Owner.com UX principles applied:**

- One button, one outcome, immediate redirect

**States:** standard — Ready · empty — Ticks missing → disabled button with the missing item named · mobile — Same


### Owner · Home

#### `SCR-O01` Today

**Surface:** Owner desktop app · **Route:** `/app` · **Density:** mixed · **Items:** 49, 50, 56, 57

**Primary user goal:** See today at a glance and the one thing that needs attention; get to the desk in one click.

**Layout & key components:**

- Top strip: outlet name, open/closed switch, busy toggle, 'Open order desk' primary
- Row of 4 stat tiles (today): revenue, orders, new vs repeat customers, identity capture % — tabular-nums, delta vs same weekday last week
- Left column: Live now card (active tables, orders waiting, slow orders) with links into the desk
- Right column: This morning's briefing card (the WhatsApp text, 'reply on WhatsApp to ask') · Insights inbox (top 3, each with a one-tap action) · Lapsed wall mini (paid; blurred with 'unlock' on free)
- Bottom: 7-day revenue sparkline by day-part
- Pre-live variant: the go-live checklist card replaces the stats row

**Modals, drawers & sheets:**

- `OV-O01a` **Insight action sheet** — Shows the insight payload + 'Apply' (calls the tool) or 'Dismiss'
- `OV-O46` **Upgrade sheet** — The clicked value shown un-blurred once + paid plan list + Start paid plan (SCR-O46)

**Owner.com UX principles applied:**

- Owner.com dashboard: summary before detail; one proactive 'opportunity' surfaced, not a feed
- Every tile links to the screen that explains it
- Paid features visible but blurred — the free plan sees what it's missing

**States:** standard — Stats + cards · empty — Day 0: 'No orders yet — put the QR on 5 tables today' with the QR PDF link · mobile — Tiles 2×2, cards stacked; desk button sticky


### Owner · Orders

#### `SCR-O02` Order desk

**Surface:** Owner desktop app · **Route:** `/app/orders` · **Density:** high · **Items:** 31, 32, 33, 36, 39, 66

**Primary user goal:** Accept, prepare, ready, complete — with the table and the customer badge visible on every card.

**Layout & key components:**

- Header: New count badge (pulsing when > 0), sound toggle, busy toggle, '+ Staff order' button, view switch (Columns · Takeaway queue · Regulars Board)
- 3 columns New / Preparing / Ready; cards: order code, table badge, customer badge (new · repeat ×N), items with variants and instructions, age timer, ETA
- New card actions: Accept (ETA chips 10·15·20·custom) · Reject (preset menu); Preparing: Mark ready; Ready: Complete (dine-in) / Collected (takeaway)
- Cancellation-request cards carry an amber stripe and Approve / Deny
- Slow-order cards turn amber after `slow_order_alert_min`; auto-accepted cards show a small 'auto' chip
- Right rail (≥ 1440): Regulars Board panel docked

**Modals, drawers & sheets:**

- `OV-O03` **Order detail drawer** — Timeline, lines, customer, payment strip and state-matched actions (SCR-O03); opens on card click
- `OV-O02a` **Reject reason menu** — Presets: out of stock · closing · suspicious · other (text)
- `OV-O02b` **Accept with ETA popover** — ETA chips + note to customer
- `OV-O04` **Staff order entry** — Two-pane POS: item tiles + slip, table picker, phone identity (SCR-O04)

**Owner.com UX principles applied:**

- Loud persistent alert until acted on; the 'New' column never auto-scrolls under your hand
- One primary button per card state; presets over free text (Owner.com Kitchen Tablet)
- Status by colour + position — cards move columns; no status dropdowns

**States:** standard — 3 columns · empty — 'Nothing cooking. Tables with a QR: 12' + link to QR sheet · mobile — Single column with a segmented New/Preparing/Ready switch; alert sound + push

#### `SCR-O03` Order detail (drawer)

**Surface:** Owner desktop app · **Route:** `/app/orders/:id` · **Density:** mixed · **Items:** 33, 36

**Primary user goal:** Everything about one round without leaving the desk.

**Layout & key components:**

- Header: code, table badge, customer badge, source (QR / staff / bot), age
- Timeline: placed → accepted (manual/auto + rule) → preparing → ready → completed; rejected/cancelled with reason and who
- Lines with variants, add-ons, instructions, per-person attribution; complimentary/void markers
- Payment strip: required when, status, gateway ref
- Footer actions matching the current state; cancel with reason

**Modals, drawers & sheets:**

- `OV-O03a` **Cancel confirm** — Reason required; warns if a payment is captured → refund needed first

**Owner.com UX principles applied:**

- Drawer, not page — the desk stays visible behind it
- Timestamps, not statuses: show times next to each step

**States:** standard — Drawer 480px · empty — n/a · mobile — Full-screen sheet

#### `SCR-O04` Staff order entry

**Surface:** Owner desktop app · **Route:** `/app/orders/new` · **Density:** high · **Items:** 34

**Primary user goal:** Punch a walk-in or a table order in under 20 seconds.

**Layout & key components:**

- Left 60%: category tabs + item tiles (name, price, veg dot, availability); search box focused on open
- Right 40%: the slip — table selector (or 'No table / takeaway'), customer phone (optional, shows badge if known), lines with steppers, instructions, subtotal
- Footer: 'Send to kitchen' (creates order, accept-gate skipped for staff) · 'Settle now' for counter

**Modals, drawers & sheets:**

- `OV-O04a` **Variant/add-on picker** — Same sheet as the customer item sheet, desktop sized
- `OV-O04b` **Table picker** — Floor mini-grid; occupied tables add a round to the existing session

**Owner.com UX principles applied:**

- Two-pane POS layout, keyboard-first (type to search, Enter adds)
- Phone field is the identity capture — show 'repeat ×4' the moment it matches

**States:** standard — Two panes · empty — Menu empty → 'Add items in Menu first' · mobile — Item grid full-screen, slip as a bottom sheet with a count badge

#### `SCR-O05` Takeaway queue

**Surface:** Owner desktop app · **Route:** `/app/orders/takeaway` · **Density:** high · **Items:** 20, 24

**Primary user goal:** Counter tokens in order: paid → preparing → ready → collected.

**Layout & key components:**

- Token cards in a row per state; big token number; 'Ready' pings the customer; 'Collected' closes it
- Paid-first badge; unpaid tokens cannot be accepted

**Modals, drawers & sheets:**

- `OV-O03` **Order detail drawer** — shared

**Owner.com UX principles applied:**

- Token number is the biggest text on the card — it is what staff shout

**States:** standard — Queue · empty — 'No takeaway orders' + counter QR link · mobile — Vertical list


### Owner · Tables

#### `SCR-O06` Floor grid

**Surface:** Owner desktop app · **Route:** `/app/tables` · **Density:** high · **Items:** 37

**Primary user goal:** See every table's state and open the one that needs you.

**Layout & key components:**

- Grid of table tiles grouped by zone; tile = name, state colour (empty · eating · needs-you), running total, seated minutes, diners count, service-call icon, rounds count
- Legend + filter chips (needs-you first)
- Header: 'Manage tables & QR' link, view toggle grid/list

**Modals, drawers & sheets:**

- `OV-O07` **Table sheet (drawer)** — Rounds, diners with badges, service calls, totals, Settle / Add round / Move / Void (SCR-O07)
- `OV-O06a` **Open table for staff order** — Empty tile click → SCR-O04 with the table preselected

**Owner.com UX principles applied:**

- Table badge is the visual spine: same badge component as the desk and the customer app
- Needs-you tiles pulse; nothing else animates

**States:** standard — Grid · empty — No tables → 'Add tables' CTA · mobile — 2-column tiles

#### `SCR-O07` Table sheet (session drawer)

**Surface:** Owner desktop app · **Route:** `/app/tables/:sessionId` · **Density:** mixed · **Items:** 26, 35, 37, 43

**Primary user goal:** Running total, rounds, diners, service calls — and settle.

**Layout & key components:**

- Header: table badge, seated since, diners (names + badges, usual order on hover)
- Rounds accordion (each round = order card summary)
- Service calls with acknowledge
- Totals block: items, discount, GST, round-off, total
- Footer: 'Settle' primary · 'Add round' · 'Move table' · 'Void'

**Modals, drawers & sheets:**

- `OV-O08` **Settle drawer** — Invoice preview, multi-mode payment rows, tip, Settle & send e-bill / Print (SCR-O08)
- `OV-O07a` **Move table picker** — Floor mini-grid of empty tables
- `OV-O07b` **Discount / coupon sheet** — Code or manual amount with reason
- `OV-O07c` **Complimentary line sheet** — Pick line + reason
- `OV-O07d` **Void confirm** — Reason required; blocked if any payment captured

**Owner.com UX principles applied:**

- The Regulars Board data (visit count, usual order) appears right here — this is where it changes service

**States:** standard — Drawer 520px · empty — n/a · mobile — Full-screen sheet with sticky Settle

#### `SCR-O08` Settle (payment drawer)

**Surface:** Owner desktop app · **Route:** `/app/tables/:sessionId/settle` · **Density:** mixed · **Items:** 40, 41, 44, 46

**Primary user goal:** Record how the bill was paid — possibly in more than one mode — issue the GST invoice, send the e-bill.

**Layout & key components:**

- Bill preview (invoice layout) with GSTIN/FSSAI, sequential number reserved on settle
- Payment rows: mode segmented (Cash · UPI · Card · Gateway link) + amount; 'Add another mode' link; remaining amount live; cash tendered → change
- Tip line (voluntary), coupon applied badge
- Primary: 'Settle & send e-bill' · Secondary: 'Print (80 mm)'
- GST missing → inline block with 'Enter GSTIN' (422 GST_REQUIRED)

**Modals, drawers & sheets:**

- `OV-O08a` **UPI QR modal** — Outlet UPI QR large for the customer to scan
- `OV-O08b` **Gateway link sent toast** — 'Payment link sent to +91…'
- `OV-O08c` **Print preview** — 80 mm HTML

**Owner.com UX principles applied:**

- Split-mode entry is a list of rows, not a mode dropdown — day-close sums payments
- Invoice preview is the confirmation; no second 'are you sure'

**States:** standard — Drawer · empty — n/a · mobile — Full-screen; keypad for amounts

#### `SCR-O09` Refund dialog

**Surface:** Owner desktop app · **Route:** `(modal)` · **Density:** low · **Items:** 45

**Primary user goal:** Refund the right amount the right way, with a reason and a name.

**Layout & key components:**

- Pick the payment to refund (list of captured payments), amount (≤ original), mode cards: Back to source (5–10 days) · Cash at counter · Adjust & replace (pick replacement order)
- Reason (required), 'refunded by' (owner in v1)
- Consequence line per mode

**Owner.com UX principles applied:**

- Irreversible → the confirm button repeats the amount ('Refund ₹340 to source')

**States:** standard — Modal · empty — No captured payments → dialog disabled · mobile — Sheet

#### `SCR-O10` Tables & QR management

**Surface:** Owner desktop app · **Route:** `/app/tables/manage` · **Density:** high · **Items:** 18, 19

**Primary user goal:** Add, rename, group and reprint tables; regenerate a leaked QR.

**Layout & key components:**

- Table list (name, kind, zone, QR version, active) with inline edit
- Row actions: reprint · regenerate (confirm: old QR stops working) · deactivate
- Header: 'Add table(s)', 'Download QR sheet', 'Order printed standees' (paid option)

**Modals, drawers & sheets:**

- `OV-O10a` **Add tables sheet** — Count + kind + zone
- `OV-O10b` **Regenerate confirm** — 'Old QR will show "ask staff"'
- `OV-O10c` **Standee order sheet** — Quantity + address → manual fulfilment
- `OV-A03.3a` **QR sheet preview** — shared

**Owner.com UX principles applied:**

- Practice table is pinned at the top with a 'test' tag

**States:** standard — List · empty — 'No tables yet' · mobile — Cards


### Owner · Regulars Board

#### `SCR-O11` Regulars Board

**Surface:** Owner desktop app · **Route:** `/app/board` · **Density:** mixed · **Items:** 38

**Primary user goal:** Who is seated right now — name, visit count, usual order — so the floor can greet them.

**Layout & key components:**

- Cards per seated diner grouped by table: name (or phone last-4), badge (new / repeat ×N), last visit, usual order (top 3 items), 'birthday this week' chip
- Header: seated count, new vs repeat split today
- Docked as a right rail on the desk at ≥ 1440; full page here

**Modals, drawers & sheets:**

- `OV-O26` **Customer profile drawer** — Visits, spend, usual order, consent, block (SCR-O26)

**Owner.com UX principles applied:**

- This is the 'feel' demo: big names, small numbers; nothing to click to get value

**States:** standard — Cards · empty — 'Nobody seated. Regulars appear here the moment they scan.' · mobile — List; also available as a WhatsApp reply via the bot


### Owner · Menu

#### `SCR-O12` Menu editor

**Surface:** Owner desktop app · **Route:** `/app/menu` · **Density:** high · **Items:** 10, 12, 16

**Primary user goal:** Edit the menu the way it appears to customers; toggle availability on the row.

**Layout & key components:**

- Left 260px: category list (drag to reorder, active toggle, '+ Category')
- Center: items table for the selected category — photo thumb, name, veg dot, price, GST, availability switch (row), bestseller tag, drag handle; row click opens the item sheet
- Right 360px (≥ 1440): live phone preview with warnings (missing price, empty category)
- Header: search, 'Bulk edit', 'Import', 'Banners', '+ Item'

**Modals, drawers & sheets:**

- `OV-O13` **Item sheet** — Name, price, veg, GST, photo, variants, add-on groups, price history (SCR-O13)
- `OV-O12a` **Availability popover** — Unavailable until: next open · custom time · until I switch it back; limited count
- `OV-O12b` **Category sheet** — Name, active, happy-hour window (v1.1 field greyed)

**Owner.com UX principles applied:**

- Availability toggle is on the row — the most frequent action is zero clicks away
- Preview pane shows the customer truth, with warnings inline (Owner.com menu manager)

**States:** standard — 3-pane · empty — 'Photograph your menu' hero with the 4 import cards · mobile — Category dropdown + item list; preview behind a button

#### `SCR-O13` Item sheet

**Surface:** Owner desktop app · **Route:** `/app/menu/items/:id` · **Density:** mixed · **Items:** 10, 11

**Primary user goal:** Name, price, veg, GST, photo, variants, add-on groups — in one scroll.

**Layout & key components:**

- Photo dropzone (optional), name, description, veg/non-veg segmented, price, GST rate select, tags (bestseller)
- Variants list (name, price, default radio) with 'Add variant'
- Add-on groups (name, min/max, add-ons with price and availability)
- Prep time, sort
- Price history link (revisions)
- Footer: Save (primary) · Delete

**Modals, drawers & sheets:**

- `OV-O13a` **Price history drawer** — Revisions table: version, price, who, when
- `OV-O13b` **Delete confirm** — Soft delete; 'orders keep their snapshot'

**Owner.com UX principles applied:**

- Money changes show a small 'price version 4 → 5' note on save — history is a feature, not a surprise

**States:** standard — Sheet 560px · empty — New item: name focused · mobile — Full-screen

#### `SCR-O14` Bulk edit grid

**Surface:** Owner desktop app · **Route:** `/app/menu/bulk` · **Density:** high · **Items:** 13

**Primary user goal:** Change many prices / GST / availability at once.

**Layout & key components:**

- Spreadsheet grid: category, name, price, GST, active, availability; multi-select rows → bulk actions bar (set GST, +/−% price, availability)
- Diff footer: '12 prices changed · 3 items deactivated' → Apply

**Modals, drawers & sheets:**

- `OV-O14a` **Apply confirm** — Shows the diff summary; writes revisions

**Owner.com UX principles applied:**

- Changes staged, applied once — same mental model as the import review grid

**States:** standard — Grid · empty — n/a · mobile — Not supported → 'Use a laptop for bulk edit'

#### `SCR-O15` Menu import

**Surface:** Owner desktop app · **Route:** `/app/menu/import` · **Density:** mixed · **Items:** 14

**Primary user goal:** Photo / PDF / CSV in; a reviewed menu out.

**Layout & key components:**

- Step 1 entry cards (photo, PDF, CSV template, re-photograph existing)
- Step 2 processing (page thumbnails, progress, ~20 s)
- Step 3 review grid (SCR-O16)

**Modals, drawers & sheets:**

- `OV-O16` **Import review grid** — Confidence-sorted staging grid with page-crop thumbnail and phone preview (SCR-O16)

**Owner.com UX principles applied:**

- Camera is a first-class input on mobile

**States:** standard — Cards · empty — n/a · mobile — Camera first

#### `SCR-O16` Import review grid

**Surface:** Owner desktop app · **Route:** `/app/menu/import/:id` · **Density:** high · **Items:** 14, 15

**Primary user goal:** Confirm what the model read — lowest confidence first — then apply as one change.

**Layout & key components:**

- Grid: category, name, veg, price, description, confidence pill; rows sorted by confidence asc; low rows tinted; inline edit
- Left: page thumbnail with the row's crop highlighted on focus
- Right (≥ 1440): phone preview updating as rows are accepted
- Re-photograph mode: rows carry action chips (update · new · gone) and the diff banner '12 prices changed, 3 new, 2 gone — apply?'
- Footer: 'Add 54 items' / 'Apply 17 changes' primary · Discard

**Modals, drawers & sheets:**

- `OV-O16a` **Row crop zoom** — Enlarged crop of the source for a row
- `OV-O16b` **Apply confirm** — Counts + 'prices become live immediately'

**Owner.com UX principles applied:**

- Low-confidence first; accept-all-high in one click; the original vs edited pair is kept (training data, invisibly)

**States:** standard — Grid + thumbnail · empty — Nothing extracted → 'Try a clearer photo' with tips · mobile — Read-only preview; 'finish on a laptop'

#### `SCR-O17` Offers banner

**Surface:** Owner desktop app · **Route:** `/app/menu/banners` · **Density:** low · **Items:** 17

**Primary user goal:** One line atop the customer menu: owner-written, day-wise or festive.

**Layout & key components:**

- Banner list with active toggle, kind, schedule summary, coupon link
- Editor sheet: text (60 chars, live preview on a phone frame), kind, days/time/dates, optional coupon

**Modals, drawers & sheets:**

- `OV-O17a` **Banner editor sheet** — Text with live phone-frame preview, kind, schedule, optional coupon

**Owner.com UX principles applied:**

- Preview in the phone frame is the form's validation

**States:** standard — List · empty — 'No banner. Try: Happy hour 3–5 pm, 20% off cold coffee' · mobile — Same


### Owner · Reports

#### `SCR-O18` Reports home (Revenue)

**Surface:** Owner desktop app · **Route:** `/app/reports` · **Density:** high · **Items:** 47, 59

**Primary user goal:** Revenue by day / day-part / hour with the free-plan basics, and the paid insights where they belong.

**Layout & key components:**

- Sticky filter bar: range (today · 7d · 30d · custom), outlet (v2), compare toggle
- KPI row: gross, net, discounts, refunds, avg ticket, orders
- Chart: revenue by day with day-part stacking; hour heat strip (dead hours highlighted)
- Tabs: Revenue · Kitchen speed · Discounts & voids · Messaging costs · Bills · Day close
- Export CSV

**Modals, drawers & sheets:**

- `OV-O18a` **Custom range picker** — Two-month calendar

**Owner.com UX principles applied:**

- Sticky filter bar; every number links to the list that makes it up
- Tabular numbers, right-aligned, one decimal max

**States:** standard — KPIs + chart · empty — 'No sales yet' · mobile — KPIs 2×2; chart scrolls horizontally

#### `SCR-O19` Kitchen speed

**Surface:** Owner desktop app · **Route:** `/app/reports/kitchen` · **Density:** high · **Items:** 59

**Primary user goal:** Accept time and prep time by hour and weekday; which items slow the kitchen.

**Layout & key components:**

- Two heatmaps (weekday × hour): avg accept sec, avg prep sec
- Table: slowest items (avg prep, count) with a link to the menu conclusion insight
- Slow-order count trend

**Owner.com UX principles applied:**

- Heatmap cells are clickable → the orders behind them

**States:** standard — Heatmaps · empty — Needs 7 days of data → explainer · mobile — Heatmap scrolls

#### `SCR-O20` Discounts & voids

**Surface:** Owner desktop app · **Route:** `/app/reports/discounts` · **Density:** high · **Items:** 47

**Primary user goal:** Discount rate and void rate over time, with the reasons.

**Layout & key components:**

- KPIs: discount %, void %, complimentary ₹
- Table by day: bills, discounts, voids, comps, refunds; reason breakdown pie-less list
- Drill to bill

**Modals, drawers & sheets:**

- `OV-O22` **Bill detail drawer** — Invoice layout, payments list, refund, resend e-bill, print

**Owner.com UX principles applied:**

- Reasons are text, not codes — the owner wrote them

**States:** standard — Table · empty — 'No discounts recorded' · mobile — Cards

#### `SCR-O21` Messaging costs

**Surface:** Owner desktop app · **Route:** `/app/reports/messaging` · **Density:** high · **Items:** 61

**Primary user goal:** What WhatsApp cost this month and why (marketing vs utility, in-window vs billable).

**Layout & key components:**

- KPIs: sends, delivered %, read %, cost ₹
- Table by purpose: sends, billable, cost; note about the 1 Oct 2026 rate change
- Link to the revenue ledger for ROI

**Owner.com UX principles applied:**

- Cost next to return — always show the ledger link

**States:** standard — Table · empty — 'No messages sent' · mobile — Cards

#### `SCR-O22` Bills

**Surface:** Owner desktop app · **Route:** `/app/reports/bills` · **Density:** high · **Items:** 41

**Primary user goal:** Find any invoice; reprint or resend.

**Layout & key components:**

- Search (invoice no, phone, table), date filter, status chips (paid · partially paid · refunded · voided)
- Table: invoice no, time, table, customer (masked), total, modes, status
- Row → bill detail drawer

**Modals, drawers & sheets:**

- `OV-O22` **Bill detail drawer** — Invoice layout, payments list, refund button, resend e-bill, print

**Owner.com UX principles applied:**

- Invoice number is the primary key the owner remembers — searchable by partial

**States:** standard — Table · empty — 'No bills yet' · mobile — Cards

#### `SCR-O23` Day close

**Surface:** Owner desktop app · **Route:** `/app/reports/day-close` · **Density:** mixed · **Items:** 46

**Primary user goal:** Close the day: expected cash vs counted, mode totals, open tabs — one screen, one button.

**Layout & key components:**

- Summary cards: orders, bills, gross, net, GST, tips
- Mode totals table from payments (cash · UPI · card · gateway) and refunds by mode
- Cash reconciliation: expected (computed) vs counted (input) → variance (signed, red if negative), note
- Open sessions list (carried)
- Primary: 'Close day' (irreversible) · History link

**Modals, drawers & sheets:**

- `OV-O23a` **Close confirm** — Repeats variance; PIN in v1.1
- `OV-O23b` **Day-close history** — Table by date with reopen (admin)

**Owner.com UX principles applied:**

- Computed numbers are read-only and large; the only input is 'cash counted'

**States:** standard — Preview · empty — 'Nothing to close today' · mobile — Same, keypad input

#### `SCR-O24` Coupons

**Surface:** Owner desktop app · **Route:** `/app/coupons` · **Density:** high · **Items:** 42

**Primary user goal:** Create and track coupon codes with limits.

**Layout & key components:**

- Table: code, type/value, validity, uses/limit, per-customer, campaign link, active toggle
- Editor sheet: code (auto-uppercase), flat/%, value, max discount, min order, validity, total uses, per customer, outlets (v2)

**Modals, drawers & sheets:**

- `OV-O24a` **Coupon editor sheet** — Code, flat/%, value, max discount, min order, validity, total uses, per-customer limit

**Owner.com UX principles applied:**

- Limits visible in the row (uses 34/100)

**States:** standard — Table · empty — 'No coupons' · mobile — Cards


### Owner · Customer book

#### `SCR-O25` Customer book home

**Surface:** Owner desktop app · **Route:** `/app/customers` · **Density:** high · **Items:** 48, 49, 50

**Primary user goal:** Who your customers are, who is slipping away, and how many you actually know.

**Layout & key components:**

- Tiles: Identity capture rate (30d) · Lapsed wall (at-risk count + ₹) · segments split (new / repeat / loyal / at-risk / lost)
- Filter bar: segment chips, tags, search (name/phone), sort (last visit · spend)
- Table: name (or masked phone), badge, visits, last visit, spend, tags; row → profile drawer
- Header: 'Import customers', 'Segments'

**Modals, drawers & sheets:**

- `OV-O26` **Customer profile drawer** — Visits, spend, usual order, consent, block (SCR-O26)
- `OV-O46` **Upgrade sheet** — The clicked value shown un-blurred once + paid plan list + Start paid plan (SCR-O46)

**Owner.com UX principles applied:**

- Free plan: the count is real, the list is blurred — the upgrade is the list
- Lapsed wall is a tile with a number and a rupee value — the emotional hook, then 'send win-back' one-tap

**States:** standard — Tiles + table · empty — 'Customers appear after the first QR order' · mobile — Tiles stacked; list

#### `SCR-O26` Customer profile (drawer)

**Surface:** Owner desktop app · **Route:** `/app/customers/:id` · **Density:** mixed · **Items:** 48

**Primary user goal:** One customer at this outlet: visits, spend, usual order, messages, consent — and block if needed.

**Layout & key components:**

- Header: name, phone (full on this screen), badge, segment, tags editor
- Stats: visits, spend, avg ticket, first/last visit, preferred day-part
- Usual order (top 3)
- Timeline: visits + messages (sent/read) + campaigns
- Consent block: marketing (source, date), utility; STOP status
- Footer: Send message (template), Block (reason)

**Modals, drawers & sheets:**

- `OV-O26a` **Block confirm** — Reason required
- `OV-O26b` **Send template sheet** — Approved templates + variables; counts against caps

**Owner.com UX principles applied:**

- Consent is shown as a fact with a date — never a toggle the owner can flip

**States:** standard — Drawer · empty — n/a · mobile — Full-screen

#### `SCR-O27` Import customers

**Surface:** Owner desktop app · **Route:** `/app/customers/import` · **Density:** mixed · **Items:** 7

**Primary user goal:** CSV in, opt-in out.

**Layout & key components:**

- Upload → column mapper (first 5 rows, dropdown per column) → validation counts → opt-in preview (locked template with outlet name) → Run
- Result: created / merged / invalid; opt-in sent count; accepted/declined updating over days

**Modals, drawers & sheets:**

- `OV-O27a` **Column mapper** — Dropdown per CSV column (name / phone / birthday / tags) over a 5-row preview; shows valid / duplicate / invalid counts live

**Owner.com UX principles applied:**

- The opt-in message is shown before the button; 'no marketing until they say yes' is on the screen

**States:** standard — Steps · empty — Upload · mobile — Upload only

#### `SCR-O28` Segments

**Surface:** Owner desktop app · **Route:** `/app/customers/segments` · **Density:** mixed · **Items:** 48

**Primary user goal:** See the system segments; build a custom rule set without SQL.

**Layout & key components:**

- System segments (new · repeat · loyal · at-risk · lost) with counts and the rule in words ('no visit in 30 days')
- Custom segment builder: rows of field · operator · value with live count; save

**Modals, drawers & sheets:**

- `OV-O28a` **Segment preview drawer** — Members list

**Owner.com UX principles applied:**

- Live count on every rule change — the count is the feedback

**States:** standard — List + builder · empty — System segments only · mobile — Read-only


### Owner · Marketing

#### `SCR-O29` Campaigns

**Surface:** Owner desktop app · **Route:** `/app/marketing` · **Density:** high · **Items:** 51, 52, 54, 55

**Primary user goal:** Every campaign with its real lift, and the automations that run by themselves.

**Layout & key components:**

- Ledger strip: 'This month WhatsApp brought back ₹X' (incremental) with a link to the ledger
- Automations row: birthday · win-back · post-first-order · dead-hour · review ask — each a card with on/off, sent, revenue
- Campaign table: name, type, audience, sent, read %, treatment vs control rate, incremental ₹, status; row → results
- Header: '+ Campaign', 'Dead hours'

**Modals, drawers & sheets:**

- `OV-O29a` **Automation settings sheet** — Enable, cooldown days, daily cap, quiet hours, template, offer
- `OV-O46` **Upgrade sheet** — The clicked value shown un-blurred once + paid plan list + Start paid plan (SCR-O46)

**Owner.com UX principles applied:**

- Attribution ledger is the retention spine (Owner.com): show ₹ before features
- Automations are cards with a switch — opinionated defaults, few knobs

**States:** standard — Strip + cards + table · empty — 'Your first campaign: win back 23 at-risk customers' one-tap starter · mobile — Cards

#### `SCR-O30` Campaign composer

**Surface:** Owner desktop app · **Route:** `/app/marketing/new` · **Density:** low · **Items:** 52, 54

**Primary user goal:** Audience → message → offer → schedule → check → send, with the holdout explained once.

**Layout & key components:**

- Left stepper (5 steps); right sticky summary sidebar (audience count, treatment/control split, est. sends after caps, est. cost, expected lift range)
- Step 1 Audience: segment cards with live counts + 'exclude messaged in last 7 days'
- Step 2 Message: template cards (approved only) with phone preview; variables form
- Step 3 Offer: none / existing coupon / new coupon inline
- Step 4 Schedule: now / date-time / send window; 'smart timing' greyed (v1.5)
- Step 5 Check: audit list (consent ✓, caps, quota, template approved) → 'Send' / 'Schedule'

**Modals, drawers & sheets:**

- `OV-O30a` **Test send sheet** — Send to the owner's own number
- `OV-O30b` **Holdout explainer popover** — '20 % won't get it so you can see the real difference'

**Owner.com UX principles applied:**

- Sticky summary sidebar (Owner.com checkout pattern) — the numbers update as you choose
- Test send exists (Owner.com's most-complained absence)

**States:** standard — Stepper · empty — No approved templates → blocked with 'templates pending approval' · mobile — Steps stacked; summary as a bottom bar

#### `SCR-O31` Campaign results

**Surface:** Owner desktop app · **Route:** `/app/marketing/:id` · **Density:** high · **Items:** 54, 55

**Primary user goal:** Did it work — treatment vs control, honestly.

**Layout & key components:**

- Header: name, sent time, status
- Funnel: targeted → sent → delivered → read → visited (treatment) with the control visit rate beside it
- Lift card: rate lift, incremental ₹, cost, ROI; 'final' badge when the window closes
- Recipients table (masked) with arm, status, converted, revenue
- Skip reasons breakdown

**Modals, drawers & sheets:**

- `OV-O31a` **Recipient drawer** — Message timeline + customer link

**Owner.com UX principles applied:**

- Show control next to treatment — the difference is the product
- Not-yet-final results say so with the date they will be

**States:** standard — Funnel + table · empty — Scheduled → countdown · mobile — Stacked

#### `SCR-O32` Dead hours

**Surface:** Owner desktop app · **Route:** `/app/marketing/dead-hours` · **Density:** mixed · **Items:** 53

**Primary user goal:** See the empty hours and fill one with a capped send to people who visit then.

**Layout & key components:**

- Weekday × hour heat strip (orders vs same-weekday baseline); dead cells highlighted
- Click a cell → panel: customers who visit that slot (count), cap slider, template, offer → 'Fill this hour'
- History of filled slots with results

**Modals, drawers & sheets:**

- `OV-O32a` **Fill confirm** — Count, cost, holdout

**Owner.com UX principles applied:**

- The chart is the form — click the hour you want to fix

**States:** standard — Heat strip · empty — Needs 14 days of data → explainer · mobile — Strip scrolls

#### `SCR-O33` Revenue ledger

**Surface:** Owner desktop app · **Route:** `/app/marketing/ledger` · **Density:** high · **Items:** 55

**Primary user goal:** 'This brought back ₹X' — by month, by campaign, by trigger, with the control baseline.

**Layout & key components:**

- Month selector
- Big number: incremental ₹ (with 'attributed' ₹ beside it, explained)
- Table: source (campaign/trigger), sends, visits, revenue, incremental, cost, ROI
- Export

**Modals, drawers & sheets:**

- `OV-O31a` **Recipient drawer** — Message timeline + customer link

**Owner.com UX principles applied:**

- Two numbers, explained once: attributed (visited after a message) vs incremental (minus what control did anyway)

**States:** standard — Table · empty — 'No campaigns yet' · mobile — Cards

#### `SCR-O34` Insights inbox

**Surface:** Owner desktop app · **Route:** `/app/insights` · **Density:** mixed · **Items:** 57, 58

**Primary user goal:** Menu conclusions, upsell pairs, kitchen slowdowns, capture rate — each with one action.

**Layout & key components:**

- List of insight cards grouped by kind: headline, value (₹ or count), period, 'Apply' (tool call) · 'Dismiss'; expired fade
- Filters: kind, status
- Never-ordered list as a table

**Modals, drawers & sheets:**

- `OV-O01a` **Insight action sheet** — shared

**Owner.com UX principles applied:**

- Every insight is a sentence with a verb; every card has exactly one action (Owner.com 'opportunities')

**States:** standard — Cards · empty — 'Insights start after 7 days of orders' · mobile — Cards

#### `SCR-O35` Briefings

**Surface:** Owner desktop app · **Route:** `/app/briefings` · **Density:** low · **Items:** 56, 65

**Primary user goal:** Read past briefings; set the time; see what the bot answered.

**Layout & key components:**

- Settings card: on/off, time, language, phone
- Archive list: date, first line, read/replied; row → full text + the bot thread transcript
- Thumbs up/down on each briefing

**Modals, drawers & sheets:**

- `OV-O35a` **Briefing detail drawer** — Text + transcript + rating

**Owner.com UX principles applied:**

- The WhatsApp thread is the product; this screen is the archive

**States:** standard — List · empty — 'First briefing tomorrow at 08:00' · mobile — Same

#### `SCR-O36` Reviews

**Surface:** Owner desktop app · **Route:** `/app/reviews` · **Density:** high · **Items:** 60

**Primary user goal:** Private feedback first; reply; see the rating trend.

**Layout & key components:**

- KPIs: avg stars, count, private (≤3★) open
- List: stars, text, customer (masked), order link, routing (Google prompted / private), reply
- Filter: unanswered private first

**Modals, drawers & sheets:**

- `OV-O36a` **Reply sheet** — Text → sent as WhatsApp utility message

**Owner.com UX principles applied:**

- Unhappy first — the recovery path is the value

**States:** standard — List · empty — 'Reviews arrive after bills are paid' · mobile — Cards


### Owner · Settings

#### `SCR-O37` Settings home

**Surface:** Owner desktop app · **Route:** `/app/settings` · **Density:** low · **Items:** —

**Primary user goal:** Find the setting in one click: outlet, ordering, payments, WhatsApp & bot, practice, plan, team.

**Layout & key components:**

- Left settings nav (sticky); right content
- Sections listed as cards with one-line status ('Gateway: Live', 'Auto-accept: on for repeat customers')

**Owner.com UX principles applied:**

- Status in the card, not behind it

**States:** standard — Nav + cards · empty — n/a · mobile — List

#### `SCR-O38` Outlet profile

**Surface:** Owner desktop app · **Route:** `/app/settings/outlet` · **Density:** low · **Items:** 2, 3

**Primary user goal:** Name, address, hours, type, branding, GST/FSSAI, invoice prefix.

**Layout & key components:**

- Form sections with save per section
- Hours grid
- Logo/cover upload

**Modals, drawers & sheets:**

- `OV-A03.1a` **Hours range editor** — shared

**Owner.com UX principles applied:**

- Save per section; unsaved-changes bar sticky at bottom

**States:** standard — Form · empty — n/a · mobile — Accordion

#### `SCR-O39` Ordering

**Surface:** Owner desktop app · **Route:** `/app/settings/ordering` · **Density:** low · **Items:** 4, 27, 39, 66

**Primary user goal:** QR mode, auto-accept rules, cancellation requests, slow-order and auto-expiry minutes.

**Layout & key components:**

- QR mode selectable cards
- Auto-accept: master switch + rule rows (repeat customers · under ₹X · after N orders · staff entries) each with its own switch; 'last changed by owner/bot' line
- Cancellation requests toggle
- Slow order alert minutes, auto-expire minutes

**Modals, drawers & sheets:**

- `OV-O39a` **Rule value sheet** — Amount / count input

**Owner.com UX principles applied:**

- Rules read as sentences: 'Auto-accept orders under ₹500'

**States:** standard — Form · empty — n/a · mobile — Same

#### `SCR-O40` Payments & gateway

**Surface:** Owner desktop app · **Route:** `/app/settings/payments` · **Density:** low · **Items:** 8, 40, 45

**Primary user goal:** Gateway status, pay-flow default, refund default, UPI ID.

**Layout & key components:**

- Gateway card: provider, status pill, 'Connect / Reconnect', merchant id
- Pay-flow default (pay now / at checkout)
- Refund default (3 modes) — 'still open per outlet type' note for us, hidden for owners
- UPI ID

**Modals, drawers & sheets:**

- `OV-A03.7a` **Gateway onboarding return modal** — shared

**Owner.com UX principles applied:**

- 'Your money goes to your account' stated on the card

**States:** standard — Form · empty — Gateway not started · mobile — Same

#### `SCR-O41` WhatsApp & owner bot

**Surface:** Owner desktop app · **Route:** `/app/settings/whatsapp` · **Density:** low · **Items:** 51, 56, 61, 65

**Primary user goal:** Owner number, briefing time, bot on/off, triggers on/off, weekly cap info; (v1.5) connect own WABA.

**Layout & key components:**

- Owner phone + 'send test'
- Briefing time, language
- Bot switch (read-only note: 'answers questions; changes come next version')
- Triggers switches
- Caps & quota read-only card
- Own WABA card (v1.5, greyed 'coming soon')

**Owner.com UX principles applied:**

- Read-only facts (caps) shown as cards, not disabled inputs

**States:** standard — Form · empty — n/a · mobile — Same

#### `SCR-O42` Practice mode

**Surface:** Owner desktop app · **Route:** `/app/settings/practice` · **Density:** low · **Items:** 6

**Primary user goal:** Turn the test table on/off; open its QR.

**Layout & key components:**

- Switch, test table QR, 'what practice orders do' 3 bullets

**Owner.com UX principles applied:**

- One switch, one QR

**States:** standard — Card · empty — n/a · mobile — Same

#### `SCR-O43` Plan, billing & export

**Surface:** Owner desktop app · **Route:** `/app/settings/plan` · **Density:** low · **Items:** 9

**Primary user goal:** See the plan, cancel anytime, export everything.

**Layout & key components:**

- Plan card (free / paid, renews, price), 'Start paid plan' (manual billing in v1 → contact sheet), 'Cancel'
- Export card: 'Export everything' → job → download link (7 days)
- Customer count (free plan)

**Modals, drawers & sheets:**

- `OV-O43a` **Cancel sheet** — Reason + 'your data stays exportable'
- `OV-O43b` **Export ready toast** — Link

**Owner.com UX principles applied:**

- Cancel and export are one click each — the promise on the pricing page

**States:** standard — Cards · empty — n/a · mobile — Same

#### `SCR-O44` Team (v1.1) · **v1.1 — reserved**

**Surface:** Owner desktop app · **Route:** `/app/settings/team` · **Density:** low · **Items:** v1.1

**Primary user goal:** Staff logins, permissions, PIN for sensitive actions.

**Layout & key components:**

- Staff list, invite, role/permissions, PIN set

**Modals, drawers & sheets:**

- `OV-O44a` **Invite staff sheet** — Phone/email + permissions

**Owner.com UX principles applied:**

- Reserved frame; v1 shows 'one owner login' note

**States:** standard — List · empty — 'Staff logins coming in v1.1' · mobile — Same

#### `SCR-O45` Printing & KOT (v1.1) · **v1.1 — reserved**

**Surface:** Owner desktop app · **Route:** `/app/settings/print` · **Density:** low · **Items:** v1.1

**Primary user goal:** Thermal print agent pairing, KOT routing, bill copies.

**Layout & key components:**

- Print agent card (Android), KOT on/off, copies

**Owner.com UX principles applied:**

- Reserved

**States:** standard — Card · empty — 'Coming in v1.1 — use the browser print for now' · mobile — Same


### Owner · Global

#### `SCR-O46` Upgrade sheet

**Surface:** Owner desktop app · **Route:** `(overlay)` · **Density:** low · **Items:** 48–60

**Primary user goal:** Turn a blurred paid tile into a paid plan without leaving the screen.

**Layout & key components:**

- What you clicked (e.g. 'Lapsed wall: 23 customers, ₹18,400') shown un-blurred once as the hook
- Paid plan list, price band, 'Talk to us' / 'Start paid plan' (manual in v1)

**Owner.com UX principles applied:**

- Show the real number once — the upgrade sells itself

**States:** standard — Sheet · empty — n/a · mobile — Sheet

*Note: Referenced as OV-O46 from other screens*

#### `SCR-O47` Notifications & alerts drawer

**Surface:** Owner desktop app · **Route:** `(overlay)` · **Density:** mixed · **Items:** 39, 60

**Primary user goal:** Slow orders, cancellation requests, private reviews, gateway issues, template status — one list.

**Layout & key components:**

- Bell in the top bar with count; drawer list grouped by today/earlier; each item deep-links

**Owner.com UX principles applied:**

- Alerts are actionable rows, not toasts that vanish

**States:** standard — Drawer · empty — 'All quiet' · mobile — Full-screen

#### `SCR-O48` Command search

**Surface:** Owner desktop app · **Route:** `(overlay)` · **Density:** low · **Items:** —

**Primary user goal:** Jump to a table, order code, customer, item or setting by typing.

**Layout & key components:**

- ⌘K palette: recent, then results grouped by type

**Owner.com UX principles applied:**

- Same search the bot uses — one index

**States:** standard — Palette · empty — Recents · mobile — Search icon → full-screen


### Customer · Order

#### `SCR-C01` Menu (after scan)

**Surface:** Customer mobile PWA · **Route:** `/v/:slug/t/:token` · **Density:** high · **Items:** 21, 22, 17, 30

**Primary user goal:** Browse and add in seconds; know which table you're on; no login.

**Layout & key components:**

- Top: outlet name + logo, table badge ('Table 4'), open/busy pill, offers banner (one line)
- Sticky category pills (horizontal scroll), search, veg-only toggle
- Item cards: photo (optional, 72px), name, veg dot, price, 'Add' (or 'Unavailable' grey); stepper appears in place after add
- Floating bottom bar when cart > 0: '3 items · ₹450 · View cart'
- Returning customer: greeting line 'Welcome back, Rahul' + 'Your usual' row (top 3)

**Modals, drawers & sheets:**

- `OV-C02` **Item sheet** — Variant radio cards, add-on groups with min/max helpers, instructions, sticky Add with live price (SCR-C02)
- `OV-C01a` **Outlet info sheet** — Address, hours, GST/FSSAI
- `OV-C01b` **Closed / busy banner** — Sticky banner explaining why ordering is paused

**Owner.com UX principles applied:**

- Table badge is the first thing you see — the anchor of trust
- Add in place (stepper morphs from the button) — no cart page detour (Owner.com lightning checkout)
- Search + pills always reachable; the bottom bar is the only floating element

**States:** standard — Menu · empty — Menu empty → 'Ask staff for a menu' · mobile — This is the mobile screen; desktop = centered 420px column

#### `SCR-C02` Item sheet

**Surface:** Customer mobile PWA · **Route:** `(sheet)` · **Density:** low · **Items:** 22, 11

**Primary user goal:** Pick variant and add-ons, write an instruction, add.

**Layout & key components:**

- Photo header (if any), name, description
- Variant radio cards, add-on groups with min/max helper ('Pick 1', 'Up to 2') and +₹ chips
- Instructions field (200 chars)
- Sticky CTA: 'Add · ₹220' with qty stepper

**Owner.com UX principles applied:**

- Price in the button updates live
- Required groups block the button with the group name, not a toast

**States:** standard — Bottom sheet 90% · empty — n/a · mobile — Sheet

#### `SCR-C03` Cart

**Surface:** Customer mobile PWA · **Route:** `/v/:slug/cart` · **Density:** mixed · **Items:** 22, 23, 28

**Primary user goal:** Confirm what's in this round and place it.

**Layout & key components:**

- Table badge header
- Lines with steppers, variant/add-on summary, instruction
- Coupon field (Apply → green discount line / inline error)
- Totals: items, CGST, SGST, total
- Primary sticky: 'Place order · ₹450'; helper line 'You pay after the kitchen accepts' (pay-now outlets) or 'Pay when you leave' (checkout outlets)
- Price-changed state: diff list with 'Update and continue'

**Modals, drawers & sheets:**

- `OV-C04` **Phone OTP sheet** — Phone → 6-box code; first-timer adds name + consent checkbox + optional birthday (SCR-C04)
- `OV-C03a` **Price changed sheet** — old vs new prices, one button

**Owner.com UX principles applied:**

- The primary button says 'Place order', never 'Pay' — payment comes after acceptance
- One screen, one button

**States:** standard — Cart · empty — 'Your cart is empty' + back to menu · mobile — Sheet-like page

#### `SCR-C04` Phone OTP (first order)

**Surface:** Customer mobile PWA · **Route:** `(sheet)` · **Density:** low · **Items:** 23

**Primary user goal:** Verify the phone in 10 seconds; first-timer gives a name and consent.

**Layout & key components:**

- Step 1: phone (+91 fixed, 10 digits, numeric keypad), 'Send code'
- Step 2: 6-box OTP with auto-advance and 60 s resend; first-timer: name field + consent checkbox ('Get my bill and offers on WhatsApp' with notice vX link), optional birthday (day/month)
- Returning: OTP only, name greeting

**Modals, drawers & sheets:**

- `OV-C04a` **Notice text sheet** — DPDP notice vX

**Owner.com UX principles applied:**

- Zero-friction: phone → code → done; name and consent are on the same step as the code, not a third screen
- Consent copy is a benefit ('get your bill on WhatsApp'), pre-checked is NOT allowed for marketing

**States:** standard — Sheet · empty — n/a · mobile — Sheet; OTP autofill from SMS/WhatsApp

#### `SCR-C05` Waiting for acceptance

**Surface:** Customer mobile PWA · **Route:** `/v/:slug/s/:token` · **Density:** low · **Items:** 23, 27

**Primary user goal:** Reassure: the order reached the kitchen; nothing is charged yet.

**Layout & key components:**

- Big status: 'Sent to kitchen' with a gentle progress indicator
- Order summary collapsed
- 'Cancel request' link (only if the outlet allows, only while waiting)
- Auto-transitions to SCR-C06 on accept, or shows the rejection reason with 'Order something else'

**Modals, drawers & sheets:**

- `OV-C14` **Cancellation request sheet** — Optional reason chips → requested / approved / denied states (SCR-C14)

**Owner.com UX principles applied:**

- Immediate feedback loop: the state changes on the screen within 2 s of the desk action

**States:** standard — Waiting · empty — n/a · mobile — Full screen; push/vibrate on accept


### Customer · Session

#### `SCR-C06` Live table (session)

**Surface:** Customer mobile PWA · **Route:** `/v/:slug/s/:token` · **Density:** mixed · **Items:** 25, 23

**Primary user goal:** Track rounds, order more, call the waiter, ask for the bill — and pay when prompted.

**Layout & key components:**

- Table badge + diners chips (you + 2 others, first names)
- Rounds list: each with status pill and ETA ('Round 2 · Preparing · ~8 min')
- Pay prompt card (pay-now outlets, after acceptance): 'Pay ₹450 for round 1' with UPI/gateway button — dismissible, re-surfaces
- Actions row: '+ Order more' (primary), 'Call waiter' (turns into 'Waiter notified ✓' for 60 s), 'Request bill'
- Running total footer → 'My bill'

**Modals, drawers & sheets:**

- `OV-C07` **Payment sheet** — Scope (this round / my items / whole table) + UPI intent buttons / gateway (SCR-C07)
- `OV-C06a` **Waiter notified toast** — inline state change, not a toast

**Owner.com UX principles applied:**

- Status by colour + position, times not jargon
- Pay prompt is a card, not a modal — never blocks ordering more

**States:** standard — Session · empty — No rounds yet (joined a table) → 'Order something' · mobile — Full screen; sticky actions


### Customer · Pay

#### `SCR-C07` Payment sheet

**Surface:** Customer mobile PWA · **Route:** `(sheet)` · **Density:** low · **Items:** 23, 26, 40

**Primary user goal:** Pay this round / my items / the whole table with the fewest taps.

**Layout & key components:**

- Scope segmented: This round · My items · Whole table (amounts live)
- Method: UPI intent buttons (GPay · PhonePe · Paytm · any UPI) → gateway checkout; card via gateway
- 'Paying at the counter instead?' link (checkout outlets / gateway down)

**Modals, drawers & sheets:**

- `OV-C08` **Payment status** — Pending with time expectation → captured / failed with next step (SCR-C08)

**Owner.com UX principles applied:**

- UPI intent first (India); amount repeated on the button ('Pay ₹450')

**States:** standard — Sheet · empty — n/a · mobile — Sheet

#### `SCR-C08` Payment status

**Surface:** Customer mobile PWA · **Route:** `/pay/:id` · **Density:** low · **Items:** 23

**Primary user goal:** Confirm the money landed — or tell them what to do.

**Layout & key components:**

- Pending: spinner + 'Confirming with your bank… usually 10 s' + polling
- Captured: green check, amount, 'Back to table'
- Failed: reason + 'Try again' / 'Pay at counter'

**Owner.com UX principles applied:**

- Never leave a pending state without a next step and a time expectation

**States:** standard — Pending/Captured/Failed · empty — n/a · mobile — Full screen


### Customer · Bill

#### `SCR-C09` My bill

**Surface:** Customer mobile PWA · **Route:** `/v/:slug/s/:token/bill` · **Density:** mixed · **Items:** 26, 28

**Primary user goal:** See every round, who ordered what, and pay the whole table or just my items.

**Layout & key components:**

- Rounds with lines; each line shows the diner's first name (per-person attribution)
- Toggle: Whole table · My items → totals recompute
- Coupon field (before payment only)
- Totals with GST lines and round-off
- Primary: 'Pay ₹1,240' (whole) / 'Pay my ₹450'; checkout outlets: 'Pay at counter' + 'Request bill'

**Modals, drawers & sheets:**

- `OV-C07` **Payment sheet** — shared

**Owner.com UX principles applied:**

- Attribution shown as names, not numbers — group dining made simple without a shared cart

**States:** standard — Bill · empty — No accepted rounds · mobile — Full screen

#### `SCR-C10` Bill paid & e-bill

**Surface:** Customer mobile PWA · **Route:** `/bills/:id` · **Density:** low · **Items:** 29, 30, 41, 60

**Primary user goal:** Show the GST invoice, send it on WhatsApp, ask for a rating.

**Layout & key components:**

- Invoice view (seller GSTIN/FSSAI, number, lines, GST, total, modes)
- First visit: 'Sent to your WhatsApp ✓' (the consent moment); repeat: 'Send to WhatsApp' button
- Rating prompt: 5 stars inline
- Add-to-home-screen card (after first order)
- 'Your visits' link

**Modals, drawers & sheets:**

- `OV-C11` **Review flow** — 5 stars → thanks (Google in v1.1) or private feedback chips + text (SCR-C11)
- `OV-C19` **Add to home screen prompt** — Native-style card with 'Add' / 'Later'

**Owner.com UX principles applied:**

- The bill is the receipt and the review ask — one screen, no chase

**States:** standard — Invoice · empty — n/a · mobile — Full screen

#### `SCR-C11` Review (routing)

**Surface:** Customer mobile PWA · **Route:** `/bills/:id/review` · **Density:** low · **Items:** 60

**Primary user goal:** 4–5★ → Google (v1.1 switch) / thanks; ≤3★ → private feedback to the owner.

**Layout & key components:**

- Stars (large)
- ≥4: 'Thank you!' + (v1.1) 'Share on Google' button
- ≤3: 'Tell us what went wrong' chips (food · speed · cleanliness · price · staff) + text; 'Send privately'

**Owner.com UX principles applied:**

- Unhappy path is private and short — recovery, not exposure

**States:** standard — Stars · empty — n/a · mobile — Full screen


### Customer · Takeaway

#### `SCR-C12` Counter takeaway

**Surface:** Customer mobile PWA · **Route:** `/v/:slug/t/:counterToken` · **Density:** mixed · **Items:** 20, 24

**Primary user goal:** Order at the counter QR, pay first, get a token, get pinged when ready.

**Layout & key components:**

- Menu (SCR-C01) with 'Takeaway' badge instead of a table
- Cart → OTP → Payment sheet (pay first) → Token screen: big token number, 'We'll ping you on WhatsApp', status Preparing → Ready → Collected

**Modals, drawers & sheets:**

- `OV-C07` **Payment sheet** — shared

**Owner.com UX principles applied:**

- Token number is huge; the phone is the buzzer

**States:** standard — Token · empty — n/a · mobile — Full screen


### Customer · Pay-only

#### `SCR-C13` Pay-only mode

**Surface:** Customer mobile PWA · **Route:** `/v/:slug/t/:token (pay_only)` · **Density:** low · **Items:** 4

**Primary user goal:** Scan, see the bill staff entered, pay — no menu.

**Layout & key components:**

- Table badge, outlet name
- Bill lines (from staff entry) and total
- OTP on first pay → identity captured
- Pay button (UPI intent)
- No session yet: 'Nothing to pay yet — ask staff' with the menu link if qrMode = both

**Modals, drawers & sheets:**

- `OV-C04` **Phone OTP sheet** — shared
- `OV-C07` **Payment sheet** — shared

**Owner.com UX principles applied:**

- Zero staff disruption: the customer never orders, only pays — the phone is still captured

**States:** standard — Bill · empty — No bill → message · mobile — Full screen


### Customer · Session

#### `SCR-C14` Cancellation request

**Surface:** Customer mobile PWA · **Route:** `(sheet)` · **Density:** low · **Items:** 27

**Primary user goal:** Ask to cancel; the owner decides.

**Layout & key components:**

- Reason chips (optional), 'Request cancellation'
- State: requested → approved / denied with the reason

**Owner.com UX principles applied:**

- Language is 'request' — expectation set correctly

**States:** standard — Sheet · empty — n/a · mobile — Sheet


### Customer · Account

#### `SCR-C15` My visits

**Surface:** Customer mobile PWA · **Route:** `/me` · **Density:** mixed · **Items:** 30

**Primary user goal:** History across every Regulars outlet; profile; consent.

**Layout & key components:**

- List of visits grouped by outlet: date, total, bill link, rating
- Profile sheet: name, birthday, language, consent (marketing / utility) with dates, 'Stop all messages'

**Modals, drawers & sheets:**

- `OV-C15a` **Profile sheet** — Name, birthday, language, consent flags with dates, Stop all messages
- `OV-C15b` **Erase my data sheet** — Request erasure → confirmation

**Owner.com UX principles applied:**

- Consent is editable by the customer here; STOP works from WhatsApp too

**States:** standard — List · empty — 'Your visits will show here' · mobile — Full screen


### Customer · States

#### `SCR-C16` Closed / busy / blocked

**Surface:** Customer mobile PWA · **Route:** `(state)` · **Density:** low · **Items:** 2, 35, 48

**Primary user goal:** Explain why ordering is paused without dead-ending.

**Layout & key components:**

- Illustration-free card: 'Kitchen is closed — opens 17:00' / 'Busy right now, ETAs are longer' (ordering allowed with extra ETA) / 'Please order at the counter' (blocked)
- Menu remains browsable

**Owner.com UX principles applied:**

- Say when it opens; keep the menu

**States:** standard — Banner + menu · empty — n/a · mobile — Full screen

#### `SCR-C17` Invalid or regenerated QR

**Surface:** Customer mobile PWA · **Route:** `(410)` · **Density:** low · **Items:** 18

**Primary user goal:** Tell them to ask staff, offer the counter.

**Layout & key components:**

- 'This QR was replaced — ask staff for the new one' + outlet name + counter link if any

**Owner.com UX principles applied:**

- Never a blank error

**States:** standard — Card · empty — n/a · mobile — Full screen

#### `SCR-C18` Practice table notice

**Surface:** Customer mobile PWA · **Route:** `(state)` · **Density:** low · **Items:** 6

**Primary user goal:** Make it obvious this is a test.

**Layout & key components:**

- Striped 'Practice table — nothing will be cooked' banner over the normal flow

**Owner.com UX principles applied:**

- Same flow, one banner

**States:** standard — Banner · empty — n/a · mobile — Same


### Admin

#### `SCR-X01` Vendors

**Surface:** Super-admin panel · **Route:** `/admin/vendors` · **Density:** high · **Items:** 1, 62

**Primary user goal:** Every vendor with plan, status, door, live date, last activity.

**Layout & key components:**

- Table with filters (status, plan, door, city), search
- Row → vendor detail
- 'Onboard manually' button

**Modals, drawers & sheets:**

- `OV-X01a` **Manual onboarding sheet** — Name, phone, email, outlet → magic link on WhatsApp

**Owner.com UX principles applied:**

- Dense table, sticky header, keyboard row navigation

**States:** standard — Table · empty — n/a · mobile — Not supported

#### `SCR-X02` Vendor detail

**Surface:** Super-admin panel · **Route:** `/admin/vendors/:id` · **Density:** mixed · **Items:** 62

**Primary user goal:** Change plan/status/feature flags, see outlets, users, usage, messages, audit.

**Layout & key components:**

- Header with status pill and actions (suspend, change plan)
- Tabs: Overview · Outlets · Users · Usage · Messages · Audit
- Feature flags list with switches (ownerBotWrites, churnMl…)

**Modals, drawers & sheets:**

- `OV-X02a` **Suspend confirm** — Reason; effect list
- `OV-X02b` **Plan change sheet** — Plan, price, renews

**Owner.com UX principles applied:**

- Everything that changes writes audit_logs and shows it in the Audit tab

**States:** standard — Tabs · empty — n/a · mobile — Not supported

#### `SCR-X03` Templates

**Surface:** Super-admin panel · **Route:** `/admin/templates` · **Density:** high · **Items:** 61

**Primary user goal:** Meta template sync and status.

**Layout & key components:**

- Table: key, kind, category, language, Meta status, sent 30d, read rate; 'Sync from Meta'
- Editor drawer: components JSON with preview

**Modals, drawers & sheets:**

- `OV-X03a` **Template editor drawer** — components + preview

**Owner.com UX principles applied:**

- Meta status is the truth; local edits mark 'pending'

**States:** standard — Table · empty — n/a · mobile — Not supported

#### `SCR-X04` Message ledger

**Surface:** Super-admin panel · **Route:** `/admin/messages` · **Density:** high · **Items:** 61

**Primary user goal:** Cross-tenant message search for support: status timeline, errors, cost.

**Layout & key components:**

- Filters: vendor, phone, status, purpose, date
- Table with status timeline chips; row → raw + timeline drawer
- DLQ tab with retry

**Modals, drawers & sheets:**

- `OV-X04a` **Message drawer** — Timeline, error, raw payload, retry

**Owner.com UX principles applied:**

- Support tool: search by phone is the first field

**States:** standard — Table · empty — n/a · mobile — Not supported

#### `SCR-X05` Webhooks

**Surface:** Super-admin panel · **Route:** `/admin/webhooks` · **Density:** high · **Items:** 61

**Primary user goal:** See raw webhook events, failures, reprocess.

**Layout & key components:**

- Filters: source, status; table; row → raw JSON drawer with 'Reprocess'

**Modals, drawers & sheets:**

- `OV-X05a` **Raw event drawer** — JSON viewer + reprocess

**Owner.com UX principles applied:**

- Raw is stored; reprocess is idempotent

**States:** standard — Table · empty — n/a · mobile — Not supported

#### `SCR-X06` Tools registry

**Surface:** Super-admin panel · **Route:** `/admin/tools` · **Density:** high · **Items:** 64

**Primary user goal:** The action registry: which tools the bot may call, side effects, deprecations.

**Layout & key components:**

- Table: key, module, side effect, requires confirmation, surfaces (api/bot/automation/ui), status; switch for bot surface
- Row → schema drawer (input JSON schema, examples)

**Modals, drawers & sheets:**

- `OV-X06a` **Tool schema drawer** — JSON schema + examples

**Owner.com UX principles applied:**

- Irreversible tools can never be set to auto — the switch is disabled with the reason

**States:** standard — Table · empty — n/a · mobile — Not supported

#### `SCR-X07` Agents

**Surface:** Super-admin panel · **Route:** `/admin/agents` · **Density:** mixed · **Items:** 65

**Primary user goal:** Prompt versions, allowed tools, guardrails, cost.

**Layout & key components:**

- Agent cards; detail: prompt versions (diff), allowed tool keys, cost/day, runs, refusal rate

**Modals, drawers & sheets:**

- `OV-X07a` **New prompt version sheet** — Text + notes → activates

**Owner.com UX principles applied:**

- Prompt changes are versions, never edits

**States:** standard — Cards · empty — n/a · mobile — Not supported

#### `SCR-X08` Audit log

**Surface:** Super-admin panel · **Route:** `/admin/audit` · **Density:** high · **Items:** 62

**Primary user goal:** Who did what, across tenants.

**Layout & key components:**

- Filters: vendor, actor, action, date; table with before/after diff drawer

**Modals, drawers & sheets:**

- `OV-X08a` **Diff drawer** — before/after JSON diff

**Owner.com UX principles applied:**

- Read-only

**States:** standard — Table · empty — n/a · mobile — Not supported

#### `SCR-X09` Platform stats

**Surface:** Super-admin panel · **Route:** `/admin` · **Density:** high · **Items:** 62

**Primary user goal:** Vendors live, orders/day, messages/day, cost, DLQ size, replication lag, DEFAULT partition rows.

**Layout & key components:**

- Stat tiles + 30-day charts; health strip (DLQ, lag, partitions)

**Owner.com UX principles applied:**

- Ops health next to growth

**States:** standard — Tiles · empty — n/a · mobile — Not supported


### Future · v1.1

#### `SCR-F01` Staff login & PIN prompt · **v1.1 — reserved**

**Surface:** Owner desktop app · **Route:** `/login (staff)` · **Density:** low · **Items:** v1.1

**Primary user goal:** Staff logins with limited permissions; PIN prompt before void/comp/day-close/refund.

**Layout & key components:**

- PIN keypad modal reused across sensitive actions

**Modals, drawers & sheets:**

- `OV-F01a` **PIN modal** — 4–6 digit keypad

**Owner.com UX principles applied:**

- Same PIN modal everywhere

**States:** standard — Modal · empty — n/a · mobile — Sheet

#### `SCR-F02` Move & merge tables · **v1.1 — reserved**

**Surface:** Owner desktop app · **Route:** `(overlay on floor)` · **Density:** low · **Items:** v1.1

**Primary user goal:** Move (v1 candidate) and merge sessions.

**Layout & key components:**

- Floor mini-grid picker; merge = pick two tiles → confirm totals

**Owner.com UX principles applied:**

- Extends OV-O07a

**States:** standard — Picker · empty — n/a · mobile — Sheet

#### `SCR-F03` Order modification after placing · **v1.1 — reserved**

**Surface:** Owner desktop app · **Route:** `(drawer)` · **Density:** mixed · **Items:** v1.1

**Primary user goal:** Edit lines before preparing; customer sees the change.

**Layout & key components:**

- Order detail drawer gains an 'Edit' state with steppers

**Owner.com UX principles applied:**

- Extends SCR-O03

**States:** standard — Drawer · empty — n/a · mobile — Sheet

#### `SCR-F04` Happy hour & price lists · **v1.1 — reserved**

**Surface:** Owner desktop app · **Route:** `/app/menu/pricing` · **Density:** high · **Items:** v1.1

**Primary user goal:** Time-windowed prices; dine-in vs takeaway lists.

**Layout & key components:**

- Price list tabs on the menu editor; category time windows

**Owner.com UX principles applied:**

- Extends SCR-O12

**States:** standard — Tabs · empty — n/a · mobile — Read-only

#### `SCR-F05` Google review routing · **v1.1 — reserved**

**Surface:** Customer mobile PWA · **Route:** `/bills/:id/review` · **Density:** low · **Items:** v1.1

**Primary user goal:** 4–5★ → one-tap Google Maps review.

**Layout & key components:**

- 'Share on Google' button on SCR-C11 positive path

**Owner.com UX principles applied:**

- Extends SCR-C11

**States:** standard — Button · empty — n/a · mobile — Same


### Future · v1.5

#### `SCR-F06` Bot write confirmation (in-app handoff) · **v1.5 — reserved**

**Surface:** Owner desktop app · **Route:** `/app/confirm/:executionId` · **Density:** low · **Items:** v1.5

**Primary user goal:** Anything with a preview (campaign, bulk change, refund) proposed by the bot lands here for one-tap confirm.

**Layout & key components:**

- Proposed action card (what, who, ₹), preview (campaign composer step 5 / bulk diff), 'Confirm' · 'Reject'; expires in 10 min

**Owner.com UX principles applied:**

- Reads free, writes confirmed, irreversible never automatic

**States:** standard — Card · empty — Expired state · mobile — Full screen from the WhatsApp link

#### `SCR-F07` AI copy & suggested campaigns · **v1.5 — reserved**

**Surface:** Owner desktop app · **Route:** `/app/marketing/suggested` · **Density:** mixed · **Items:** v1.5

**Primary user goal:** Hinglish copy with mandatory approval; suggested campaigns as cards.

**Layout & key components:**

- Suggestion cards → composer prefilled; copy editor with approve

**Owner.com UX principles applied:**

- Extends SCR-O29/O30

**States:** standard — Cards · empty — n/a · mobile — Cards

#### `SCR-F08` Own WhatsApp number (WABA connect) · **v1.5 — reserved**

**Surface:** Owner desktop app · **Route:** `/app/settings/whatsapp/connect` · **Density:** low · **Items:** v1.5

**Primary user goal:** Meta embedded signup; quality rating card.

**Layout & key components:**

- Connect card → Meta flow → status

**Owner.com UX principles applied:**

- Extends SCR-O41

**States:** standard — Card · empty — n/a · mobile — Same

#### `SCR-F09` Points loyalty · **v1.5 — reserved**

**Surface:** Customer mobile PWA · **Route:** `/me/points` · **Density:** low · **Items:** v1.5

**Primary user goal:** Fixed-mechanics points, replayable from events.

**Layout & key components:**

- Balance, history, redeem on bill

**Owner.com UX principles applied:**

- Fixed mechanics (Owner.com lesson)

**States:** standard — Card · empty — n/a · mobile — Same


### Future · v2

#### `SCR-F10` Hotel room tab · **v2 — reserved**

**Surface:** Owner desktop app · **Route:** `/app/tables (kind room)` · **Density:** mixed · **Items:** v2

**Primary user goal:** Room QR, room tab across a stay, checkout settle.

**Layout & key components:**

- Table sheet variant with stay dates and multiple sessions

**Owner.com UX principles applied:**

- Extends SCR-O07

**States:** standard — Drawer · empty — n/a · mobile — Sheet

#### `SCR-F11` Outlet switcher (multi-outlet) · **v2 — reserved**

**Surface:** Owner desktop app · **Route:** `(top bar)` · **Density:** low · **Items:** v2

**Primary user goal:** Switch outlet; roll-up reports.

**Layout & key components:**

- Top-bar outlet dropdown; reports gain an outlet filter

**Owner.com UX principles applied:**

- Extends top bar

**States:** standard — Dropdown · empty — n/a · mobile — Sheet

#### `SCR-F12` Inventory · **v2 — reserved**

**Surface:** Owner desktop app · **Route:** `/app/inventory` · **Density:** high · **Items:** v2

**Primary user goal:** Stock, recipes, purchase orders, movements ledger.

**Layout & key components:**

- Items table, movements ledger, PO editor

**Owner.com UX principles applied:**

- New module; orders/menu untouched

**States:** standard — Table · empty — n/a · mobile — Read-only

#### `SCR-F13` Table booking · **v2 — reserved**

**Surface:** Customer mobile PWA · **Route:** `/o/:slug/book` · **Density:** low · **Items:** v2

**Primary user goal:** Reserve a table.

**Layout & key components:**

- Date/time/party picker

**Owner.com UX principles applied:**

- New

**States:** standard — Form · empty — n/a · mobile — Same


---

## 2b. Overlay index (drawers, sheets, modals)

| ID | Name | Used by | Description |
|---|---|---|---|
| `OV-S01a` | Demo booking sheet | SCR-S01, SCR-S05, SCR-S06 | Slide-over with 4 fields (name, phone, outlet, city) + calendar embed; submits without leaving the page |
| `OV-S01b` | Video/product tour modal | SCR-S01 | 90-second screen recording; closes on Esc |
| `OV-S02a` | Screenshot lightbox | SCR-S02 | Click any screenshot to enlarge |
| `OV-S03a` | Message-cost calculator | SCR-S03 | Small inline calculator: customers × messages/month → ₹ estimate |
| `OV-S07a` | Booked confirmation | SCR-S07 | Inline replace of the form: date/time, add-to-calendar, WhatsApp confirmation note |
| `OV-A01a` | Forgot password sheet | SCR-A01 | Email → reset link; success replaces the form |
| `OV-A02a` | Existing account modal | SCR-A02 | Email already used → 'log in instead' with the email prefilled |
| `OV-A03a` | Skip confirmation | SCR-A03, SCR-A03.6 | Only for GST (step 6): 'You can skip now; you will need it before your first invoice' |
| `OV-A03.1a` | Hours range editor | SCR-A03.1, SCR-O38 | Add a second range for split shifts (e.g. 08–11, 17–23) |
| `OV-O16` | Import review grid | SCR-A03.2, SCR-O15 | Confidence-sorted staging grid with page-crop thumbnail and phone preview (SCR-O16) |
| `OV-A03.3a` | QR sheet preview | SCR-A03.3, SCR-O10 | PDF preview in a modal with print |
| `OV-A03.7a` | Gateway onboarding return modal | SCR-A03.7, SCR-O40 | Status + 'what's pending' checklist from the provider |
| `OV-O27a` | Column mapper | SCR-A03.8, SCR-O27 | Dropdown per CSV column (name / phone / birthday / tags) over a 5-row preview; shows valid / duplicate / invalid counts live |
| `OV-O01a` | Insight action sheet | SCR-O01, SCR-O34 | Shows the insight payload + 'Apply' (calls the tool) or 'Dismiss' |
| `OV-O46` | Upgrade sheet | SCR-O01, SCR-O25, SCR-O29 | The clicked value shown un-blurred once + paid plan list + Start paid plan (SCR-O46) |
| `OV-O03` | Order detail drawer | SCR-O02, SCR-O05 | Timeline, lines, customer, payment strip and state-matched actions (SCR-O03); opens on card click |
| `OV-O02a` | Reject reason menu | SCR-O02 | Presets: out of stock · closing · suspicious · other (text) |
| `OV-O02b` | Accept with ETA popover | SCR-O02 | ETA chips + note to customer |
| `OV-O04` | Staff order entry | SCR-O02 | Two-pane POS: item tiles + slip, table picker, phone identity (SCR-O04) |
| `OV-O03a` | Cancel confirm | SCR-O03 | Reason required; warns if a payment is captured → refund needed first |
| `OV-O04a` | Variant/add-on picker | SCR-O04 | Same sheet as the customer item sheet, desktop sized |
| `OV-O04b` | Table picker | SCR-O04 | Floor mini-grid; occupied tables add a round to the existing session |
| `OV-O07` | Table sheet (drawer) | SCR-O06 | Rounds, diners with badges, service calls, totals, Settle / Add round / Move / Void (SCR-O07) |
| `OV-O06a` | Open table for staff order | SCR-O06 | Empty tile click → SCR-O04 with the table preselected |
| `OV-O08` | Settle drawer | SCR-O07 | Invoice preview, multi-mode payment rows, tip, Settle & send e-bill / Print (SCR-O08) |
| `OV-O07a` | Move table picker | SCR-O07 | Floor mini-grid of empty tables |
| `OV-O07b` | Discount / coupon sheet | SCR-O07 | Code or manual amount with reason |
| `OV-O07c` | Complimentary line sheet | SCR-O07 | Pick line + reason |
| `OV-O07d` | Void confirm | SCR-O07 | Reason required; blocked if any payment captured |
| `OV-O08a` | UPI QR modal | SCR-O08 | Outlet UPI QR large for the customer to scan |
| `OV-O08b` | Gateway link sent toast | SCR-O08 | 'Payment link sent to +91…' |
| `OV-O08c` | Print preview | SCR-O08 | 80 mm HTML |
| `OV-O10a` | Add tables sheet | SCR-O10 | Count + kind + zone |
| `OV-O10b` | Regenerate confirm | SCR-O10 | 'Old QR will show "ask staff"' |
| `OV-O10c` | Standee order sheet | SCR-O10 | Quantity + address → manual fulfilment |
| `OV-O26` | Customer profile drawer | SCR-O11, SCR-O25 | Visits, spend, usual order, consent, block (SCR-O26) |
| `OV-O13` | Item sheet | SCR-O12 | Name, price, veg, GST, photo, variants, add-on groups, price history (SCR-O13) |
| `OV-O12a` | Availability popover | SCR-O12 | Unavailable until: next open · custom time · until I switch it back; limited count |
| `OV-O12b` | Category sheet | SCR-O12 | Name, active, happy-hour window (v1.1 field greyed) |
| `OV-O13a` | Price history drawer | SCR-O13 | Revisions table: version, price, who, when |
| `OV-O13b` | Delete confirm | SCR-O13 | Soft delete; 'orders keep their snapshot' |
| `OV-O14a` | Apply confirm | SCR-O14 | Shows the diff summary; writes revisions |
| `OV-O16a` | Row crop zoom | SCR-O16 | Enlarged crop of the source for a row |
| `OV-O16b` | Apply confirm | SCR-O16 | Counts + 'prices become live immediately' |
| `OV-O17a` | Banner editor sheet | SCR-O17 | Text with live phone-frame preview, kind, schedule, optional coupon |
| `OV-O18a` | Custom range picker | SCR-O18 | Two-month calendar |
| `OV-O22` | Bill detail drawer | SCR-O20, SCR-O22 | Invoice layout, payments list, refund button, resend e-bill, print |
| `OV-O23a` | Close confirm | SCR-O23 | Repeats variance; PIN in v1.1 |
| `OV-O23b` | Day-close history | SCR-O23 | Table by date with reopen (admin) |
| `OV-O24a` | Coupon editor sheet | SCR-O24 | Code, flat/%, value, max discount, min order, validity, total uses, per-customer limit |
| `OV-O26a` | Block confirm | SCR-O26 | Reason required |
| `OV-O26b` | Send template sheet | SCR-O26 | Approved templates + variables; counts against caps |
| `OV-O28a` | Segment preview drawer | SCR-O28 | Members list |
| `OV-O29a` | Automation settings sheet | SCR-O29 | Enable, cooldown days, daily cap, quiet hours, template, offer |
| `OV-O30a` | Test send sheet | SCR-O30 | Send to the owner's own number |
| `OV-O30b` | Holdout explainer popover | SCR-O30 | '20 % won't get it so you can see the real difference' |
| `OV-O31a` | Recipient drawer | SCR-O31, SCR-O33 | Message timeline + customer link |
| `OV-O32a` | Fill confirm | SCR-O32 | Count, cost, holdout |
| `OV-O35a` | Briefing detail drawer | SCR-O35 | Text + transcript + rating |
| `OV-O36a` | Reply sheet | SCR-O36 | Text → sent as WhatsApp utility message |
| `OV-O39a` | Rule value sheet | SCR-O39 | Amount / count input |
| `OV-O43a` | Cancel sheet | SCR-O43 | Reason + 'your data stays exportable' |
| `OV-O43b` | Export ready toast | SCR-O43 | Link |
| `OV-O44a` | Invite staff sheet | SCR-O44 | Phone/email + permissions |
| `OV-C02` | Item sheet | SCR-C01 | Variant radio cards, add-on groups with min/max helpers, instructions, sticky Add with live price (SCR-C02) |
| `OV-C01a` | Outlet info sheet | SCR-C01 | Address, hours, GST/FSSAI |
| `OV-C01b` | Closed / busy banner | SCR-C01 | Sticky banner explaining why ordering is paused |
| `OV-C04` | Phone OTP sheet | SCR-C03, SCR-C13 | Phone → 6-box code; first-timer adds name + consent checkbox + optional birthday (SCR-C04) |
| `OV-C03a` | Price changed sheet | SCR-C03 | old vs new prices, one button |
| `OV-C04a` | Notice text sheet | SCR-C04 | DPDP notice vX |
| `OV-C14` | Cancellation request sheet | SCR-C05 | Optional reason chips → requested / approved / denied states (SCR-C14) |
| `OV-C07` | Payment sheet | SCR-C06, SCR-C09, SCR-C12, SCR-C13 | Scope (this round / my items / whole table) + UPI intent buttons / gateway (SCR-C07) |
| `OV-C06a` | Waiter notified toast | SCR-C06 | inline state change, not a toast |
| `OV-C08` | Payment status | SCR-C07 | Pending with time expectation → captured / failed with next step (SCR-C08) |
| `OV-C11` | Review flow | SCR-C10 | 5 stars → thanks (Google in v1.1) or private feedback chips + text (SCR-C11) |
| `OV-C19` | Add to home screen prompt | SCR-C10 | Native-style card with 'Add' / 'Later' |
| `OV-C15a` | Profile sheet | SCR-C15 | Name, birthday, language, consent flags with dates, Stop all messages |
| `OV-C15b` | Erase my data sheet | SCR-C15 | Request erasure → confirmation |
| `OV-X01a` | Manual onboarding sheet | SCR-X01 | Name, phone, email, outlet → magic link on WhatsApp |
| `OV-X02a` | Suspend confirm | SCR-X02 | Reason; effect list |
| `OV-X02b` | Plan change sheet | SCR-X02 | Plan, price, renews |
| `OV-X03a` | Template editor drawer | SCR-X03 | components + preview |
| `OV-X04a` | Message drawer | SCR-X04 | Timeline, error, raw payload, retry |
| `OV-X05a` | Raw event drawer | SCR-X05 | JSON viewer + reprocess |
| `OV-X06a` | Tool schema drawer | SCR-X06 | JSON schema + examples |
| `OV-X07a` | New prompt version sheet | SCR-X07 | Text + notes → activates |
| `OV-X08a` | Diff drawer | SCR-X08 | before/after JSON diff |
| `OV-F01a` | PIN modal | SCR-F01 | 4–6 digit keypad |

---

## 3. Navigation architecture

### 3.1 Owner desktop app (`apps/owner-web`)

```
Left sidebar (fixed 232 px, collapsible to 64 px icons)
├── Today                    SCR-O01
├── Orders                   SCR-O02  (badge: New count)      sub: Takeaway queue O05 · Staff order O04
├── Tables                   SCR-O06                          sub: Manage tables & QR O10
├── Regulars Board           SCR-O11                          (also a docked rail on Orders ≥ 1440)
├── Menu                     SCR-O12                          sub: Bulk edit O14 · Import O15/O16 · Banners O17
├── Reports                  SCR-O18                          tabs: Kitchen O19 · Discounts O20 · Messaging O21 · Bills O22 · Day close O23 · Coupons O24
├── Customers     [paid]     SCR-O25                          sub: Profile O26 · Import O27 · Segments O28
├── Marketing     [paid]     SCR-O29                          sub: Composer O30 · Results O31 · Dead hours O32 · Ledger O33 · Insights O34 · Briefings O35 · Reviews O36
└── Settings                 SCR-O37                          sub: Outlet O38 · Ordering O39 · Payments O40 · WhatsApp & bot O41 · Practice O42 · Plan O43 · Team O44 (v1.1) · Printing O45 (v1.1)

Top bar (56 px): outlet name + open/closed switch · busy toggle · ⌘K search (O48) · bell (O47) · owner menu (logout, plan)
Global overlays: Upgrade sheet (OV-O46 / SCR-O46) · Notifications drawer (O47) · Command search (O48)
```

**Key transitions**
- Login/Signup → wizard (`/setup`) until `outlets.status = live`, then always → Today.
- Today → Orders is one click (primary button) and one keyboard shortcut (`O`).
- Orders card click → Order detail drawer (stays on Orders). Table badge click anywhere → Table sheet drawer (SCR-O07). Customer badge click anywhere → Customer profile drawer (SCR-O26).
- Table sheet → Settle drawer → invoice issued → drawer closes, floor tile turns empty, toast "Bill #INV-2627-0042 sent" with Undo-less confirmation.
- Insight card `Apply` → runs the tool → inline success on the card; never navigates away.
- Paid screens on the free plan render with real counts and blurred rows; any click → Upgrade sheet.
- Alerts (slow order, cancellation request, private review, gateway issue) → bell drawer → deep link to the exact drawer.

**Mobile (owner on a phone)**: bottom tab bar Today · Orders · Tables · More. Orders gets sound + push. Reports, Menu bulk edit, Import review are read-only with "finish on a laptop".

### 3.2 Customer PWA (`apps/customer-web`)

```
Entry: scan → /v/{slug}/t/{token}
  dine-in + qrMode order|both  → C01 Menu → C02 Item sheet → C03 Cart → C04 OTP (first time) → C05 Waiting → C06 Live table ⇄ C01 (order more)
                                         C06 → C07 Payment sheet → C08 Status → C06
                                         C06 → C09 My bill → C07 → C10 Bill paid → C11 Review → C15 My visits
  counter token                → C12 Takeaway: C01 → C03 → C04 → C07 (pay first) → token screen → ready → collected
  qrMode pay_only              → C13 Pay-only bill → C04 → C07 → C10
  states                        → C16 closed/busy/blocked banner · C17 replaced QR · C18 practice banner
Persistent: floating cart bar (menu only) · table badge in every header · 'My visits' from the bill screen
No bottom tab bar — the session is the navigation. Back always returns to the live table screen.
```

### 3.3 Public site + signup (`apps/site` → `apps/owner-web/setup`)

```
Top bar: logo · Product ▾ (mega-menu S10) · For ▾ · Pricing · Demo · Log in · [Start free]
Any CTA → /signup (A02) → /setup (A03.0 … A03.9) → /setup/live (A04) → /app (O01)
Demo/manual doors: magic link on WhatsApp → /setup at the saved step
```

### 3.4 Super-admin (`apps/admin-web`)

```
Left nav: Stats X09 · Vendors X01/X02 · Templates X03 · Messages X04 · Webhooks X05 · Tools X06 · Agents X07 · Audit X08
Desktop only; dense tables; every mutation confirms and audits.
```

---

## 4. Design system & theme foundation inputs

### 4.1 Global patterns (build these once; every screen composes them)

| Pattern | Used on | Spec notes |
|---|---|---|
| **Table badge** | every owner screen, every customer header | Rounded rectangle, monospace-ish numerals, kind icon (table / room / counter), state colour (empty · eating · needs-you); sizes S/M/L. The visual spine. |
| **Customer badge** | desk cards, board, table sheet, profile | Pill: `new` / `repeat ×N`; birthday-week variant; click → profile drawer |
| **Status pill** | orders, payments, campaigns, messages, bills | One component, one colour scale: neutral / info / progress / success / warning / danger. Position + colour, never text alone |
| **Stat tile** | Today, reports, customer book, campaigns | Label, big tabular number, delta chip, sparkline slot; blurred variant for paid-on-free |
| **Card** | desk order card, table tile, insight, automation, plan | Border 1 px, radius 10 px, no shadow at rest; only the "needs you" state lifts (shadow + pulse). Not everything is a card — tables and lists are not |
| **Drawer (slide-over)** | order detail, table sheet, settle, customer profile, bill, message | Right, 480–560 px, scrim 40 %, sticky header + footer action bar, Esc closes; stacks at most 2 deep (drawer → confirm modal) |
| **Bottom sheet** | customer item, OTP, payment, cancellation; owner on mobile | 90 % height max, drag handle, sticky CTA |
| **Modal (confirm)** | refund, void, day-close, regenerate QR, reject | Title states the consequence with the number ("Refund ₹340 to source"); primary repeats the verb; irreversible = danger style |
| **Data table** | reports, bills, customers, campaigns, admin | Sticky header, 40 px rows, right-aligned numerics with `tabular-nums`, row click → drawer, bulk-select bar appears on selection |
| **Grid (editable)** | bulk edit, import review | Spreadsheet keyboard model; staged changes + one Apply |
| **Filter bar** | reports, customers, campaigns, admin | Sticky under the page header; chips for enums, range picker, search; active filters shown as removable chips |
| **Stepper (wizard)** | setup, campaign composer, import | Left rail on desktop / top progress on mobile; one primary per step; skip is secondary text |
| **Sticky summary sidebar** | campaign composer, settle drawer, wizard preview | Numbers update live as choices change (Owner.com checkout) |
| **Phone preview frame** | menu editor, import review, banner editor, wizard, composer | 360×720 frame rendering the real customer components |
| **Segmented control** | QR mode, payment mode, scope (this round / my items / whole table) | ≤ 4 options; selection cards when each option needs a sentence |
| **Selection card** | outlet type, QR mode, refund mode, audience | Icon + title + one consequence line; radio semantics |
| **Inline error / helper** | all forms | Under the field; no toasts for validation |
| **Toast** | success only (sent, saved, closed) | 4 s, bottom-left on desktop, top on mobile; never for errors |
| **Empty state** | every list | One sentence + one action, no illustration required |
| **Heat strip / heatmap** | dead hours, kitchen speed, reports | Cells clickable; single sequential scale; the chart is the form |
| **Alert row** | notifications drawer, desk amber stripes | Actionable, deep-links, dismiss = act |
| **Blurred paid tile** | free plan | Real value rendered under a frosted layer with a lock chip; click → upgrade sheet |

### 4.2 Density groups → token sets

| Group | Base type | Row/line height | Spacing rhythm | Screens |
|---|---|---|---|---|
| High-density | 13 px / 1.4 | 40 px table rows, 36 px grid rows | 4 px | reports, bills, customers list, campaigns, menu editor, order desk, floor, admin |
| Mixed | 14 px / 1.5 | 44 px | 8 px | Today, drawers, board, results, insights |
| Low-density | 16 px / 1.5 (customer) · 15 px (owner forms) | 48 px inputs, 52 px primary buttons | 8 / 16 px | wizard, settings, composer, customer flow, site |

### 4.3 Colour roles the theme must define (semantic, not hue)

- **Ground / panel / sheet** (three surfaces), **line / line-strong**, **ink / ink-2 / ink-3**.
- **Accent** (one brand hue; primary buttons, active nav, links) — keep it off the status scale.
- **Status scale (semantic, fixed):** empty/neutral · eating/info · preparing/progress · ready/success · needs-you/warning · rejected/danger. Table tiles, order cards, status pills and customer-side status all read from this one scale.
- **Paid/locked** frosted overlay token.
- Both themes (light default for customer + site; owner app supports dark for evening service).

### 4.4 Typography roles

Display (site hero, big numbers on tiles), Heading, Body, Label (uppercase, tracked, 11 px), Numeric (tabular, used for money, tokens, invoice numbers, timers), Mono-ish for order codes and table names.

### 4.5 Motion

Only three: card moves between desk columns (200 ms), needs-you pulse (1.2 s loop, stops on hover), drawer slide (240 ms). Respect reduced-motion. No skeleton shimmer on the desk — stale data is shown with a small "updating" dot instead.

### 4.6 Figma file structure (suggested)

Pages: `00 Foundations` (tokens, type, colour roles) · `01 Components` (§4.1) · `02 Site` (S) · `03 Onboarding` (A) · `04 Owner — Operate` (O01–O11) · `05 Owner — Menu` (O12–O17) · `06 Owner — Reports` (O18–O24) · `07 Owner — Grow` (O25–O36) · `08 Owner — Settings & global` (O37–O48) · `09 Customer` (C) · `10 Admin` (X) · `11 Reserved` (F). Frame names = screen IDs; overlay components named by OV-ID.
