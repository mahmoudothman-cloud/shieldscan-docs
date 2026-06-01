# Credential Revocation Flow: Implementation Plan

**Status:** Ready for Stage 3 cross-repo implementation. Pre-implementation grounded against credential revocation design doc `0e55a4f` Y/Q-chain + pre-verification V-FA-V-FJ. Plan structure grounded against verified ProjectCredential + DELETE endpoint + Task 4.5 cancel infrastructure + engine V-FG already-shipped code shapes.

**Date:** 2026-05-28.

**Authority:** Credential revocation design doc canonical authority (shieldscan-docs commit `0e55a4f`; Y1+Y2+Y3+Q1-Q8 architectural-decision authority); pre-verification surface report (this session; V-FA-V-FJ grounding); R2 implementation plan structural precedent (commit `721f788`; 357 LoC); V10 implementation plan structural precedent (commit `7c4fe75`; 304 LoC); ADR-015 implementation plan structural precedent (commit `00dd2d1`; 278 LoC); Task 7.6 implementation plan structural precedent (commit `40606c5`; 269 LoC); 50+ cumulative session-tail framing-drift discipline (no new revocation drifts pre-verification; clean entry).

**Related:** Stage 3 cross-repo implementation trigger phrase: ***"Resume credential revocation — Stage 3 cross-repo implementation"*** (after this plan lands).

---

## 1. Authority + Scope Lock

**Y1+Y2+Y3 (locked at design doc `0e55a4f` §1):** Y1 ADR-015 addendum shape (Q7 b dependent); Y2 direct Q-chain (V-FF+V-FG infrastructure-already-shipped); Y3 ADR-015 audit-emission + Task 4.5 cancel-fanout joint analog.

**In scope (Stage 3 sub-steps; Q8 a 2-commit shape):**

1. shieldscan-docs `SPECIFICATION.md` §13 ADR-015 Addendum: Credential Lifecycle Extension (Stage 3 Commit 1)
2. shieldscan-api `src/app/services/audit.py` `ProjectAction.PROJECT_CREDENTIAL_REVOKED` enum value (Stage 3 Commit 2)
3. shieldscan-api `src/app/services/orchestrator.py` OR `scans.py` `find_in_flight_scans_by_project(project_id)` helper (Y-FILE-LOCATION resolves at Stage 3 Commit 2 execution; default `orchestrator.py`) (Stage 3 Commit 2)
4. shieldscan-api `src/app/routes/projects.py:462-610` DELETE endpoint cascade-cancel extension (Stage 3 Commit 2)
5. shieldscan-api `tests/routes/test_projects.py` revoke + cascade + idempotency tests (Stage 3 Commit 2)

**Out of scope (forward-pinned per design doc §6):** soft-delete revocation column (Q2 γ lock); new POST `.../credentials/revoke` endpoint (Q3 β lock); engine `cancel_reason` field (Q5 a ZERO engine scope lock); `Scan.cancel_reason` DB column (Q5 a); `SCAN_CREDENTIAL_REVOKED` enum (Q6 b single-enum lock); project-id-scoped Pub/Sub channel (Q4 a); DB-direct UPDATE bypassing Task 4.5 (Q4 a); new ADR-028 (Q7 b addendum lock); symbolic engine commit (Q8 a); credential re-activation (Q2 γ trade-off); org-wide credential revocation (per-project scope only); credential-revoke audit retention + analytics.

**Engine ZERO scope:** V-FG empirical infrastructure-already-shipped (`processor.go:181-322` per-job `jobCtx` + `spawnCancelWatcher` + `ctx.Done` + 3 goleak-clean exit paths + `emitCancellation`); credential-revocation-cancel handled identically to user-initiated cancel; NO engine repo changes. Q8 a 2-commit shape rejects symbolic engine commit as ceremonial.

**Brainstorming chain Q-locks recap:** Y1 + Y2 + Y3 + Q1 (b) axes 1+2+3 v1 + Q2 (γ) hard-delete-semantics + Q3 (β) DELETE-extension + Q4 (a) orchestrator-fan-out + Q5 (a) no-cancel-reason + Q6 (b) `PROJECT_CREDENTIAL_REVOKED`-only + Q7 (b) ADR-015 addendum + Q8 (a) 2-commit per design doc `0e55a4f` §1 + §3.

## 2. Pre-Implementation State

### 2.1 API Infrastructure Ready Matrix (V-FC-V-FH Pre-Verification Findings)

**API side ready (V-FC + V-FD + V-FE + V-FF pre-built):**

- `src/app/models/projects.py`: `ProjectCredential` model with `TimestampMixin` + `TenantMixin` + per-project UNIQUE (Drift #41 M1; migration `d4f6b1e9a527`); NO lifecycle column changes needed per Q2 γ
- `src/app/routes/projects.py:462-610`: existing PATCH `/credentials` (UPSERT + `PROJECT_CREDENTIAL_SET` audit) + DELETE `/credentials` (hard delete + `PROJECT_CREDENTIAL_DELETED` audit + 204 idempotent)
- `src/app/services/audit.py`: `ProjectAction` enum with `PROJECT_CREDENTIAL_SET` + `PROJECT_CREDENTIAL_DELETED`; `ScanAction` enum with `SCAN_CANCELED` + `SCAN_CREDENTIAL_DECRYPTED` + `SCAN_PRESIGNED_URL_GENERATED` + 5 more
- `src/app/models/scans.py`: `ScanStatus` enum with `QUEUED` + `RECONNING` + `RUNNING` + `ANALYZING` (active) + `COMPLETED` + `PARTIAL` + `FAILED` + `CANCELED` (terminal); `CANCELED` canonical per Task 4.5
- `src/app/routes/scans.py:246-340`: `cancel_scan` endpoint (Task 4.5) — PG state flip + `SCAN_CANCELED` audit + `CancelPublisher.publish_cancel` + idempotency
- `src/app/services/scan_queue.py:224`: `CancelPublisher.publish_cancel(scan_id)` → Redis Pub/Sub signal

**API side pending (Stage 3 Commit 2 scope):**

- `src/app/services/audit.py`: `ProjectAction.PROJECT_CREDENTIAL_REVOKED = "project.credential_revoked"` enum value (~1-2 LoC)
- `src/app/services/orchestrator.py` OR `scans.py`: `find_in_flight_scans_by_project(project_id)` helper (~15-25 LoC; Y-FILE-LOCATION resolves at execution)
- `src/app/routes/projects.py:462-610`: DELETE endpoint extension — emit `PROJECT_CREDENTIAL_REVOKED` BEFORE delete; preserve `PROJECT_CREDENTIAL_DELETED`; cascade-cancel loop after delete (~30-50 LoC)
- `tests/routes/test_projects.py`: revoke audit emission + cascade `SCAN_CANCELED` + scan state transitions + idempotency tests (~40-60 LoC)

**Docs side pending (Stage 3 Commit 1 scope):**

- `SPECIFICATION.md` §13 ADR-015 Addendum: Credential Lifecycle Extension (~8-15 LoC)

### 2.2 Engine Infrastructure (V-FG; ZERO scope)

**Engine side ZERO scope per Q1 (b) axis 4 + V-FG empirical finding:**

- `internal/worker/processor.go:181-322`: per-job `jobCtx` + `spawnCancelWatcher` + `ctx.Done` + 3 goleak-clean exit paths (Task 4.5 infrastructure; ADR-021 Rule 2 compliant)
- `internal/worker/cancel_watcher`: Pub/Sub cancel signal consumer
- credential-revocation-cancel = user-initiated-cancel from engine perspective (no payload distinction per Q5 a)

**NO engine repo changes in this task.**

### 2.3 Architectural Analog to ADR-015 + Task 4.5 (V-FH + V-FF grounded)

Revocation task structurally combines two prior analogs:

- ADR-015 enablement audit-emission shape: `SCAN_CREDENTIAL_DECRYPTED` + orchestrator-as-sole-writer ↔ `PROJECT_CREDENTIAL_REVOKED` + DELETE-endpoint-as-sole-writer
- Task 4.5 cancel-fanout shape: scan-id-scoped `CancelPublisher.publish_cancel` + `ctx.Done` propagation ↔ project-id-scoped fan-out via `find_in_flight_scans_by_project` + loop `CancelPublisher` 1:1 reuse

Plan structure inherits ADR-015 implementation plan `00dd2d1` + R2 implementation plan `721f788` shape.

## 3. Architectural Decisions (Plan-Level Locks)

Brainstorming chain Y1+Y2+Y3+Q1-Q8 architectural decisions locked at design doc `0e55a4f`. Plan-level refinements captured below.

### 3.1 Q1 (b) Axes 1+2+3 v1; Axis 4 ZERO Scope

**Implementation surface:** Axes 1 (revocation API endpoint) + 2 (in-flight scan handling) + 3 (scan-job cancellation via Task 4.5 reuse) all api-side. Axis 4 (worker-side revocation handling) ZERO scope per V-FG.

**Empirical scope reduction validated:** ~4-6h cross-repo forecast → ~1.5-2.5h api-primary + docs.

### 3.2 Q2 (γ) Hard-Delete Semantics — Implementation Site

**No `ProjectCredential` model changes.** Existing `TimestampMixin` + `TenantMixin` + per-project UNIQUE preserved. Existing hard-replace UPSERT + hard-delete pattern preserved.

**Migration impact:** ZERO. No DB schema changes.

### 3.3 Q3 (β) DELETE Endpoint Extension

**Implementation surface:** `src/app/routes/projects.py` DELETE `/credentials` at lines 462-610 area. Sequence per Y-AUDIT-EMISSION-ORDER (a) default:

```python
# Pseudo-code structure
async def delete_project_credential(...):
    existing = await db.scalar(...)  # find ProjectCredential by project_id
    if not existing:
        return Response(status_code=204)  # Y-IDEMPOTENCY-SEMANTICS (a): silent 204; no audit

    # Q6 b: PROJECT_CREDENTIAL_REVOKED audit FIRST (Y-AUDIT-EMISSION-ORDER a)
    await audit(session=db, organization_id=..., actor_id=actor_id,
                action=ProjectAction.PROJECT_CREDENTIAL_REVOKED,
                resource_type="project", resource_id=str(project_id),
                ip_address=..., user_agent=..., details={...})

    # Q2 γ: existing hard-delete pattern preserved
    await db.delete(existing)
    await db.commit()

    # Preserve existing PROJECT_CREDENTIAL_DELETED audit
    await audit(session=db, organization_id=..., actor_id=actor_id,
                action=ProjectAction.PROJECT_CREDENTIAL_DELETED,
                resource_type="project", resource_id=str(project_id),
                ip_address=..., user_agent=..., details={...})

    # Q4 a: cascade-cancel fan-out (reuses Task 4.5 cancel_scan flow)
    in_flight_scans = await find_in_flight_scans_by_project(db, project_id)
    for scan in in_flight_scans:
        await cancel_scan(db=db, scan_id=scan.id, actor_id=actor_id, ...)
        # cancel_scan internally: PG state flip → CANCELED + SCAN_CANCELED audit + CancelPublisher signal

    return Response(status_code=204)
```

~30-50 LoC delta.

**Plan-level Y-decisions:**

- Y-AUDIT-EMISSION-ORDER (a) default — REVOKED → delete → DELETED → cascade (revoke-action precedes delete-effect; cascade after credential row gone)
- Y-IDEMPOTENCY-SEMANTICS (a) default — silent 204 for already-deleted (no audit emission; preserves existing pattern)

### 3.4 Q4 (a) Orchestrator-Side Fan-Out — Implementation Site

**Implementation surface:** `find_in_flight_scans_by_project(db, project_id)` helper.

```python
async def find_in_flight_scans_by_project(
    db: AsyncSession, project_id: UUID,
) -> list[Scan]:
    """Find all scans in active (non-terminal) states for a project.
    Active states per V-FE state machine: QUEUED, RECONNING, RUNNING, ANALYZING.
    Excludes terminal states: COMPLETED, PARTIAL, FAILED, CANCELED.
    """
    stmt = select(Scan).where(
        Scan.project_id == project_id,
        Scan.status.in_([
            ScanStatus.QUEUED,
            ScanStatus.RECONNING,
            ScanStatus.RUNNING,
            ScanStatus.ANALYZING,
        ]),
    )
    result = await db.scalars(stmt)
    return list(result.all())
```

~15-25 LoC.

**Plan-level Y-decision Y-FILE-LOCATION:** (a) `orchestrator.py` (default; parallel to existing dispatch logic + `find_*`-style helpers) OR (b) `scans.py` (parallel to scan CRUD) OR (c) `scan_queue.py` (parallel to `CancelPublisher` co-location). Default (a) per design doc §4.2 lean; resolve at Stage 3 Commit 2 C2.2 execution per file-organization-convention surface.

**Cascade loop integration:** in DELETE endpoint after credential delete, loop `find_in_flight_scans_by_project` result + invoke existing `cancel_scan(scan_id)` per scan. `cancel_scan` internally handles PG state flip + `SCAN_CANCELED` audit + `CancelPublisher` signal. Task 4.5 reuse 1:1.

### 3.5 Q5 (a) No Cancel-Reason Field — ZERO Engine Scope Preserved

**Decision:** Engine cancel event payload unchanged. No `JobCancel.reason` field; no `Scan.cancel_reason` DB column.

**Q1 (b) axis 4 ZERO scope preserved in implementation.**

### 3.6 Q6 (b) Single Audit Enum Addition — Implementation Site

**Implementation:** `ProjectAction` enum addition at `src/app/services/audit.py`:

```python
class ProjectAction(str, Enum):
    PROJECT_CREDENTIAL_SET = "project.credential_set"
    PROJECT_CREDENTIAL_DELETED = "project.credential_deleted"
    PROJECT_CREDENTIAL_REVOKED = "project.credential_revoked"  # NEW; Q6 b
```

~1-2 LoC.

**Cascade scan audits reuse `SCAN_CANCELED` (existing Task 4.5 enum value).** No `SCAN_CREDENTIAL_REVOKED`.

### 3.7 Q7 (b) ADR-015 Addendum — Implementation Site (Stage 3 Commit 1)

**SPECIFICATION.md §13 addition:** ADR-015 Addendum: Credential Lifecycle Extension after existing ADR-015 + ADR-015 Pre-Signed URL addendum (chronological order).

**Addendum content scope (~8-15 LoC):**

- Credential lifecycle revocation semantics (hard-delete + cascade-cancel per Q2 γ + Q3 β + Q4 a locks)
- Audit emission pattern (`PROJECT_CREDENTIAL_REVOKED` + cascade `SCAN_CANCELED` per Q6 b)
- Axis 4 ZERO engine scope justification (V-FG empirical finding)
- Cross-reference to existing Task 4.5 cancellation infrastructure

**Total Stage 3 Commit 1 LoC delta:** ~8-15 LoC.

### 3.8 Q8 (a) 2-Commit Cross-Repo Shape — Sequencing

**Commit 1 (docs FIRST; ~8-15 LoC):** SPEC §13 ADR-015 Addendum: Credential Lifecycle Extension (mirrors ADR-015 enablement multi-addendum precedent `9a57865` + R2 multi-addendum continuation `8f71b01`).

**Commit 2 (api SECOND; ~80-130 LoC):** `audit.py` `ProjectAction` enum + `orchestrator.py`/`scans.py` `find_in_flight_scans_by_project` helper + `routes/projects.py` DELETE endpoint extension + `test_projects.py` revoke + cascade + idempotency tests.

**Engine NO COMMIT per Q8 (a) empirical-scope-honesty.** V-FG infrastructure-already-shipped means engine has nothing to change. Symbolic engine commit rejected as ceremonial.

**Cross-reference shape:** Commit 1 docs references future api hash via placeholder; Commit 2 api references docs Commit 1 hash concretely + engine "no commit per V-FG" note.

## 4. Stage 3 Sub-Step Breakdown

### Stage 3 Commit 1 — Docs (SPEC ADR-015 Addendum) (~15-25min)

**C1.1** Locate ADR-015 section in `SPECIFICATION.md`; identify existing ADR-015 + Pre-Signed URL addendum chronological order for new Credential Lifecycle Extension addendum insertion. grep ADR-015 references.

**C1.2** Append ADR-015 Addendum: Credential Lifecycle Extension after existing ADR-015 + ADR-015 Pre-Signed URL addendum. Content per design doc §3.7. ~8-15 LoC.

**C1.3** Pre-commit verification: `grep "ADR-015 Addendum" SPECIFICATION.md` (verify ≥3 matches: existing ADR-015-enablement + R2 Pre-Signed URL + new Credential Lifecycle Extension); `wc -l SPECIFICATION.md` (delta ~8-15 LoC).

**C1.4** Single atomic commit; cross-references future api Commit 2 hash via placeholder + engine no-commit note.

**Total Stage 3 Commit 1 LoC delta:** ~8-15 LoC.

### Stage 3 Commit 2 — API (audit + orchestrator/scans helper + DELETE extension + tests) (~60-90min)

**C2.1** `audit.py`: `ProjectAction.PROJECT_CREDENTIAL_REVOKED` enum value addition. ~1-2 LoC.

**C2.2** `orchestrator.py` OR `scans.py`: `find_in_flight_scans_by_project` helper per Y-FILE-LOCATION (a default; verify at execution per file-organization convention surface). ~15-25 LoC.

**C2.3** `routes/projects.py` DELETE endpoint extension at lines 462-610 area per Y-AUDIT-EMISSION-ORDER (a) + Y-IDEMPOTENCY-SEMANTICS (a) defaults: emit `PROJECT_CREDENTIAL_REVOKED` audit BEFORE delete; preserve `PROJECT_CREDENTIAL_DELETED`; cascade-cancel loop after delete. ~30-50 LoC.

**C2.4** `tests/routes/test_projects.py`: `test_delete_credential_emits_revoked_audit` + `test_delete_credential_cascades_in_flight_scan_cancel` + `test_delete_credential_idempotency_silent_204` + `test_delete_credential_no_in_flight_scans_no_cascade`. Mock `cancel_scan` + `CancelPublisher`; assert audit rows + scan state transitions + idempotency. ~40-60 LoC.

**C2.5** Pre-commit verification: `pytest tests/routes/test_projects.py -v` (existing tests green + 4 new revoke+cascade+idempotency tests green); `pytest tests/routes/` green (module-level no regressions); `pytest` green (full suite no regressions).

**C2.6** Single atomic commit; cross-references docs Commit 1 hash concretely + engine no-commit per V-FG note.

**Total Stage 3 Commit 2 LoC delta:** ~80-130 LoC.

### Stage 3 Aggregate LoC Forecast

Total across 2 commits: ~88-145 LoC (docs ~8-15 + api ~80-130). Smaller than R2 Stage 3 (~200-313); significantly smaller than V10 Stage 3 (~500-770) per axis 4 ZERO scope + reuse pattern.

## 5. D-Deviation Tracking Framework

Per Task 7.6 + ADR-015 + V10 + R2 D-PLAN tracking precedent.

**Pre-execution drifts catalogued:** None (pre-verification surface report this session caught zero drifts; calibration accurate; cumulative count stays at 50).

**Expected Stage 3 D-deviation count:** LOW. Pre-verification grounded all V-items + canonical authority text; strong dual analog (ADR-015 + Task 4.5) directly applicable; bounded LoC scope; engine ZERO scope eliminates concurrency drift surface. Expected ~1-3 drifts at execution per typical pattern (compare R2 Stage 3 trio surfaced ZERO drifts entire trio; V10 Stage 3 surfaced 3 pre-execution static + 0 post-execution; ADR-015 Stage 3 surfaced 5 drifts). Revocation should land closer to R2 ZERO-drift level given strong dual analog template.

**Plan-level Y-decisions to resolve at execution:**

- Y-FILE-LOCATION: `orchestrator.py` (a) vs `scans.py` (b) vs `scan_queue.py` (c); default (a) per parallel-to-dispatch-logic; resolve at Stage 3 Commit 2 C2.2 execution
- Y-AUDIT-EMISSION-ORDER: REVOKED→delete→DELETED→cascade (a) vs alternatives; default (a) per revoke-action-precedes-delete-effect; resolve at Stage 3 Commit 2 C2.3 execution
- Y-IDEMPOTENCY-SEMANTICS: silent 204 (a) vs always-audit (b) vs 404-not-found (c); default (a) per existing DELETE pattern preservation; resolve at Stage 3 Commit 2 C2.3 execution

## 6. Out of Scope (per design doc §6 + plan-level refinements)

1. Soft-delete revocation column on ProjectCredential (Q2 γ lock)
2. New POST `.../credentials/revoke` endpoint (Q3 β lock; forward-pinned if production-safety surfaces)
3. Engine `cancel_reason` field at `JobCancel` event (Q5 a ZERO engine scope lock)
4. `Scan.cancel_reason` DB column (Q5 a; audit join suffices)
5. `SCAN_CREDENTIAL_REVOKED` enum value (Q6 b single-enum lock)
6. Project-id-scoped Pub/Sub channel (Q4 a)
7. DB-direct UPDATE bypassing Task 4.5 (Q4 a)
8. New ADR-028 "Credential Revocation Lifecycle" (Q7 b addendum lock)
9. Symbolic engine commit (Q8 a empirical-scope-honesty lock)
10. Credential re-activation flow (Q2 γ trade-off; forward-pinned if rotate-via-reactivate surfaces)
11. Org-wide credential revocation (per-project scope only; forward-pinned if multi-project rotation surfaces)
12. Credential-revoke audit retention + analytics (out of v1 scope)
13. shieldscan-api modifications outside `audit.py` + `orchestrator.py`/`scans.py` + `routes/projects.py` + tests
14. shieldscan-engine modifications (Q8 a engine ZERO scope; nothing to modify)

## 7. Forward-Pins

**Pre-execution forward-pins (Stage 3 entry):**

1. **Stage 3 trigger phrase:** ***"Resume credential revocation — Stage 3 cross-repo implementation"***
2. **Y-FILE-LOCATION decision context:** `find_in_flight_scans_by_project` helper at `orchestrator.py` (a) vs `scans.py` (b) vs `scan_queue.py` (c); default (a) per parallel-to-dispatch; resolve at C2.2
3. **Y-AUDIT-EMISSION-ORDER decision context:** REVOKED→delete→DELETED→cascade (a) default per revoke-action-precedes-delete-effect; resolve at C2.3
4. **Y-IDEMPOTENCY-SEMANTICS decision context:** silent 204 (a) default per existing pattern; resolve at C2.3
5. **Design doc canonical authority:** `0e55a4f` §3 + §4 verbatim drafts

**Post-Stage-3 forward-pins:**

6. **Separate POST `.../credentials/revoke` endpoint task** — Q3 α forward-pin if production-safety surfaces
7. **Org-wide credential revocation task** — out-of-v1-scope forward-pin
8. **Credential re-activation flow task** — Q2 γ trade-off forward-pin (typically YAGNI)

## 8. Cross-References

**Engine commits:**

- `99e2c31` (F2 close; latest engine state; ZERO engine scope per Q8 a)
- `3ccf5b8` (R2 Stage 3 C3; `http_fetcher` + DRIFT-LOG inline R2 LANDED)
- `ad7cc94` (V10 Stage 3 C2)
- `b48fef8` (ADR-015 Stage 3 C3; Task 4.5 cancel infrastructure foundation per V-FG)

**Docs commits:**

- `0e55a4f` (revocation Stage 1 design doc; this plan's canonical authority)
- `88b192c` (R2 P5.A; latest docs state pre-revocation)
- `b25e9ba` (R2 design doc; structural precedent)
- `721f788` (R2 plan; structural precedent for this plan)
- `b344d0c` (ADR-015 design doc; Q6 a forward-pin canonical authority)
- `00dd2d1` (ADR-015 plan; structural precedent)
- `9a57865` (ADR-015 Stage 3 C1; multi-addendum precedent for Q7 b)
- `8f71b01` (R2 Stage 3 C1; multi-addendum continuation)
- `40606c5` (Task 7.6 plan; structural precedent)

**API commits:**

- `824853c` (R2 Stage 3 C2; orchestrator + `SCAN_PRESIGNED_URL_GENERATED` enum precedent for Q6 b `ProjectAction` analog)
- `742faed` (ADR-015 Stage 3 C2; orchestrator credential-decrypt + `SCAN_CREDENTIAL_DECRYPTED` + Drift #42 emission-site refactor precedent)
- Task 4.5 `cancel_scan` endpoint + `CancelPublisher` (V-FF canonical authority; `routes/scans.py:246-340` + `services/scan_queue.py:224`)

**SPEC sections:**

- §13 ADR-013 (sole-writer canonical; ProjectCredential lifecycle authority)
- §13 ADR-015 + addendums (Q7 b new addendum target)
- §13 ADR-015 v1 audit-only scope (this task lifts revocation forward-pin)

**Source authorities (Stage 3 sub-step targets):**

- shieldscan-api `src/app/services/audit.py` (C2.1 modification target)
- shieldscan-api `src/app/services/orchestrator.py` (C2.2 modification target; default Y-FILE-LOCATION a)
- shieldscan-api `src/app/services/scans.py` (C2.2 alt target; Y-FILE-LOCATION b)
- shieldscan-api `src/app/services/scan_queue.py` (C2.2 alt target; Y-FILE-LOCATION c; co-located with `CancelPublisher`)
- shieldscan-api `src/app/routes/projects.py` (C2.3 modification target; DELETE endpoint extension)
- shieldscan-api `tests/routes/test_projects.py` (C2.4 modification target)
- shieldscan-api `src/app/models/projects.py` (`ProjectCredential` model; consumed unchanged per Q2 γ)
- shieldscan-api `src/app/routes/scans.py:246-340` (Task 4.5 `cancel_scan`; reused via Q4 a)
- shieldscan-api `src/app/services/scan_queue.py:224` (`CancelPublisher`; reused via Q4 a)

**Pre-verification artifacts:**

- V-FA-V-FJ surface report (this session)

**Cumulative drift count:** 50 catches at execution time (zero new pre-verification drifts; clean entry into Stage 3).
