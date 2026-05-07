# Koan Cloud — Specifications

Architecture and roadmap documents for **Koan Cloud**, the managed SaaS offering of Kōan.

These docs are the working source of truth for Alexis & Nicolas. They evolve as decisions land — each doc is versioned (`v0.1`, `v0.2`…) when its content materially changes.

## Reading order

### Round 2 docs (current — read these first)

| # | Document | Purpose | Audience |
|---|----------|---------|----------|
| 06 | [R1 decisions locked](06-r1-decisions-locked.md) | TL;DR of every decision Alexis committed in Round 1. The new source of truth. | Both |
| 07 | [Multi-tenant worker model](07-multi-tenant-worker-model.md) | Replaces 1-Railway-per-customer with a stateless worker pool + per-mission isolation. State backend abstraction in `koan-private`. | Nicolas (lead) |
| 08 | [Platformization and repo layout](08-platformization-and-repos.md) | Anantys Stack as a SaaS toolkit. Koan Cloud as a separate repo with its own DB, auth, and user base. | Alexis (lead) |
| 09 | [Pricing tiers and design-partner deal](09-pricing-and-design-partner.md) | Plan structure (Solo $99 / Team $299 / Scale $799), Anthropic margin model, and a concrete $25K founding-customer deal proposal. | Both |
| 10 | [Open questions, round 2](10-open-questions-round-2.md) | Follow-up questions opened by R1 decisions + revised Sprint 1 plan. | Alexis to answer |

### Round 1 docs (historical — superseded for some content)

| # | Document | Status |
|---|----------|--------|
| 00 | [Vision and Epic #1](00-vision-and-epic-1.md) | ✅ Still accurate — the product vision is unchanged |
| 01 | [Architecture eagle view](01-architecture-eagle-view.md) | ⚠️ Partially superseded — see 07 + 08 |
| 02 | [Anantys stack reuse](02-anantys-stack-reuse.md) | ⚠️ Partially superseded — see 08 (no monolith extension) |
| 03 | [Tenant runtime](03-tenant-runtime.md) | ⚠️ Superseded — see 07 (no 1-service-per-customer) |
| 04 | [Roadmap and Sprint 1](04-roadmap-and-sprint-1.md) | ⚠️ Phases still hold; Sprint 1 plan revised in [10 §G](10-open-questions-round-2.md) |
| 05 | [Open questions, round 1](05-open-questions.md) | ✅ Closed — all answered. Frozen for archeology. |

The R1 docs each carry a banner at the top noting what's superseded and where to read the current direction.

## Related GitHub issues

- **[#1 — RFC: Koan Cloud SaaS architecture and roadmap](https://github.com/Anantys/koan-private/issues/1)** — Original strategy doc.
- **[#2 — Koan Cloud Dashboard v1](https://github.com/Anantys/koan-private/issues/2)** — Dashboard sub-issue.

## Status

🟡 **Draft v0.2** — written 2026-05-07. Awaiting Alexis's answers to [Round 2 open questions](10-open-questions-round-2.md) before promoting to v1 and locking Sprint 1.

## Quick architectural snapshot (one paragraph)

Koan Cloud is a separate product from Anantys Invest, in its own repo (`koan-cloud`), with its own DB and user base. It consumes shared Anantys infrastructure (mail, billing) via a thin internal API surface (`/anantys-stack/*`) on the existing `anantys-back` Flask app. It schedules autonomous coding missions onto a stateless **multi-tenant Kōan worker pool** (Railway service, N replicas) that pulls from a shared Redis queue. Each mission runs in an isolated subprocess with its tenant's tokens and a fresh ephemeral workspace. State (missions, journal, memory) lives in MySQL via a new `StateBackend` abstraction in `koan-private` that lets self-hosted Koan keep its file-based behavior unchanged. Customers see a Greptile-style Next.js dashboard, no Telegram, no Anthropic key juggling — pricing aligns to Anthropic's grid × ~3 margin with hard-cap overage. Chat is post-MVP; MVP UX = dashboard + GitHub @mentions.
