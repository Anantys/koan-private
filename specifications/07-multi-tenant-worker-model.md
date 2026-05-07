# 07 — Multi-tenant worker model

> Replaces the "1 Railway service per customer" model from [03 — Tenant runtime (v0.1)](03-tenant-runtime.md).
> Driven by Alexis's R1 decision on Q17.

## The problem with 1 service per customer

- Linear cost floor: 100 customers × ~$10/mo Railway floor = $1 000/mo before any usage.
- Operational burden scales linearly: 100 services to monitor, redeploy, update.
- Idle tenants (sleep, weekends, vacations) burn the cost floor for zero value.
- Railway API rate limits become a real concern at scale.
- Forces us to charge a high MRR floor just to absorb infrastructure waste.

## The model

A **stateless Koan worker pool** processes missions for many tenants out of a **shared queue**, with strong **per-mission context isolation**.

```
┌────────────────────────────────────────────────────────────────┐
│  Control plane  (in koan-cloud backend repo)                   │
│  ──────────────                                                │
│  • Tenant registry (MySQL)                                     │
│  • Mission inbox (per-tenant)                                  │
│  • Stripe billing, GitHub OAuth, dashboard API, mail           │
│  • Schedules recurring missions                                │
│  • Enqueues missions into the shared queue                     │
└──────────────────────┬─────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────┐
│  Mission queue (Redis Streams or BullMQ on existing Redis)     │
│  ───────────────────────────────────────                       │
│  Item = {tenant_id, mission_id, priority, attempt, lease_at}   │
│  Visibility timeout for crash safety.                          │
└──────────────────────┬─────────────────────────────────────────┘
                       │ workers poll/claim
                       ▼
┌────────────────────────────────────────────────────────────────┐
│  Koan worker pool (Railway service "koan-worker", N replicas)  │
│  ─────────────────────────                                     │
│  Per claimed mission:                                          │
│    1. Hydrate tenant state (MySQL + S3) into KOAN_ROOT=/tmp/X  │
│    2. Spawn isolated subprocess (clean env, scoped tokens)     │
│    3. Clone repo into ephemeral workspace                      │
│    4. Run Koan agent loop for THIS ONE MISSION (1-shot)        │
│    5. Persist new state back to MySQL + S3                     │
│    6. Tear down /tmp/X completely                              │
│    7. Loop                                                     │
└────────────────────────────────────────────────────────────────┘
```

## Isolation guarantees

| Layer | Mechanism | Guarantee |
|---|---|---|
| Process | Each mission spawns a fresh `python -m koan.app.cloud_worker --mission <id>` subprocess | Memory cannot leak between missions |
| Filesystem | `KOAN_ROOT=/tmp/koan-{mission_id}/`, deleted after the mission | No cross-mission file leakage |
| Repo workspace | Cloned per mission into `/tmp/koan-{mission_id}/repo`, never reused | No git-state contamination |
| Network — GitHub | Each mission gets the **tenant's own** GitHub token, only its repos accessible | Cross-tenant API isolation |
| Network — Anthropic | Single master key, but every API call carries `metadata.user_id = tenant_id` | Tenant-attributed billing + AUP audit |
| Memory / learnings | Loaded from MySQL/S3 at mission start, written back at end | Persistent across missions but scoped per tenant |
| Logs | Tagged with `tenant_id`, `mission_id` at every line | Per-tenant log grep, never mixed |

**Crucially:** the worker process itself is **shared** across tenants over its lifetime, but at any instant it is processing **one tenant's mission with that tenant's data and tokens only**. Between missions, the worker holds no tenant state.

This is the same isolation model as a stateless web request handler in a Flask gunicorn worker — Anantys Invest already operates on this pattern at scale.

## Tenant state — where does it live?

Today (self-hosted) Koan stores everything as **files** under `KOAN_ROOT/instance/`:

| Today (file-based) | In cloud (DB-backed) | Why |
|---|---|---|
| `instance/missions.md` | `koan_mission` table (status, body, project, assignee) | Queryable from dashboard, durable across worker churn |
| `instance/journal/YYYY-MM-DD/{project}.md` | `koan_journal_entry` table | Same |
| `instance/memory/projects/{name}/learnings.md` | `koan_memory` table (TEXT) or S3 blob | Per-tenant, must persist between missions |
| `instance/config.yaml`, `projects.yaml`, `soul.md` | `koan_tenant_config` table | Edited from dashboard, atomic updates |
| `instance/.koan-status`, `.koan-pause`, `.koan-focus` | `koan_tenant_runtime` table | Real-time status the dashboard polls |
| Repo workspace clone | Ephemeral `/tmp` per mission | No persistence needed |

**Recommendation:** introduce a **state backend abstraction** in `koan/`:

```python
# koan/app/state_backend.py (NEW)
class StateBackend(Protocol):
    def read_missions(self) -> str: ...
    def write_missions(self, content: str) -> None: ...
    def append_journal(self, project: str, entry: str) -> None: ...
    def read_memory(self, project: str) -> str: ...
    def write_memory(self, project: str, content: str) -> None: ...
    # ... etc

class FileBackend(StateBackend):       # OSS / self-hosted — today's behavior
    ...

class CloudBackend(StateBackend):      # MySQL + S3
    ...
```

Selected by `KOAN_STATE_BACKEND=files|cloud` env var. The OSS Koan keeps file behavior; cloud workers use the DB backend. **No fork**, just dependency injection.

## Mission lifecycle in cloud mode

```
1. Trigger: dashboard POST, GitHub @mention webhook, or scheduled cron
   ↓
2. Control plane creates row in koan_mission (status=pending) + enqueues to Redis
   ↓
3. Worker claims (visibility timeout 30 min, configurable)
   ↓
4. Worker spawns subprocess:
       python -m koan.app.cloud_worker
           --tenant {id} --mission {id} --provider claude
   ↓
5. Subprocess:
   • Pulls tenant config + memory + projects.yaml from CloudBackend
   • Clones repos (shallow, only the projects this mission touches)
   • Runs ONE iteration of the agent loop (mission_runner.py invocation)
   • Writes journal + updated memory + mission status back via CloudBackend
   • Exits 0 (success) or non-zero (retry / fail)
   ↓
6. Control plane updates Stripe usage, dashboard reflects new state
```

**Key shift from self-hosted:** today Koan loops *forever*, picks the next mission, contemplates between missions, runs cron-like recurring tasks itself. In cloud mode, the **scheduling is owned by the control plane**, not the worker. The worker is a one-shot mission executor. Recurring missions, contemplative sessions, focus mode timers — all become control-plane scheduled jobs that enqueue concrete missions.

## Resource model & scaling

- **Workers:** start with 2 Railway replicas, autoscale on queue depth (`koan-worker` Railway service). Each replica handles 1–N concurrent subprocesses based on Railway plan size.
- **Per-mission cost:** ~Anthropic tokens + a few seconds CPU + clone bandwidth. No 24/7 idle cost.
- **Cost floor at 1000 customers:** ~3–5 workers × Railway plan + DB + Redis + S3 = **maybe $200/mo total infra** vs $10 000/mo with 1-per-customer model.
- **Concurrency bounds per tenant:** `koan_mission` table enforces "max N concurrent missions for tenant X" at claim time (e.g., Starter: 1 mission, Scale: 5 concurrent). Prevents runaway tenants from starving others.

## What about the chat / interactive surface?

Multi-tenant workers are great for autonomous async missions. They are **not great** for interactive chat, where each user keystroke can't trigger a fresh subprocess clone-repo cycle.

Three options for chat (relevant only post-MVP since chat is MVP+1):

1. **Sticky interactive worker pool** — when the user opens chat, an interactive worker is reserved for them, repo + memory loaded, idle timeout 5 min. Released when chat closes.
2. **Full-context streaming** — chat sends the full tenant context to Anthropic each turn (no on-disk repo, just files served via tools). Simpler but more token-expensive.
3. **Chat = "queue this question as a mini-mission"** — every chat message becomes a mission, processed by the same worker pool. Higher latency (5–30 s per message) but no special infra.

Recommendation: option 3 for early chat (post-MVP), option 1 only if latency complaints emerge. We don't decide this now.

## What about repo size?

Big repos (>100 MB) are slow to clone every mission. Mitigations available **without** breaking the model:

- **Shallow clones** (`--depth 1`) for read-only missions — fast, ~1–10 s for typical repos.
- **Per-tenant repo cache** on the worker: a local clone kept warm, fetched + reset between missions. Sticky-worker placement ("tenant X usually goes to worker N") via consistent hashing of `tenant_id`.
- **S3 repo snapshots** if cloning becomes the bottleneck.

These are optimizations to apply *if needed*, not upfront work.

## Failure & retry

- Worker dies mid-mission: visibility timeout expires, mission goes back to queue, another worker claims it.
- Mission idempotency: each mission has an `attempt` counter; if attempt > 3, mark failed, alert control plane, surface in dashboard.
- Tenant data corruption: every state write is atomic (MySQL transactions, S3 overwrite-with-version). Daily DB snapshots restore the worst case.

## What changes in the OSS `koan-private` repo

Net-new modules:

- `koan/app/state_backend.py` — protocol + `FileBackend` extracted from current behavior + `CloudBackend` (MySQL/S3 implementation)
- `koan/app/cloud_worker.py` — the one-shot mission executor entrypoint
- DB migration scripts (Alembic — see Q in [10](10-open-questions-round-2.md))

Modified modules (refactor to use `state_backend` instead of direct file I/O):

- `missions.py`, `memory_manager.py`, `journal/*`, `pause_manager.py`, `focus_manager.py`, `passive_manager.py`, `usage_tracker.py`, hooks runner

The agent loop (`run.py`, `mission_runner.py`, `iteration_manager.py`, prompts, skills) stays untouched — it sees the state backend through the same API.

OSS users continue with `KOAN_STATE_BACKEND=files` (default) and notice nothing.

## Trade-offs we accept

| Trade-off | Why we accept it |
|---|---|
| Loss of "single Koan process per customer feels" — agent doesn't run continuously between missions | We gain 50× cost efficiency. Async-mission model still feels alive to the customer (missions run, journal updates, PRs appear). |
| Cold-start cost on every mission (load state, clone repo) | A 5–30 s overhead is fine for autonomous missions that take minutes/hours anyway. |
| State migration to DB requires touching many Koan core modules | Lift the state-backend abstraction now and we get clean separation forever. |
| Loss of "I can ssh into my Railway service" debuggability | Workers are stateless and ephemeral. We invest in centralized log aggregation and a dashboard "see what worker N is doing right now" view. |
| One Anthropic key for everyone → noisy-neighbor risk on rate limits | Mitigation: per-tenant request budget enforced before the API call (don't queue 100 concurrent missions for one tenant). Anthropic also offers per-key rate-limit increases for paying business customers. |

## Trade-offs we explicitly do NOT accept

- ❌ Sharing **agent process memory** across tenants — never. The subprocess boundary is non-negotiable.
- ❌ Sharing **GitHub tokens** across tenants — never. Each subprocess only sees its tenant's token in env.
- ❌ Sharing **the agent's local Claude Code CLI session state** across tenants — fresh CLI invocation per mission.
- ❌ Sharing **a single MySQL row** across tenants — every state row carries `tenant_id`, every query filters by it, RBAC at the data layer.

## What's in scope for Sprint 1 vs later

- **Sprint 1 (Phase 0 POC):** prove a single mission can be run by a "fake worker" subprocess against tenant state in MySQL — even if MySQL is just a Docker container, even if "scheduling" is a manual `python -m ...` call. We're validating the architecture, not shipping it.
- **Phase 1:** real state backend, real worker pool, real queue, ship one tenant end-to-end.
- **Phase 2+:** multi-tenant for real, with the control plane orchestrating, dashboard surfacing.

See [10 — Open questions round 2](10-open-questions-round-2.md) for what's blocking Sprint 1.
