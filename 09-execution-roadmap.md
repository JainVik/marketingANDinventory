# 09 — Execution Roadmap: Day One → Production → Users

The end-to-end path for building this startup, written for our actual situation: 3 people, different cities, AI writing most of the code, first city pilot. The reason teams "disappear in between" is never lack of skill — it's missing **gates** (no definition of done), **horizontal building** (all backend, then all frontend, then integration hell), and **no external deadline** (nobody waiting for a demo). This roadmap fixes all three: every phase has an exit gate, everything is built in vertical slices, and a weekly demo ritual creates the deadline.

Timeboxes assume part-time/evening work by 3 people with AI doing the heavy coding. Full-time compresses ~2×.

---

## The map

```
Phase 0  Decide & commit           week 0–1     gate: signed one-pager + not-list
Phase 1  Validate with real cafes  week 1–3     gate: 5 pilot commitments
Phase 2  Design on paper           week 2–4     gate: docs pack + clickable wireframes
Phase 3  Foundations               week 4–6     gate: walking skeleton live on staging
Phase 4  Vertical slices           week 6–14    gate: per-slice DoD, weekly demo
Phase 5  Hardening                 week 14–16   gate: release checklist green
Phase 6  Pilot                     week 16–20   gate: 5 cafes live, metrics real
Phase 7  Operate & iterate         ongoing      gate: v1.1 decided by data, not vibes
```

Phases 1 and 2 overlap deliberately. Nothing else does.

---

## Phase 0 — Decide & commit (week 0–1)

The cheapest place to fix a mistake is before anything exists.

1. **One-pager**: problem, one-sentence positioning ("Your cafe's own ordering app + a marketing manager that runs itself on WhatsApp — flat ₹X/month, no commission"), target customer (independent cafes/QSRs in ONE named city), success metric for the pilot (e.g., "10 cafes live, 30% of their walk-in customers captured, 15% repeat-order lift in 8 weeks").
2. **The not-list** — write down what v1 will NOT do (we did: no payments, no delivery, no bookings, no multi-outlet, no native apps). This list is the #1 defense against disappearing: every "wouldn't it be cool if" gets checked against it.
3. **Team contract**: who owns what (e.g., A = product + vendor side + AI-code review, B = customer PWA + design, C = backend/infra + WhatsApp pipeline), decision rule when you disagree (product owner decides after hearing both, 24h max), equity/vesting conversation NOW while everyone is friends, weekly cadence (one fixed 1-hour call: demo → decide → assign).
4. **Tooling**: one GitHub org + monorepo, one project board (GitHub Projects — keep it where the code is), one WhatsApp/Discord group, one shared drive for docs. Nothing else.
5. **Name check**: domain + trademark search for the working name; don't bikeshed longer than a day — pick and move.

**Gate:** all three founders have signed off on the one-pager and the not-list (literally: names at the bottom). If you can't agree here, you've saved yourselves a year.

## Phase 1 — Validate with real cafes (week 1–3, overlaps Phase 2)

You are one WhatsApp message away from your customer. Use that before writing code.

1. **Talk to 12–15 cafe owners** in the launch city. Not a survey — sit at the counter at 4pm. Ask: How do you know if a customer ever comes back? What do you pay Zomato monthly? Have you ever messaged customers about an offer? What POS/billing do you use, what does it cost, what do you hate about it? Would you pay ₹1,500/mo for X? Watch faces on the price question.
2. **Pre-sell the pilot**: "We're launching with 10 cafes in [city] — free for 3 months, then ₹X/mo, cancel anytime, we set up your whole menu for you." Get a yes in writing (WhatsApp text counts).
3. **Test the QR behavior**: put a dummy QR on 2 friendly cafes' tables linking to a one-page menu (even a Google Doc). Watch how many people scan. This one hack de-risks the entire product thesis for ₹0.
4. **Kill criteria** (write them down): if fewer than 5 of 15 owners commit, or nobody blinks at losing customer data, stop and re-scope before building.

**Gate:** 5 written pilot commitments + interview notes summarized into the PRD (what changed?).

## Phase 2 — Design the system on paper (week 2–4)

This is where we already are. The docs pack (01–08) IS this phase for the backend. Complete it with:

1. **UX flows before screens**: draw the 6 critical flows as boxes-and-arrows — scan→order (customer), OTP login, order board (vendor), menu setup (vendor), campaign send (vendor), onboarding (admin). Every screen in the flow gets one line: what the user sees, what they tap, what can go wrong.
2. **Low-fi wireframes** of the ~15 core screens (Figma free tier, or even paper photographed). Do NOT design pixel-perfect UI now; do decide information hierarchy. Show 2 cafe owners the vendor screens — they will instantly tell you what's confusing.
3. **Brand minimum**: name, logo, 2 colors, 1 font. One day, not one month.
4. **Walking-skeleton definition**: agree exactly what "one request end to end" means (Phase 3 gate) so foundations have a finish line.
5. **Freeze**: docs 01–06 get a `v1.0` git tag. From now on scope changes are pull requests to the docs, discussed at the weekly call — not silent additions.

**Gate:** docs tagged, wireframes clickable-ish, two cafe owners have seen the vendor screens.

## Phase 3 — Foundations (week 4–6)

The unglamorous fortnight that decides whether week 12 is fun or hell. Build NO features yet.

1. **Scaffold the monorepo** exactly per docs/02 §2 (workspaces, apps/api, 3 frontends, packages/shared).
2. **CI from the first commit**: lint + typecheck + test on every PR; auto-deploy `main` to staging. If deploy isn't automated in week 4 it never gets automated.
3. **Environments**: local docker-compose (postgres 16 + pgvector, redis), staging VPS + a second Supabase project (or branch), prod untouched until Phase 5.
4. **The skeleton**: env config (zod-validated), error envelope + central handler, request logging with requestId, auth (OTP + JWT + refresh rotation), tenantContext middleware, one protected route, Socket.IO handshake, one BullMQ job that logs, seed script with 2 fake cafes.
5. **AI-coding discipline starts here**: CLAUDE.md at repo root; every session scoped to one module; every AI PR reviewed by a human founder (the one who did NOT prompt it); tests required in the same PR; conventional commits.
6. **Walking skeleton demo**: on staging, a seeded customer logs in via OTP, hits a protected endpoint, sees seeded menu data in a bare-bones PWA page, and an event shows up in the vendor dashboard skeleton via socket. Ugly is fine. **End to end is the point.**

**Gate:** the walking skeleton runs on staging, CI is green, all three founders have run it locally.

## Phase 4 — Vertical slices (week 6–14)

The core rule: **never build a layer, always build a slice.** A slice = schema + API + UI + tests + deployed to staging + demoed. "All models first, then all APIs, then frontend" is how teams disappear — integration debt piles up invisibly and morale dies with no demos. Slice order (each 1–2 weeks, matches README build order):

| # | Slice | Demo at the weekly call |
|---|---|---|
| S1 | Vendor profile + catalog CRUD + stock states | Owner builds a real cafe's menu in 15 min |
| S2 | Public storefront + menu + cart (PWA) | Scan QR on a phone → browse → cart |
| S3 | **Order placement + order board** (transactions, idempotency, sockets) — the hard one, budget 2 weeks | Live order placed on one phone appears on the vendor screen in <2s, stock decrements, double-tap creates one order |
| S4 | Order lifecycle + customer status screen + auto-expiry | Full placed→completed run; cancel race demo |
| S5 | Reviews (verified) + rating display | Review only possible after completed order |
| S6 | Customer profiles + segments + vendor customer list | "This cafe has 12 repeat customers" from real seed orders |
| S7 | WhatsApp pipeline (queue, worker, webhook, opt-in/out, message log) on the wholesale BSP | Real order-status message arrives on a real phone |
| S8 | Automated triggers (birthday, win-back, post-first-order) + quotas + dedupe | Cron fires on staging, dedupe proven by running it twice |
| S9 | Manual campaigns + templates + counts | Vendor sends a segmented promo; honest skipped counts |
| S10 | Admin panel + onboarding flow + subscription states | Onboard a brand-new cafe end to end in 30 min |

Working rules for this phase:

- **Weekly demo is sacred.** Whatever exists gets demoed on staging every week, even broken. The demo is the deadline; the deadline is the anti-disappearance device.
- **Definition of done per slice**: docs/05 edge cases for that module handled (🧪 tests green), tenant-leak test added, deployed, demoed. Not "code written."
- **One slice in flight per person, max.** Parallelize across slices only when they don't share files (e.g., S5 + S6).
- **Change log discipline**: mid-build ideas go to a `later.md` file, not into the sprint. Review it monthly; most entries die of embarrassment.
- **AI workflow per slice**: (1) prompt Claude Code with the doc references (per README), (2) review the diff like a hostile senior engineer, (3) run the edge-case tests, (4) the non-prompting founder approves the PR. AI writes fast; unreviewed AI code is where startups quietly rot.

**Gate:** all 10 slices demoed; a stranger can order coffee from a seeded cafe on staging with nobody helping them.

## Phase 5 — Hardening (week 14–16)

1. **Release checklist** = docs/06 §10 gates: tenant-leak matrix green, `npm audit` clean, all 🧪 tests green, boot-fails-on-bad-config proven, load smoke (50 concurrent orders on `limited` stock → zero oversells/dupes).
2. **UAT with 2 friendly cafes on staging**: real menu, real staff phone, fake customers (you three + friends). Watch them use the order board during a fake rush. Every confusion = a UI bug, log it.
3. **Bug bash**: one evening, all three founders + friends try to break it (double-taps, airplane mode mid-order, absurd inputs, two tabs, back button).
4. **Ops readiness**: Sentry wired and alerting to your group chat, /healthz monitored (UptimeRobot free), Supabase PITR verified by actually restoring once to staging, runbook.md (deploy, rollback, "site is down" steps, WhatsApp quality-rating drop response), rate limits verified.
5. **Legal/compliance minimum**: ToS + privacy policy (DPDP-aware, lawyer-reviewed if affordable), WhatsApp templates approved in Meta, business entity + GST registration for billing, vendor agreement template (the pilot terms).

**Gate:** checklist green, UAT cafes said "I'd use this," runbook exists, rollback rehearsed once.

## Phase 6 — Pilot launch (week 16–20)

1. **Onboard the 5 committed cafes, one at a time, in person.** Done-for-you: you photograph the menu, you build the catalog, you print and laminate the QRs, you train the staff for 20 minutes at 4pm. First real order happens while you're standing there.
2. **War-room week 1**: watch every order live; call the owner every evening ("anything weird?"). Fix small annoyances within 24h — pilot-stage responsiveness is marketing.
3. **Measure from day one** (the numbers that decide v1.1): scans→orders conversion, orders/cafe/day, % customers opted into WhatsApp, message delivery/read rates, repeat-order rate at 2/4/8 weeks, review submission rate, vendor dashboard daily-active (do owners actually open it?).
4. **The attributed-revenue ledger must work now** — "this platform made you ₹X" is what converts the free pilot into paid.
5. **Weekly iteration cadence continues**: demo call becomes metrics call (15 min metrics → decide → assign).
6. **Start the paid conversation at week 6 of the pilot**, before the free period ends: "It's ₹X/mo from next month — here's your ledger." Target: ≥60% convert. Below 40% = pricing or value problem; diagnose before scaling.

**Gate:** 5+ cafes live, ≥1 has organically told another cafe about it (the only growth signal that matters at this stage), conversion conversation started.

## Phase 7 — Operate & iterate (ongoing)

1. **Support becomes a system**: a dedicated WhatsApp number for vendors, response SLA you actually keep (Owner.com's #1 complaint is the post-launch support cliff — do not repeat it), FAQ doc that grows from every ticket.
2. **Incident discipline**: severity levels, who's on call (rotate weekly among three), post-mortems without blame in `incidents/`.
3. **Roadmap by data**: v1.1 candidates (Razorpay/UPI + in-chat payments, abandoned-cart recovery ladder, 2nd-visit nudge, slow-day boost, Tech Provider migration + per-vendor WABA, points loyalty) get ranked by pilot metrics, not by what's fun to build. One release train per 2–4 weeks, always through the same CI gates.
4. **Scale trigger**: second cohort (10–15 cafes, same city) only after ≥60% paid conversion and support load per cafe is known. Second CITY only after the sales+onboarding playbook is written down well enough that a hired person (not a founder) can run it.
5. **Keep the docs honest**: every shipped change updates docs 01–06 in the same PR. The docs pack remains the contract — for the AI and for you.

---

## The anti-disappearance rules (the whole doc in 8 lines)

1. Every phase has a written gate; you don't move without it — and you don't stall past a timebox without a decision either.
2. Build vertical slices, never horizontal layers. Demo weekly on staging, no exceptions, ugly allowed.
3. The not-list is law; new ideas go to `later.md`, not into the sprint.
4. AI writes the code; a human who didn't prompt it reviews every PR; tests ship in the same PR.
5. Talk to cafe owners before, during, and after — 15 interviews before code, 2 at wireframes, 2 at UAT, all 5 weekly in pilot.
6. CI/CD and staging exist before the first feature. Deploy is boring or it doesn't happen.
7. Measure from the first pilot order; let the ledger and the metrics pick v1.1.
8. One fixed weekly call: demo → decide → assign. If the call dies, the startup is dying — treat a skipped call as an incident.
