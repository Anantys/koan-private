# 09 — Pricing tiers and design-partner deal

> Addresses Alexis's R1 questions on pricing (Q5, Q6, Q7) and the explicit ask in Q4: *"What rationale to sell to a customer who pays us to develop it? 10K, 50K?"*

## Part 1 — Plan tiers (production pricing)

### The math behind plan pricing

Decided in R1:

- **Hosted credits** model: Anantys holds the Anthropic key, customer never sees it.
- **Pricing baseline:** Anthropic price grid × margin multiplier + plan limits.
- **Reference:** Greptile.com pricing for shape, expand for Koan's broader scope.

### What Greptile charges (as a calibration anchor)

Greptile pricing (public, as of early 2026):

- **Hobby:** Free — 25 reviews/month, public repos only.
- **Pro:** ~$30/seat/month — 200 reviews/month, private repos.
- **Team:** ~$70/seat/month — unlimited reviews, team features.
- **Enterprise:** custom — SSO, audit logs, on-premise.

Greptile is **per seat** because their unit of consumption is "a code review event" tied to a developer. **Koan is fundamentally different**: our unit is a **mission**, often run by an autonomous agent over hours, not tied to a specific seat. Pricing structure has to reflect that.

### Recommended pricing structure (proposal — to validate)

Two axes, like AWS: **a tier (capacity envelope)** + **metered overage capacity**. With overage **disabled at MVP** (hard cap, per Q8 decision).

| | **Solo** | **Team** | **Scale** | **Enterprise** |
|---|---|---|---|---|
| **Monthly** | **$99** | **$299** | **$799** | Custom |
| **Annual (–30%)** | $69/mo billed yearly | $209/mo billed yearly | $559/mo billed yearly | Custom |
| Repositories plugged | 3 | 10 | 30 | Unlimited |
| Concurrent missions | 1 | 3 | 10 | Custom |
| Missions / month | 100 | 500 | 2000 | Unlimited |
| Recurring tasks | 5 | 20 | 100 | Unlimited |
| Anthropic token allowance | ~$30 cost / mo | ~$100 cost / mo | ~$300 cost / mo | Custom |
| Skills available | All core + 0 custom | All core + 5 custom | All core + unlimited custom | All + custom |
| Support | Community / email | Email, 48h | Email, 24h + Slack channel | Dedicated CSM |
| GitHub integration | OAuth + PAT | OAuth + PAT | OAuth + PAT, GitHub App once shipped | GitHub App + SAML |
| Audit logs | — | 30 days | 1 year | Forever + export |

Anthropic price as of early 2026 (Opus / Sonnet/-mini blend):

- Sonnet input ~$3 / Mtokens, output ~$15 / Mtokens
- Opus input ~$15 / Mtokens, output ~$75 / Mtokens

A typical Koan mission spends roughly $0.10–$1.00 of Anthropic depending on size and model. Allowances above translate to:

- Solo $99 → ~30–300 missions of token cost included → bounded by mission count (100), gives Koan a comfortable margin even on token-heavy missions.
- Team $299 → ~100–1000 missions of token cost → 500 missions limit binds first.
- Scale $799 → ~300–3000 missions → 2000 missions binds first.

**Margin model:** plan revenue covers Anthropic cost × ~3 (3× multiplier as a rule of thumb), leaving room for compute, infra, support, and product investment.

### Why these numbers and not $89 / $249 / $499

Two reasons to push slightly higher than Alexis's brief:

1. **Greptile is single-feature** (code review) at $30/seat. Koan is **autonomous agent + persistent memory + multi-repo** — far broader scope. Charging less than 3× a single-feature competitor signals "Koan is a Greptile-class product" when in fact it's "Cursor-meets-Devin-class."
2. **Anchoring against the $200 Claude Max license:** customers will mentally compare. Solo at $99 is well below — easy yes. Team at $299 = $99 × 3 with 5× the productivity = obvious upgrade. Scale at $799 captures "I'm running Koan as part of my engineering process."

These are still **proposals**. The hard validation comes from selling to 10 prospects in Sprint 2–4.

### Plan limit knobs (config-driven from day 1)

Per Alexis's request (Q26): make limits **configurable** so we can re-tier without code changes. Proposed `koan_plan` table:

```
id, name, monthly_price_usd, annual_price_usd,
max_repos, max_concurrent_missions, max_missions_per_month,
max_recurring_tasks, anthropic_budget_usd_per_month,
allowed_skills_csv, support_tier, audit_log_days
```

Adding a new tier or moving a limit = SQL update. No deploy.

---

## Part 2 — Design-partner ("Founding Customer") deal

Alexis flagged in Q4 that a customer may want to **sponsor development**. Question: what's the rationale? what's the price?

### The framing — what we sell them, in plain words

> *"For 12 months, you get an unlimited Koan Cloud instance, dedicated infrastructure, a direct Slack channel with the founders, and a seat on our roadmap. You're not buying a finished product — you're buying influence over how the product is built. After GA, you keep 50 % off forever."*

This is a **sponsorship** disguised as a contract — and that's fine. Both sides know what they're signing up for. Standard practice for early B2B SaaS (HashiCorp, Vercel, Linear, Greptile all did this).

### What the design partner gets (the deliverables list)

1. **Free unlimited Koan Cloud access** for 12 months — no plan limits, no token cap.
2. **Dedicated workers** — their missions never wait in a shared queue. They get their own Railway worker pool.
3. **White-glove onboarding** — a 90-minute call with Alexis, hands-on integration, custom mission templates for their stack.
4. **Direct founders' line** — private Slack/Discord channel; SLA: <2h response during business hours.
5. **Roadmap influence** — quarterly check-ins where they prioritize 2 features for the next quarter (within reason; they don't get to dictate, but they get a vote).
6. **"Founding Customer" badge** in marketing once we go public (optional, with their consent).
7. **50 % lifetime discount** on whichever plan they pick after the 12-month sponsorship ends. Forever.
8. **Right-of-first-refusal** on enterprise features (SSO, on-prem, custom skills) — they get them before general availability if asked.
9. **Money-back guarantee** if MVP isn't shipped by month 4. Not "satisfaction guarantee" — "we didn't ship" guarantee. This signals confidence.

### What we ask for in return

1. **Money** (see pricing below).
2. **Real usage** — they actually run Koan on their codebase, give us real-world signal. No vanity logo deals.
3. **Weekly 30-min sync** for the first 8 weeks — we need feedback fast during MVP build.
4. **Public testimonial / case study** at GA — quotable, with their name and logo, post-launch.
5. **Bug-report priority** — they tag issues "founding customer" and we treat them as P1.

### Pricing the deal

The right number depends on **what kind of company**. Three tiers of sponsor:

| Sponsor profile | What they look like | Price (12 mo, paid up front) |
|---|---|---|
| **Friend / individual** | Solo dev or pre-seed founder, wants to support, has limited budget | **$5 000 – $10 000** |
| **Funded startup** (Seed / Series A) | 10–50 engineers, raised < $20M, willing to sponsor a tool that 10× their dev throughput | **$20 000 – $30 000** |
| **Scaleup** (Series B+ / profitable) | 50–500 engineers, has real engineering budget, wants direct relationship with the founders | **$40 000 – $60 000** |
| **Enterprise** (later, post-MVP) | 500+ engineers, procurement-driven, needs SSO/audit/on-prem | $100K+ — out of scope for now |

For a **first** design partner, my recommendation:

- **Aim for the "funded startup" band: $25 000 for 12 months**, paid upfront in two installments ($15K at signing, $10K at MVP delivery).
- **Floor: $15K.** Below this it's not worth the dedicated infra + founders' time.
- **Ceiling: $40K.** Above this they expect SSO/SOC2/SLAs we can't honor in MVP.

### Why $25K is defensible

- $25K / 12 months = ~$2 100 / month for an unlimited tier that retails at $799/mo. So they're paying ~3× the highest public tier — fair for "preferred treatment" + dedicated infra + roadmap influence.
- 1 senior engineer's salary for 1 month is $15K–$25K loaded. They're spending what they'd spend on hiring one engineer for a month, but they're getting **autonomous coding leverage for a year**. ROI math is easy.
- $25K covers: ~$2K/month of Anthropic at heavy use × 12 = $24K. Margin on the deal is mostly time investment (founders' attention), not infra. We don't lose money.

### Why **not** charge $50K

- We don't have SLAs, observability, multi-region, audit logs, SSO. If we charge enterprise prices, we owe enterprise features. We don't have them.
- $50K signals "you're our priority" — and we'll have N other customers too. Better to underpromise on price and overdeliver on attention.

### Why **not** charge $5K

- Cheap deals attract people who want a discount, not partnership. Self-selection matters.
- The first design partner sets the floor for every deal after. $5K means no one ever pays >$5K in this segment.

### What this deal is NOT

- ❌ A free trial. Trials don't pay attention. Cash on the table = aligned incentives.
- ❌ A consulting contract. We're not building features for them; we're building a product they get to influence.
- ❌ A revenue share. Don't muddy the model.
- ❌ Equity for service. Ever. Stay clean.

### When to sign one

**Now.** A signed founding customer contract:

- **Validates** the offering before we've shipped — the strongest possible PMF signal at this stage.
- **De-risks the timeline** — the customer has skin in the game, they push us to actually ship.
- **Funds part of MVP** — $25K covers significant Anthropic + Railway burn during dev.
- **Provides real-world usage data** that shapes Phase 0 / Phase 1 better than dogfooding alone.

Our advice: pitch the first sponsor in **Sprint 1**, ideally before any other engineering work. The conversation itself sharpens the spec.

### Suggested pitch script (rough)

> *"We're building an autonomous coding agent that runs on your repos 24/7. Picks issues from your backlog, writes the code, opens draft PRs you review. Persistent memory of your codebase. Greptile-meets-Devin scope.*
>
> *MVP ships in 3–4 months. We're picking 1–3 founding customers to sponsor the build at $25K for 12 months. You get unlimited usage, dedicated infrastructure, weekly syncs with the founders, and a vote on roadmap. After GA, 50 % off forever.*
>
> *We'll give you a money-back guarantee if we don't ship by [date]. Want a 30-min call?"*

Two paragraphs. Names what they get. Offers a way out. Asks for the call.

### Numbers to watch

If we land **3 founding customers at $25K** = $75K in cash before MVP ships. That funds 3 months of full-time engineering for 2 people + infra. Not nothing.

If we land **1 sponsor at $25K**: validation + ~1 month of runway. Worth it.

If we land **0 sponsors**: a strong signal that the pitch needs sharpening before we keep building. Better to find out now than after MVP.
