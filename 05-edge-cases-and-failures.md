# 05 — Edge-Case & Failure Catalog (v1)

Binding failure catalog: the things AI-written code silently gets wrong. Every case must have explicit handling in code AND (where marked 🧪) an automated test. When implementing a module, re-read that module's section first. Format: **Case → Required behavior.** DB terms are Postgres (`docs/03`).

---

## 1. Auth & sessions

1. Expired access JWT → 401 `TOKEN_EXPIRED`; frontends auto-refresh once and retry; refresh also failed → hard logout. 🧪
2. Malformed/absent JWT → 401 `UNAUTHENTICATED`, never a 500. 🧪
3. Refresh-token reuse (rotated token replayed) → revoke the entire family, all devices out. 🧪
4. OTP: wrong code 3× → invalidate OTP. Resend inside 60 s → 429. > 5 OTPs/phone/hour → 429. Codes single-use, hashed in Redis. 🧪
5. OTP phone equals an owner's `users.phone` → still a *customer* identity; separate JWT. The owner-bot thread is keyed on `users.phone` only for inbound WhatsApp, never for OTP login.
6. Owner disabled or vendor suspended while a JWT is valid → `tenantContext` checks `vendors.status` (cached 60 s); writes → 403. 🧪
7. First-timer OTP without consent checkbox → account created, `consent_marketing = false`; utility messages (e-bill, status) still allowed under `consent_utility` recorded at OTP. 🧪
8. Signup email already used → 409 `DUPLICATE`; demo/manual door for an existing email → attach to existing vendor, never create a second.

---

## 2. Sessions & rounds (Postgres transaction; `docs/03` §2b.2)

1. **Two guests at the same table submit rounds concurrently** → both transactions `SELECT … FOR UPDATE` the session row; both run guarded stock decrements; both insert `orders`; `running_total` is `UPDATE … SET running_total = running_total + $`. Zero lost rounds. 🧪
2. **Round submitted while cashier is settling** → settle transaction locks the session row and flips `status='settled'`; the round transaction sees it after the lock → 409 `SESSION_CLOSED`. 🧪
3. **Double-tap on submit** → same `Idempotency-Key` → `UNIQUE (outlet_id, idempotency_key)` violation (SQLSTATE 23505) → return the existing order with 200 + `Idempotency-Replayed: true`. 🧪
4. **Stock race on limited items** → `UPDATE menu_items SET available_count = available_count - $qty WHERE id = $1 AND (availability <> 'limited' OR available_count >= $qty)`; `rowCount = 0` → 409 `OUT_OF_STOCK` naming the item. NEVER select-then-update. 🧪
5. **Price changed while items were in the cart** → the same UPDATE also matches `price_version = $cartVersion`; mismatch → 409 `PRICE_CHANGED` with fresh prices; client shows old vs new and re-confirms. 🧪
6. **Outlet closed / busy after menu load** → placement returns 423 `VENDOR_CLOSED`; open/busy computed server-side at placement. Busy mode adds `busy_extra_eta_min` to the ETA shown. 🧪
7. **Rapid call-waiter / request-bill taps** → max 1 pending `service_calls` row per session per 60 s; later taps return 200 with the existing pending row. 🧪
8. **Second QR scan on an occupied table** → same session returned (one active session per table, partial unique index). A different phone joining adds a `dining_session_customers` row with badge new/repeat. 🧪
9. **Table reassignment** → `table_moves` row + `dining_sessions.table_id` update inside one transaction; rounds and totals untouched. Target table occupied → 409 `TABLE_OCCUPIED`. 🧪
10. **Regenerated QR token scanned** → 410 GONE with "ask staff for the new QR". 🧪
11. **Practice table** → session/orders/bills carry `is_practice = true`; no WhatsApp, no events consumed by marketing, excluded from day-close and reports. 🧪
12. **Pay-only outlet** (qrMode `pay_only`) → customer scan opens the bill view of the staff-entered session; no cart. If no session exists on that table → "nothing to pay yet, ask staff".

---

## 3. Order desk & kitchen

1. **Two staff accept the same order** → guarded `UPDATE … WHERE id = $1 AND status = 'placed'`; loser gets `rowCount = 0` → 409 `CONFLICT_STATE`; UI refreshes silently. 🧪
2. **Auto-accept** → at placement, if `outlets.auto_accept_enabled` and any active rule matches (repeat customer / grand_total < value / customer's order_count ≥ n) → the same transaction sets `accepted_at`, `accept_mode='auto'`, `accept_rule`, writes event `auto_accept_fired`. Never for practice orders, never when busy mode is on. 🧪
3. **Customer cancellation request vs staff accept** → request only sets `cancel_requested_at`; the order keeps its state. Owner decides; approve → guarded cancel + stock restore; deny → notify customer. Request after `preparing` → 409 `CANCEL_NOT_ALLOWED`. 🧪
4. **Staff cancels after accepting** → allowed from `accepted`/`preparing` with reason; stock restored in the same transaction; if a pay-now payment was captured → a refund is *required* before the bill can settle (bill shows "refund pending"). 🧪
5. **Order stuck in `placed`** → auto-expiry job cancels after `placed_auto_expire_min` (default 15) with `cancelled_by='system'`, restores stock, notifies the customer. Uses the partial index `orders_stuck_ix`. 🧪
6. **Slow order** → job flags orders past `slow_order_alert_min` since acceptance once (`slow_alerted_at`), emits `slow:order`. 🧪
7. **Dine-in tab left open overnight** → 05:00 IST job lists sessions active > 12 h for owner audit; nothing auto-settles. Day-close records them in `open_sessions_carried`.
8. **Takeaway token** → `ops.next_seq('token:{outlet}:{date}')`; resets daily; "collected" is a timestamp on the session.

---

## 4. Billing, payments, GST, day-close

1. **Concurrent settlements → sequential invoice numbers** → `ops.next_seq('invoice:{outlet}:{fy}')` is one atomic `INSERT … ON CONFLICT … RETURNING`; `UNIQUE (outlet_id, financial_year, invoice_seq)` is the backstop. Never `count()+1`. 🧪
2. **FY rollover (1 April)** → `app.fy_label()` computes the key; sequence restarts at 1 automatically. 🧪
3. **No GSTIN at first settle** → 422 `GST_REQUIRED`; the wizard blocks "go live" checklist item until entered. 🧪
4. **Rounding** → CGST/SGST at 2.5 % each on taxable total; grand total rounded to the rupee; `round_off` signed paise recorded. Tip is voluntary only; no service charge field exists. 🧪
5. **Coupon abuse** → `UPDATE coupons SET used_count = used_count + 1 WHERE id = $1 AND (total_uses IS NULL OR used_count < total_uses)`; `rowCount = 0` → 422 `INVALID_COUPON`. Per-customer limit = `COUNT(*) FROM coupon_redemptions WHERE coupon_id, customer_id` inside the same transaction. Coupon applied after any payment → 422. 🧪
6. **Bill paid in two modes** (₹500 UPI + ₹340 cash) → two `payments` rows; `bills.paid_amount` incremented by each; status `partially_paid` until `paid_amount >= grand_total`. Day-close sums **payments**, never bills. 🧪
7. **Gateway webhook arrives twice / out of order** → `webhook_dedupe (source, event_id)` → second is 202 no-op. `captured` after `failed` for the same gateway order → captured wins (idempotent by `gateway_payment_id`). 🧪
8. **Customer pays but the webhook is late** → payment stays `pending`; the customer page polls `GET /payments/{id}`; a reconcile job queries the gateway after 3 min. Staff may not settle while a gateway payment is `pending`. 🧪
9. **Refund** → dialog preselects the mode the payment came in (gateway → `back_to_source`, cash/UPI/card at counter → `cash`); `adjust_replace` always offered; `outlets.refund_default` overrides the preselection if set. Row = `payments(kind='refund', of_payment_id, refund_mode, reason, refund_by)`; back-to-source calls the gateway adapter and stays `pending` until its webhook; cash refund is `captured` immediately with `recorded_by`; adjust-replace links `replacement_order_id`. Refund > original payment → 422. 🧪
10. **Pay whole table vs own items** → `scope='own_items'` sums lines where `order_lines.customer_id = me`; a line with no customer belongs to "whole table". Two diners paying "own items" concurrently → row lock on the bill. 🧪
11. **Day-close race** → `UNIQUE (outlet_id, local_date)` → second click 409 `DAY_ALREADY_CLOSED`. Negative cash variance stored signed and flagged. 🧪
12. **Void after payment** → not allowed; refund first, then void. Void writes `audit_logs`. 🧪
13. **Complimentary line** → `comp_reason` mandatory (CHECK on `bill_discounts`); line keeps its price, discount row carries the amount so the discount-rate report is honest. 🧪

---

## 5. Customers, consent, imports

1. **Inbound STOP / UNSUBSCRIBE / बंद** → before anything else: `customers.stop_at`, `consent_marketing=false`, `customer_consent_events` row, `conversations.opted_out`; every queued marketing message to that number skipped with `opted_out`. 🧪
2. **Imported list** → rows land as `customer_profiles.lifecycle='imported'`, `consent_marketing=false`; the only message allowed is the opt-in template. Marketing to an imported number without a recorded opt-in → hard block in the worker (`no_consent`) + alert. 🧪
3. **Same phone imported by two vendors** → one `customers` row, two `customer_profiles`; consent recorded per capture event with `vendor_id`; STOP is global. 🧪
4. **Customer erasure request** → job per `docs/03` §4.7; vendor stats remain, PII gone; `customer_consent_events` kept as evidence. 🧪
5. **Blocked customer** → can still browse; placement returns 423 `VENDOR_CLOSED` variant `BLOCKED` (message: "please order at the counter"); no marketing. 🧪
6. **Birthday without year, Feb 29** → send on Feb 28 in non-leap years.
7. **Segment recompute** → nightly `UPDATE customer_profiles SET segment = …` from rules (`at_risk_days`, `loyal_visits`); `segment_rule_version` stamped so a rule change is auditable.

---

## 6. WhatsApp pipeline & Meta

1. **Meta 5xx/429** → BullMQ backoff (5 attempts, jitter) → `failed_retryable` → DLQ; never blocks HTTP. 🧪
2. **Error 131049 (per-user marketing cap)** → `status='skipped'`, `skip_reason='meta_cap_131049'`; campaign counts reflect it; retry the customer next campaign, not now. 🧪
3. **Weekly per-customer cap (2)** → checked in the worker at send time from `customer_profiles.marketing_this_week`; skip `weekly_cap`. 🧪
4. **Cron re-run / worker crash** → `message_dedupe` PK on `{customer}:{purpose}:{date}` → duplicate insert fails → no second birthday message. 🧪
5. **24 h window** → inbound message sets `conversations.window_expires_at = now()+24h`; utility sends inside the window are marked `in_window=true, billable=false`; outside → template category pricing recorded. 🧪
6. **Vendor revokes WABA permissions (v1.5)** → 401/403 from Cloud API → `whatsapp_channels.status='suspended'` + dashboard banner; sends skipped `vendor_suspended`.
7. **Template paused/rejected by Meta** → `message_templates.meta_status`; campaigns using it cannot be scheduled (422) and running ones pause.
8. **Holdout** → arm assigned deterministically by `hash(holdout_seed || profile_id) % 100 < holdout_percent`; control rows get no message but a `campaign_recipients` row and `holdout_assigned` event; the attribution job treats both arms identically. 🧪
9. **Attribution window** → a visit within `attribution_window_days` after send creates `revenue_attributions`; one bill can be attributed to at most one campaign (`UNIQUE (bill_id, campaign_id)` + first-touch rule). 🧪
10. **Dead-hour filler cap** → `dead_cap` recipients max, chosen by `visit_hours[hour]` desc; a customer messaged in the last 7 days is excluded. 🧪

---

## 7. Owner bot & tools

1. **Bot asked to change something in v1** → tool `side_effect <> 'read'` → `tool_executions.status='denied'`, reply "coming in the next version"; never silently ignored. 🧪
2. **Bot question about another outlet / vendor** → `ToolExecutionContext.tenant` is bound from the session; the tool runs under RLS; no data, plain "I only see Brewhouse Jaipur". 🧪
3. **LLM proposes tool args with a foreign `outletId`** → schema validation + tenant check reject; logged as `guard` step. 🧪
4. **Reply to a briefing older than 7 days** → new agent session; briefing context reloaded from `briefings.snapshot`.
5. **Cost guard** → `agents.max_cost_paise_per_day` per vendor; over → polite refusal, `outcome='refused'`.
6. **Confirmation (v1.5)** → `awaiting_confirmation` expires in 10 min; a late "YES" → "that request expired, ask again".

---

## 8. Menu import (AI extraction)

1. **Blurry photo** → low `confidence` rows first in the grid; rows < 0.5 require explicit acceptance before apply. 🧪
2. **Price formats "150/-", "Rs 150.00"** → normalised to integer paise; unparseable → row `price NULL`, apply blocked until fixed. 🧪
3. **Re-photograph diff** → rows matched to existing items by normalised name; `action` = update/create/delete; owner applies as one transaction; every price change writes `menu_item_revisions`. 🧪
4. **Duplicate category names** → merged into the existing category.

---

## 9. Multi-tenancy (leaks are P0)

1. **Vendor A token + vendor B resource id** → RLS returns no row → 404 `NOT_FOUND` (never 403). 🧪
2. **Socket rooms** → join only after JWT/session-token ownership check. 🧪
3. **Customer data** → vendors only ever see `customer_profiles`; phone masked (`98•••••210`) in list responses; full phone only on the profile detail of an identified customer. 🧪
4. **Worker code path** → `asWorker()` outside `jobs/` is an ESLint error; every worker query names `vendor_id` explicitly. 🧪

---

## 10. Money, time & text

1. **Integer paise everywhere** (`app.paise`); any float in a money path is review-blocking. 🧪
2. **UTC storage, IST display**; `local_date`/`day_part` come from the DB trigger — app code never computes them. 🧪
3. **Phone normalisation** → E.164 (`+91…`) at every intake; the DB domain rejects anything else.

---

## 11. Infrastructure & fallbacks

1. **Redis outage** → ordering and billing continue (they don't need Redis); WhatsApp sends queue in `messages(status queued)` and drain when Redis returns; OTP requests return 503. 🧪
2. **Postgres pooler in transaction mode** → no `SET` outside `SET LOCAL`, no named prepared statements, no `LISTEN`. A violation shows up as cross-request tenant bleed — the leak matrix catches it. 🧪
3. **Partition missing (clock skew / job failure)** → DEFAULT partition catches the row; alert on non-empty DEFAULT; `ops.ensure_month_partitions` on next cron.
4. **Customer offline at placement** → PWA keeps the cart in localStorage; retry with the same idempotency key never double-orders. 🧪
5. **Gateway down** → `POST /payments` returns 503 with "pay at counter"; staff can still settle by cash/UPI.
