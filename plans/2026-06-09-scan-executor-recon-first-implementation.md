# M8.1β.2 — Scan-Executor Recon-First Architecture (ADR-028): Implementation Plan

**Status:** Ready for Stage 3 cross-repo 4-commit implementation. Pre-implementation grounded against M8.1β.2 Stage 1 design doc `3f07611` Q1-Q10 locks + V-W refinements + Task 8.3α infrastructure (`fc75a98` + `05023f4`) + M8.1β.1 lifecycle CLOSED (`bb3e75f` + `d773776` + `9ccde1a`).

**Date:** 2026-06-09.

**Authority:** M8.1β.2 Stage 1 design doc canonical authority (shieldscan-docs commit `3f07611`; 252 LoC; Q1-Q10 locks + V-W refinements + Drift #60 6/6 closure + Drift #61 catalogue); M81_PV + M81B_PV + V-W pre-verification surface reports (prior + this session); Task 8.3α implementation plan structural precedent (commit `dba6a7c`; 422 LoC; closest analog at architectural-decision-density level); source-ingestion fix plan structural precedent (commit `04f44a9`; 426 LoC); revocation plan structural precedent (commit `fdad021`; 338 LoC); R2 plan structural precedent (commit `721f788`; 357 LoC); SPEC §13 ADR-013 + ADR-014 + ADR-017 + ADR-022 + M8.1α/β.1 addendums + ADR-028 (Stage 1 design doc target; Stage 3 Commit 1 implementation target); 61+ cumulative session-tail framing-drift discipline; milestone-completion-constraint locked.

**Related:** Stage 3 cross-repo implementation trigger phrase: ***"Resume M8.1β.2 — Stage 3 cross-repo 4-commit implementation"*** (after this plan lands).

---

## 1. Authority + Scope Lock

Q1-Q10 locks (locked at Stage 1 design doc `3f07611` §1 + §3): see design doc §1 + §3 for full architectural rationale.

**In scope (Stage 3 sub-steps; Q9 (b) 4-commit shape):**

1. shieldscan-docs SPEC §13 ADR-028 + ADR-022 addendum continuation + ADR-020 closure documentation + TOOL-ARCH §8.2+§10.3+§12.3 + SPEC §1885+ speculative scaffolding resolution markers (Stage 3 Commit 1; ~80-150 LoC)
2. shieldscan-engine `internal/worker/processor.go` dispatch case `engine="recon"` → `RunRecon` (Stage 3 Commit 2; ~10-25 LoC)
3. shieldscan-engine `DRIFT-LOG.md` Drift #60 6/6 closure entry (Stage 3 Commit 2; ~25-45 LoC)
4. shieldscan-engine tests: recon-engine dispatch unit + integration extension (Stage 3 Commit 2; ~15-30 LoC)
5. shieldscan-api `src/app/services/orchestrator.py` `SCAN_TYPE_TOOLS` web-ScanType rename (remove subfinder + httpx from QUICK + FULL_WEB + FULL_WEB_SOURCE + FULL_SPECTRUM); phase-1 web-ScanType recon ScanJob dispatch + audit emission; NEW `dispatch_phase2` method per §3.7 + Y-AUDIT-LOOKUP-QUERY-SHAPE + Y-AUTHIDENTITY-RECONSTRUCTION-SHAPE + Y-RECON-SCANJOB-IDEMPOTENCY-KEY (Stage 3 Commit 3; ~150-250 LoC)
6. shieldscan-api `src/app/services/audit.py` `SCAN_DISPATCHED_PHASE2 = "scan.dispatched_phase2"` enum addition per V-WC (Stage 3 Commit 3; ~1-5 LoC)
7. shieldscan-api `tests/services/test_orchestrator.py` phase-1 dispatch unit + dispatch_phase2 unit + audit emission tests + recon-disabled-non-web tests (Stage 3 Commit 3; ~150-200 LoC)
8. shieldscan-api `src/app/services/completions_consumer.py` `_handle_attack_surface` extension to invoke `orchestrator.dispatch_phase2` in new session after UPSERT commit + fail-loud-audit on failure per Y-PHASE2-DISPATCH-FAILURE-AUDIT-SHAPE (Stage 3 Commit 4; ~50-100 LoC)
9. shieldscan-api `tests/services/test_completions_consumer.py` phase-2 trigger unit tests (Stage 3 Commit 4; ~80-150 LoC)
10. shieldscan-api `tests/integration/` NEW end-to-end test ScanCreate → recon emission → AttackSurface UPSERT → phase-2 dispatch → tool ScanJobs created (Stage 3 Commit 4; ~80-150 LoC)

**Out of scope (forward-pinned per design doc §6):** Per-target rate-limiting; phase-2 dispatch retry logic; ADR-018 Streams+consumer-groups migration; speculative scaffolding deprecation pass; Task 8.3β attack-surface endpoint; production-readiness audit; M9 entry; ADR-018 + ADR-019 promotions.

**Brainstorming chain Q1-Q10 locks + V-W refinements recap (per Stage 1 design doc §1):** Y-EXPAND-LOCATION (c) hybrid follow-up dispatch; Y-DISPATCH-MODEL (a) per-(target, tool) + (a.ii) target_hash + (i) empty-correct; Y-CONCURRENCY-MODEL (a) per-worker only + ADR-020 closure; Y-MIGRATION-NEEDED (a) NO migration; Y-AUDIT-TRAIL-CHANGES (b) two-phase + (b.i) + rich details; Y-RECON-ENGINE-NAME (a) engine="recon" + (a.ii) implicit; Y-DISPATCH-PHASE-COORDINATION (c) dispatch_phase2 + (c.ii) sequential + (c.ii.B) fail-loud + Q7.4 Option α audit-log lookup (V-WD refinement); Y-ADR-NUMBER ADR-028 (V-WB lock); Q-STAGE3-DECOMPOSITION (b) 4-commit; Q-MIGRATION-PATH (c) bounded-staleness.

## 2. Pre-Implementation State

### 2.1 Engine + API + Docs Infrastructure Ready Matrix (M81_PV + M81B_PV + V-W Pre-Verification Findings)

**Engine side ready:**

- `internal/tools/recon/recon.go`: `RunRecon` already wired with `completionsPub` + `scanID` + `orgID` + `log` signature (Task 8.3α C2 `fc75a98`); `EventAttackSurface` emission operational; `ReconResult.LiveHosts` rich shape persists
- `internal/worker/processor.go`: existing dispatch case structure (`registry.Get(engine) → ToolRunner.Run`); needs new case for `engine="recon"` → invoke `RunRecon` directly (NOT via registry per ADR-022)
- `internal/redis/pubsub.go`: `CompletionsPublisher.PublishAttackSurface` already operational (Task 8.3α C2)

**Engine side pending (Stage 3 Commit 2 scope):**

- `internal/worker/processor.go`: NEW dispatch case `engine="recon"` → invoke `RunRecon` (~10-25 LoC; preserves ADR-022 — recon stays helper not ToolRunner)
- `DRIFT-LOG.md`: Drift #60 6/6 closure entry (~25-45 LoC; M8.1α + M8.1β.1 + this commit)
- tests: dispatch-recon-engine unit test + integration extension (~15-30 LoC)

**API side ready (V-W refinements):**

- `src/app/services/orchestrator.py`: `SCAN_TYPE_TOOLS` current state (M8.1β.1 `d773776`; depcheck + nuclei + zap canonical); `_build_config_block` extension precedent (M8.1β.1); `ScanOrchestrator.dispatch(scan, identity, request_ip=None, user_agent=None) -> list[ScanJob]` signature per V-XC
- `src/app/services/audit.py`: `ScanAction(str, PyEnum)` + 6 existing entries (`SCAN_DISPATCHED`, `SCAN_CANCELED`, `SCAN_COMPLETED`, `SCAN_FAILED`, `SCAN_COMPARED`, …) per V-WC; addition pattern straightforward
- `src/app/models/audit.py`: `AuditLog` with `actor_id` (UUID nullable FK users.id) + `action` (String(100)) + `resource_type` (String(50)) + `resource_id` (UUID nullable) + `details` (JSONB) + `created_at` per V-XB; **audit-log-lookup canonical pattern: `WHERE resource_type='scan' AND resource_id=scan_id AND action='scan.dispatched'`** (no direct `scan_id` FK; canonical via `resource_id`)
- `src/app/services/completions_consumer.py`: `_handle_attack_surface` UPSERT operational (Task 8.3α C3 `05023f4`); new-session-after-commit pattern is extension surface

**API side pending (Stage 3 Commits 3-4 scope):**

- `src/app/services/audit.py`: `SCAN_DISPATCHED_PHASE2 = "scan.dispatched_phase2"` enum addition (~1-5 LoC)
- `src/app/services/orchestrator.py`: `SCAN_TYPE_TOOLS` web-ScanType rename + phase-1 web-ScanType recon dispatch + NEW `dispatch_phase2` method (~150-250 LoC)
- `tests/services/test_orchestrator.py`: phase-1 + dispatch_phase2 + audit emission + recon-disabled-non-web tests (~150-200 LoC)
- `src/app/services/completions_consumer.py`: post-UPSERT-commit `orchestrator.dispatch_phase2` invocation + fail-loud-audit (~50-100 LoC)
- `tests/services/test_completions_consumer.py` + `tests/integration/`: phase-2 trigger + end-to-end ScanCreate → recon → AttackSurface → phase-2 → tool ScanJobs tests (~160-300 LoC)

**Docs side pending (Stage 3 Commit 1 scope):**

- `SPECIFICATION.md` §13: NEW ADR-028 canonical text + ADR-022 addendum continuation (Drift #60 6/6 closure) + ADR-020 promotion-trigger closure documentation (~80-150 LoC)
- `TOOL-ARCHITECTURE.md`: §8.2 + §10.3 + §12.3 SPECULATIVE markers updated to reference ADR-028 as canonical authority (mechanical update; ~10-25 LoC)
- `SPECIFICATION.md` §1885+: M8 invocation pattern speculative markers resolution (mechanical; ~5-15 LoC)

### 2.2 Architectural Analog Strength

**Strong dual-side analog:** Task 8.3α infrastructure (engine `RunRecon` + `EventAttackSurface` emission + api `completions_consumer` + `AttackSurface` UPSERT) is the foundation for Q1 hybrid follow-up dispatch composition. M8.1β.2 extends this with ONE NEW component layer (api `orchestrator.dispatch_phase2` method + `completions_consumer` hook).

**D-deviation forecast implication:** LOWER bound expected per strong-dual-side-analog hypothesis (~0-3 drifts at Stage 3 execution) — confirmed by Task 8.3α (1 drift trio total) + M8.1α (0 drifts trio) + M8.1β.1 (0 drifts trio) cumulative validation.

### 2.3 V-W + V-X Refinement Implications for Stage 3

- V-WB ADR-028 locked: Stage 3 Commit 1 docs lands ADR-028 at SPEC §13 with full canonical text per Stage 1 design doc §3.8 + rejected alternatives + composition with prior ADRs.
- V-WC `SCAN_DISPATCHED_PHASE2 = "scan.dispatched_phase2"`: Stage 3 Commit 3 audit registration is 1-5 LoC enum addition + import update.
- V-WD Q7.4 Option α audit-log lookup: Stage 3 Commit 3 `dispatch_phase2` implementation queries `AuditLog` inline; partial `AuthIdentity` reconstruction per Y-AUTHIDENTITY-RECONSTRUCTION-SHAPE; Drift #61 documented in commit body.
- V-XB AuditLog lookup canonical pattern: `resource_type='scan' AND resource_id=scan_id AND action='scan.dispatched'` (NOT a direct `scan_id` FK); execution-territory precision for Y-AUDIT-LOOKUP-QUERY-SHAPE.

## 3. Architectural Decisions (Plan-Level Locks)

Brainstorming chain Q1-Q10 architectural decisions locked at Stage 1 design doc `3f07611` §1 + §3.1-§3.10. Plan-level refinements captured below.

### 3.1 Y-AUDIT-LOOKUP-QUERY-SHAPE

**Plan-level Y-decision:** (a) direct ORM query; (b) repository pattern method; (c) service-layer method.

Default (a) direct ORM query per `completions_consumer` existing pattern (`db.execute + select`); resolve at Stage 3 Commit 3 execution. Pseudo-code (V-XB-grounded canonical lookup):

```python
async def dispatch_phase2(self, *, db: AsyncSession, scan_id: UUID) -> list[ScanJob]:
    """Phase-2 dispatch: re-dispatch per-(target, tool) ScanJobs after recon completion.
    Per Q1 + Q7 + Q7.4 V-WD refinement (audit-log lookup for identity context).
    """
    # Q7.4 Option α: audit-log lookup for identity context (Drift #61 V-WD refinement).
    # V-XB canonical pattern: AuditLog has resource_type/resource_id (not direct scan_id FK).
    dispatch_audit = await db.execute(
        select(AuditLog)
        .where(
            AuditLog.resource_type == "scan",
            AuditLog.resource_id == scan_id,
            AuditLog.action == ScanAction.SCAN_DISPATCHED.value,
        )
        .order_by(AuditLog.created_at.desc())
        .limit(1)
    )
    dispatch_row = dispatch_audit.scalar_one_or_none()
    if dispatch_row is None:
        # Implicit-ordering-dependency violation; fail-loud per Q7 (c.ii.B)
        await self._fail_scan_phase2(db, scan_id, reason="no_dispatch_audit_found")
        return []
    # Y-AUTHIDENTITY-RECONSTRUCTION-SHAPE (a) minimal:
    identity = _reconstruct_identity_for_phase2(dispatch_row)
    ...
```

### 3.2 Y-AUTHIDENTITY-RECONSTRUCTION-SHAPE

**Plan-level Y-decision:** (a) minimal — actor_id-only reconstruction; (b) partial with `api_key_prefix` rehydration from `details`; (c) lazy reconstruction at audit-emission only.

Default (a) per minimal-reconstruction principle — `actor_id` is load-bearing for phase-2 audit emission; `api_key_prefix` not required for `orchestrator.dispatch` internal context (already lives in `details` JSONB for phase-1 audit row).

Per V-XC `AuthIdentity` shape needs verification at C3.4: the dispatch signature consumes `identity: AuthIdentity`; reconstruction must yield a valid `AuthIdentity` object with at least the fields `dispatch` accesses. Resolve exact reconstruction shape at C3.4 execution.

### 3.3 Y-RECON-SCANJOB-IDEMPOTENCY-KEY

**Plan-level Y-decision:** (a) `{scan_id}:recon:{ts}`; (b) `{scan_id}:recon:none:{ts}`; (c) `{scan_id}:recon:{root_domain_hash}:{ts}`.

Default (a) per simplicity — phase-1 is always single recon per scan; consistent with per-tool format minus target hash. (Phase-2 tool ScanJobs adopt `{scan_id}:{tool}:{sha256(target_url)[:16]}:{ts}` per Q2 (a.ii) at C3.4.)

### 3.4 Y-PHASE2-DISPATCH-FAILURE-AUDIT-SHAPE

**Plan-level Y-decision:** (a) generic `{phase, error_type, error_message}`; (b) diagnostic-rich `{phase, attempted_target_count, attempted_tool_count, exception_class, exception_msg}`; (c) new event type `SCAN_PHASE2_DISPATCH_FAILED`.

Default (b) per diagnostic-richness for operational debugging; uses existing `SCAN_FAILED` action + rich details JSON; preserves audit event type minimalism.

### 3.5 SCAN_TYPE_TOOLS Web-ScanType Transformation

Per Q6 (a) + (a.ii) lock:

| ScanType | Pre-M8.1β.2 | Post-M8.1β.2 |
|---|---|---|
| QUICK | `(subfinder, httpx, nuclei, sslyze)` | `(nuclei, sslyze)` |
| FULL_WEB | `(subfinder, httpx, nuclei, zap, wapiti, nikto, nmap, sslyze)` | `(nuclei, zap, wapiti, nikto, nmap, sslyze)` |
| FULL_WEB_SOURCE | `(subfinder, httpx, nuclei, zap, wapiti, nikto, nmap, sslyze, semgrep, gitleaks, depcheck)` | `(nuclei, zap, wapiti, nikto, nmap, sslyze, semgrep, gitleaks, depcheck)` |
| FULL_SPECTRUM | `(subfinder, httpx, nuclei, zap, wapiti, nikto, sqlmap, nmap, sslyze, semgrep, gitleaks, depcheck, trivy-fs, corstest, checkov)` | `(nuclei, zap, wapiti, nikto, sqlmap, nmap, sslyze, semgrep, gitleaks, depcheck, trivy-fs, corstest, checkov)` |
| API | unchanged: `(nuclei, zap, corstest, sqlmap)` | unchanged |
| MOBILE | unchanged: `(mobsf,)` | unchanged |
| CONTAINER | unchanged: `(trivy-container, checkov)` | unchanged |

Drift #60 recon-orphan sub-category resolved **STRUCTURALLY** (subfinder + httpx no longer in any `SCAN_TYPE_TOOLS` entry).

### 3.6 Phase-1 Web-ScanType Recon Dispatch (Q6 (a.ii))

```python
# In ScanOrchestrator.dispatch(), after validating scan_type and credentials:
_WEB_SCAN_TYPES: Final[frozenset[ScanType]] = frozenset({
    ScanType.QUICK,
    ScanType.FULL_WEB,
    ScanType.FULL_WEB_SOURCE,
    ScanType.FULL_SPECTRUM,
})

if scan.scan_type in _WEB_SCAN_TYPES:
    # Phase-1: dispatch single recon ScanJob + audit job_count=1+len(tools)
    recon_job = ScanJob(
        organization_id=scan.organization_id,
        scan_id=scan.id,
        engine="recon",
        target_url=scan.project.target_url,
        status="queued",
        idempotency_key=f"{scan.id}:recon:{unix_ts}",  # Y-RECON-SCANJOB-IDEMPOTENCY-KEY (a)
    )
    self.db.add(recon_job)
    # NOTE: phase-1 SCAN_DISPATCHED audit retains job_count for tool jobs;
    # the recon job is a phase-1 marker. Phase-2 emits SCAN_DISPATCHED_PHASE2
    # with the actual tool-dispatch summary. See §3.7 audit shape.
    jobs = [recon_job]
    # ... wire only the recon_job at this dispatch; tool jobs created at phase-2 ...
```

### 3.7 Phase-2 Audit Emission (Q5 (b.i))

After `dispatch_phase2` succeeds:

```python
await audit(
    db,
    organization_id=scan.organization_id,
    actor_id=identity.actor_id,  # reconstructed per Y-AUTHIDENTITY-RECONSTRUCTION-SHAPE
    action=ScanAction.SCAN_DISPATCHED_PHASE2,
    resource_type="scan",
    resource_id=scan.id,
    ip_address=None,  # not available at phase-2; consumer-driven dispatch
    user_agent=None,
    details={
        "scan_type": scan.scan_type.value,
        "priority": scan.priority,
        "phase": "tools",
        "target_count": target_count,
        "tool_count": tool_count,
        "job_count": target_count * tool_count,
        "recon_event_id": str(dispatch_row.id),
    },
)
```

### 3.8 Completions Consumer Phase-2 Trigger Hook (Q7 (c.ii))

```python
# In _handle_attack_surface, AFTER the existing UPSERT commit:
async def _handle_attack_surface(self, event: dict[str, Any]) -> None:
    async with self._session_factory() as session:
        # Existing Task 8.3α UPSERT path (preserved verbatim)
        ...
        await session.commit()

    # Q7 (c.ii) sequential sessions — open NEW session for phase-2 dispatch
    try:
        async with self._session_factory() as dispatch_session:
            from app.services.orchestrator import ScanOrchestrator
            orch = ScanOrchestrator(db=dispatch_session, redis=self._redis)
            await orch.dispatch_phase2(scan_id=UUID(event["scan_id"]))
            await dispatch_session.commit()
    except Exception as exc:
        # Q7 (c.ii.B) fail-loud-audit per Y-PHASE2-DISPATCH-FAILURE-AUDIT-SHAPE (b)
        await self._fail_scan_phase2_with_audit(
            scan_id=UUID(event["scan_id"]),
            exc=exc,
        )
```

### 3.9 Q-STAGE3-DECOMPOSITION (b) 4-Commit Cross-Repo Sequencing

**Commit 1 (docs FIRST; ~85-150 LoC):** ADR-028 canonical text + ADR-022 addendum continuation + ADR-020 closure documentation + speculative-scaffolding resolution markers.

**Commit 2 (engine SECOND; ~50-100 LoC):** `processor.go` recon dispatch case + DRIFT-LOG Drift #60 6/6 closure entry + tests.

**Commit 3 (api orchestrator THIRD; ~265-423 LoC):** `SCAN_TYPE_TOOLS` web-ScanType rename + phase-1 dispatch + `dispatch_phase2` method + `SCAN_DISPATCHED_PHASE2` audit registration + tests.

**Commit 4 (api completions_consumer + e2e FOURTH; ~210-400 LoC):** `_handle_attack_surface` post-UPSERT-commit `dispatch_phase2` trigger + fail-loud-audit + e2e integration tests.

**Sequencing rationale:** Canonical authority before implementation; engine recon dispatch enables phase-1 ScanJob to execute; api orchestrator creates `dispatch_phase2` infrastructure; api completions_consumer wires trigger + closes chain with e2e tests.

### 3.10 Test Strategy Per Commit

**C2 engine tests:** mock `RunRecon` to capture dispatch + assert `engine="recon"` routes outside registry + verify completion event emission.

**C3 api orchestrator tests (~3-5 tests):** phase-1 web-ScanType emits ONLY recon ScanJob + non-web ScanType unchanged behavior + `dispatch_phase2` reads AttackSurface + creates N×M ScanJobs + emits `SCAN_DISPATCHED_PHASE2` + empty-AttackSurface case (zero phase-2 ScanJobs + audit still emitted per Q5 (b.i)) + fail-loud on no-dispatch-audit-found.

**C4 completions_consumer + e2e tests:** phase-2 trigger fires after UPSERT commit + sequential-session pattern (handler session commits AttackSurface before dispatch session opens) + dispatch failure path emits fail-loud audit + e2e ScanCreate → mocked recon emission → AttackSurface UPSERT → phase-2 dispatch → tool ScanJobs persisted with correct idempotency keys.

## 4. Stage 3 Sub-Step Breakdown

### Stage 3 Commit 1 — Docs (~30-45min)

**C1.1** Locate SPEC §13 ADR-027 + identify ADR-028 insertion point (after ADR-027; consistent with ADR-022 + addendums precedent).

**C1.2** Author ADR-028 canonical text per Stage 1 design doc §3.1-§3.10 + §1 summary. ~50-80 LoC.

**C1.3** Append ADR-022 addendum continuation: Drift #60 6/6 closure documentation + recon-orphan sub-category resolution at this ADR. ~10-20 LoC.

**C1.4** Author ADR-020 promotion-trigger closure documentation: DRIFT-LOG L349 hypothesized M8.1 as promotion-trigger; empirically didn't fire per Q3 lock cascade. ~10-15 LoC.

**C1.5** Update TOOL-ARCHITECTURE.md §8.2 + §10.3 + §12.3: prepend SPECULATIVE markers with "→ ADR-028 canonical authority; this scaffolding pre-dates ADR-013 sole-writer and is INFORMATIONAL not BINDING." ~10-20 LoC.

**C1.6** Update SPEC §1885+: M8 invocation pattern speculative resolution. ~5-15 LoC.

**C1.7** Pre-commit verification + single atomic docs commit. Cross-references future engine + api commit hashes via placeholders.

**Total Stage 3 Commit 1 LoC delta:** ~85-150 LoC.

### Stage 3 Commit 2 — Engine (~20-30min)

**C2.1** `processor.go`: NEW dispatch case `engine="recon"` → invoke `RunRecon` (NOT via registry per ADR-022). Pseudo-code:

```go
// Step 5b (NEW): recon-engine special case per ADR-028 + ADR-022.
if job.Engine == "recon" {
    result, err := recon.RunRecon(
        jobCtx, job.Target.Domain, defaultLimit,
        progressPub, p.completionsPub,
        job.ScanID, job.OrganizationID, p.log,
    )
    if err != nil {
        return p.emitFailure(ctx, progressPub, job, start, err)
    }
    // Emit job_completed with the subdomain count for SSE visibility;
    // findings stay empty per ADR-022 recon-emits-target-discovery-not-findings.
    return p.emitJobCompleted(ctx, progressPub, job, start, len(result.LiveHosts))
}

// existing dispatch via registry.Get continues for tool engines
runner, err := p.registry.Get(job.Engine)
...
```

~10-25 LoC.

**C2.2** `DRIFT-LOG.md`: Drift #60 6/6 closure entry (newest-on-top per precedent). ~25-45 LoC.

**C2.3** Tests: dispatch-recon-engine unit + integration extension. ~15-30 LoC. Verify `RunRecon` invoked + completion emission + no registry.Get call.

**C2.4** Pre-commit verification: `go vet ./...` + `go test -race ./...` (27-package baseline preserved).

**C2.5** Single atomic engine commit. Cross-references docs Commit 1 + api Commit 3+4 placeholders.

**Total Stage 3 Commit 2 LoC delta:** ~50-100 LoC.

### Stage 3 Commit 3 — API Orchestrator (~60-90min)

**C3.1** `audit.py`: `SCAN_DISPATCHED_PHASE2 = "scan.dispatched_phase2"` enum addition per V-WC. ~1-5 LoC.

**C3.2** `orchestrator.py`: `SCAN_TYPE_TOOLS` web-ScanType rename per §3.5 (subfinder + httpx removed from QUICK + FULL_WEB + FULL_WEB_SOURCE + FULL_SPECTRUM). ~4-8 LoC.

**C3.3** `orchestrator.py`: Phase-1 web-ScanType recon ScanJob dispatch per §3.6 (`_WEB_SCAN_TYPES` frozenset + recon ScanJob branch + Y-RECON-SCANJOB-IDEMPOTENCY-KEY (a)). ~30-60 LoC.

**C3.4** `orchestrator.py`: NEW `dispatch_phase2` method per §3.1 + V-WD audit-log lookup (V-XB canonical pattern `resource_type='scan' AND resource_id=scan_id`) + Y-AUTHIDENTITY-RECONSTRUCTION-SHAPE (a) minimal + Y-PHASE2-DISPATCH-FAILURE-AUDIT-SHAPE (b) diagnostic-rich. Includes per-(target, tool) ScanJob construction with `{scan_id}:{tool}:{sha256(target_url)[:16]}:{ts}` idempotency-key per Q2 (a.ii). ~80-150 LoC.

**C3.5** `tests/services/test_orchestrator.py`: NEW tests:
1. `test_dispatch_phase1_web_emits_only_recon_job` — QUICK/FULL_WEB/FULL_WEB_SOURCE/FULL_SPECTRUM emit ONLY recon ScanJob at phase-1
2. `test_dispatch_phase1_non_web_unchanged` — API/MOBILE/CONTAINER dispatch all SCAN_TYPE_TOOLS engines directly (no phase-1 split)
3. `test_dispatch_phase2_queries_audit_log_for_identity` — V-XB canonical pattern verified
4. `test_dispatch_phase2_creates_per_target_tool_scanjobs` — N×M ScanJob persistence + idempotency-key format
5. `test_dispatch_phase2_empty_attack_surface` — zero ScanJobs created + SCAN_DISPATCHED_PHASE2 emitted with target_count=0
6. `test_dispatch_phase2_emits_scan_dispatched_phase2_audit` — audit details shape
7. `test_dispatch_phase2_fail_loud_when_no_dispatch_audit_found` — Q7 (c.ii.B) verification

~150-200 LoC (per ~50 LoC per test calibration).

**C3.6** Pre-commit verification: `pytest tests/services/test_orchestrator.py` + full suite (expect 573 baseline + new tests; ZERO regressions).

**C3.7** Single atomic api commit. Cross-references docs Commit 1 + engine Commit 2 hashes concretely.

**Total Stage 3 Commit 3 LoC delta:** ~265-423 LoC.

### Stage 3 Commit 4 — API Completions Consumer + E2E (~45-60min)

**C4.1** `completions_consumer.py`: `_handle_attack_surface` extension per §3.8 (sequential sessions per Q7 (c.ii); fail-loud-audit per Y-PHASE2-DISPATCH-FAILURE-AUDIT-SHAPE (b) if dispatch raises). Add `_fail_scan_phase2_with_audit` helper. ~50-100 LoC.

**C4.2** `tests/services/test_completions_consumer.py`: NEW tests for phase-2 trigger from UPSERT completion + dispatch-failure path emits fail-loud audit + sequential-session pattern verification (handler session commits BEFORE dispatch session opens). ~80-150 LoC.

**C4.3** `tests/integration/test_recon_first_e2e.py` (NEW file): end-to-end test ScanCreate → recon ScanJob → engine RunRecon (mocked via fakeredis publish) → EventAttackSurface → AttackSurface UPSERT → `dispatch_phase2` → per-(target, tool) ScanJobs created → SCAN_DISPATCHED_PHASE2 audit emitted. ~80-150 LoC.

**C4.4** Pre-commit verification: `pytest tests/services/test_completions_consumer.py` + `pytest tests/integration/` + full suite.

**C4.5** Single atomic api commit. Cross-references docs Commit 1 + engine Commit 2 + api Commit 3 hashes concretely.

**Total Stage 3 Commit 4 LoC delta:** ~210-400 LoC.

### Stage 3 Aggregate LoC Forecast

Total across 4 commits: ~610-1073 LoC. Comparable to Task 8.3α Stage 3 trio (+785 LoC) extended by phase-2 dispatch + integration tests + audit-log-lookup complexity. Calibration: ADR-style decision-document implementations may push upper to ~1100-1300 LoC at execution per test-LoC inflation pattern + diagnostic-rich audit.

## 5. D-Deviation Tracking Framework

Per Task 8.3α + M8.1α + M8.1β.1 D-PLAN tracking precedent.

**Pre-execution drifts catalogued (cumulative count 61):** Drift #58 (Task 8.3α) + Drift #59 (Task 8.3α C2 Y-RECON-PUBLISHER-WIRING +3 params soft drift) + Drift #60 (6-engine surface; 6/6 closure at this ADR) + Drift #61 (V-WD identity field empirical drift; Q7.4 refined to Option α).

**Expected Stage 3 D-deviation count:** LOWER bound (~0-3 drifts at execution) per strong-dual-side-analog hypothesis confirmed empirically across prior 3 lifecycles. Honest forecast: 0 drifts at C1 docs (annotation-heavy); 0-1 at C2 engine (small footprint); 1-2 at C3 api orchestrator (largest scope; `dispatch_phase2` novel pattern + V-XB audit-log-lookup canonical-pattern refinement); 0-1 at C4 api completions_consumer + e2e (integration-test novel territory).

**Plan-level Y-decisions to resolve at execution:**

- **Y-AUDIT-LOOKUP-QUERY-SHAPE:** (a) direct ORM query (default; V-XB canonical pattern grounded) vs (b) repository vs (c) service-layer; resolve at C3.4
- **Y-AUTHIDENTITY-RECONSTRUCTION-SHAPE:** (a) minimal (default) vs (b) partial-with-prefix vs (c) lazy; resolve at C3.4 per V-XC `AuthIdentity` empirical shape
- **Y-RECON-SCANJOB-IDEMPOTENCY-KEY:** (a) simple `{scan_id}:recon:{ts}` (default) vs (b) explicit-none vs (c) root-domain-hash; resolve at C3.3
- **Y-PHASE2-DISPATCH-FAILURE-AUDIT-SHAPE:** (b) diagnostic-rich (default) vs (a) generic vs (c) new-event-type; resolve at C4.1

## 6. Out of Scope (per design doc §6 + plan-level refinements)

1. Per-target execution rate-limiting (Q3 forward-pin)
2. Phase-2 dispatch retry logic (Q7 forward-pin)
3. ADR-018 Streams+consumer-groups migration
4. Speculative scaffolding deprecation pass (TOOL-ARCH + SPEC text cleanup post-M8.1β.2)
5. Task 8.3β attack-surface endpoint (forward-pinned post-M8.1β.2)
6. Production-readiness audit (Q10 forward-pin)
7. M9 entry (M8 closure at M8.1β.2 + Task 8.3β)
8. ADR-018 + ADR-019 promotions (M8.1β.2 only closes ADR-020)
9. shieldscan-api modifications outside `services/orchestrator.py` + `services/audit.py` + `services/completions_consumer.py` + `tests/`
10. shieldscan-engine modifications outside `internal/worker/processor.go` + `DRIFT-LOG.md` + tests
11. shieldscan-docs modifications outside SPEC §13 + TOOL-ARCH §8/§10/§12 + SPEC §1885+

## 7. Forward-Pins

**Pre-execution forward-pins (Stage 3 entry):**

1. Stage 3 trigger phrase: ***"Resume M8.1β.2 — Stage 3 cross-repo 4-commit implementation"***
2. Y-AUDIT-LOOKUP-QUERY-SHAPE decision context preserved (a default; resolve C3.4)
3. Y-AUTHIDENTITY-RECONSTRUCTION-SHAPE decision context preserved (a default; resolve C3.4)
4. Y-RECON-SCANJOB-IDEMPOTENCY-KEY decision context preserved (a default; resolve C3.3)
5. Y-PHASE2-DISPATCH-FAILURE-AUDIT-SHAPE decision context preserved (b default; resolve C4.1)
6. Stage 1 design doc canonical authority: `3f07611` §3 + §4 verbatim drafts

**Post-Stage-3 forward-pins:**

7. ***"Begin Task 8.3β attack-surface endpoint task"*** — activation paired with this M8.1β.2 lifecycle close
8. ***"Begin per-target execution rate-limiting"*** — Q3 forward-pin
9. ***"Begin phase-2 dispatch retry logic"*** — Q7 forward-pin
10. ***"Begin speculative scaffolding deprecation pass"*** — TOOL-ARCH + SPEC text cleanup
11. ***"Begin production-readiness audit"*** — Q10 forward-pin
12. ***"Begin M9 entry"*** — after M8 declared CLOSED

**Discipline-level forward-pins:**

13. ***"Integrate audit-driven model+spec orphan check into pre-verification template"*** — Drift #60 rule-of-three trigger
14. ***"Adopt DEFERRED-EMPIRICAL marking for concrete-empirical-field Q-decisions"*** — Drift #61 discipline-level meta-pattern

**In-scope forward-pin closures at Stage 3:** Drift #60 6/6 closure operationally settled (engine `processor.go` recon dispatch case at C2 + api `SCAN_TYPE_TOOLS` rename at C3 + DRIFT-LOG documentation at C2). ADR-020 promotion-trigger closure documented at C1.

## 8. Cross-References

**Engine:** `fc75a98` (Task 8.3α Stage 3 C2; `RunRecon` emission infrastructure for C2 reuse); `9ccde1a` (M8.1β.1 DRIFT-LOG; latest engine state pre-this-lifecycle); `a0bff50` (source-ingestion fix Stage 3 C3); `internal/worker/processor.go` (C2 modification target); `DRIFT-LOG.md` (C2 modification target).

**Docs:** `3f07611` (M8.1β.2 Stage 1 design doc; this plan's canonical authority); `bb3e75f` (M8.1β.1 ADR-022 addendum continuation); `fb8cff9` (M8.1α ADR-022 addendum); `0e5249e` (Task 8.3α P5.A); `0030319` (Task 8.3α design doc structural precedent); `dba6a7c` (Task 8.3α plan precedent at 422 LoC; closest analog); `90fc933` (source-ingestion fix design doc); `04f44a9` (source-ingestion fix plan at 426 LoC); `fdad021` (revocation plan at 338 LoC); `721f788` (R2 plan at 357 LoC). `SPECIFICATION.md` (C1 modification target; §13 ADR-028 + ADR-022 addendum + ADR-020 closure + §1885+ speculative); `TOOL-ARCHITECTURE.md` (C1 modification target; §8.2 + §10.3 + §12.3 speculative markers).

**API:** `d773776` (M8.1β.1 SCAN_TYPE_TOOLS rename + variant config; latest api state pre-this-lifecycle); `05023f4` (Task 8.3α Stage 3 C3; completions_consumer + AttackSurface UPSERT; C4 modification target); `2b36d62` (M8.1α SCAN_TYPE_TOOLS rename); `8dbcbab` (source-ingestion fix Stage 3 C2). Source authorities: `src/app/services/orchestrator.py` (C3 modification target; `ScanOrchestrator.dispatch` + `dispatch_phase2` + `SCAN_TYPE_TOOLS`); `src/app/services/audit.py` (C3.1 modification target; `ScanAction` enum); `src/app/services/completions_consumer.py` (C4.1 modification target; `_handle_attack_surface` extension); `src/app/models/audit.py` (V-XB AuditLog reference); `tests/services/test_orchestrator.py` (C3.5 extension target); `tests/services/test_completions_consumer.py` (C4.2 extension target); `tests/integration/` (C4.3 new file target).

**SPEC sections:** ADR-028 NEW at §13 (canonical authority target); §13 ADR-013 sole-writer (preserved); §13 ADR-014 mixed-primitives (preserved); §13 ADR-017 sequencing (not invoked); §13 ADR-020 reserved-now-closed-empirically at this ADR; §13 ADR-022 + addendums (recon-helper preserved; Drift #60 6/6 closure documented).

**Pre-verification artifacts:** M81_PV + M81B_PV + M81B1_PRE + V-W + V-X surface reports.

**Cumulative drift count:** 61 catches at execution time (Drift #58 + #59 + #60 + #61 catalogued); Drift #60 6/6 closure documented at this lifecycle; Drift #61 V-WD refinement integrated.

**Milestone-completion-constraint context:** locked; M8.1β.2 + Task 8.3β multi-stage close required before M8 declarable CLOSED + M9 entry.
