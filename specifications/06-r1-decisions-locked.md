# 06 — Round 1 decisions (locked)

This doc consolidates every decision Alexis made on the Round 1 questions. It is the **authoritative summary** — when in doubt, this is what we agreed.

Source: [05-open-questions.md](05-open-questions.md), Round 1 answers.

---

## Strategy & branding

- **Product codename:** `Koan Cloud` for the MVP. No marketing name yet — naming work happens after MVP is functional.
- **Domain:** `koan.cloud` is the working domain.
  - `koan.cloud` → marketing/CTA/subscription site (Greptile-style landing)
  - `koan.cloud/dashboard` → customer dashboard (the app)
- **No free tier.** B2B pricing only. Possibly a 14-day Stripe trial before billing kicks in.
- **Target launch:** MVP-functional **2–4 months from now** (target end of summer 2026, ideally before).
- **Pricing reference:** **Greptile.com** is our pricing model template. Their per-seat + usage limits structure → adapt for Koan's broader scope. See [09 — Pricing and design partner](09-pricing-and-design-partner.md).

## Anthropic & token model

- **Hosted credits, not BYOK.** Anantys holds the master Anthropic API key. Customer never sees Anthropic — we are the product.
- **Pricing baseline:** Anthropic price grid × **margin multiplier** + plan-level limits (missions/month, repos plugged, recurring tasks scheduled, specific skills).
- **Hard cap on overage** (MVP): instance auto-pauses when allowance exhausted. Customer must upgrade. Post-MVP: "buy more credits" top-up.
- **Annual pricing:** ~30% discount for yearly billing (matches existing Anantys rule).

## Architecture (the big shifts)

- **Koan Cloud is a separate product** — its own repo, its own DB, its own user base, its own auth.
  - **No shared session, no shared user table** with Anantys Invest. Different universes.
- **Anantys-back evolves into the "Anantys Stack"** — a SaaS Cloud Building Toolkit. Shared services (mail, billing, websocket, encryption) are exposed via internal API endpoints under a new prefix (`/anantys-stack/*` — naming TBC).
  - Long-term goal: spin up multiple SaaS products from this stack.
  - Short-term: pragmatic — expose 2–3 endpoints needed by Koan Cloud, refactor over time.
- See **[08 — Platformization and repos](08-platformization-and-repos.md)**.

- **Multi-tenant worker pool** instead of 1 Railway service per customer.
  - Stateless workers pick missions from a shared queue, run them with strong context isolation, persist tenant state to a DB.
  - See **[07 — Multi-tenant worker model](07-multi-tenant-worker-model.md)**.

## Dashboard & frontend

- **Stack:** Next.js, brand new app in `koan-cloud` repo (not nested in any Anantys frontend).
- **Design system:** reuse the Anantys design system from `investmindr-nextapp` (extracted as a package — see open Q in [10](10-open-questions-round-2.md)).
- **Aesthetic:** Greptile-inspired dark UI **applied through the Anantys design system**.
- **Mobile:** responsive only, mobile-polish deferred to post-MVP.
- **Chat / chatbot:** **deferred to MVP+1**. MVP user interaction = dashboard + GitHub mentions on issues/PRs.
- **WebSocket reuse:** keep the existing Anantys WS infra (the `RAILWAY_TARGET=websocket` server). Likely add a `/koan/chat` namespace post-MVP.
- **Investmindr chatbot component:** consider re-using or duplicating the React chatbot component when chat ships.

## GitHub integration

- **MVP:** customer pastes a GitHub Personal Access Token (PAT). OAuth flow comes shortly after but PAT is the day-1 path.
- **Post-MVP:** native GitHub App (a "Koan bot" any GitHub user can install), enabling richer webhooks and finer-grained scopes.
- **Repos per tier:** configurable per-tier from day 1, exact numbers to be defined later.
- **GitHub scopes:** mirror what self-hosted Koan needs (TBC during Phase 0).
- **CI cost worries (broken Actions, expensive jobs):** not Koan's responsibility.

## Lifecycle & operations

- **Update strategy:** open — Nicolas to decide between control-plane redeploy vs per-instance auto-update.
- **Backups:** likely DB-mirror all Koan `.md` state (missions, journal, memory) — design proposed in [07](07-multi-tenant-worker-model.md).
- **Payment failure:** 3-day grace, mail reminders day 1 and day 3, then suspend.
- **Cancellation:** instance shuts down at the last day of the billing period (no grace).
- **Region:** EU only at MVP.
- **Active hours scheduling:** all tiers run 24/7 by default; customer-facing settings let users define heavy/light hours and pause windows.
- **On-call, SLAs, status page:** out of scope for the spec. Handled later.
- **Sprint cadence:** **1 week**.
- **Where issues live:** `koan-private` for OSS Koan code only; **Koan Cloud SaaS work lives in the new `koan-cloud` repo**.

## Stuff to handle "for free"

These weren't framed as questions but Alexis confirmed/added:

- **Welcome mission** = repo exec summary with most important findings. First-impression magic.
- **Bad PR risk:** Koan already defaults to draft PRs and `auto_merge: false` — fine as is.
- **Cost monitoring** (per-tenant Anthropic spend outliers): build the alert before we have outliers. Belongs in the control plane from day 1.

## Things Alexis explicitly punted

- Annual pricing fine-tuning beyond "30% off"
- Token allowance per tier (validate after Phase 0 / dogfooding)
- On-call rotation and SLAs (out of spec)
- Status page (not MVP)
- Slash command UX (chat is post-MVP, so this question shifts)
- Live logs SSE/WS (post-MVP)

These are in the [Round 2 open questions](10-open-questions-round-2.md) only where they block Sprint 1; otherwise they sit in a backlog until needed.
