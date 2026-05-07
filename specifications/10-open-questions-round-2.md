# 10 — Open questions, round 2

Round 1 answers cleared the strategic direction. Round 2 surfaces the new questions opened by those decisions. Same convention: drop a `**Decision (Alexis):**` line under each question.

The list is shorter than R1. Most of the architectural fog has lifted; what remains is calibration, sizing, and a few specific design choices.

---

## A. Anantys Stack carve-out

### Q-R2-1 — Naming the platform layer
Alexis offered `anantys-cloud`, `anantys-os`, `anantys-stack`. I recommend **Anantys Stack** ([08 — Platformization](08-platformization-and-repos.md)). Confirm or pick.

### Q-R2-2 — Internal API auth: server-to-server token, mTLS, or JWT?
Koan Cloud calls Anantys Stack endpoints (mail, billing). Three options:

- (A) Static S2S bearer token, rotatable from env. Simplest.
- (B) mTLS between services. Most secure.
- (C) Short-lived JWT issued by an Anantys Stack auth endpoint.

`Lean: (A) for MVP, plan migration to (C) post-MVP.` Pragmatic, gets us shipping.

### Q-R2-3 — Where do Stripe webhooks land first?
Stripe sends `customer.subscription.created` etc. Two routes:

- (A) Stripe → `anantys-back` webhook → `anantys-back` decides which product subscription this is → forwards to the right consumer (Anantys Invest or Koan Cloud).
- (B) Stripe → directly to each product's webhook URL (Stripe supports multiple endpoints).

`Lean: (B).` Simpler, no extra hop, each product owns its plan IDs and reacts directly. Anantys Stack only abstracts the *creation* of checkout sessions, not the receiving of webhooks.

### Q-R2-4 — How many Anantys Stack endpoints does Koan Cloud actually need at MVP?
Proposed in [08](08-platformization-and-repos.md):

- `POST /mail/send`
- `POST /billing/checkout-session`
- `POST /billing/portal-session`
- `GET  /billing/subscription/{customer_id}`

Anything missing? E.g., do we need user-search-across-products, shared Mailgun unsubscribe, GDPR deletion request fan-out?
*Genuinely open — list anything you want available from day 1.*

### Q-R2-5 — Multi-domain Mailgun
You suggested using Mailgun's multi-domain feature so emails come from `noreply@koan.cloud` while reusing the existing API key. Confirm: is the Anantys Mailgun account already on a plan that supports multiple domains, and do we need to add `koan.cloud` as a sending domain (DNS records, DKIM)?
*Operational checklist item.*

---

## B. Repo creation and ownership

### Q-R2-6 — Repo `koan-cloud`: who creates it, when, and where?
- Org: `Anantys` (private)?
- Default branch: `main`
- Initial structure: per [08](08-platformization-and-repos.md) §"What `koan-cloud` repo actually contains"
- License: proprietary (this is the closed-source SaaS, distinct from the OSS `koan-private`)

`Lean: create now in Sprint 1.` Naming "koan-cloud" or something else?

### Q-R2-7 — Anantys design system: extract as a package?
The investmindr-nextapp design system needs to be importable from `koan-cloud/frontend`. Options:

- (A) Yarn/PNPM workspace inside a (new) `anantys-monorepo` umbrella repo. Big refactor.
- (B) Publish `@anantys/design-system` to a private npm registry (GitHub Packages). Medium refactor, fastest payoff.
- (C) Symlink / git-submodule the design-system folder. Hacky.
- (D) Copy components on Sprint 1, plan extraction later.

`Lean: (D) for Sprint 1, (B) by end of Phase 2.` We don't want a refactor of investmindr blocking Koan Cloud start.

### Q-R2-8 — Backend language for `koan-cloud`: Python (Flask) or Node (Next.js full-stack)?
Anantys uses Flask. Koan workers are Python. Argument for Flask: stack uniformity, share patterns (logging, services, registry). Argument for Node: dashboard is Next.js anyway, full-stack TypeScript reduces context-switching.

`Lean: Flask backend + Next.js frontend.` Same shape as Anantys. The dashboard team works in TS, the backend team in Python. Both already familiar.

### Q-R2-9 — Frontend deploy target
Vercel? Railway static? Cloudflare Pages?
*Genuinely open. Vercel is the path of least resistance for Next.js but adds another vendor.*

---

## C. Multi-tenant worker design (the big one)

### Q-R2-10 — Confirm the multi-tenant worker architecture
Proposed in [07 — Multi-tenant worker model](07-multi-tenant-worker-model.md). Big shifts:

- Stateless worker pool, shared mission queue.
- Per-mission subprocess isolation.
- Tenant state in MySQL + S3.
- One Anthropic key for everyone, with `metadata.user_id = tenant_id` per call.

Sign off, push back, or ask for changes?

### Q-R2-11 — State backend abstraction in `koan-private`
Lift `missions.py`, `memory_manager.py`, `journal/*`, `pause_manager.py`, etc. behind a `StateBackend` protocol. OSS users keep `FileBackend`; cloud uses `CloudBackend` (MySQL + S3).

This is a real refactor — touches ~10 modules in `koan/app/`. Estimated 1–2 weeks for one engineer.

`Strong recommendation: do it cleanly, in koan-private, behind a flag.` It's the right architecture even if cloud never ships.

### Q-R2-12 — Mission queue technology
- (A) Redis Streams (lightweight, ordered, with consumer groups).
- (B) BullMQ on Redis (more features, JS ecosystem — feels weird in a Python stack).
- (C) PostgreSQL `LISTEN/NOTIFY` (no Redis needed but Anantys is on MySQL — won't work).
- (D) Simple polling on a `koan_mission` MySQL table.

`Lean: (A) Redis Streams.` Mature, simple, Python-native (`redis-py`), already deployed for Anantys. We add a new Redis instance for `koan-cloud`.

### Q-R2-13 — Persistent storage layer
Confirm the split:

- **MySQL** for missions, journal, memory (TEXT columns), tenant config, runtime status. Query-friendly from dashboard.
- **S3-compatible** (Cloudflare R2 or AWS S3) for repo workspace snapshots if we need them, large memory blobs > 1 MB, daily backups.
- **Redis** for mission queue, ephemeral session locks, rate limiting.

`Lean: as written.` Anything else needed?

### Q-R2-14 — Worker base image: extend `koan-private` Dockerfile?
The worker is `python -m koan.app.cloud_worker`, so its image = the existing Koan Docker image + cloud entry point + DB drivers. Two ways:

- (A) Extend `koan-private/Dockerfile` with a `cloud-worker` build target.
- (B) New `Dockerfile.worker` in `koan-cloud/workers/` that `FROM`s the OSS Koan image.

`Lean: (B).` Keeps OSS Dockerfile clean, cloud-specific deps stay in `koan-cloud`.

### Q-R2-15 — How many concurrent missions does a worker run?
A Railway worker replica on a 2GB plan can run, say, 4 concurrent subprocesses without trouble. Should we:

- (A) 1 mission per worker (simpler, scale by adding replicas).
- (B) N missions per worker (better resource use, more bookkeeping).

`Lean: (A) at MVP`, switch to (B) if Railway cost becomes a bottleneck.

### Q-R2-16 — Cron-style recurring missions
Today, Koan's `loop_manager.py` schedules contemplative sessions and recurring missions itself. In multi-tenant cloud, who owns the clock?

- (A) Control plane runs a scheduler (cron or Celery beat or `ScheduleWakeup`-style) that enqueues recurring missions.
- (B) A dedicated `koan-scheduler` service (one Railway replica).

`Lean: (A) for MVP — add a `flask koan-cloud schedule-tick` cron in the control plane.` (B) only if scheduling logic gets complex enough to warrant a separate service.

### Q-R2-17 — What happens to per-tenant `auto_update.py`?
In OSS Koan, instances pull updates from `main`. In cloud mode, workers are immutable images deployed by Railway.

`Strong recommendation: disable `auto_update.py` when `KOAN_STATE_BACKEND=cloud`.` Confirmed?

### Q-R2-18 — Anthropic key model — single key vs Workspace-per-tenant?
Anthropic offers Workspaces (sub-organizations) under a master org. Each Workspace has its own key, billing line, AUP attribution.

- (A) Single master key for all tenants. `metadata.user_id = tenant_id` per call. Simplest.
- (B) One Workspace per tenant. Strongest audit story (Anthropic invoices itemize per Workspace). More setup overhead.

`Lean: (A) for MVP, evaluate (B) at 50 paying tenants.` (B) becomes valuable if AUP enforcement gets real or we want fine-grained Anthropic-side rate limits per tenant.

---

## D. Tenant data model

### Q-R2-19 — Database choice for `koan-cloud`
Anantys uses MySQL. Stick with MySQL for stack uniformity?

`Strong recommendation: yes, MySQL.` Match Anantys conventions, share migration tooling (Alembic), share ops know-how.

### Q-R2-20 — Encryption-at-rest for sensitive fields
GitHub PATs, Anthropic per-tenant metadata (if any), instance bearer tokens. Reuse Anantys's `services/encryption/` helpers via Anantys Stack, or duplicate in `koan-cloud`?

`Lean: duplicate the helper at MVP, share via Anantys Stack later.` Avoids cross-repo coupling on a critical path.

### Q-R2-21 — How do we map "Stripe customer ↔ Koan tenant"?
- Stripe customer is created in `koan-cloud` (its own customer namespace, separate from Anantys Invest).
- `KoanTenant` row stores `stripe_customer_id` and `stripe_subscription_id`.
- Webhooks from Stripe land in `koan-cloud`, look up tenant, update status.

Confirm this is the model (separate Stripe customer namespaces per product).

---

## E. GitHub integration

### Q-R2-22 — PAT vs OAuth for Sprint 1 POC
You said MVP starts with PAT, OAuth later. For Phase 0 POC: should the POC tenant use a PAT (faster) or do we go OAuth from day 1?

`Lean: PAT for Phase 0`, OAuth as the second deliverable.

### Q-R2-23 — GitHub mentions trigger missions (since chat is post-MVP)
You said MVP user-interaction = dashboard + GitHub @mentions. We have `koan/app/github_command_handler.py` already. In cloud:

- (A) Each tenant registers a webhook on each plugged repo, fires into `koan-cloud` backend, which enqueues a mission.
- (B) Polling: control plane periodically fetches notifications via `gh api` per tenant.

`Lean: (A) webhook` — real-time and free. Adds a webhook registration step at provisioning.

### Q-R2-24 — GitHub App migration timeline
Native GitHub App is post-MVP. When? Quarter after GA?
*Genuinely open.*

---

## F. Dashboard (without chat)

### Q-R2-25 — Dashboard scope without chat
With chat moved to MVP+1, the dashboard has fewer surfaces. MVP set:

- **Onboarding:** GitHub OAuth/PAT, plan selection (Stripe checkout), repo picker, "we're setting up your instance" wait screen
- **Overview:** tenant status, today's missions, recent journal entries, current usage / budget
- **Missions:** list + detail views, "queue new mission" form (replaces chat for now)
- **Activity:** journal feed (live-ish via polling, not WS)
- **Repos:** list of plugged GitHub repos, add/remove
- **Settings:** GitHub token mgmt, soul.md edit, pause/focus toggles, per-project config
- **Billing:** Stripe portal link, current usage, plan + usage limits

Sound right? Anything to add or cut?

### Q-R2-26 — "Queue new mission" form vs free-text command
Without chat, the dashboard needs a way to queue missions. Two UX approaches:

- (A) Structured form: project picker, mission text, priority. Boring but clear.
- (B) Single text input that accepts `/skill-name [args]` or natural-language mission. More elegant.

`Lean: (A) at MVP`, (B) when chat ships and we share the input component.

### Q-R2-27 — Real-time updates without chat WS — what cadence?
Without WS, the dashboard polls. Cadence:

- Mission status: every 10s while a mission is active, otherwise 60s.
- Journal entries: every 60s.
- Usage / budget: every 5min.

`Lean: as written.` Heavy enough? Light enough?

### Q-R2-28 — Public marketing site at `koan.cloud`
The dashboard is at `koan.cloud/dashboard`. The marketing site at `koan.cloud/` is a separate concern.

Question: who builds the marketing site, and on what stack?

- (A) Same Next.js app as the dashboard, marketing routes at `/`, dashboard at `/dashboard`. One deploy.
- (B) Static site (Astro, Eleventy, plain HTML) at `koan.cloud`, dashboard separately at `koan.cloud/dashboard` via reverse proxy.

`Lean: (A) for Sprint 1+2`, polish the marketing pages closer to launch.

---

## G. Sprint 1 — revised plan (final time-box)

Given all the R1 + R2 answers, Sprint 1's deliverables shift. Proposed (1 week):

| ID | Deliverable | Owner | Definition of done |
|---|---|---|---|
| S1-01 | **Create `koan-cloud` repo** with skeleton (backend + frontend folders, Dockerfiles, README). | Alexis | Repo exists, basic CI green |
| S1-02 | **Anantys Stack POC: 2 endpoints exposed** (`/mail/send`, `/billing/checkout-session`) under `/anantys-stack/*` in `anantys-back`, behind S2S token | Alexis | curl-able from outside, tested |
| S1-03 | **Phase 0 POC: 1 mission run by a fake worker subprocess against MySQL-backed state** (manual, no real queue, no real provisioning) | Nicolas | A `python -m koan.app.cloud_worker --tenant X --mission Y` end-to-end success |
| S1-04 | **Pricing decision finalized** (Solo/Team/Scale numbers, Anthropic multiplier confirmed) | Alexis | Numbers in [09](09-pricing-and-design-partner.md) committed; if revised, doc updated |
| S1-05 | **Stripe products created in test mode** with the 3 (or 4) tier prices | Alexis | Visible in Stripe test dashboard |
| S1-06 | **GitHub OAuth dev app registered**, callback URL chosen, tokens stored in 1Password | Alexis | App ID + secret recorded |
| S1-07 | **Design partner pitch sent** to at least 1 prospect | Alexis | Email sent or call booked |
| S1-08 | **All R2 questions answered** | Alexis | This doc has `Decision:` lines under every question |
| S1-09 | **Phase 1 task list opened as issues in `koan-cloud` repo** | Alexis | One issue per substantive task, prioritized |
| S1-10 | **Phase 1 task list opened as issues in `koan-private` repo** | Nicolas | Same — focused on state-backend refactor + cloud_worker entry point |

Goal: by end of Sprint 1, both engineers know exactly what to do for the next 3 sprints, the architecture is settled, and we have at least 1 sponsor conversation in flight.

---

## H. Things we still don't know that we don't know

These aren't questions — they're acknowledgments of unknowns we'll discover by doing:

- **Real-world Anthropic spend per typical mission.** We have estimates, not data. Phase 0 dogfooding will calibrate.
- **Railway's actual behavior at 50 services + autoscaling worker pool.** Their API may surprise us.
- **GitHub PAT vs OAuth scope friction in real onboarding.** Some users hate creating PATs; we'll see.
- **Whether the multi-tenant worker model has a hidden bottleneck** at, say, 100 concurrent missions. Probably fine, but unproven.
- **Whether Mailgun multi-domain on the existing account scales to two products of different volumes** without reputation issues.

Punt: discover, then react. Don't plan for the third surprise before encountering the first.
