# M10.D Tool Health — Stage 1 Lock-Record (light Mode-2 compressed per DQ4)

**Status:** Stage 1 locks ratified; Stage 3 **single-C1 forthcoming** (C1+C2 fallback if aggregation earns its own test surface); **NO C0 — no ADR, no migration, Redis-read-only.** The **FINAL M10 sub-milestone** — closing it closes M10 (Report Architecture) entirely. Fourth and last M10 sub-milestone; the leanest shape in the arc. 66 cumulative drift preserved (0 catalogued so far).

**Date:** 2026-07-31.

**Authority:** M10 decomposition `41ad3e6` (M10.D = Task 10.6; minimal per DQ4; DQ5 final trigger); M10.D entry probe (minimality **CONFIRMED** — the health signal EXISTS: engine workers publish `shieldscan:workers:{worker_id}` via SETEX 60s TTL / 20s refresh; SPEC line 567 `GET /orgs/:org_id/tools/health` org-scoped; distinct from `/ready`; no migration, no ADR; api-side first-of-kind = `redis.scan_iter` over `shieldscan:workers:*`, no new dep); M10.D Mode-2 brainstorming CLOSED (Gate-1 / 5 locks); M10.A + M10.C C0-less lock-record precedent. 66 cumulative drift discipline; 12-instance averted-prediction lineage; 2-instance ADR-mislabel lesson; 3-instance date-stamp lesson.

---

## 1. Gate-Lock Table (1 gate / 5 locks)

### Gate 1 — Semantics + Endpoint
- **G1.1 TWO-STATE availability (honest):** per tool → `available` (≥1 **live** worker advertises it via the heartbeat `engines` list) / `unavailable` (none). **NO fabricated "degraded"** — the heartbeat carries worker-liveness + engines-list, *not* per-tool health; a heuristic "degraded" would invent a state the signal cannot support (the direct echo of M10.C's `not_assessed` honesty). Degraded forward-pinned to a real per-tool health signal (e.g. engine-emitted error rates).
- **G1.2 RESPONSE SHAPE:** worker list (`worker_id` / `hostname` / last-seen `registered_at` / advertised `engines`) **+** derived per-tool availability roll-up (each engine → `available`/`unavailable` + live-worker count).
- **G1.3 LIVENESS = TTL-presence:** a worker is live iff its key is present within the SETEX 60s TTL (crash → key expires ≤60s; **the TTL IS the liveness contract**). Surface `registered_at`/last-seen for observability; **no second staleness threshold** beyond the TTL.
- **G1.4 ORG-SCOPED PATH, GLOBAL DATA:** `GET /orgs/{org_id}/tools/health`, membership-gated. The worker fleet is **GLOBAL** (workers serve all tenants) — org-scoping is **auth-gating, not data-partitioning** (the same global-behind-org-path shape as M10.C compliance reference data).
- **G1.5 UNGATED + NO-C0 + SINGLE-C1:** ships ungated (tier-enforcement forward-pinned to billing — **4th instance** after M10.A /fix + M10.B reports + M10.C compliance); no C0 (no ADR, no migration); single compressed C1 (C1+C2 fallback only if the aggregation earns its own test surface).

---

## 2. Architecture

A **read-and-aggregate** endpoint over the **existing** engine heartbeat signal (`shieldscan:workers:*`) — **NO engine changes, NO signal written.** The api reads via `redis.scan_iter("shieldscan:workers:*")` + per-key GET + JSON-parse, then aggregates.

**The probe's structural finding (M10.D's scope-delta):** the heartbeat publishes **worker-liveness + engines-list**, NOT a native `healthy/degraded/unhealthy` three-state. So per-tool status is **DERIVED** from fleet liveness (a tool is available if any live worker advertises it), and the plan-literal's three-state is honestly collapsed to two-state (G1.1) rather than fabricating the missing middle — each M10 sub-milestone surfaced one such finding (M10.A 7-vs-4, M10.B 5-vs-4 + dual-mechanism, M10.C 3-vs-2 + global-data; M10.D = signal-shape-vs-three-state).

**Distinct from `/ready` (no duplication):** `/ready` (`main.py`) is the **API pod's own** infra liveness (postgres+redis dep probes, k8s 503 gate); `/tools/health` is the customer/ops view of the **engine worker fleet's** per-tool availability. Different subjects, different sources.

---

## 3. Signal Contract (source: engine `internal/worker/heartbeat.go`)

- **Key:** `shieldscan:workers:{worker_id}` (const `workerKeyPrefix = "shieldscan:workers:"`).
- **Write:** `SETEX`, **60s TTL, refreshed every 20s** (`DefaultHeartbeatTTL` / `DefaultHeartbeatInterval`; 3× safety margin). **TTL-only expiry** — no `Del` on shutdown; crash and clean-shutdown are symmetric (key gone ≤60s).
- **Payload (JSON):** `{ hostname, started_at, concurrency, engines, worker_id, registered_at }` — `registered_at` is RFC3339Nano (sub-second liveness). `engines` = `registry.Engines()`, the list of tools/engines a live worker advertises — **the per-tool roll-up key source** (G1.2).

---

## 4. Endpoint (1 org-scoped GET; ungated; no audit per V-JJH)

- `GET /orgs/{org_id}/tools/health` — `require_org_membership()` + `get_redis`; returns the worker list + derived per-tool availability roll-up (G1.2). No audit rows (GET — V-JJH precedent).

---

## 5. Stage 3 (no-C0; single compressed C1)

- **C1** — `routes/tools.py` (the endpoint) + a thin aggregation helper (inline unless it earns `services/tool_health.py`) + `tests/routes/test_tools.py` (fixture seeds fake `shieldscan:workers:*` keys; asserts two-state roll-up + live-worker counts + TTL-expiry → unavailable + membership 404). ~120-180 LoC. Then **P5.A ×3** (docs annotations + api DRIFT-LOG + engine cross-ref). Closes M10 entirely.

---

## 6. Forward-Pins

- `degraded` per-tool state → a **real per-tool health signal** (engine-emitted error rates / probe results), not a heuristic over liveness.
- Tier-gating → billing milestone (**4th instance**: /fix + reports + compliance + tool-health).
- Drain-status (`worker:$id` HSET `status "draining"`, CLAUDE.md Gotcha 8) → fold into availability **only if a live written signal** (confirm at C1 per V-MMD); else forward-pin.

---

## 7. V-MM Pre-C1 DEFERRED-EMPIRICAL Grounding List

- **V-MMA:** exact heartbeat key + payload JSON field names from engine `internal/worker/heartbeat.go` (`WriteOnce` payload) — ground the api-side parse on the **real** field names, not this doc's paraphrase.
- **V-MMB:** `redis.scan_iter` idiom + the `get_redis` dependency shape (this is the **first `scan_iter` in the api** — confirm redis-py async client exposes it; per-key GET vs MGET).
- **V-MMC:** `require_org_membership()` + `APIRouter(prefix="/orgs/{org_id}/...")` shape from `routes/compliance.py` + `routes/reports.py`.
- **V-MMD [KEY — decides whether availability considers drain]:** the drain-key `worker:$id` HSET `status "draining"` — is it a **LIVE written signal** (grep engine for the HSET) or aspirational/ops-runbook-only? Fold into availability only if live; else forward-pin.
- **V-MME:** the canonical `engines`-list values (the ~9 engine identifiers from `registry.Engines()`) — the roll-up keys for the per-tool availability map.
