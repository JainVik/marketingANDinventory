# 06 — Security Checklist

Binding security requirements for v1. Items marked 🧪 must have automated tests; items marked 🚦 are release gates (deploy blocks until green).

## 1. Input handling & injection

- Every endpoint zod-parses body, params, AND query before the controller runs; unknown keys stripped (`.strict()` where shape is closed). 422 on failure. 🧪
- **NoSQL injection:** with zod coercing types, objects can't smuggle into string fields — but belt-and-braces: `express-mongo-sanitize` strips `$`/`.` keys from all inputs, and no query filter is ever built by spreading raw request objects (`{ ...req.query }` into a Mongoose filter is a review-blocking defect). 🧪 (test: `{"phone": {"$ne": null}}` on OTP login → 422, not a query)
- No string-built queries anywhere; `$where`, `$function`, `$accumulator` are forbidden operators (ESLint ban + code review).
- Regex from user input (search fields) → escape regex metacharacters via shared util; cap search string length (100) — ReDoS guard. 🧪
- File uploads: presigned URLs constrain content-type and size (≤ 2 MB); on save, server re-validates the stored object's magic bytes and re-serves only from the public bucket domain — user-controlled URLs are never stored (only keys we issued). SVG uploads forbidden (script vector).
- JSON body limit 100 kB; array length caps in zod on every list field (items ≤ 50, addonIds ≤ 20, etc.).

## 2. XSS & frontend

- React's default escaping is the mechanism; `dangerouslySetInnerHTML` is banned (ESLint error) — no exceptions in v1 (no rich text exists).
- All user-generated text (reviews, notes, item names) rendered as text nodes only. 🧪
- CSP on all three frontends: `default-src 'self'`, images from our storage domain, `connect-src` API + socket origin; no inline scripts (Vite builds comply).
- `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin` via helmet.
- Never reflect request data into error messages verbatim beyond zod paths; error `message` strings come from our enum, not from input.

## 3. AuthZ — permission matrix (enforced by `authorize()` + service-level ownership asserts)

| Capability | customer | vendor_staff | vendor_admin | super_admin |
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

- `vendorId` always from JWT (tenantContext), never from client input on `/vendor/*`. Grep-able invariant: `req.body.vendorId|req.params.vendorId|req.query.vendorId` must not appear in vendor modules (ESLint custom rule).
- Public projections: discovery/storefront endpoints use explicit `.select()` whitelists. A projection test asserts no `subscription`, `quotas`, `settings`, or contact-list fields ever appear in public JSON. 🧪

## 5. Secrets, tokens, crypto

- Passwords: argon2id (memory 64 MB, time 3); no MD5/SHA-anything for passwords. OTPs & refresh tokens stored hashed (SHA-256 is fine here — high entropy inputs).
- JWT RS256; private key only on API hosts via env/secret store; `kid` header for future rotation; access 15 min; no sensitive data in claims beyond `sub/role/vendorId`.
- Refresh cookie: `httpOnly`, `Secure`, `SameSite=Strict`, path-scoped to `/api/v1/auth`.
- Webhook: verify `X-Hub-Signature-256` (timing-safe compare) before parsing; Meta verify-token random ≥ 32 bytes.
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
- Mongo Atlas: IP allowlist to API hosts, TLS, least-privilege DB user (no admin), separate users for api vs migrations.

## 8. Privacy & data protection (India — DPDP Act awareness)

- We hold personal data: phone, name, birthday, order history. Principles baked into v1:
  - Collect minimum (birthday is optional, no year, no address in v1).
  - WhatsApp marketing strictly on recorded, revocable consent (timestamp + source stored). Opt-out honored globally and immediately. 🧪
  - Vendors see masked phones; full numbers never leave the platform (no export endpoints in v1). 🧪
  - PII redaction in logs: pino redact paths for phone/name/birthday everywhere; request bodies never logged raw on auth/order/customer routes. 🧪
  - Deletion path: customer account deletion (admin-mediated in v1) anonymizes user + customer_profiles (`name: 'Deleted user'`, phone removed/hashed) while keeping order aggregates for vendor accounting.
  - Data lives in-region where practical (Atlas Mumbai region).
- Disclaimer: DPDP compliance details (notices, grievance officer, etc.) need legal review before public launch — this doc covers engineering posture only.

## 9. Auditability & monitoring

- `audit_logs` for every privileged mutation: subscription changes, review hide/unhide, staff add/remove, template changes, campaign sends, manual vendor edits by admin.
- Auth anomalies logged with IP/device: refresh-reuse events, repeated OTP failures, login backoff triggers; daily digest to super_admin in v1 (no SIEM yet).
- Sentry alerts on: 5xx rate spike, DLQ growth, webhook signature failures, OTP budget breach.

## 10. Release gates summary 🚦

1. Tenant-leak matrix green.
2. `npm audit` high = 0; gitleaks clean.
3. All 🧪 items in this doc + `05-edge-cases-and-failures.md` implemented and green.
4. `.env.example` complete; boot fails on missing config (proven by a CI test).
5. Load smoke: 50 concurrent order placements against staging with `limited` stock — zero oversells, zero duplicate orders.
