# 08 — Platformization and repo layout

> Driven by Alexis's R1 decisions on Q10–Q14: **Koan Cloud is its own product, not an extension of `anantys-back`.** Anantys evolves into a SaaS platform that hosts multiple products.

## The mental model Alexis described

> *"Anantys is the company name (like Alphabet or Microsoft). Anantys Invest is our first SaaS product. The backend stack has reusable parts. We want to decouple the Anantys Stack from the Anantys Invest app code, moving step by step toward modularization, so we can write and deploy as many SaaS products as we want."*

Three layers, three lifetimes:

```
┌──────────────────────────────────────────────────────────────────┐
│  LAYER 1 — Anantys Stack (the platform / toolkit / "OS")         │
│                                                                  │
│  Reusable infrastructure that any Anantys-built SaaS product     │
│  consumes. Not customer-facing.                                  │
│                                                                  │
│  • Mail-as-a-service (Mailgun wrapper, multi-domain)             │
│  • Billing-as-a-service (Stripe wrapper, subs, metering)         │
│  • WebSocket hub (Flask-Sock + Redis sessions, multi-product)    │
│  • Encryption-at-rest helpers                                    │
│  • Authentication primitives (NOT user identity — those are      │
│    per-product)                                                  │
│  • Rate-limiting, greylisting, abuse guards                      │
│  • Internal API endpoints under /anantys-stack/* prefix          │
└──────────────────────────────────────────────────────────────────┘
              ▲                                ▲
              │ HTTP (S2S token)               │ HTTP (S2S token)
              │                                │
┌────────────────────────────┐    ┌────────────────────────────┐
│  LAYER 2A — Anantys Invest │    │  LAYER 2B — Koan Cloud     │
│  (existing customer-facing │    │  (new customer-facing      │
│   investment SaaS)         │    │   developer-tool SaaS)     │
│                            │    │                            │
│  Own user DB               │    │  Own user DB               │
│  Own auth (Google, Apple,  │    │  Own auth (GitHub OAuth,   │
│   email/password)          │    │   GitHub PAT)              │
│  Investment domain logic   │    │  Multi-tenant Koan workers │
│  Frontends                 │    │  Tenant control plane      │
│                            │    │  Greptile-style dashboard  │
└────────────────────────────┘    └────────────────────────────┘
```

## Naming "the platform layer"

Alexis offered three options: `anantys-cloud`, `anantys-os`, `anantys-stack`.

**Recommendation: `Anantys Stack`.**

- `anantys-cloud` reads as "another product of Anantys" — confusing because Koan Cloud is also "of Anantys."
- `anantys-os` is bold but premature — we don't have an OS, we have a toolkit.
- `anantys-stack` describes exactly what it is: a stack of reusable SaaS infrastructure pieces. Boring and accurate.

The internal API prefix on the existing Flask app: `/anantys-stack/*`. This is **the only thing we need to commit to today** — naming the toolkit itself is editable later.

## Repo layout — current and target

### Today

```
github.com/Anantys/
├── anantys-back        — Flask monolith: Anantys Invest backend + frontends as submodules
├── koan-private        — OSS Kōan (single-tenant, file-based, Telegram bridge)
└── (other internal repos)
```

### Target after Epic #1

```
github.com/Anantys/
├── anantys-back        — Flask app, BUT with a new /anantys-stack/* internal-API surface
│                          (mail, billing, WS) carved out as reusable modules.
│                          Anantys Invest stays where it is.
├── koan-private        — OSS Kōan, plus state-backend abstraction + cloud_worker
│                          entrypoint. Self-hosted users are unchanged.
├── koan-cloud          — NEW: koan.cloud customer-facing product
│                          • Backend: control plane, tenant registry, mission scheduler
│                          • Frontend: Next.js dashboard SPA
│                          • Calls anantys-back's /anantys-stack/* for mail + billing
│                          • Calls koan-private workers via Redis queue
│                          • Owns its own MySQL DB and user identity
└── (eventually) anantys-stack — extracted into its own repo when ready
```

`koan-cloud` has the most significant new code surface. `anantys-back` gets a minor carve-out. `koan-private` gets a real refactor (state backend) but no fork.

## What "Anantys Stack" exposes (initial scope)

For Epic #1, only what Koan Cloud needs from day 1:

| Endpoint | Purpose | Auth |
|---|---|---|
| `POST /anantys-stack/mail/send` | Send a transactional email (template, recipient, vars) | S2S bearer token |
| `POST /anantys-stack/billing/checkout-session` | Create a Stripe checkout session | S2S bearer token |
| `POST /anantys-stack/billing/portal-session` | Create a Stripe customer portal session | S2S bearer token |
| `GET  /anantys-stack/billing/subscription/{customer_id}` | Read subscription status | S2S bearer token |
| Webhook receiver in `anantys-back` for Stripe → forward decoded events to Koan Cloud webhook URL | Stripe billing events | Stripe signature |
| `WS   /anantys-stack/ws/{namespace}` | Multi-product WS namespace (used post-MVP for chat) | JWT issued by the consuming product |

Anything else the consumer needs (auth, user identity, repo state) lives in **the consumer's own repo** — Koan Cloud has its own user table, its own Stripe customer mapping, its own session.

This deliberately keeps the platform layer thin. We resist the temptation to "make Anantys Stack do auth" — different products have different auth needs, and centralizing auth means rewriting it before we have product-market fit on the second product.

## Why Koan Cloud has its own DB and user base (recap)

This is the biggest divergence from doc 02 (v0.1). The reasons:

1. **Different user populations.** Anantys Invest serves retail investors (in French, with banking + portfolios). Koan Cloud serves software developers (in English, with GitHub repos). Zero overlap in identity attributes, plans, or onboarding.
2. **Different operational models.** Anantys Invest is a stable monolith with daily deploys; Koan Cloud will iterate fast with experimental features. Coupling them slows both down.
3. **Future portability.** If we ever sell or spin off Koan Cloud, having it as a separate repo with a separate DB is a precondition, not an afterthought.
4. **Smaller blast radius.** A bug in Koan Cloud's auth doesn't take down Anantys Invest sessions.

The price we pay: small duplication of "user" model, login flows, session handling. Acceptable.

## Authentication strategy in `koan-cloud`

- **Primary signup path:** GitHub OAuth (full identity, repo permissions, single click).
- **Fallback (rare):** classical email + password.
- **Session:** JWT issued by `koan-cloud` backend, scoped to `*.koan.cloud`.
- **No SSO with Anantys Invest.** A user with both products has two accounts. We may unify later via Anantys Stack, but not for Epic #1.

## What `koan-cloud` repo actually contains

Proposed structure (TBC, see [open Q in 10](10-open-questions-round-2.md)):

```
koan-cloud/
├── backend/                          # Python (Flask, mirroring Anantys conventions)
│   ├── app/
│   │   ├── __init__.py               # create_app()
│   │   ├── routes/
│   │   │   ├── api/                  # JSON API consumed by dashboard SPA
│   │   │   ├── auth/                 # GitHub OAuth + email/password
│   │   │   ├── webhooks/             # Stripe webhook receiver (forwarded by Anantys Stack)
│   │   │   └── public/               # marketing site routes (or static, or separate)
│   │   ├── models/
│   │   │   ├── user.py               # Koan Cloud user model
│   │   │   ├── tenant.py             # KoanTenant — runtime config, plan, tokens
│   │   │   ├── mission.py            # mirrors koan_mission table
│   │   │   ├── memory.py             # tenant memory blobs
│   │   │   └── journal.py            # journal entries
│   │   └── services/
│   │       ├── mission_scheduler.py  # enqueues missions to Redis for workers
│   │       ├── tenant_provisioning.py
│   │       ├── github_integration.py # OAuth + token mgmt + repo listing
│   │       ├── usage_tracker.py      # token + mission counters per tenant
│   │       └── anantys_stack_client.py  # HTTP client to Anantys Stack
│   └── tests/
│
├── frontend/                         # Next.js dashboard SPA
│   ├── app/                          # App Router
│   ├── components/
│   ├── lib/
│   └── design-system/                # Symlink/package import from investmindr-nextapp DS
│
├── workers/                          # Optional: deployable wrappers around koan-private workers
│   ├── Dockerfile.worker             # Builds the Koan worker image (consumes koan-private)
│   └── entrypoint.sh
│
├── infra/
│   ├── railway.json                  # Railway config
│   ├── migrations/                   # Alembic
│   └── docker-compose.yml            # local dev (MySQL + Redis + worker + backend + frontend)
│
└── README.md
```

The frontend likely deploys to Vercel or Railway static, the backend to Railway, and `koan-private` workers to a Railway worker service. Three deployable units.

## How the design system is shared

Two clean options:

### (A) Extract design system as an internal npm package (recommended)

```
investmindr-nextapp/
└── packages/
    └── design-system/        # publishable internal package
        ├── components/
        ├── tokens/
        └── package.json      # @anantys/design-system
```

`koan-cloud/frontend` adds `"@anantys/design-system": "*"` as a dependency (via Yarn/PNPM workspaces or a private npm registry).

Pro: single source of truth, design changes propagate.
Con: requires a tiny package extraction effort upfront.

### (B) Copy components into `koan-cloud/frontend/components/`

Pro: zero infra cost.
Con: drift over time, double maintenance.

**Recommendation: (A).** The investment is small (~half a day) and pays back forever.

## Migration plan for the platformization (light touch)

We are not refactoring Anantys-back into a SaaS toolkit during Epic #1. We are:

1. **Creating** `/anantys-stack/*` blueprint in `anantys-back` with the 4–5 endpoints listed above.
2. **Keeping** Anantys Invest behavior unchanged.
3. **Moving** zero existing code in week 1 — just wrapping `mail_service` and `stripe_service` behind HTTP endpoints.
4. **Documenting** the Anantys Stack contract in a single doc inside `anantys-back/documentation/anantys-stack.md`.

Future migrations (extract to its own repo, build proper SDK, formalize multi-product auth) happen **after** Koan Cloud has paying customers and we've proven the platformization vision works.

## Risks of repo split

| Risk | Mitigation |
|---|---|
| **Cross-repo coordination** when changing the `/anantys-stack/*` contract | Treat it as a public API. Versioned endpoints (`/v1/`), breaking changes require a notice + migration. |
| **CI complexity** running tests across repos | Each repo runs its own CI. Integration tests in `koan-cloud` mock Anantys Stack endpoints; real integration validated via staging environment. |
| **Anantys Stack endpoint goes down → Koan Cloud loses billing/mail** | Monitor Anantys Stack as Tier-1 dependency. If Anantys Invest is up, Anantys Stack is up. They cohabit. |
| **Two MySQL databases to admin** | Different host or same host, different schema. Acceptable cost. |
| **Auth duplication tempting "let's centralize" rewrites** | Resist. Each product owns its identity until we have a strong reason to unify. |

## What we don't do (and why)

- ❌ Build an SDK / Python package for Anantys Stack endpoints in week 1. We use plain `requests.post(...)` and worry about an SDK after the third consumer.
- ❌ Microservice architecture for Anantys Stack. It stays inside `anantys-back` (one Flask app, two URL prefixes). Splitting into its own repo is a future exercise.
- ❌ Move Anantys Invest auth/mail/billing code. It already works. We just add a thin HTTP layer in front.
- ❌ Shared Redis between Anantys Invest and Koan Cloud workers in week 1. Spin up a fresh Redis for `koan-cloud` to keep blast radius isolated; the Anantys Stack WS hub remains on its own Redis.
