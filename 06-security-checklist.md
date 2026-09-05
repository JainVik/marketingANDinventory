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
- Global `customers` table: readable only via the customer's own JWT (`app.customer_id`) or through the vendor's `customer_profiles` row (policy `customers_read`). 🧪
- Public projections: storefront endpoints select explicit column lists. A projection test asserts no `gateway_*`, `sub_*`, settings, or customer fields appear in public JSON. 🧪

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
- Postgres (Supabase): TLS required; three roles — `regulars_app` (RLS enforced, no DDL, no DELETE), `regulars_worker` (BYPASSRLS, jobs only), `regulars_ml` (read-only); migrations run with the direct URL under a separate migrator user; pooler URL at runtime. Network restrictions to API hosts where the plan allows.

## 8. Privacy & data protection (India — DPDP Act awareness)

- We hold personal data: phone, name, birthday, order history. Principles baked into v1:
  - Collect minimum (birthday is optional, no year, no address in v1).
  - WhatsApp marketing strictly on recorded, revocable consent (timestamp + source stored). Opt-out honored globally and immediately. 🧪
  - Consent is purpose-level (marketing / utility / model-training) with a notice version, kept as current flags on `customers` plus the append-only `customer_consent_events` ledger. Imported lists get no marketing until an opt-in event exists. 🧪
  - Vendors see masked phones in lists; the vendor export (#9) contains the vendor's own `customer_profiles` (the list is theirs) but never other tenants' data. 🧪
  - PII redaction in logs: pino redact paths for phone/name/birthday; request bodies never logged raw on auth/order/customer routes; `app.events.payload` carries no PII by contract (payload schemas are PII-free, tested). 🧪
  - Erasure path (`docs/03` §4.7): `customers` phone hashed + name/birthday nulled, `customer_profiles` PII nulled, `messages` content nulled, bill phone masked to last-4 (GST record), lake tombstones. `customer_consent_events` kept as evidence.
  - ML/lake export strips PII at the CDC boundary via a per-table column allowlist; free-text (bot input, message text) exported only where `ml.ai_feedback.training_eligible = true`.
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
