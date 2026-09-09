# 06 — Security Checklist

Binding security requirements for v1. Items marked 🧪 must have automated tests; items marked 🚦 are release gates (deploy blocks until green).

## 1. Input handling & injection

- Every endpoint zod-parses body, params, AND query before the controller runs; unknown keys stripped (`.strict()` where shape is closed). 422 on failure. 🧪
- **SQL injection:** every query goes through Drizzle's query builder or the `sql` tagged template (parameterised). String concatenation into `sql.raw()` with request data is a review-blocking defect; `sql.raw` is allowed only for identifiers from an allowlist (sort columns) and is grep-audited in CI. 🧪 (test: `phone = "' OR 1=1 --"` on OTP login → 422, and the query log shows a bound parameter)
- Segment rule DSL (`segments.definition`) compiles to SQL through a whitelist of fields and operators — never by interpolating the JSON. 🧪
- Search fields use `ILIKE` with escaped `%`/`_` or trigram indexes; cap search string length (100). 🧪
- File uploads: presigned URLs constrain content-type and size (≤ 2 MB); on save, server re-validates the stored object's magic bytes and re-serves only from the public bucket domain — user-controlled URLs are never stored (only keys we issued). SVG uploads forbidden (script vector).
- JSON body limit 100 kB; array length caps in zod on every list field (items ≤ 50, addonIds ≤ 20, etc.).

## 2. XSS & frontend

- React's default escaping is the mechanism; `dangerouslySetInnerHTML` is banned (ESLint error) — no exceptions in v1 (no rich text exists).
- All user-generated text (reviews, notes, item names) rendered as text nodes only. 🧪
- CSP on all three frontends: `default-src 'self'`, images from our storage domain, `connect-src` API + socket origin; no inline scripts (Vite builds comply).
- `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin` via helmet.
- Never reflect request data into error messages verbatim beyond zod paths; error `message` strings come from our enum, not from input.

## 3. AuthZ — permission matrix (enforced by `authorize()` + service-level ownership asserts)

| Capability | customer | staff (v1.1) | owner | super_admin |
|---|---|---|---|---|
| Place/cancel own order, review own completed order | ✅ | — | — | — |
| View/act on vendor's orders; stock toggle | — | ✅ | ✅ | — |
| Catalog CRUD, profile, QR, staff mgmt | — | ❌ | ✅ | — |
| Customer list, campaigns, triggers, quota | — | ❌ | ✅ | — |
| Reply to reviews | — | ❌ | ✅ | — |
| Vendor onboarding, subscription status, hide reviews, templates, cities, cross-tenant logs | — | — | — | ✅ |

- Object-level checks are separate from role checks and live in services: "order belongs to my vendor", "review belongs to my order". Role middleware alone is never sufficient. 🧪
- Suspended vendor: all writes 403; storefront delisted; reads allowed. `past_due`: banner only. 🧪

## 4. Tenant isolation (release gate 🚦)

The integration suite contains a **tenant-leak matrix test**: for EVERY vendor-scoped resource type (orders, items, categories, reviews, customers, campaigns, staff, stats, quota, message logs), vendor A's token requests vendor B's resource by real ID → expect 404, and vendor A's list endpoints seeded with B's data → expect zero B rows. This suite is generated from a resource registry so a new module can't ship without joining it. 🧪🚦

- **Row-Level Security is the wall.** Every table with a `vendor_id` column has `FORCE ROW LEVEL SECURITY` and a generated policy keyed on `app.current_vendor_id()`; the API role `regulars_app` cannot bypass it. The leak-matrix test list is generated from `information_schema.columns WHERE column_name = 'vendor_id'` at test time, so a new table joins the gate on creation. 🧪🚦
- `vendorId` always from JWT (tenantContext) into `withTenant()`; never from client input on `/vendor/*`. ESLint rules: no `req.*.vendorId` in vendor modules; no `db.` import in services (they take `tx`); no `asWorker()` outside `jobs/`.
- Pooler discipline: transaction-mode pooling means `SET LOCAL` only — a plain `SET` would leak tenant context to the next request. Lint + a test that runs two tenants over one pooled connection. 🧪
- Global customer identity is not in `app` at all — see §4b. 🧪
- Public projections: storefront endpoints select explicit column lists. A projection test asserts no `gateway_*`, `sub_*`, settings, or customer fields appear in public JSON. 🧪

## 4b. Identity isolation — the PII wall (release gate 🚦)

Tenancy stops vendor A reading vendor B. This stops **the application itself** reading the phone book. It is a Postgres privilege, not a code convention, so no forgotten filter can defeat it.

- `regulars_app` holds **no table grant in schema `pii`**. Gate test: `SELECT * FROM pii.customers` as `regulars_app` must raise `42501`, and `has_table_privilege('regulars_app','pii.customers','SELECT')` must be false for every table in `pii`. Generated from `pg_tables WHERE schemaname='pii'`, so a new identity table joins the gate on creation. 🧪🚦
- The only doors are the view `pii.customer_public` (no phone, no name) and `pii.reveal_identity()`. Both are enumerated in the gate; adding a third door requires editing this list. 🧪
- **Reveal requires a prior order.** `pii.reveal_identity()` raises `42501` unless the calling vendor holds a `customer_profiles` row with `revealed_at IS NOT NULL`. Tested per-vendor: A having revealed customer X must not let B read X. 🧪🚦
- Every reveal writes `audit_logs (action='customer.reveal')`. A vendor bulk-reading identities shows up as a rate spike on one action key — alert at 50 reveals/minute.
- One connection pool, one module: `regulars_identity` may be imported only inside `modules/identity/`. ESLint `no-restricted-imports`, plus a test that greps the build output. 🧪
- **Documented exceptions** (do not "fix" these): `bills.customer_phone`/`customer_name` — a GST invoice must name the buyer, and that customer ordered there by definition; `conversations.phone`, `messages.phone`, `messages.raw` — the sender needs a real number, so `regulars_app` gets a column-level grant that excludes them and the owner UI masks to `98•••••210`. A test asserts those three columns are absent from `regulars_app`'s privileges. 🧪
- The lake never sees `pii`: the logical replication publication covers `app` and `ml` only, and CI fails if a `pii.*` table appears in `pg_publication_tables`. 🧪🚦
- **`06-pii-wall-smoke.sql`** runs all of the above against a freshly migrated database. It is the executable form of this section — run it in CI after migrations, and treat any deviation as a release blocker. 🚦

## 5. Secrets, tokens, crypto

- Passwords: argon2id (memory 64 MB, time 3); no MD5/SHA-anything for passwords. OTPs & refresh tokens stored hashed (SHA-256 is fine here — high entropy inputs).
- JWT RS256; private key only on API hosts via env/secret store; `kid` header for future rotation; access 15 min; no sensitive data in claims beyond `sub/role/vendorId`.
- Refresh cookie: `httpOnly`, `Secure`, `SameSite=Strict`, path-scoped to `/api/v1/auth`.
- Webhooks: verify `X-Hub-Signature-256` (Meta) / provider signatures (Razorpay, Cashfree) with timing-safe compare before parsing; Meta verify-token random ≥ 32 bytes; every raw payload stored in `webhook_events`, deduped by `(source, event_id)`.
- Gateway and WABA credentials are never stored in the DB — `*_ref` columns hold vault keys (Supabase Vault / env-injected secrets); the API reads them only inside the payment/whatsapp adapters.
- Owner bot: tool calls run with the session's tenant context, never with arguments the model supplies for `vendorId`/`outletId`; write tools are denied in v1 at the registry level (`tools.surface_bot`), not by prompt. 🧪
- All secrets via env (zod-validated at boot); `.env*` gitignored; secret scanning (gitleaks) in CI. 🚦
- TLS everywhere (Caddy auto-HTTPS); HSTS; no HTTP listener in production.

## 6. Rate limiting & abuse (Redis-backed, per route class)

| Route class | Limit (v1 defaults) | Keyed by |
|---|---|---|
| `POST /auth/otp/request` | 5/hour + 60s cooldown | phone AND IP |
| `POST /auth/otp/verify` | 3 attempts per OTP; 10/hour | phone AND IP |
| `POST /auth/login` | 10/15 min, then backoff | email AND IP |
| `POST /orders` | 5/min | customer |
| Review create/edit | 5/hour | customer |
| Campaign send | 10/day | vendor |
| Public discovery/menu | 120/min | IP |
| Everything else authed | 300/min | user |

- Fail-closed on auth routes if Redis is down; fail-open elsewhere (see 05 §9.2). 429s include `Retry-After`. 🧪
- OTP abuse is also a **cost attack** (each OTP = paid WhatsApp message): daily platform-wide OTP budget alarm.

## 7. Dependency & platform hygiene

- `npm audit` (fail on high) + Dependabot/Renovate weekly in CI. 🚦
- Node LTS only; `engines` pinned; lockfile committed; no postinstall-script packages added without review (`--ignore-scripts` in CI installs).
- Docker: non-root user, distroless/slim base, read-only fs where possible.
- CORS: exact-origin allowlist from env (the three frontend origins); credentials true only for auth paths; no `*`.
- Postgres (Supabase): TLS required; **four** roles — `regulars_app` (RLS enforced, no DDL, no DELETE, **no grant on `pii`**), `regulars_identity` (the only reader of `pii`, used by one pool in `modules/identity`), `regulars_worker` (BYPASSRLS, jobs only), `regulars_ml` (read-only, never granted `pii`); migrations run with the direct URL under a separate migrator user; pooler URL at runtime. Network restrictions to API hosts where the plan allows.

## 8. Privacy & data protection (India — DPDP Act awareness)

- We hold personal data: phone, name, birthday, order history. Principles baked into v1:
  - Collect minimum (birthday is optional, no year, no address in v1).
  - WhatsApp marketing strictly on recorded, revocable consent (timestamp + source stored). Opt-out honored globally and immediately. 🧪
  - Consent is purpose-level (marketing / utility / model-training) with a notice version, kept as current flags on `customers` plus the append-only `customer_consent_events` ledger. Imported lists get no marketing until an opt-in event exists. 🧪
  - Vendors see masked phones in lists; the vendor export (#9) contains the vendor's own `customer_profiles` (the list is theirs) but never other tenants' data. 🧪
  - PII redaction in logs: pino redact paths for phone/name/birthday; request bodies never logged raw on auth/order/customer routes; `app.events.payload` carries no PII by contract (payload schemas are PII-free, tested). 🧪
  - Erasure path (`docs/03` §4.7): `customers` phone hashed + name/birthday nulled, `customer_profiles` PII nulled, `messages` content nulled, bill phone masked to last-4 (GST record), lake tombstones. `customer_consent_events` kept as evidence.
  - ML/lake export strips PII at the CDC boundary via a per-table column allowlist; free-text (bot input, message text) exported only where `ml.ai_feedback.training_eligible = true`.
  - **Identity is physically separated inside the database** (schema `pii`, §4b): a compromised API credential yields orders and totals, never a phone book. Customer refresh tokens and device rows live there too, so an erasure signs every device out.
  - **A vendor sees a person only after that person ordered with them** (item 67). Before that, browsing and visits are aggregates. This is a privacy property of the product, not only a feature — say it plainly in the customer notice.
  - **Browse tracking** (item 68) records behaviour against a random device id with no fingerprinting, no free text and no search strings. It begins before the customer has identified themselves, so the **notice must cover it at the point of scan**, not at OTP. ⚠ Lawyer sign-off required before launch (`docs/01` §6). Unlinked visitors and their browse rows are deleted after 400 days; `browse_events` detail after 90.
  - Data lives in-region (Supabase Mumbai `ap-south-1`).
- Disclaimer: DPDP compliance details (notices, grievance officer, etc.) need legal review before public launch — this doc covers engineering posture only.

## 9. Auditability & monitoring

- `app.audit_logs` (partitioned, append-only) for every privileged mutation: settings changes (incl. auto-accept), refunds, voids, day-close, customer block, subscription changes, template changes, campaign sends, admin edits. Every tool execution links its audit row (`tool_execution_id`).
- Auth anomalies logged with IP/device: refresh-reuse events, repeated OTP failures, login backoff triggers; daily digest to super_admin in v1 (no SIEM yet).
- Sentry alerts on: 5xx rate spike, DLQ growth, webhook signature failures, OTP budget breach.

## 10. Release gates summary 🚦

1. Tenant-leak matrix green.
2. `npm audit` high = 0; gitleaks clean; `sql.raw` audit clean.
3. All 🧪 items in this doc + `05-edge-cases-and-failures.md` implemented and green.
4. `.env.example` complete; boot fails on missing config (proven by a CI test).
5. Load smoke: 50 concurrent order placements against staging with `limited` stock — zero oversells, zero duplicate orders, zero duplicate invoice numbers under 20 concurrent settles.
6. `db/seed/smoke.sql` green against the migrated staging database (RLS, counters, partitions, append-only guards).
7. **PII wall green** (§4b): `regulars_app` has zero privileges in schema `pii`; reveal-before-order raises 403; no `pii.*` table in the lake publication; `phone`/`raw` absent from `regulars_app`'s column grants on the WhatsApp ledger.
8. Browse-tracking notice text signed off by the lawyer, and the telemetry endpoint proven non-blocking (an induced telemetry failure does not fail an order placement).
