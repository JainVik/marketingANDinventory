# 05 — Edge-Case & Failure Catalog

This is the document the user asked for by name: the things AI-written code silently gets wrong. Every case below is **binding**: each one must have explicit handling in code AND (where marked 🧪) an automated test. When implementing a module, re-read that module's section here first.

Format: **Case → Required behavior.**

---

## 1. Auth & sessions

1. Expired access JWT → 401 `TOKEN_EXPIRED`; frontends auto-refresh once and retry the request transparently; refresh also failed → hard logout to login screen. 🧪
2. Malformed/absent/garbage JWT (`null`, `undefined`, truncated) → 401 `UNAUTHENTICATED`, never a 500 stack trace. 🧪
3. Refresh-token reuse (old rotated token replayed — possible theft) → revoke entire token family, all devices logged out. 🧪
4. OTP: wrong code 3× → invalidate that OTP, require new request. Resend inside 60s cooldown → 429. >5 OTP requests/phone/hour → 429. Codes are single-use and hashed in Redis. 🧪
5. OTP requested for a phone that is a vendor-staff/admin identity → still fine (separate identity), creates/logs into a *customer* account.
6. Vendor user disabled (`status: 'disabled'`) or vendor suspended while their JWT is still valid → auth middleware re-checks user status? No — too costly per request. Instead: access tokens are 15 min; suspension takes effect within that window, AND all *write* vendor routes check `subscription.status` via tenantContext (cached 60s). Suspended vendor write → 403. 🧪
7. Same user logged in on two devices — allowed; refresh families are per-device; logout only kills that device's family.
8. Deleted/archived vendor's staff tries to log in → 403 with clear message, not 500 on a null vendor.

## 2. Order placement

1. **Double-tap / double-submit:** same `Idempotency-Key` arrives twice (or concurrently). First creates; second returns the same order (`200`, replay header). Concurrent replay must not create two orders — unique index `{vendorId, idempotencyKey}` is the last line of defense; catch `E11000` → fetch and return the winner. 🧪
2. **Stock race — the classic:** two customers order the last unit simultaneously. Guarded `$inc` with `$gte: qty` inside the transaction (03 §10.1); loser gets 409 `OUT_OF_STOCK` naming the item; cart UI removes/adjusts the line. NEVER read-check-then-decrement. 🧪
3. Item deactivated / deleted / out-of-stock between menu load and order submit → per-line validation at placement → 409 with per-item detail; client shows which lines died. 🧪
4. Price or variant changed after menu load (`snapshotVersion` mismatch) → 409 `PRICE_CHANGED` returning fresh prices; client re-renders cart with old→new diff and asks the customer to confirm. Never silently charge the new price. 🧪
5. Vendor closes store (or hours end, or gets suspended) after menu load → placement returns 423 `VENDOR_CLOSED`. Open/closed is computed server-side at placement time only — client clock is untrusted. 🧪
6. Cart totals: client-sent prices are display-only; server recomputes everything from DB. Any mismatch is not an error — server result wins silently (client already confirmed via PRICE_CHANGED flow if relevant). 🧪
7. Addon rules violated (minSelect/maxSelect, addon from another item's group, unknown addonId) → 422 with paths. 🧪
8. qty limits: qty ≥ 1, ≤ 20 per line, ≤ 50 items per order, ≤ 200-char note — zod-enforced.
9. Order placed at 23:59:50, vendor closes 00:00 → whatever passes the placement-time check stands; vendor can still reject.
10. Transaction commits but response is lost (network drop) → client retries with same idempotency key → gets the existing order. This is WHY the key is required. 🧪
11. Mongo transaction aborts on transient error (`TransientTransactionError`) → retry the whole transaction up to 3× with jitter; still failing → 500 + alert. Use the driver's `withTransaction` which does this — do not hand-roll.

## 3. Order lifecycle & the live board

1. Two staff accept the same order simultaneously → guarded transition (03 §10.2); loser gets 409 `CONFLICT_STATE` + current state; UI refreshes the card. 🧪
2. Customer cancels at the same moment vendor accepts → same mechanism; exactly one wins; the loser's UI shows the outcome. 🧪
3. Vendor cancels after accepting (ran out of stock) → allowed from `accepted`/`preparing` with mandatory reason; restores `limited` stock (03 §10.4); customer notified via socket + WhatsApp `order_status` template. 🧪
4. Illegal jumps (`placed → ready`, `completed → anything`, `rejected → accepted`) → 409. The transition map lives in `packages/shared` as THE single source; both frontends derive buttons from it. 🧪
5. Order stuck in `placed` (vendor asleep): auto-expiry job cancels `placed` orders older than X min (config, default 15) with reason `vendor_no_response`, notifies customer, restores stock. 🧪
6. Order stuck in `ready` forever → daily job auto-completes `ready` orders older than 12h (`by: 'system'`) so review eligibility and stats aren't blocked; flagged in vendor stats.
7. Vendor dashboard offline (socket dead, laptop closed) → new-order alerting still works because the board re-polls every 30s on focus; socket is not the source of truth. Missed-order risk is mitigated by 5's auto-expiry.
8. Clock skew: never compare client timestamps to server ones; all state timing is server-side.

## 4. Inventory & catalog

1. `limited` stock hits 0 via orders → flip to `out_of_stock` (separate guarded update); storefront shows it greyed, not hidden. Restock → vendor sets state back explicitly.
2. Vendor sets stock while orders are in flight → decrements and manual sets are both atomic ops on the same fields; last write wins is acceptable ONLY for manual sets; decrements must never be lost — hence `$inc`, never `set(count-qty)` from a read. 🧪
3. Deleting a category containing items → forbidden (409) until items are moved/deleted — no cascade deletes.
4. Deleting an item referenced by past orders → soft problem: orders hold snapshots, so hard delete is safe for orders, but keep `isActive:false` soft-delete as default; hard delete only if never ordered.
5. Image upload succeeded but item save failed → orphaned file; weekly job garbage-collects storage objects not referenced by any document. Never block the user on this.
6. Two admins editing the same item → last-write-wins BUT `snapshotVersion` still increments per write, so carts detect it; no optimistic-lock UI in v1.

## 5. WhatsApp pipeline

1. BSP/Meta API down or 5xx/429 → retry with exponential backoff (5 attempts, jitter), then `failed_retryable` → dead-letter queue; DLQ inspected via admin log screen; NEVER lose the message silently and NEVER block the HTTP request that caused it. 🧪
2. Permanent failures (invalid number, unregistered WhatsApp, template rejected/paused by Meta) → `failed_permanent`, no retry; campaign counts reflect it. 3+ permanent failures for one phone → mark profile `unreachable`, skip in future sends.
3. **Consent:** checked in the worker at send time; opted-out or never-opted-in → `skipped` with reason (campaign shows honest counts). Opt-out mid-campaign is honored for messages not yet sent. 🧪
4. Opt-out (`STOP` reply or profile toggle) is global per customer: applies across ALL vendors and platform triggers; only transactional order-status + OTP remain (with a config to disable even those on request). 🧪
5. Weekly per-customer cap and vendor daily quota exceeded → `skipped` with reasons; campaign UI shows "targeted 400, sent 320, skipped-quota 80" — vendors must see honest numbers. 🧪
6. Birthday cron runs, then re-runs after a crash → `dedupeKey` unique index makes second insert fail → skip; NO double birthday message. All cron sends are dedupe-keyed. 🧪
7. Feb-29 birthdays → send on Feb-28 in non-leap years (explicit rule; don't let a date library decide silently).
8. Customer changed phone number → profile phone is denormalized at last order; sends use profile phone; a bounced permanent failure + newer user phone triggers profile refresh on next order.
9. Webhook: bad/absent signature → 401, log, drop. Duplicate delivery callbacks (Meta retries) → status updates are idempotent (only move forward: sent→delivered→read). Unknown `providerMessageId` → log, 200, drop. 🧪
10. Webhook endpoint must respond < 5s: verify → enqueue → 200. Processing is async.
11. Template variable mismatch (missing variable, too-long value) → validation at campaign creation, not at send time. Send-time render failure anyway → `failed_permanent` + alert (means template drift between DB and Meta).
12. 24-hour session rule: we only send pre-approved template messages, so the 24h free-form window is irrelevant in v1 — code must not assume free-form messaging works. Inbound customer replies (other than STOP) get one auto-reply template pointing at the vendor's phone, max once per 24h per customer.

## 6. Reviews & ratings

1. Review for someone else's order / non-completed order → 403/409; eligibility = order.customerId === me AND status completed. 🧪
2. Double review (double-tap or two tabs) → unique `{orderId}` index; catch E11000 → 409 `ALREADY_REVIEWED`. 🧪
3. Edit after 24h window → 403; window enforced server-side from `editableUntil`, not client clock.
4. Review hidden by admin → excluded from rating recompute; recompute runs on hide/unhide too. 🧪
5. Rating average with 0 reviews → display "New", never NaN/0★. 🧪
6. Order auto-completed by system (§3.6) still grants review eligibility — deliberate.
7. Review text: strip/escape HTML on render (frontends never `dangerouslySetInnerHTML`); emoji fine; length capped 1000.

## 7. Multi-tenancy (leaks are P0)

1. Vendor A token + vendor B resource ID → **404** (not 403 — don't confirm existence). Enforced by `vendorId` in every query filter, from JWT only. 🧪 (systematically, for every vendor resource — see 06 §4)
2. Vendor staff of A added by email that already exists as admin of B → reject (one vendor identity per email in v1).
3. Campaign/customer-profile queries missing `vendorId` filter → impossible by construction (`scopedModel`), and the custom ESLint rule + code review catch violations.
4. Socket rooms: joining `vendor:{otherId}` or `order:{notMine}` → join handler validates against JWT; silent refuse + log. 🧪
5. Discovery endpoints (public, cross-tenant by nature) expose ONLY whitelisted public fields (projection), never quotas/subscription/settings/phone lists. 🧪

## 8. Money, time, text

1. All money integer paise; any float in a money path is a review-blocking defect. Display formatting (₹, lakh separators) only at the UI edge via one shared `formatINR()`. 🧪
2. All comparisons/storage UTC; IST conversion only at display and at cron *scheduling* boundaries (birthday cron runs 08:00 IST regardless of server TZ; container TZ is pinned UTC). 🧪
3. `openingHours` crossing midnight (18:00–01:00) → supported by allowing close < open, meaning "next day"; open/closed computation handles it. 🧪
4. Names/notes in Hindi/emoji → full Unicode; DB and JSON handle it; length limits count code points, not bytes.
5. Phone normalization: everything to E.164 (+91 default country code) via one shared util at every intake point (OTP, staff, profile). `9876543210`, `09876...`, `+91 98765...` are the same number. 🧪

## 9. Infrastructure failures

1. Mongo primary election / connection drop mid-request → driver retries reads/writes (retryable writes on); transactions retried per §2.11; still failing → 500 envelope, `/readyz` flips, orchestrator alerts. Requests never hang past a 10s server-side timeout.
2. Redis down → degrade deliberately: OTP issuance fails (503 with honest message), rate limiting **fails closed on auth routes, fails open elsewhere**, queues buffer in memory? NO — BullMQ needs Redis; message-producing actions (campaign send) return 503; order flow (which doesn't need Redis) keeps working. This asymmetry is deliberate: ordering is the business; messaging can wait. 🧪
3. Worker process crash mid-job → BullMQ redelivers (job `attempts` config); all job handlers are idempotent (dedupe keys, guarded updates) so redelivery is safe. 🧪
4. API restart/deploy → graceful shutdown: stop accepting, finish in-flight (≤ 10s), close sockets (clients auto-reconnect), drain worker current job. Socket clients resubscribe rooms on reconnect (client-side handler required).
5. Storage (images) down → menus render with placeholder images (frontends must have `onError` fallback); uploads fail with honest 503.
6. Disk/DB full, Atlas quota hit → writes fail loudly; `/readyz` red; this is an ops alert, not something code should "handle" silently.

## 10. Frontend-specific

1. Stale PWA build after deploy → service-worker update flow: detect new SW, show "Update available" toast, reload on tap. Never let a week-old client talk to a changed API silently — API changes stay backward-compatible within v1, breaking changes bump `/api/v2`.
2. Order status screen left open overnight → socket reconnect + refetch on `visibilitychange`; show data-freshness, never a frozen "preparing" from yesterday (see §3.6 auto-complete).
3. Double-click on "Place order" → button disabled on first click AND idempotency key protects the backend. Both, always. 🧪
4. Back button after placing order → history replace to the order-status page so re-submit is impossible.
5. Cart in localStorage from a previous menu version → zod-parse on load; invalid/mismatched schema → clear cart, toast. Corrupt storage never crashes the app. 🧪
6. Two tabs, same vendor dashboard → both receive socket events; guarded transitions make double actions safe; UI treats 409 `CONFLICT_STATE` as a refresh signal, not an error toast.
7. Offline at "Place order" tap → network error → keep cart intact, show retry; the idempotency key makes retry safe.
8. Geolocation denied → city picker still fully functional; geolocation is a convenience, never a requirement.

## 11. Cron & background jobs

1. Every cron job: dedupe-keyed effects, safe to run twice, safe to skip once (next run self-heals — e.g., segment recompute recomputes everything, not deltas).
2. Jobs are scheduled in one place (`jobs/schedule.ts`); a job that runs > 2× its expected duration logs a warning; overlapping runs of the same job are prevented by a BullMQ job-id lock.
3. Segment recompute at 03:00 IST vs an order completing at 03:00:01 → event-driven update wins later; cron and event both write `segmentComputedAt` and use freshest-wins guarded update.
