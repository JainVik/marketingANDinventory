# Restaurant Platform — AI Build Documentation Pack

A complete feeding-document set for building the platform with AI coding tools (Claude Code, Copilot, Cursor). The docs are written to be **binding specs**, not inspiration: the AI reads them, then codes against them.

## What's in the pack

```
CLAUDE.md                          ← put at the REPO ROOT — Claude Code auto-reads it every session
docs/ (or repo root)
├── 01-product-scope.md            ← WHAT v1 is (and the explicit not-list)          [feed to AI]
├── 02-architecture.md             ← stack, monorepo, layers, tenancy, WhatsApp rail [feed to AI]
├── 03-database-schema.md          ← every collection, index, and atomicity rule     [feed to AI]
├── 04-api-contract.md             ← endpoints, envelope, error codes, sockets       [feed to AI]
├── 05-edge-cases-and-failures.md  ← the failure catalog — requirements with tests   [feed to AI]
├── 06-security-checklist.md       ← injection, RBAC, tenant-leak gate, releases     [feed to AI]
├── 07-market-research.md          ← India POS/WhatsApp/AI market — for the FOUNDERS
├── 08-owner-com-teardown.md       ← Owner.com product teardown + verdict — FOUNDERS
├── 09-execution-roadmap.md        ← day one → production, phases & gates — FOUNDERS
├── 10-backend-technical-roadmap.md← backend decisions + 15-step build order — BOTH
├── 11-v1-master-feature-specification.md ← Master V1 Feature Spec (POS, Invoicing, QR, WABA) [feed to AI]
└── 12-ux-wireframe-flow-map.md    ← Screen-by-Screen UX flows & Mermaid maps        [feed to AI]
```

Docs 01–06, 11, 12 + CLAUDE.md are the AI's binding contract. Docs 07–09 are founder strategy references (don't paste them into coding sessions — their conclusions are already folded into 01–06, 11). Doc 10 bridges both: humans read it to understand the order; point the AI at it when scaffolding.

## How to use with Claude Code

1. Create the repo, copy `CLAUDE.md` to the root and `docs/` alongside it, commit.
2. Start sessions with a scoped instruction, e.g.:
   > "Read CLAUDE.md and docs/02. Scaffold the monorepo exactly as specified — workspaces, apps/api bootstrap (loaders, middlewares, error handler, env config), packages/shared with the order-state machine, roles, and error-code enums. No feature modules yet."
3. Then build **one module per session**, always pointing at the docs:
   > "Implement the auth module. Requirements: docs/02 §6, docs/04 §3, docs/05 §1, docs/06 §5–6. Include the 🧪 tests."
4. Suggested build order (each step is shippable and testable):
   auth → vendors+catalog → public storefront APIs → orders (the hard one: transactions + idempotency + sockets) → reviews → customer profiles/segments → WhatsApp pipeline (queue, worker, webhook) → campaigns/triggers → admin panel → PWA polish.
5. After each module, run the AI's self-review step (bottom of CLAUDE.md) and the test suite before moving on.

## Keeping the docs honest

- These docs are the contract. When a real decision changes (e.g., you add Razorpay), **update the doc in the same PR** — stale specs are worse than none.
- Anything marked 🚦 in doc 06 is a launch blocker. Anything marked 🧪 must exist as an automated test.
- Decisions already locked: MongoDB + Mongoose, customer side as PWA (QR-first), pay-at-counter in v1, official WhatsApp Business Platform via a BSP, single city, manual vendor onboarding.

## Open items for you (not for the AI)

- Pick the BSP (Gupshup / Interakt / AiSensy / Twilio — compare current per-message marketing/utility pricing for India and onboarding speed) and register the WABA + templates.
- Platform name + domain (replace "LocalServe" everywhere).
- Subscription plan pricing (`basic` / `growth`) and what the daily message quota is per plan.
- City list for launch, and your first 3–5 pilot cafes (their menus become your seed data).
- Legal: terms for vendors, privacy policy, DPDP notice — needs a human/lawyer, not AI code.
