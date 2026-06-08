# Task 8.3α — AttackSurface Persistence Consumer: Design

**Status:** Brainstorming chain 12-Q-locks complete; pre-verification grounded across two sessions (T83_PV + T83A_PV); ready for Stage 2 plan landing + Stage 3 cross-repo (docs → engine → api) implementation.

**Date:** 2026-05-30.

**Authority:** T83_PV pre-verification surface report (this session; V-LB-V-LI; Drift #58 candidate surfaced at V-LI MISSING persistence path) + T83A_PV pre-verification surface report (this session; V-MA-V-MI; reframed Drift #58 as two-layer manifestation per V-MD wire-shape gap) + Q-chain 12 locks this session: Y-CONSUMER-LOCATION (α) extend-completions-channel + Y-WIRE-SHAPE (a) new `EventAttackSurface` + Q-ENGINE-EMIT-CALLSITE (a) RunRecon-internal + Q-EVENT-PAYLOAD-FIELDS rich subdomain rows + Q-UPSERT-PATTERN `ON CONFLICT DO UPDATE` on `uq_scan_subdomain` + Q-LIFECYCLE FastAPI lifespan via `completions_consumer` extension + Q-RLS-GUC org-scoped SET + Q-TEST-PATTERN fakeredis + Q-MULTI-PROCESS-POSTURE (b) UPSERT-idempotency-by-construction + Q-DOCS-LOCATION (c) dual TOOL-ARCH §8 + SPEC §7 + Q-PHASE-0-V2 NO + Q-COMMITS 3-commit (docs → engine → api). SPEC §13 ADR-013 + ADR-014 + ADR-017; source-ingestion fix design doc structural precedent `90fc933`; R2 design doc precedent `b25e9ba`; revocation design doc precedent `0e55a4f`; existing `completions_consumer.py` V-MB architectural analog (601 LoC).

**Related:** Implementation plan landing trigger phrase: ***"Begin Task 8.3α implementation plan landing"*** (after this design doc lands).

---

## 1. Authority + 12-Q-Chain Summary

**Y-CONSUMER-LOCATION (locked by V-MC + V-MG ADR-014 constraints) (α) extend completions Pub/Sub channel:** per-scan progress Streams subscription is unworkable for a persistence consumer without ADR-018 consumer-groups (forward-pinned; not yet built); the completions Pub/Sub is the ADR-014-compliant channel for cross-scan persistence-targeted events.

**Y-WIRE-SHAPE (locked by separation-of-concerns) (a) new `EventAttackSurface` event type:** the existing `EventReconCompleted` carries progress-stream SSE semantics (aggregate counts); the new dedicated event carries persistence semantics on the completions channel with rich subdomain rows. Multi-purpose-payload rejected.

**Brainstorming Q-chain (12 locks; high confidence per pre-verification grounded reality):**

- **Q-ENGINE-EMIT-CALLSITE (a) RunRecon-internal** — consolidates the rich-data publish with recon-completion semantics; emits to the completions Pub/Sub channel after `EventReconCompleted` fires on the existing per-scan progress stream
- **Q-EVENT-PAYLOAD-FIELDS** — `{scan_id, organization_id, root_domain, subdomains: [{url, status, status_code, tech_stack, last_probed_at}]}`; complete ORM-mapping per V-LC `AttackSurface` column set; `LiveHost` rich shape (V-MD) becomes wire-borne
- **Q-UPSERT-PATTERN** — `INSERT … ON CONFLICT (scan_id, subdomain) DO UPDATE` on rich fields per `uq_scan_subdomain` unique constraint; mirrors `routes/projects.py:519` credentials UPSERT precedent (V-ME)
- **Q-LIFECYCLE** — FastAPI lifespan integration via `completions_consumer.py` `_handle()` routing extension; no new process; inherits V-MF deployment shape
- **Q-RLS-GUC** — `SET app.current_org_id` GUC from `event.organization_id` pre-write per V-MB existing pattern
- **Q-TEST-PATTERN** — fakeredis + direct `_handle` invocation per V-MH `test_completions_consumer.py` precedent
- **Q-MULTI-PROCESS-POSTURE (b) UPSERT-idempotency-by-construction** — `uq_scan_subdomain` semantics make duplicate-event processing inherently idempotent (N>1 uvicorn workers safe by construction); cleaner than completions_consumer's "honest acknowledgment + ADR-018 forward-pin" inheritance
- **Q-DOCS-LOCATION (c) dual addendums TOOL-ARCH §8 + SPEC §7** — TOOL-ARCH §8 documents engine emission (tool-architecture territory); SPEC §7 documents wire shape (API/wire territory); full canonical authority preservation
- **Q-PHASE-0-V2 NO** — pre-verification fully grounded all empirical concerns (completions_consumer architecture + event wire shapes + UPSERT pattern + test pattern); no execution-time-unknown empirical gaps remaining
- **Q-COMMITS 3-commit cross-repo (docs → engine → api)** — event-emits-before-consumer-test ordering; engine emission lands first so api consumer tests can verify against the real payload shape

**Forward-pin chain closure:** Drift #58 (two-layer manifestation: orphaned `AttackSurface` table + orphaned engine wire shape) operationally settles with Stage 3 implementation. **Layer A** gap (engine emission shape) repaired by Commit 2 (`EventAttackSurface` emit); **Layer B** gap (api consumer absent) repaired by Commit 3 (`completions_consumer` extension).

**Out of 8.3α scope:** Task 8.3β GET endpoint (next task; mechanical compressed-lifecycle on now-populated `AttackSurface` rows); M8.1 recon-invocation (arc-evolution-pivot territory; brainstorming forward-pinned to a fresh session; `AttackSurface` rows stay empty operationally until M8.1 lands; 8.3α emission-side ready for when M8.1 lands); ADR-018 consumer-groups (forward-pinned per ADR-014 architectural decision territory); per-scan progress-Stream subscription consumers (unworkable today without ADR-018; deferred).

## 2. Pre-Verification Findings (V-LB-V-LI + V-MA-V-MI)

**Task 8.3 acceptance criteria (V-LB):** SPEC §6 line 633-654 + IMPLEMENTATION-PLAN.md Task 8.3 Step 1+2; GET endpoint returns aggregated `{root_domain, total_discovered, live, dead, subdomains: [{url, status, status_code, tech_stack, vulnerability_count, last_probed_at}]}`; aggregation logic joins `AttackSurface` + vulnerabilities for `vulnerability_count` (8.3β concern; 8.3α populates the rows).

**AttackSurface ORM (V-LC):** per-subdomain rows; columns `id`/`scan_id`/`root_domain`/`subdomain`/`full_url`/`status`/`status_code`/`tech_stack`/`last_probed_at`; unique constraint `uq_scan_subdomain (scan_id, subdomain)`; `SubdomainStatus` enum `LIVE`/`DEAD`/`TIMEOUT`; `TimestampMixin` + `TenantMixin` (RLS).

**`completions_consumer.py` architecture (V-MB):** 601 LoC; channel `shieldscan:completions` Pub/Sub; lifespan-integrated background `asyncio.create_task`; per-event session via `session_factory`; SET `app.current_org_id` GUC pre-write; single-purpose dispatcher on `event_type`; ADR-013 sole-writer + ADR-017 sequencing-locked.

**Stream-vs-Pub/Sub topology (V-MC + V-MG ADR-014):** per-scan `shieldscan:progress:{scan_id}` = Streams (SSE-replay-required); `shieldscan:completions` = Pub/Sub (broadcast fire-and-forget); ADR-018 Streams+consumer-groups forward-pinned for future migration. Per-scan-stream subscription unworkable for a persistence consumer today; completions Pub/Sub is the ADR-014-compliant channel for AttackSurface persistence.

**Drift #58 wire-shape gap (V-MD):** engine emits aggregate-only payloads (`EventLivenessProbed: {count}` + `EventReconCompleted: {subdomain_count, live_count, status}`); `ReconResult.LiveHosts` rich shape (`URL` + `StatusCode` + `Tech` + `Title` + `Webserver` + `ContentType`) is **never** published — only returned to the caller (test-only). `AttackSurface` column population requires NEW engine wire-shape extension (Layer A of Drift #58); api consumer absence is Layer B.

**AttackSurface UPSERT convention (V-ME):** zero existing writes; closest precedent at `routes/projects.py:519` PATCH credentials UPSERT against `uq_project_credentials_project_id`; `AttackSurface` uses `uq_scan_subdomain` for the same `INSERT … ON CONFLICT DO UPDATE` pattern.

**Service lifecycle (V-MF):** `main.py` `@asynccontextmanager lifespan` — Redis init + `CompletionsConsumer` construction + `start()` spawn + shutdown drain. 8.3α inherits identical pattern (no new process; consumer extension lives in `completions_consumer.py`).

**Test pattern (V-MH):** fakeredis + extracted `_handle(event)` direct invocation; 6+ scenario tests; per-event `AttackSurface` assertion fixtures.

**Forward-pin chain (V-MI):** zero prior references to AttackSurface consumer; greenfield territory; no architectural-decision-deferred-here.

**Drift #58 catch-class:** Same as Drift #54 (stored-design-intent-with-unimplemented-mechanism). Manifestation difference: Drift #54 was orphaned-column; Drift #58 is orphaned-table-PLUS-orphaned-wire-shape (two-layer manifestation; single cataloging entry per consolidated catch-class discipline). Cumulative count: 57 → **58**.

## 3. Architectural Decisions

Cross-references Q-chain locks (§1) + pre-verification findings (§2).

### 3.1 Y-CONSUMER-LOCATION (α) Extend Completions Pub/Sub Channel

**Rationale:** ADR-014 mixed-primitive lock canonically separates Pub/Sub (broadcast) from Streams (per-scan replay). Persistence-targeted cross-scan events fit the Pub/Sub completion channel pattern (existing precedent: `completions_consumer` drains `job_completed` + `partial_findings` events). Per-scan-stream subscription consumers are unworkable today (ADR-018 forward-pinned) and would deny the canonical completions-channel-for-persistence pattern.

**Rejected:** (β) NEW `services/attack_surface_consumer.py` subscribing to per-scan progress Streams — requires ADR-018 consumer-groups; subscription-discovery problem (which scans, when to start, how to clean up); architectural-decision-deferred-here. (γ) Hybrid dual-emit (Streams + Pub/Sub) — ADR-014 explicitly rejected; dual-emit invites drift.

**Implementation surface:** engine publishes new `EventAttackSurface` to `shieldscan:completions`; api `completions_consumer._handle()` dispatches `event_type == "attack_surface"` → `_handle_attack_surface(event)` handler that UPSERTs `AttackSurface` rows.

### 3.2 Y-WIRE-SHAPE (a) New `EventAttackSurface` Event Type

**Rationale:** existing `EventReconCompleted` carries progress-stream SSE semantics (aggregate counts intended for client real-time updates). Persistence-targeted rich subdomain rows have different semantics (persisted state, not transient progress). Separating event types preserves the single-purpose-event invariant; multi-purpose payloads invite drift.

**Rejected:** (b) extend existing `EventReconCompleted` with rich subdomain rows — overloads SSE-targeted event; rejects separation-of-concerns.

**Wire shape:**

```json
{
  "event_type": "attack_surface",
  "scan_id": "uuid",
  "organization_id": "uuid",
  "root_domain": "example.com",
  "subdomains": [
    {
      "url": "https://api.example.com",
      "status": "live",
      "status_code": 200,
      "tech_stack": ["nginx", "Node.js", "React"],
      "last_probed_at": "2026-05-30T14:32:00Z"
    }
  ],
  "timestamp": "2026-05-30T14:32:01Z"
}
```

### 3.3 Q-ENGINE-EMIT-CALLSITE (a) RunRecon-Internal

**Decision:** new event emit at the same callsite as the existing successful `EventReconCompleted` (recon.go after httpx phase succeeds). Consolidates the rich-data publish with recon-completion semantics; the emission carries the same `ReconResult.LiveHosts` data that's already constructed for the return value.

**Publish target:** the **completions Pub/Sub channel** (`shieldscan:completions`), distinct from the per-scan progress Stream the existing `EventReconCompleted` writes to. Requires a `CompletionsPublisher` interface (or equivalent) wired into `RunRecon`'s signature alongside the existing `ProgressPublisher`.

**Status-only events skipped:** when recon fails (subfinder failure / no_subdomains / httpx_failed), `LiveHosts` is empty; no `EventAttackSurface` emission (existing `EventReconCompleted` failure-status events on the progress stream remain unchanged).

### 3.4 Q-EVENT-PAYLOAD-FIELDS

**Decision:** rich subdomain rows mapping `LiveHost` fields → `AttackSurface` columns 1:1:

| LiveHost | AttackSurface | Wire field |
|---|---|---|
| `URL` | `full_url` (and `url` on the wire) | `url` |
| `StatusCode` | `status_code` | `status_code` |
| `StatusCode > 0` → `LIVE`; else `TIMEOUT` (derived) | `status` | `status` |
| `Tech` | `tech_stack` (JSONB) | `tech_stack` |
| event timestamp | `last_probed_at` | (consumer uses outer `timestamp`) |

`subdomain` field on `AttackSurface` is derived from `URL` host component at consumer side. `root_domain` is the recon input domain, carried at outer payload level.

### 3.5 Q-UPSERT-PATTERN

**Decision:** `INSERT … ON CONFLICT (scan_id, subdomain) DO UPDATE` per the `uq_scan_subdomain` unique constraint. Mirrors `routes/projects.py:519` PATCH credentials UPSERT precedent. Updates on conflict: `full_url`, `status`, `status_code`, `tech_stack`, `last_probed_at`, `updated_at`. Preserves `id`, `created_at`, `root_domain`.

**Atomicity:** per-event single-transaction batch — all subdomain rows in one event commit together. Failure rolls back all rows; Pub/Sub redelivery re-fires the event.

### 3.6 Q-LIFECYCLE — Inherit from `CompletionsConsumer`

**Decision:** no new process or subscriber. `completions_consumer.py` `_handle()` dispatcher extension adds the third `event_type` branch (`attack_surface`) alongside `job_completed` + `partial_findings`. The new `_handle_attack_surface(event)` handler is a method on `CompletionsConsumer`. Lifespan integration in `main.py` is untouched.

### 3.7 Q-RLS-GUC

**Decision:** `SET app.current_org_id` from `event.organization_id` before any RLS-scoped write per V-MB's existing pattern. Parameterized: `text("SET app.current_org_id = :org_id"), {"org_id": str(org_id)}`. Mirrors `identity.py`, `audit.py`, `db/policies.py` convention.

### 3.8 Q-TEST-PATTERN

**Decision:** fakeredis + direct `_handle(event)` invocation; per-scenario `AsyncSession` + Org/Scan fixture setup. Mirrors `test_completions_consumer.py` 6-scenario shape extended with:

1. Happy-path: single subdomain → AttackSurface row inserted
2. Multi-subdomain: 5+ subdomain rows in one event → all inserted
3. UPSERT idempotency: duplicate event → same rows; no errors
4. Mixed-state subdomains: `live` + `timeout` derived statuses preserved
5. RLS isolation: cross-org event consumed under wrong GUC → no rows visible to the wrong tenant
6. Malformed event: missing required fields → logged + skipped (loop survives)

### 3.9 Q-MULTI-PROCESS-POSTURE (b) UPSERT-Idempotency-by-Construction

**Decision:** `uq_scan_subdomain` semantics make duplicate-event processing inherently idempotent. N>1 uvicorn workers each consuming the same `EventAttackSurface` execute the same `ON CONFLICT DO UPDATE`; the final row state is deterministic regardless of execution order.

**Rationale:** cleaner than `completions_consumer.py`'s "honest acknowledgment + ADR-018 forward-pin" inheritance for `job_completed` events. No multi-process posture concession needed; UPSERT-on-natural-key handles it.

### 3.10 Q-DOCS-LOCATION (c) Dual TOOL-ARCH §8 + SPEC §7 Addendums

**Decision:**

- **TOOL-ARCHITECTURE.md §8 addendum:** AttackSurface Emission Lock — engine `recon.go` publishes `EventAttackSurface` to the completions channel after httpx phase; rich payload from `ReconResult.LiveHosts`.
- **SPECIFICATION.md §7 addendum:** `EventAttackSurface` wire-shape canonical — JSON schema + ADR-014 Pub/Sub channel reference + ADR-013 sole-writer continuity (api is sole writer of `AttackSurface` rows; events drive persistence).

**Rationale:** TOOL-ARCH governs engine-emission territory; SPEC §7 governs wire-shape territory. Dual addendum preserves the canonical-authority split without forcing a single-doc compromise.

### 3.11 Q-PHASE-0-V2 NO

**Decision:** no Phase 0 v2 empirical session. Pre-verification (T83_PV + T83A_PV) fully grounded all empirical concerns (`completions_consumer` architecture; event wire shapes; UPSERT pattern; test pattern). No execution-time-unknown empirical gaps remain.

### 3.12 Q-COMMITS 3-Commit Cross-Repo (docs → engine → api)

**Decision:** 3-commit lifecycle in order: docs (canonical authority) → engine (emission) → api (consumer).

**Cross-reference shape:** Commit 1 docs references future engine + api hashes via placeholders; Commit 2 engine references docs Commit 1 hash concretely + api placeholder; Commit 3 api references both docs + engine hashes concretely.

**Rationale:** engine emission lands before api consumer tests so the consumer can be tested against the actual real payload shape (avoids the wire-shape-drift class of bugs that surface when consumer tests use a hand-rolled fixture).

## 4. Cross-Repo Implementation Surface

### 4.1 Docs Side (Stage 3 Commit 1)

- **TOOL-ARCHITECTURE.md §8 addendum:** AttackSurface Emission Lock — engine `recon.go` publishes `EventAttackSurface` to completions channel after httpx phase; rich payload from `ReconResult.LiveHosts`. ~15-25 LoC.
- **SPECIFICATION.md §7 addendum:** `EventAttackSurface` wire shape canonical — JSON schema + ADR-014 Pub/Sub channel reference + ADR-013 sole-writer continuity. ~15-30 LoC.

**Total docs delta:** ~30-55 LoC.

### 4.2 Engine Side (Stage 3 Commit 2)

- `internal/events/events.go`: NEW `EventAttackSurface` event type constant + JSON envelope/struct (~10-15 LoC)
- `internal/redis/completions.go` (or equivalent): `CompletionsPublisher` interface widened (or new `PublishAttackSurface` method) so `RunRecon` can write to `shieldscan:completions` without depending on the per-scan progress stream wrapper. May reuse existing primitive if already exposed. (~10-20 LoC)
- `internal/tools/recon/recon.go`: extend `RunRecon` signature to accept a completions-publisher; after httpx success, build `AttackSurface` payload from `ReconResult.LiveHosts` + publish; preserve existing per-scan-stream `EventReconCompleted` emission unchanged (~30-50 LoC)
- Tests: emission unit test + integration extension confirming completions channel receives event with rich payload (~20-30 LoC)

**Total engine delta:** ~70-115 LoC.

### 4.3 API Side (Stage 3 Commit 3)

- `src/app/services/completions_consumer.py`:
  - `_handle()` dispatch routing extension to accept `event_type == "attack_surface"`
  - NEW `_handle_attack_surface(event)` handler with payload validation + UPSERT logic + RLS GUC SET pre-write (~80-150 LoC)
- `src/app/schemas/attack_surface.py` (NEW; OPTIONAL): Pydantic schema for incoming event payload validation. Smaller alternative: in-handler `dict.get`-style validation if creating a NEW schema file is over-coverage. Decide at execution. (~30-60 LoC if chosen)
- `tests/services/test_completions_consumer.py` extension: NEW `attack_surface` event scenario tests (6 tests per §3.8); fakeredis fixtures + per-event AttackSurface assertions (~80-150 LoC)

**Total api delta:** ~160-360 LoC.

### 4.4 Aggregate Stage 3 LoC

Total: **~260-530 LoC** across 3 commits. Comparable to source-ingestion fix Stage 3 (~193-358 forecast; +1035 actual) + revocation Stage 3 (~88-145) + R2 Stage 3 (~200-313). Source-ingestion fix calibration update applies: novel-pattern + comprehensive-tests may push upper to **~400-700 LoC** at execution.

## 5. Phase Structure

Stage 1 design doc (THIS COMMIT) → Stage 2 plan landing → Stage 3 3-commit cross-repo (docs → engine → api) → Stage 4 Phase 5 sub-phases.

## 6. Out of Scope

1. **Task 8.3β GET endpoint** (next-task; mechanical compressed-lifecycle on populated rows)
2. **M8.1 recon-invocation production wiring** (arc-evolution-pivot; brainstorming forward-pinned to a fresh session)
3. **ADR-018 Streams+consumer-groups migration** (forward-pinned; out-of-this-task)
4. **Per-scan progress-Stream subscription consumers** (Y-CONSUMER-LOCATION (β); deferred until ADR-018)
5. **`EventReconCompleted` enrichment** (Y-WIRE-SHAPE (b); rejected — separation-of-concerns)
6. **`AttackSurface` schema migrations** (table + RLS already exist per V-LC)
7. **Engine recon-pipeline retries / fault-tolerance** (existing recon flow preserved unchanged)
8. **Findings-table join for `vulnerability_count` aggregation** (Task 8.3β endpoint concern)
9. **Multi-tenant scan isolation testing beyond the existing RLS pattern** (covered by existing test infrastructure)
10. **ADR-013 sole-writer pattern modifications** (preserved; api remains sole `AttackSurface` writer)
11. **Engine `processor.go` modifications** (`recon.go` is the callsite; `processor.go` unchanged)
12. **Operational deployment posture changes** (inherits existing `completions_consumer` FastAPI lifespan)

## 7. Forward-Pins

**Pre-execution forward-pins (Stage 2 + 3 entry):**

- Stage 2 plan landing trigger: ***"Begin Task 8.3α implementation plan landing"***
- Stage 3 entry trigger (post-plan): ***"Resume Task 8.3α — Stage 3 cross-repo implementation"***

**Post-Stage-3 forward-pins:**

1. ***"Begin Task 8.3β attack-surface endpoint task"*** — mechanical compressed-lifecycle on populated `AttackSurface` rows
2. ***"Begin M8.1 scan-executor brainstorming"*** — arc-evolution-pivot; recon-invocation production wiring
3. ***"Begin ADR-018 Streams+consumer-groups migration"*** — Y-CONSUMER-LOCATION (β) trigger
4. ***"Begin `EventReconCompleted` enrichment"*** — Y-WIRE-SHAPE (b) trigger if dual-purpose-payload need surfaces
5. ***"Begin AttackSurface `vulnerability_count` join"*** — Task 8.3β endpoint scope
6. ***"Begin recon-retry/fault-tolerance"*** — operational hardening territory
7. ***"Begin per-scan-stream-persistence-consumer migration"*** — Y-CONSUMER-LOCATION (β) + ADR-018

**In-scope forward-pin closures at Stage 3:** Drift #58 two-layer-manifestation operationally settled with Stage 3 (Layer A engine wire-shape repair + Layer B api consumer repair).

## 8. Cross-References

**Engine:**

- `a0bff50` (source-ingestion fix Stage 3 C3; recon.go callsite + EventReconCompleted preservation pattern)
- `internal/tools/recon/recon.go` (4 emit sites; new completions emission threaded into the successful httpx-completion path)
- `internal/events/events.go` (new event type registration)
- `internal/redis/` (completions publisher primitive surface)

**Docs:**

- `dacf5bb` (Task 8.2 retirement; latest docs state)
- `ac82d48` (source-ingestion fix Stage 4 P5.A close)
- `90fc933` (source-ingestion fix design doc structural precedent + Drift #54 catch-class lineage)
- `0e55a4f` (revocation design doc precedent)
- `b25e9ba` (R2 design doc precedent)
- `TOOL-ARCHITECTURE.md` §8 (engine emission canonical authority target)
- `SPECIFICATION.md` §7 + §13 ADR-013 + ADR-014 + ADR-017 (wire shape + sole-writer + mixed-primitives + sequencing canonical)

**API:**

- `8dbcbab` (source-ingestion fix Stage 3 C2; latest api state)
- `src/app/services/completions_consumer.py` (V-MB canonical architectural analog; 601 LoC)
- `src/app/models/recon.py` (V-LC AttackSurface ORM + SubdomainStatus)
- `src/app/routes/projects.py:519` (V-ME UPSERT precedent)
- `src/app/main.py` lifespan (V-MF deployment shape inheritance)

**SPEC sections:**

- §6 line 633-654 (Task 8.3β endpoint wire shape; future consumer of populated rows)
- §13 ADR-013 (sole-writer)
- §13 ADR-014 (mixed-primitives Pub/Sub vs Streams)
- §13 ADR-017 (sequencing — not invoked for AttackSurface UPSERT)
- §13 ADR-018 forward-pinned (Streams + consumer-groups)

**Drift catalog:**

- Drift #58 (two-layer manifestation: Layer A engine wire-shape gap + Layer B api consumer absence; single-entry consolidated catch-class per Drift #54 catch-class lineage; cumulative count 57 → 58)
- Related Drift #54 (source-ingestion fix; orphaned column variant; same catch-class)

**Cumulative drift count:** 58 catches at execution time (Drift #58 catalogued at T83_PV + T83A_PV; clean Stage 1 design doc entry).

---

## 9. Phase 5 Annotations

### §9.A Stage 3 Trio Lifecycle Close (2026-05-30)

Stage 3 cross-repo trio operationally closed Drift #58 (two-layer manifestation: orphaned `AttackSurface` table + orphaned engine wire shape; stored-design-intent-with-unimplemented-mechanism catch-class continuation from Drift #54):

- **Stage 3 C1 (shieldscan-docs `721ba02`):** `TOOL-ARCHITECTURE.md` §8.5 AttackSurface Emission Lock + `SPECIFICATION.md` §7.6 EventAttackSurface Wire Shape dual addendums; canonical authority both layers; **+66 LoC** (TOOL-ARCH +16 + SPEC +50)
- **Stage 3 C2 (shieldscan-engine `fc75a98`):** `events.go` `EventAttackSurface` + `SubdomainRow` structs + `recon.go` `RunRecon`-internal emission BEFORE `EventReconCompleted` + `CompletionsPublisher.PublishAttackSurface` method + 6 fixture updates + 3 new tests; **+212 / −6 LoC**; 27 packages green; Drift #59 catalogued at execution
- **Stage 3 C3 (shieldscan-api `05023f4`):** `completions_consumer.py` `_handle()` routing extension + NEW `_handle_attack_surface` handler with atomic-per-event UPSERT + RLS GUC `SET` + 7 scenario tests; **+507 LoC**; 571 full suite passed (564 baseline + 7 new exactly); ZERO regressions

### §9.B Drift catches at Stage 3 execution (1 total)

**Drift #59 — Y-RECON-PUBLISHER-WIRING signature extension precision (catalogued at C2 execution):** Plan `dba6a7c` §3.7 + §4.3 framed "plumb publisher into recon" as singular; empirical reality required +3 params (`completionsPub` + `scanID` + `orgID`) at `RunRecon` signature; design doc `0030319` §3.3 pseudo-code implicitly referenced `scanID`/`orgID`. Soft drift: plan-vs-design-doc internal inconsistency rather than plan-vs-empirical-reality miss. Same catch-class as Drifts #53 / #55 / #56 (parameter / file-naming-precision). Resolved inline at engine C2 by extending `RunRecon` signature; M8.1 forward-pin gap meant no production callers required updating; only 6 test fixtures updated.

**Pattern signal:** this is the 3rd instance of plan-precision-vs-design-doc-precision drift; if a 4th surfaces, a discipline-level **"plan-doc consistency check pre-implementation"** forward-pin is warranted (rule-of-three trigger).

**Drift catch summary (Stage 3 lifecycle):** 1 drift at execution (#59); matches plan §5 LOW forecast (~0-2 drifts). Lower than source-ingestion fix Stage 3 (3 drifts; novel-pattern). Cumulative session-tail framing-drift count: **58 → 59**.

### §9.C Y-decision resolutions at execution (4 total)

- **Y-EVENT-PUBLISH-PATH → (b1) second typed `PublishAttackSurface` method on existing `CompletionsPublisher`.** Plan default landed per V-OF empirical (existing `Publish(ctx, JobCompletedEvent)` hard-typed; b1 viable; `EventAttackSurface` won't need `Validate` since constructed in-code); avoids generalizing publisher dispatch.
- **Y-RECON-PUBLISHER-WIRING → (a) plumb 3 params through `RunRecon` signature.** Plan default landed per V-OG + V-OH empirical (RunRecon today: ctx + domain + limit + ProgressPublisher + logger; no scanID/orgID/completionsPub); 6 test fixtures updated; no production callsite per M8.1 gap; Drift #59 surfaced (plan said "plumb publisher"; reality required +3 params).
- **Y-UPSERT-ATOMICITY → (c) atomic-per-event single commit.** Plan default landed; multi-row `pg_insert` + `on_conflict_do_update` with single `session.commit()` per event; `uq_scan_subdomain` semantics inherently idempotent (Q-MULTI-PROCESS-POSTURE b lock holds operationally).
- **Y-EVENT-PAYLOAD-VALIDATION → (b) permissive parse (PIVOTED from c).** Plan default was (c) hybrid; V-PE empirical drove pivot to (b) — existing `completions_consumer` uses permissive `dict.get` + UUID try/except + no Pydantic schemas; consistency-of-convention preferred over hybrid; no new `schemas/` file added.

### §9.D Strong dual-side analog hypothesis validation

Plan §2.2 hypothesis: Task 8.3α has STRONG dual-side analog (`completions_consumer.py` V-MB + `EventReconCompleted` V-MD patterns) unlike source-ingestion fix (net-new pattern at engine layer); D-deviation forecast LOW (~0-2).

**Empirical outcome: hypothesis CONFIRMED.** Stage 3 trio total drift count = 1 (#59 at engine C2; ZERO at docs C1; ZERO at api C3). Below source-ingestion fix (3 drifts at Stage 3) by analog-mitigation margin.

**Pattern recognition signal:** strong-dual-side-analog tasks have empirically validated LOW drift forecasts at this arc maturity level.

**Discipline implication:** when both engine and api have direct architectural precedent + pre-verification surfaces no novel territory, drift forecast can be confidently LOW; novel-pattern tasks (source-ingestion fix Stage 3 C3 #57 framework-extension catch-class) remain higher-forecast territory.

### §9.E LoC forecast-vs-actual honest calibration

Stage 3 aggregate forecast: ~250-435 LoC (design doc §4.4 + plan §4 aggregate); actual: **+785 LoC** (C1 +66 + C2 +212 + C3 +507). Significant overshoot driven by:

- **C3 test density:** +349 LoC across 7 tests (~50 LoC per test with cross-state assertions); plan-forecast ~80-150
- **C2 engine code density:** +212 across 4 files including `statusFromStatusCode` helper + 6 fixture updates + 3 new tests; plan-forecast ~60-95
- **C1 SPEC wire-shape density:** +66 LoC including full JSON wire shape + field semantics; plan-forecast ~30-55

**Calibration update (4th instance of test-LoC undershoot pattern):**

- Pseudo-code-heavy plans default to ~350-450 LoC (already calibrated)
- Test-LoC for 5-7 scenarios with cross-state assertions: ~200-350 LoC (NOT ~80-150); **~50 LoC per test default**
- Stage 3 commits with helper-function additions + N-site test fixture updates: forecast ~150-250% of plan estimate
- SPEC wire-shape addendums: ~50 LoC density when full JSON schema + field semantics present

**Updated forward calibration:** novel-pattern Stage 3 ~400-800 LoC aggregate; strong-dual-side-analog Stage 3 ~500-800 LoC (this task) due to comprehensive-test density; comprehensive-test-coverage default ~250-400 LoC at C3 layer.

### §9.F Forward-pin chain operational closure

**Drift #58 root-cause repair (operational closure):**

- **Two-layer canonical authority canonicalized to implementation-lock:** TOOL-ARCH §8.5 + SPEC §7.6 at `721ba02`
- **Layer A engine emission:** `events.go` + `recon.go` + `CompletionsPublisher.PublishAttackSurface` at `fc75a98`
- **Layer B api consumer:** `completions_consumer` routing + `_handle_attack_surface` UPSERT at `05023f4`
- **End-to-end pipe operational STRUCTURALLY:** engine emits `EventAttackSurface` (when `RunRecon` invoked from production) → completions Pub/Sub → api consumer dispatch → `AttackSurface` ORM UPSERT with RLS
- **Operational activation gate:** **M8.1 recon-invocation production wiring**; 8.3α infrastructure ready-when-M8.1-lands; `AttackSurface` rows stay empty until M8.1 wires `RunRecon` at production callsite supplying `completionsPub` + `scanID` + `orgID`

**Forward-pins preserved (8 post-Stage-4):**

1. ***"Begin Task 8.3β attack-surface endpoint task"*** — mechanical compressed-lifecycle on populated AttackSurface rows; ready post-Stage-4-close; activation paired with M8.1
2. ***"Begin M8.1 scan-executor brainstorming"*** — arc-evolution-pivot; recon-invocation production wiring; primary M8 closure blocker
3. ***"Begin ADR-018 Streams+consumer-groups migration"*** — Y-CONSUMER-LOCATION β trigger
4. ***"Begin EventReconCompleted enrichment"*** — Y-WIRE-SHAPE b trigger if dual-purpose-payload need surfaces
5. ***"Begin AttackSurface vulnerability_count join"*** — Task 8.3β endpoint scope
6. ***"Begin recon-retry/fault-tolerance"*** — operational hardening
7. ***"Begin per-scan-stream-persistence-consumer migration"*** — Y-CONSUMER-LOCATION β + ADR-018
8. ***"Begin plan-doc consistency check pre-implementation discipline"*** — Drift #59 rule-of-three trigger if 4th plan-vs-design-doc precision drift surfaces

### §9.G Phase 5 sub-phase outcome

**Outcome γ (P5.A only; P5.B-E deferred).** Rationale: Drift #58 operationally settled at Stage 3 trio close; no forward-pin chain pivots discovered post-Stage-3; Task 8.3β + M8.1 forward-pins preserved cleanly; Stage 3 trio cross-references locked in commit chain (`721ba02` + `fc75a98` + `05023f4`). P5.B (forward-pin chain reconciliation) not required — chain captured in §9.F. P5.C-E (post-lifecycle artifact updates) not required — no orphaned artifacts. Mirrors 5-instance precedent (R2 + revocation + cancel-helper + source-ingestion fix + this) for outcome γ at P5.A close.

Task 8.3α lifecycle Stages 1+2+3+4 **OPERATIONALLY CLOSED**. Next triggers per milestone-completion-constraint: Task 8.3β GET endpoint (mechanical compressed-lifecycle on populated rows) + M8.1 scan-executor brainstorming (arc-evolution-pivot territory) before M8 declarable CLOSED + M9 entry.
