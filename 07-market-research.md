# 07 — Market Research: Indian Restaurant POS × WhatsApp Marketing Funnel × AI

Research date: **August 27, 2026.** Sources cited inline; vendor-claimed numbers marked. Companion to docs 01–06 — §9–§11 feed directly back into product/architecture decisions.

---

## 1. Executive summary

The thesis behind our product — small cafes need app-grade ordering + a retention funnel they can't build themselves — is **validated by the market on both ends**:

- **In the US, Owner.com is the proof at scale**: direct ordering + automated retention marketing for independent restaurants, flat $499/mo subscription, no commission → $1B valuation (May 2025), ~$80.6M ARR, 10,000+ restaurants (https://sacra.com/c/owner/).
- **In India, nobody bundles it yet.** Petpooja owns the POS (~100k outlets) but its marketing is fragmented paid add-ons with no journeys/AI. Reelo owns SMB retention (32k+ restaurants, ₹3,250/outlet/mo) but **takes no orders**. DotPe has ordering+POS but shallow campaign automation (Reelo integrates *on top of* it). Thrive — the one direct-ordering-vs-Zomato play — **shut down in 2024** because it monetized by commission with no SaaS anchor (founder post-mortem: https://chechani.substack.com/p/the-thrive-chapter-closes-what-i).
- **GoKwik proves the WhatsApp funnel mechanics** we want, in D2C: phone-first identity → abandoned-checkout recovery on WhatsApp → in-chat checkout → COD confirmation → order-status journeys → segmented win-backs. ~15,000 brands, $2B+ GMV, $481M valuation (June 2025).

The wedge: **an Owner.com-shaped product recomposed for India** — QR-PWA ordering (dine-in/pickup first) + auto-enrolled customer graph + a default-on, AI-run WhatsApp lifecycle engine — sold as one flat subscription (~₹2–5k/outlet/mo), designed around Meta's post-July-2025 per-message pricing (utility-window economics, not blast marketing).

---

## 2. Market context (India)

- Food services: ₹5.69 lakh crore (2024) → ₹7.76 lakh crore by 2028, 8.1% CAGR; organized share 43.8%→52.9% (NRAI IFSR 2024 via https://mediabrief.com/nrai-releases-india-food-services-report-2024/).
- Restaurant-management software: USD 303.6M (2025) → USD 1.58B (2033), 23.2% CAGR; cloud = 60% (https://www.grandviewresearch.com/horizon/outlook/restaurant-management-software-market/india).
- **POS penetration is low**: summing all major vendors' claims gives well under ~300k software-billed F&B outlets vs lakhs of organized restaurants + millions of unorganized eateries — single-digit penetration, near-zero among micro-eateries [inference from vendor claims].
- **Aggregator squeeze — our emotional sales hook**: Zomato 18–25% commission + 18% GST on commission; Swiggy 18–28%. Effective cut on a ₹500 order ≈ ₹155 (~31%) after commission+GST+packaging. A 30-orders/day restaurant pays ~₹1.2L/month total aggregator cost (https://www.dineopen.com/blog/reduce-zomato-swiggy-commission-restaurants.html). Swiggy shut **Minis** (its zero-commission storefront) Aug 2025 — aggregators have exited merchant-SaaS, leaving direct-ordering to POS/SaaS players (https://entrackr.com/snippets/swiggy-to-shut-down-digital-storefront-platform-minis-by-aug-10-9457703).
- ONDC: 3–5% total cost vs 25–30%+, magicpin largest food buyer app (150k orders/day Oct 2024), but volumes remain small, Ola exited Dec 2025; POS integration incomplete — watch, don't bet v1 on it.
- How POS is sold: field sales + resellers (Petpooja pays resellers 15%/conversion). Nobody has cracked self-serve onboarding for micro-eateries.
- Churn drivers for incumbents: add-on cost creep (modules ₹1–3k/mo each), per-aggregator integration fees ₹2–5k, 10–20% annual hikes, hardware bundling, lock-in contracts, weak off-hours support, ₹5–20k data-migration cost to switch.

## 3. Indian POS landscape (condensed)

| Vendor | Segment | Pricing (verify at contact) | Marketing/CRM reality |
|---|---|---|---|
| **Petpooja** (~100k outlets; ₹137 Cr Series C, Sept 2025, ~₹910 Cr val) | SMB mass market, Tier-2/3 strong | Core ₹10k / Growth ₹20k / Scale ₹30k per yr + add-ons + hardware; Year-1 TCO est. ₹80k–2.3L | Basic CRM native; loyalty in ₹20k tier; WhatsApp via partners (Reelo). G2 4.7★ but complaints: add-on creep, support gaps |
| **Restroworks** (ex-POSist; 25k+ outlets, 50 countries) | Enterprise chains | Quote-only; est. ₹50k–2L/yr/outlet, 2–3yr contracts | Deep native CRM/loyalty; ease-of-use rated 2.7/5; not an SMB product |
| **DotPe + Rista** (Temasek/Google-backed, $58M B) | Micro-merchants (free+2–3% commission) up to enterprise (McDonald's, Haldiram's) | Free start, 2–3% online-order commission; Rista quote-only | WhatsApp bills, segmentation, tier loyalty comparatively native; low-end product is storefront-not-POS |
| **UrbanPiper** (40k+ outlets; Swiggy+Zomato both invested) | Middleware king; Prime POS down-market | Prime POS ₹8,475/yr; Hub est. ₹4–15k/mo | Thin — ops/integration positioning, no retention layer. Also Meraki (branded ordering) + Orderline AI (AI phone orders) |
| **LimeTray** | Mid-market/chains (Burger King India) | Quote-only | The old ordering+CRM bundle: website, ordering, loyalty, email/SMS automation; no AI/WhatsApp-native layer; steep learning curve |
| **TMBill** (14k+ outlets) | Budget QSR/cafes | ₹5,999–8,999/yr | **Only incumbent advertising native WhatsApp marketing**; ONDC listed; polarized support reviews |
| **Ciferon** | Budget single-outlet, Tier-2 | ~₹7,500/yr all-inclusive + add-ons | SMS credits bundled; loyalty wallet add-on |
| **QueueBuster** ($8.16M A) | Horizontal retail+F&B | Quote-only | Native CRM/loyalty, khata; restaurant share unclear |
| **GoFrugal ServeEasy** | Inventory-heavy, bakery/production | ₹8,999–24,999/yr | Loyalty w/ SMS; retail DNA, dated UX |
| **SlickPOS** | Freemium micro | Free–~₹1,200/mo | Reliability complaints ("orders vanish") |
| **eZee Optimus** (Yanolja) | Hotel F&B | ₹2,700–4,500/mo | Hotel-first, over-engineered for cafes |
| Paytm/Pine Labs | Payments-led billing | — | Not restaurant-depth (no KOT/recipe/aggregator sync) |

Zomato killed its own POS (Base, ~2019-20); both aggregators now express restaurant-SaaS ambitions only through their UrbanPiper stakes.

**Challenger positioning already in market:** DineOpen ₹300/mo, Restrofi ₹699/mo, BillFeeds ₹999/mo — "all features included, zero transaction fees, no contracts, works on your Android phone." This is the price-rhetoric environment we launch into.

## 4. The retention/CRM layer sitting on top of POS

- **Reelo** (Ahmedabad; $1M from Gokul Rajaram, Feb 2024): loyalty (points/cashback/tiers, auto-enroll), WhatsApp/SMS/email campaigns + templates, triggered journeys (welcome/birthday/win-back), RFM segmentation, feedback→Google review routing, referrals, "Write with AI" copy. **35+ POS integrations incl. Petpooja, Restroworks, DotPe** — reads transactions from the POS. Pricing: free tier; Growth **₹3,250/outlet/mo (₹39,000/yr)** — i.e., the marketing layer costs ~4× Petpooja Core. Claimed "204x ROI" campaigns [vendor]. **Ceiling: takes no orders — no PWA, no payments** (https://reelo.io/petpooja/, https://reelo.io/pricing/).
- **Xeno**: enterprise AI CRM (Taco Bell, Nando's, Biryani By Kilo) — persona AI, inactive detection, send-time optimization. Not SMB.
- **EasyRewardz**: enterprise loyalty, retail-heavy. Not SMB.
- **Thrive (Hashtag Loyalty)**: direct ordering at ~3% commission, Jubilant-backed — **shut down 2024**. Lessons: commission monetization on thin food margins fails; no in-store anchor; investor fear of the duopoly (https://chechani.substack.com/p/the-thrive-chapter-closes-what-i).
- **magicpin**: discovery/vouchers + largest ONDC food seller app — demand channel, but the customer belongs to magicpin, not the restaurant.

## 5. GoKwik teardown (the funnel playbook)

**Company**: f. 2020 (Chirag Taneja et al.); $481M post-money (June 2025, ₹112 Cr growth round); ~15,000 brands; $2B+ GMV; FY24 revenue ₹85 Cr, loss ₹85 Cr; acquired **Tellephant (June 2023) → became KwikChat → rebranded KwikEngage (2024)**; acquired Return Prime (Sept 2024). 2026: launched Kwik Ads (agentic Meta-ads AI) and Kwik Ship (AI shipping/NDR) (https://entrackr.com/decoding/decoding-gokwiks-extended-series-b-round-current-valuation-and-captable-9441350, https://inc42.com/buzz/ecommerce-enabler-gokwik-acquires-chat-commerce-startup-tellephant/).

**Product suite**: KwikCheckout (one-click checkout, address prefill for ~90% via network, Smart COD suite, RTO engine −40% [vendor], 250+ discount templates); KwikPass (anonymous-visitor identification → phone-verified leads); **KwikEngage** (WhatsApp/email/SMS/IG/RCS journeys, WhatsApp Flows, in-chat Instant Checkout, AI chatbot 80%+ auto-resolution, AI voice-call recovery, claimed 90–98% open, 25–30% cart recovery, 20–28× ROAS [vendor]); Return Prime; Kwik Ads; Kwik Ship. Shopify app pricing: checkout free–$29.99/mo + **2.5% on prepaid**; KwikEngage $40/mo (India) + per-message.

**The funnel, step-by-step (transferable core):**
1. **Phone-first identity before payment** — OTP login is step one of checkout; KwikPass fingerprints returning shoppers → every abandoner is reachable. *(We already designed this: customer OTP login before order.)*
2. **Abandonment trigger** — webhook fires the moment checkout is exited → WhatsApp template with product card + deep link back to a **prefilled** checkout, optional coupon.
3. **In-chat completion** — checkout inside WhatsApp: one-tap COD confirm or UPI intent link (documented in the Fire-Boltt case: 2× cart recovery, −40% CAC, 324k WhatsApp leads in 45 days [vendor]).
4. **COD→prepaid (C2P)** — post-order "Pay now & save ₹50" with 3-minute countdown + UPI link.
5. **Escalation** — AI cross-channel retries (~90% delivery), then AI voice-bot call for high-value carts.
6. **Order-status journeys** — confirmation→shipped→delivered on the same thread; −40% WISMO tickets; thread becomes the marketing channel.
7. **Win-back/repeat** — behavioral segments, purchase-history recommendations, festival broadcasts; SAADA: 2.5× retention yr 1, 7× ROI vs email [vendor].

**Transfers to restaurants**: patterns 1–7 nearly verbatim, with one critical change — **the decay window compresses**: food intent expires by the next mealtime, so the recovery ladder is ~10-min nudge → same-meal offer → next-mealtime win-back, NOT 24–72h discount ladders (which train discount-waiting on thin margins). **Doesn't transfer**: the cross-merchant identity network (needs thousands of merchants), RTO/returns economics (no return leg in food), partial-COD/BNPL (tickets too small), deep browse-retargeting (menus browsed in seconds; frequency caps punish it).

**Adjacent stacks**: BIK.ai (IG-comment→DM automation, "66x ROI" [vendor]), Zoko (deepest Shopify-catalog-in-WhatsApp; $39.99–404.99/mo), LimeChat (enterprise L3 AI, publishes the C2P playbook), QuickReply.ai (~$35–199/mo SMB), Interakt (₹3,795/mo Growth; catalog+payments), Haptik (JioMart WhatsApp grocery; SMB AI-agent packs from ₹10,000).

## 6. WhatsApp Business Platform — the facts our funnel must be built on

**Pricing (India, current):** Meta switched to **per-message pricing July 1, 2025** (every delivered template charged; free-form replies free). Rates: **Marketing ₹0.8631** (since Jan 1, 2026; was ₹0.7846), **Utility ₹0.115** — **FREE inside an open 24h customer-service window** — Auth ~₹0.115, Service (replies) free. +18% GST. Price changes now land only on quarter boundaries with notice (https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing, https://msg91.com/guide/whatsapp-pricing-update-2026-and-save-with-msg91).

**Free entry point:** a conversation started from a **Click-to-WhatsApp ad or FB Page CTA opens a 72-hour window where ALL messages (marketing included) are free**. India CTWA cost-per-conversation est. ₹8.5–34 [unverified].

**Sending limits & quality:** business-initiated unique-user limits 250 → 1k → 10k → 100k → unlimited; since **Oct 2025 limits are portfolio-level** (numbers in one Business Manager share the highest tier — new numbers skip warm-up). Quality rating Green/Yellow/Red per number; low-quality templates get paused 3h → 6h → permanently disabled.

**India frequency cap (critical):** since Feb 2024 Meta caps how many marketing templates a user receives **from ALL businesses combined** (undisclosed, est. ~2/day [unverified]). Capped sends fail with **error 131049** — unbilled but silently undelivered; festive blasts can lose 10–25% delivery [anecdotal]. Design consequence: sparse, high-relevance marketing wins; blasts fail structurally. (Context: Meta has paused ALL marketing templates to US numbers since Apr 2025 — they will protect UX aggressively.)

**Capabilities we can use:** WhatsApp **Flows** (in-chat multi-screen forms — booking, feedback, even ordering; free), **catalogs + in-chat carts**, **in-chat payments in India via Razorpay/PayU (any UPI app, since Sept 2023)**, Business Calling API (July 2025; ~$0.0053/min India), CTWA ads. General-purpose AI chatbots are banned on the platform since Jan 2026; business-specific bots are fine.

**Regulatory:** TRAI/DLT does **not** apply to WhatsApp (SMS/voice only) — a real onboarding advantage vs SMS. What applies: Meta's Business Messaging Policy (documented opt-in, enforced by quality ratings/bans) + **DPDP Act 2023** (consent, purpose limitation, withdrawal). Keep per-contact consent proof: timestamp, source, wording — our schema already stores this.

**BSP economics (1,000 marketing msgs/mo example):** raw Meta ≈ ₹1,018 incl. GST. MSG91 = pass-through (₹0 platform fee). Gupshup ≈ +$0.001/msg. AiSensy ≈ ₹3,060/mo all-in (₹1.09/msg + ₹1,500 platform). Wati ≈ ₹3,900+. Twilio: $0.005/msg fee — wrong rail for India. **The platform-fee layer is the margin our product captures.**

**Build path (from the research):**
1. **Prototype on a wholesale pass-through rail** (MSG91 or Gupshup) — ship in days, no Meta App Review.
2. **In parallel, file to become a Meta Tech Provider** (free; business verification → app review with demo videos → live mode → **Embedded Signup**; realistic 1–4 weeks). Then per-message cost = raw Meta rates and we onboard vendors' WABAs self-serve.
3. **Architecture: one WABA per vendor** (via Embedded Signup) as the target — own display name per cafe, isolated quality rating/blast radius, own template set; portfolio-level limits mean new numbers inherit our tier. Platform-shared number only as the v1 stopgap (our current docs assume this) — plan the migration.
4. Funnel economics: drive opt-ins via QR/CTWA (72h free window), deliver order-status as **free in-window utility**, hold paid marketing to 2–4 sends/user/month. A cafe's steady-state Meta bill: ₹500–1,000/mo — leaving room for a ₹999–2,999/mo subscription with pass-through message billing.

## 7. Global reference models

- **Owner.com** — the blueprint. $499/mo flat (or $249 + 5%); website+ordering+branded app+**automated email/SMS funnel** (abandoned cart, win-back, occasions, welcome) off a unified CRM; **deliberately opinionated templates, not a campaign builder** — enabling cross-network A/B testing no single restaurant could run; AI writing assistant; roadmap "AI CMO" agent that proposes campaigns for one-tap approval. Claimed: +30% traffic in 28 days, app doubles reorder rate [vendor]. Criticisms: $499 heavy for thin margins, delivery dependency on DoorDash/Uber, Popmenu lawsuit over its lead-gen grader (Apr 2025) (https://sacra.com/c/owner/, https://research.contrary.com/company/owner).
- **Toast IQ** (Oct 2025): conversational AI assistant on the POS — proactive recommendations, NL analytics, direct actions; AI menu-upsell +6% AOV [vendor].
- **Square** (Oct 2025): AI voice ordering answering 100% of calls; Square AI assistant; Marketing/Loyalty cross-sell on one customer directory.
- **SpotOn Marketing Assist** ($95/mo, Mar 2024): auto-generates a monthly calendar of campaigns + "Slow Day Boost"; automated campaigns' open rates 23% above manual [vendor]. Proof that "AI runs the calendar, owner approves" works at SMB price.
- **Incentivio**: ML churn prediction ("97% accuracy" [vendor]) → automated win-back journeys; restaurants lose 30–40% of top customers yearly.
- **Flipdish** (UK/IE): white-label ordering + auto win-back for lapsed customers — "own your customer data" sold against aggregators. **ChatFood** (MENA): the WhatsApp/IG-native ordering archetype.

**Retention economics (why the funnel is the product):** 65–80% of restaurant revenue is repeat customers; ~70% of first-time diners never return; acquisition costs 5–7× retention; +5% retention → +25–95% profit (Bain, via https://www.restroworks.com/blog/customer-retention-statistics-restaurants/); loyalty members visit ~20% more often [vendor-aggregated].

## 8. AI & automation actually shipping (2025–26)

Shipped and mainstream: **AI campaign copy** (Owner, SpotOn, Reelo "Write with AI"); **churn/RFM scoring** (Incentivio, Xeno; Reelo only static RFM); **AI WhatsApp ordering bots** (AiSensy, Gupshup, Zoko, Haptik/Yellow.ai — horizontal tools, no cafe-vertical bundle); **AI voice phone ordering** (Loman.ai $3.5M seed, 1,500+ restaurants; Square voice — US-centric; India cafes are WhatsApp-first so low priority); **agentic assistants** (Toast IQ, Owner's AI-CMO direction). Frontier not yet in India SMB: predictive churn at SMB price, vernacular/Hinglish AI copy, dynamic offers, demand forecasting.

## 9. Best approaches for us (ranked synthesis)

1. **Own the order to own the funnel.** The QR-PWA is not the product — the customer graph it captures is. Reelo's ceiling is not owning ordering; Owner's success is owning it.
2. **Flat subscription, never commission.** Owner scaled on $499/mo; Thrive died on 3% commission. India band: **₹2–5k/outlet/mo** (Reelo ₹3,250 is the anchor; DineOpen ₹300 is the floor rhetoric). Keep message costs pass-through.
3. **WhatsApp-native lifecycle engineered around Meta's pricing physics**: order status = free in-window utility; QR/CTWA opt-in = 72h free window; marketing sparse (2–4/user/mo — also dodges the India frequency cap); templates pre-approved and quality-monitored.
4. **Opinionated default-on journeys, not a campaign builder.** Welcome → 2nd-visit nudge (70% never return!) → win-back at 21/45 days → birthday → slow-day boost. Owner/SpotOn prove owners won't operate marketing tools. Our v1 spec (3 automated triggers + manual segmented campaigns) is directionally right; add the 2nd-visit nudge and slow-day boost to the roadmap.
5. **Compressed recovery ladder for food**: cart/session abandonment recovery at ~10 min (same meal), then next-mealtime, then weekly win-back. Never 72h discount ladders.
6. **RFM + churn-risk as the brain, framed simply**: "12 regulars at risk this week — send them this?" — one-tap approve on the owner's own WhatsApp. This is the agentic "AI marketing manager" positioning (Owner is building it for the US; nobody has it in India).
7. **One-tap reorder in chat**: "Repeat your last order? [Yes → UPI link]" — GoKwik's Instant Checkout pattern; restaurants are low-SKU/repetitive, so it's stronger here than in D2C.
8. **Zero-friction loyalty**: phone captured at order auto-enrolls; PWA + WhatsApp thread IS the loyalty card. Never require an app install.
9. **Attributed-revenue ledger in the product**: "this platform made you ₹X this month" — every surviving retention product (Reelo, Xeno) leads with ROI attribution; it's the SaaS-churn killer.
10. **Sell the ~31% aggregator math, deliver retention.** The hook that converts owners is Zomato commission savings; the LTV comes from the funnel.
11. **Petpooja integration later, not war**: dine-in bills logged in the POS must eventually flow into our customer graph (Reelo's 35-POS integration net built its distribution). v2 consideration.
12. **City-density playbook**: our city-by-city plan matches how every hyperlocal play compounds (referrals, offer partnerships, word-of-mouth).
13. **Vernacular AI copy** (Hinglish/regional per city) — cheap for us, unshipped by anyone in India SMB.
14. **AI menu Q&A bot on the vendor's WhatsApp** — the 80%-automatable query layer (GoKwik/LimeChat pattern); v1.5+, after the transactional funnel works. (Note: must be business-specific — generic AI chatbots are banned on the platform.)

## 10. White space & positioning

**Verified competitive map**: Petpooja = POS gravity, marketing fragmented/no AI. Reelo = retention, no ordering. DotPe = ordering+POS, shallow automation, chain-skewed ("reputation issues" could NOT be verified — don't repeat that claim). LimeTray = old bundle, chains, no AI. TMBill = only POS with native WhatsApp marketing, budget tier. Xeno/EasyRewardz = enterprise. Thrive = dead. **Nobody bundles QR ordering + auto-enrolled loyalty + default-on AI WhatsApp lifecycle for independent cafes at SMB price. That is the product.**

**Positioning sentence**: "Your cafe's own ordering app + a marketing manager that runs itself on WhatsApp — flat ₹X/month, no commission, your customers stay yours."

**Risks**: Petpooja deepening native CRM (their add-on architecture suggests slowness); Reelo adding ordering ($1M funding constrains them); Meta repricing quarterly (marketing +10% Jan 2026 — design for utility-window economics); frequency cap tightening; a well-funded player (GoKwik/Interakt/Jio) verticalizing into restaurants.

## 11. Changes to feed back into our build docs

1. **WhatsApp architecture (docs/02 §7)**: keep the channel-agnostic NotificationService, but plan the BSP ladder — wholesale pass-through rail (MSG91/Gupshup) for v1 → Meta Tech Provider + Embedded Signup → **per-vendor WABA** as the target architecture (our current single-platform-WABA assumption becomes the stopgap, with migration in mind: store `wabaId`/`phoneNumberId` per vendor from day one, even if all rows point at the platform's).
2. **Funnel additions to product scope (docs/01 §2.6)**: add (a) abandoned-cart/session recovery with the compressed ladder, (b) one-tap reorder journey, (c) 2nd-visit nudge, (d) slow-day boost, (e) attributed-revenue ledger on the vendor dashboard, (f) frequency-cap-aware send logic (handle error 131049 as `skipped_meta_cap` in message_logs — new skipReason enum value).
3. **Utility-window optimization (docs/02 §7, 03 §8)**: track the 24h service-window state per customer thread; prefer sending order-status inside it (₹0); log the window state on each send for cost analytics.
4. **CTWA/QR free-entry**: vendor QR should support wa.me deep links (chat-first entry) in addition to PWA links — opens the 72h free window and captures opt-in in one scan. Worth an experiment flag in v1.
5. **Consent record (docs/03 §1)**: already compliant (status/updatedAt/source) — add the consent wording version for DPDP evidence.
6. **In-chat payments**: Razorpay/PayU WhatsApp payments exist — pairs with the v1.1 Razorpay milestone; design order links to support UPI intent.
7. **Pricing input**: subscription band ₹999–2,999/mo launch (city pilot) with message costs pass-through at Meta rates; the "basic/growth" plan quotas in docs/03 map to marketing-message quotas.

## 12. Source index (primary)

POS market: entrackr.com (Petpooja Series C) · techjockey.com (Petpooja/PrimePOS/TMBill/GoFrugal/eZee/LimeTray) · techcrunch.com (UrbanPiper) · dineopen.com blogs (pricing/commission/ONDC guides — competitor-authored, cross-check) · restrofi.com · softwaresuggest.com · reelo.io/pricing · mediabrief.com (NRAI IFSR 2024) · grandviewresearch.com. GoKwik: gokwik.co product pages + success stories (True Elements, Fire-Boltt, SAADA, mCaffeine) · inc42.com (Tellephant) · entrackr.com (captable) · apps.shopify.com (KwikCheckout/KwikEngage pricing). WhatsApp: developers.facebook.com/documentation (pricing, messaging limits) · msg91.com · gupshup.io support · twilio.com · aisensy.com/pricing · interakt.shop/pricing · richautomate.in (rate hike, benchmarks, DLT explainer) · chatarmin.com (limits) · 360dialog.com (Tech Provider). Convergence: sacra.com/c/owner · research.contrary.com/company/owner · pos.toasttab.com (Toast IQ) · spoton.com (Marketing Assist) · incentivio.com (churn) · getxeno.com · chechani.substack.com (Thrive post-mortem) · qsrweb.com (2026 AI trends) · restroworks.com/blog (retention stats).
