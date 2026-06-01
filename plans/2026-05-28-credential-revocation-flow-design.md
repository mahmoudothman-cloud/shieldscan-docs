# Credential Revocation Flow: Design

**Status:** Brainstorming chain Y1+Y2+Y3+Q1-Q8 complete this session. ADR-015 Q6 (a) revocation forward-pin operationally settled. Ready for implementation plan landing.

**Date:** 2026-05-28.

**Authority:** Y1 (ADR-015 addendum per Q7) + Y2 (direct Q-chain per V-FF+V-FG infrastructure-already-shipped) + Y3 (ADR-015 audit-emission + Task 4.5 cancel-fanout analogs) + Q1 (b) + Q2 (γ) + Q3 (β) + Q4 (a) + Q5 (a) + Q6 (b) + Q7 (b) + Q8 (a) brainstorming chain locks this session; pre-verification surface report V-FA-V-FJ (this session); ADR-015 Q6 (a) forward-pin canonical text (`plans/2026-05-21-adr-015-enablement-design.md`); ADR-015 design doc structural precedent (`b344d0c`; 361 LoC); R2 design doc structural precedent (`b25e9ba`; 252 LoC).

**Related:** Implementation plan landing trigger phrase: ***"Begin credential revocation implementation plan landing"*** (after this design doc lands).

---

## 1. Authority + Brainstorming Chain Summary

**Y1 (locked by Q7) ADR-015 addendum shape:** Credential Lifecycle Extension addendum extends existing ADR-015 (decrypted-credentials-in-Redis-transit) canonical authority with lifecycle-revocation semantics. Mirrors ADR-015 enablement multi-addendum pattern (`9a57865`).

**Y2 (locked by V-FF+V-FG empirical reality) Direct Q-chain:** No Phase 0 v2 territory needed. V-FF confirms api-side Task 4.5 cancel infrastructure shipped (`CancelPublisher` + Redis Pub/Sub + Scan/ScanJob `CANCELED` state). V-FG confirms engine-side abort paths shipped (per-job `ctx.Done` + `spawnCancelWatcher` + 3 goleak-clean exit paths). Cancellation primitives empirically verified; revocation reuses existing infrastructure.

**Y3 (locked by analog availability) ADR-015 + Task 4.5 joint analog:** ADR-015 audit-emission shape (`SCAN_CREDENTIAL_DECRYPTED` + orchestrator-as-sole-writer pattern) applies for revocation audit; Task 4.5 cancel-fanout shape (`CancelPublisher.publish_cancel` scan-id signal) applies for cascade-cancel. No new analog template needed.

**Brainstorming Q-chain locks:**

- **Q1 (b) Axes 1+2+3 v1; axis 4 ZERO scope per V-FG:** 4-axis enumeration from ADR-015 Q6 (a). Axes in scope: (1) revocation API endpoint via Q3 β DELETE extension; (2) in-flight scan handling via Q4 a orchestrator fan-out; (3) scan-job cancellation via Task 4.5 reuse. Axis 4 (worker-side revocation handling) reduces to ZERO scope per V-FG empirical finding — engine treats credential-revocation-cancel identically to user-initiated cancel; no event-payload distinction needed.
- **Q2 (γ) Hard-delete semantics:** Existing DELETE endpoint semantics retained; revocation = delete + cascade-cancel. Preserves ProjectCredential per-project UNIQUE invariant (Drift #41 M1 canonical) without partial-index relaxation. Audit log is canonical revocation history (`PROJECT_CREDENTIAL_REVOKED` + cascade `SCAN_CANCELED` entries). Cannot re-activate revoked credential; user re-sets via PATCH UPSERT after revoke.
- **Q3 (β) Extend existing DELETE endpoint to cascade-cancel:** Single endpoint; consistent with Q2 γ hard-delete-as-revoke. Existing `PROJECT_CREDENTIAL_DELETED` audit preserved for backward-compat semantics; new `PROJECT_CREDENTIAL_REVOKED` audit emitted at revoke-action; existing `SCAN_CANCELED` audit emitted per cascade-canceled scan via Task 4.5 reuse.
- **Q4 (a) Orchestrator-side fan-out via in-flight-scans query + loop CancelPublisher:** Add new `find_in_flight_scans_by_project(project_id) -> list[Scan]` helper; loop existing `CancelPublisher.publish_cancel(scan_id)` per in-flight scan; reuse Task 4.5 `cancel_scan` transactional flow (PG state flip → `CANCELED` + audit row + signal). Per-project in-flight scan count typically small (1-3); fan-out overhead negligible.
- **Q5 (a) No cancel-reason field:** Q1 (b) axis 4 ZERO scope preserved. Engine consumers don't distinguish revocation-cancel from user-cancel at event-payload level. Audit log (`PROJECT_CREDENTIAL_REVOKED` + cascade `SCAN_CANCELED` rows) provides full forensic chain. YAGNI applied.
- **Q6 (b) PROJECT_CREDENTIAL_REVOKED only; reuse SCAN_CANCELED for cascade:** Single new `ProjectAction` value at `audit.py`; cleaner semantic boundary (revoke is project-level; cascade-cancel is scan-level via existing Task 4.5 enum); audit query patterns preserved.
- **Q7 (b) ADR-015 Addendum: Credential Lifecycle Extension:** Lifecycle is operationally coupled to transit; addendum keeps credential-management story in ADR-015 canonical authority. Bounded SPEC delta (~8-15 LoC). Mirrors ADR-015 enablement multi-addendum precedent (`9a57865`).
- **Q8 (a) 2-commit (docs + api) ONLY; engine no-change empirically grounded:** V-FG infrastructure-already-shipped means engine has nothing to change. Symbolic engine commit rejected as ceremonial. Phase 5 sub-phases separate commit per V10/ADR-015 precedent.

**Forward-pin closure:** ADR-015 Q6 (a) revocation forward-pin (3-session deferral via ADR-015 → V10 → R2 → revocation) operationally settles at this task's Stage 3 implementation.

## 2. Pre-Verification Findings

Pre-verification (this session) grounded credential revocation actual scope + ProjectCredential lifecycle + API state + cancellation primitives + audit pattern before brainstorming. Critical findings:

**V-FB ADR-015 Q6 (a) verbatim:** *"Revocation flow (revocation API endpoint + in-flight scan handling + scan-job cancellation + worker-side revocation handling) is multi-axis territory; reserved for separate task. Trigger phrase preserved."* 4 canonical axes enumerated.

**V-FC ProjectCredential lifecycle (current):** `Base + TimestampMixin + TenantMixin` → `created_at` + `updated_at`; NO `deleted_at`, NO `revoked_at`, NO status enum. Per-project UNIQUE constraint canonical (Drift #41 M1; migration `d4f6b1e9a527`). Hard-replace UPSERT + hard-delete pattern. Q2 γ hard-delete-semantics lock preserves this without modification.

**V-FD API credential endpoints (current):** PATCH `/credentials` (UPSERT; `PROJECT_CREDENTIAL_SET` audit) + DELETE `/credentials` (hard delete; `PROJECT_CREDENTIAL_DELETED` audit; idempotent 204). No GET; no list. Q3 β DELETE-extension scope: add cascade-cancel + new `PROJECT_CREDENTIAL_REVOKED` audit + preserve `PROJECT_CREDENTIAL_DELETED`.

**V-FE Scan + ScanJob state machine:** `ScanStatus` has `CANCELED` canonical terminal state (per Task 4.5). Cancel state machine documented at `routes/scans.py:114-122`. No new state needed.

**V-FF Scan-job cancellation primitives (api side; ALREADY SHIPPED):** `cancel_scan` endpoint (Task 4.5) at `routes/scans.py:246-340`; PG state flip → `CANCELED` + `SCAN_CANCELED` audit + `CancelPublisher.publish_cancel` via Redis Pub/Sub; idempotent on already-`CANCELED`; `_maybe_complete_scan` short-circuits on `CANCELED`. `CancelPublisher` at `services/scan_queue.py:224`. Q4 a reuse target.

**V-FG Worker-side abort paths (engine; ALREADY SHIPPED):** `processor.go:181-322` per-job `jobCtx` + `cancel`; `spawnCancelWatcher` consumes Pub/Sub → calls `cancel()` on `jobCtx`; cancel-vs-error-vs-success discrimination at runner-return via `errors.Is(jobCtx.Err(), context.Canceled)`; 3 leak-clean exit paths per ADR-021 Rule 2; `emitCancellation` emits user-initiated-cancel completion event. ZERO engine changes required for revocation per Q1 (b) axis 4 ZERO scope.

**V-FH Audit pattern (current + gap):** `ScanAction` has `SCAN_DISPATCHED` + `SCAN_CANCELED` + `SCAN_COMPLETED` + `SCAN_FAILED` + `SCAN_COMPARED` + `SCAN_CREDENTIAL_DECRYPTED` + `SCAN_PRESIGNED_URL_GENERATED` + `SCAN_JOB_RETRIED`. `ProjectAction` has `PROJECT_CREDENTIAL_SET` + `PROJECT_CREDENTIAL_DELETED`. Gap: `PROJECT_CREDENTIAL_REVOKED` net-new. Q6 b adds this single enum value at `audit.py`.

**V-FI SPEC + ADR coverage (current + gap):** `SPECIFICATION.md` credential infrastructure references at lines 183/286/324/488/1070 (orchestration + transit); revocation prose at 1252-1297 concerns JWT/api_keys (ADR-019 territory; NOT ProjectCredential). ADR-015 v1 explicitly scoped "audit-only; revocation forward-pinned." Q7 b addendum lands ADR-015 Credential Lifecycle Extension to close gap.

**V-FJ Cross-task forward-pin references:** 3 canonical references to *"Begin credential revocation flow task"* across ADR-015 plan/design + engine DRIFT-LOG. No pre-existing partial implementations; no competing forward-pins. Clean entry.

## 3. Architectural Decisions

Cross-references Q1-Q8 brainstorming chain locks (§1) + pre-verification findings (§2).

### 3.1 Q1 (b) Task Scope Decomposition

**Decision:** Axes 1+2+3 v1 (revocation API endpoint + in-flight scan handling + scan-job cancellation via Task 4.5 reuse); Axis 4 (worker-side revocation handling) ZERO scope per V-FG empirical finding.

**Empirical scope reduction:** Original ADR-015 Q6 (a) forecast "~4-6h cross-repo multi-axis" reduces to ~2-3h api-primary + docs because V-FG infrastructure-already-shipped collapses axis 4.

**Rejected alternatives:** Q1 (a) "all 4 axes v1" (axis 4 already-shipped; no work); Q1 (c) "axes 1+3 only" (defers in-flight handling; incomplete revocation semantics).

### 3.2 Q2 (γ) Hard-Delete Semantics

**Decision:** Existing DELETE endpoint semantics retained; revocation = delete + cascade-cancel; ProjectCredential row hard-deleted; no `revoked_at` column; no status enum.

**Per-project UNIQUE invariant preserved:** `ProjectCredential.project_id unique=True` canonical (Drift #41 M1; migration `d4f6b1e9a527`). (γ) preserves this without partial-index complexity.

**Audit log canonical revocation history:** `PROJECT_CREDENTIAL_REVOKED` audit row + cascade `SCAN_CANCELED` audit rows together provide forensic chain. Row-level revocation history (`revoked_at` OR status) would duplicate without operational benefit.

**Trade-off acknowledged:** Cannot re-activate revoked credential. User re-sets via PATCH UPSERT after revoke (existing flow). Credential lifecycle is typically rotate-not-reactivate.

**Rejected alternatives:** Q2 (α) `revoked_at` column + partial index on `(project_id) WHERE revoked_at IS NULL`; Q2 (β) status enum + status-aware UNIQUE. Both require UNIQUE constraint relaxation with subtle UPSERT semantics implications.

### 3.3 Q3 (β) DELETE Endpoint Extension

**Decision:** Extend existing DELETE `/orgs/:org_id/projects/:project_id/credentials` endpoint at `routes/projects.py:462-610` with cascade-cancel logic. Endpoint semantics shift from "remove credential" to "revoke credential (remove + cascade-cancel in-flight scans)".

**Audit emission at endpoint:**

1. `PROJECT_CREDENTIAL_REVOKED` (new; Q6 b lock) emitted FIRST as revoke-action audit
2. ProjectCredential row hard-deleted from DB (Q2 γ; existing pattern)
3. `PROJECT_CREDENTIAL_DELETED` (existing; preserved per backward-compat) emitted after delete
4. Cascade-cancel loop per Q4 a: emit `SCAN_CANCELED` audit (Task 4.5 reuse) + `CancelPublisher.publish_cancel` per in-flight scan

**Idempotency preservation:** Existing DELETE 204 idempotency preserved (already-deleted credential returns 204 silently); revoke audit only emitted on actual revocation (credential row existed).

**Rejected alternatives:** Q3 (α) new POST `.../credentials/revoke` endpoint (separate verb; production-safety-maximalist); Q3 (γ) PATCH with revoke flag (awkward semantics).

### 3.4 Q4 (a) Orchestrator-Side Fan-Out

**Decision:** Add new orchestrator helper `find_in_flight_scans_by_project(project_id) -> list[Scan]` at `services/orchestrator.py` OR `services/scans.py` (location TBD at execution per file-organization conventions); loop existing `CancelPublisher.publish_cancel(scan_id)` per in-flight scan; reuse Task 4.5 `cancel_scan` flow which handles PG state flip → `CANCELED` + `SCAN_CANCELED` audit + signal in one transaction.

**In-flight definition:** `Scan.status IN (QUEUED, RECONNING, RUNNING, ANALYZING)` per V-FE state machine (active states excluding terminal).

**Cascade scope:** Per-project (not org-wide). All scans of the affected project in active states get fan-out cancel.

**Implementation surface:** ~15-25 LoC helper + ~10-15 LoC loop integration in revoke endpoint.

**Rejected alternatives:** Q4 (b) project-id-scoped Pub/Sub channel (broader infrastructure); Q4 (c) DB-direct UPDATE (bypasses Task 4.5 signal-then-state-flip discipline).

### 3.5 Q5 (a) No Cancel-Reason Field

**Decision:** Engine cancel event payload unchanged. No `cancel_reason` field at `JobCancel` event OR `Scan.cancel_reason` DB column.

**Q1 (b) axis 4 ZERO scope preserved.** Engine consumers don't distinguish revocation-cancel from user-cancel at event-payload level.

**Forensic chain via audit log:** `PROJECT_CREDENTIAL_REVOKED` row + temporally-adjacent `SCAN_CANCELED` rows + actor correlation reconstruct revocation-cause from cancel event.

**Rejected alternatives:** Q5 (b) event-payload `cancel_reason` (violates Q1 b); Q5 (c) DB-column `Scan.cancel_reason` (rarely needed; audit join suffices).

### 3.6 Q6 (b) Single Audit Enum Addition

**Decision:** Add `ProjectAction.PROJECT_CREDENTIAL_REVOKED = "project.credential_revoked"` at `src/app/services/audit.py`. Cascade-canceled scans emit existing `ScanAction.SCAN_CANCELED` (Task 4.5 reuse).

**Forensic correlation pattern:** Time-window join on `(PROJECT_CREDENTIAL_REVOKED, SCAN_CANCELED)` audit rows by project + actor + timestamp reconstructs revocation cascade.

**Rejected alternatives:** Q6 (a) dual audit values (`SCAN_CREDENTIAL_REVOKED` + `PROJECT_CREDENTIAL_REVOKED`; enum proliferation); Q6 (c) `SCAN_CREDENTIAL_REVOKED` only (counter-conventional; revoke is project-level).

### 3.7 Q7 (b) ADR-015 Addendum

**Decision:** Land ADR-015 Addendum: Credential Lifecycle Extension at `SPECIFICATION.md` §13 after existing ADR-015 + addendums (chronological order per Task 7.5e + ADR-015 + R2 addendum precedent).

**Addendum content scope (~8-15 LoC):**

- Credential lifecycle revocation semantics (hard-delete + cascade-cancel per Q2 γ + Q3 β + Q4 a locks)
- Audit emission pattern (`PROJECT_CREDENTIAL_REVOKED` + cascade `SCAN_CANCELED` per Q6 b)
- Q1 b axis 4 ZERO engine scope justification (V-FG empirical finding)
- Cross-reference to existing Task 4.5 cancellation infrastructure

**Rejected alternatives:** Q7 (a) new ADR-028 "Credential Revocation Lifecycle" (separate canonical authority; lifecycle distinct-from-transit framing); Q7 (c) dual ADR-013+ADR-015 addendums (ADR-013 already extended via R2 Pre-Signed URL addendum; over-coverage).

### 3.8 Q8 (a) 2-Commit Cross-Repo Shape

**Decision:** 2-commit (docs + api) ONLY. Engine no-change per Q1 (b) axis 4 ZERO scope + V-FG empirical infrastructure-already-shipped.

**Commit 1 (docs):** SPEC §13 ADR-015 Addendum landing (~8-15 LoC).

**Commit 2 (api):** `routes/projects.py` DELETE endpoint extension + `services/audit.py` `ProjectAction` enum + orchestrator/scans helper + test extensions (~80-130 LoC).

**Engine commit rejected as ceremonial:** Symbolic cross-repo coordination shape would distort honest scope-reality. V10 + R2 + ADR-015 trio precedent is shape, not requirement; empirical scope drives commit count.

**Phase 5 sub-phases:** Separate commit per V10/ADR-015 P5 precedent; bounded ~30-60min.

## 4. Cross-Repo Implementation Surface

### 4.1 Docs Side (Stage 3 Commit 1)

**SPECIFICATION.md §13 additions:** ADR-015 Addendum: Credential Lifecycle Extension after existing ADR-015 + Pre-Signed URL addendum (~8-15 LoC).

**Mirrors ADR-015 enablement multi-addendum precedent at `9a57865` + R2 multi-addendum precedent at `8f71b01`.**

**Total docs delta:** ~8-15 LoC.

### 4.2 API Side (Stage 3 Commit 2)

**`src/app/services/audit.py`:** `ProjectAction.PROJECT_CREDENTIAL_REVOKED = "project.credential_revoked"` enum value (~1-2 LoC).

**`src/app/services/scans.py` OR `src/app/services/orchestrator.py`:** `find_in_flight_scans_by_project(project_id)` helper (~15-25 LoC; location TBD per file-organization at execution; lean `orchestrator.py` for parallel to existing dispatch logic).

**`src/app/routes/projects.py`:** DELETE `/credentials` endpoint extension at lines 462-610 area — emit `PROJECT_CREDENTIAL_REVOKED` audit BEFORE delete; preserve existing hard-delete; emit `PROJECT_CREDENTIAL_DELETED` audit AFTER delete (existing); cascade-cancel loop after delete: `for scan in find_in_flight_scans_by_project(): cancel_scan(scan)` (reuses Task 4.5 flow including `SCAN_CANCELED` audit + `CancelPublisher` signal) (~30-50 LoC).

**`tests/routes/test_projects.py`:** Test cases — revoke endpoint emits `PROJECT_CREDENTIAL_REVOKED` + `PROJECT_CREDENTIAL_DELETED` + cascade-canceled scans emit `SCAN_CANCELED` + in-flight scans transition to `CANCELED` + idempotency preserved (already-deleted returns 204; no double-audit) (~40-60 LoC).

**Total api delta:** ~80-130 LoC.

### 4.3 Engine Side (Q8 a; ZERO scope)

NO engine changes. V-FG infrastructure-already-shipped (`processor.go:181-322` cancel-watcher + `ctx.Done` propagation + `emitCancellation`) handles credential-revocation-cancel identically to user-initiated cancel.

### 4.4 Aggregate Stage 3 LoC

Total across 2 commits: ~88-145 LoC (docs ~8-15 + api ~80-130). Smaller than R2 Stage 3 (~200-313 LoC) per axis 4 ZERO scope + reuse pattern; significantly smaller than V10 Stage 3 (~500-770 LoC).

## 5. Phase Structure

Per Q8 (a) 2-commit cross-repo + Y2 direct Q-chain (no Phase 0 v2):

### Stage 1 — Design Doc Landing (THIS COMMIT; this session)

Lands design doc at `plans/2026-05-28-credential-revocation-flow-design.md`. ~280-330 LoC (similar density to ADR-015 + R2 + V10 design docs; smaller Q-chain enumeration since direct-Q-chain reduces architectural-decision §3 footprint vs Phase-0-v2 tasks).

### Stage 2 — Implementation Plan Landing (~30-45min same session if budget allows OR next session)

Lands plan doc at `plans/2026-05-28-credential-revocation-flow-implementation.md` per Task 7.6 + ADR-015 + R2 plan precedent shapes. ~220-280 LoC.

### Stage 3 — 2-Commit Cross-Repo Implementation (~1.5-2.5h; smaller than R2 Stage 3)

Per §4.

### Stage 4 — Phase 5 Sub-Phases

Per Task 7.6 + ADR-015 + R2 Phase 5 precedent. Expected outcome dispositions: P5.A drift annotations (anticipated LOW drift count per pre-verification zero-drift entry); P5.B/C/D/E outcome γ likely.

## 6. Out of Scope

1. Soft-delete revocation column on ProjectCredential (Q2 γ hard-delete-semantics lock)
2. New POST `.../credentials/revoke` endpoint (Q3 β DELETE-extension lock; separate-verb-maximalist forward-pin if production-safety surfaces)
3. Engine `cancel_reason` field at `JobCancel` event (Q5 a ZERO engine scope lock)
4. `Scan.cancel_reason` DB column (Q5 a; audit join suffices)
5. `SCAN_CREDENTIAL_REVOKED` audit enum value (Q6 b single-enum lock; reuse `SCAN_CANCELED`)
6. Project-id-scoped Pub/Sub cancel channel (Q4 a orchestrator-fan-out lock)
7. DB-direct UPDATE in-flight scans bypassing Task 4.5 (Q4 a discipline-preservation)
8. New ADR-028 "Credential Revocation Lifecycle" (Q7 b addendum lock; lifecycle-coupled-to-transit framing)
9. Symbolic engine commit for cross-repo trio pattern-continuity (Q8 a empirical-scope-honesty lock)
10. Credential-revoke audit retention + analytics (out of v1 scope)
11. Re-activation of revoked credential (Q2 γ trade-off; user re-sets via PATCH UPSERT after revoke)
12. Org-wide credential revocation (per-project scope only; org-wide forward-pinned if production need surfaces)

## 7. Forward-Pins

**Pre-execution forward-pins (Stage 2 entry):**

1. **Stage 2 plan trigger phrase:** ***"Begin credential revocation implementation plan landing"***
2. **Y-FILE-LOCATION decision context:** `find_in_flight_scans_by_project` helper at `orchestrator.py` vs `scans.py` (resolve at Stage 3 Commit 2 execution; default `orchestrator.py` per parallel-to-dispatch-logic)
3. **Design doc canonical authority:** This commit's hash for Stage 2 plan reference

**Post-Stage-3 forward-pins:**

4. **Separate POST `.../credentials/revoke` endpoint task** — Q3 α forward-pin if production-safety surfaces semantic-muddling concern with DELETE-extension
5. **Org-wide credential revocation task** — out-of-v1-scope forward-pin if multi-project credential rotation surfaces
6. **Credential re-activation flow task** — Q2 γ trade-off forward-pin if rotate-via-reactivate pattern surfaces (typically YAGNI; rotate-via-replace is standard)

## 8. Cross-References

**Engine commits:**

- `99e2c31` (F2 close; latest engine state; ZERO engine scope for revocation per Q8 a)
- `ad7cc94` (V10 Stage 3 C2; latest engine code touched pre-F2)
- `b48fef8` (ADR-015 Stage 3 C3; SQLMap cookie + Drift #44 + integration test V4 baseline)
- `3ccf5b8` (R2 Stage 3 C3; `http_fetcher` + DRIFT-LOG inline R2 LANDED)
- Task 4.5 `processor.go` cancel infrastructure (V-FG canonical authority for Q1 b axis 4 ZERO scope)

**Docs commits:**

- `88b192c` (R2 P5.A; latest docs state)
- `b25e9ba` (R2 design doc; structural precedent)
- `b344d0c` (ADR-015 design doc; Q6 a forward-pin canonical authority)
- `9a57865` (ADR-015 Stage 3 C1; multi-addendum precedent shape for Q7 b)
- `8f71b01` (R2 Stage 3 C1; multi-addendum continuation precedent)

**API commits:**

- `824853c` (R2 Stage 3 C2; orchestrator dispatch + audit + `SCAN_PRESIGNED_URL_GENERATED` enum precedent for Q6 b `ProjectAction` analog)
- `742faed` (ADR-015 Stage 3 C2; orchestrator credential-decrypt + `SCAN_CREDENTIAL_DECRYPTED` enum precedent)
- Task 4.5 `cancel_scan` endpoint + `CancelPublisher` (V-FF canonical authority for Q4 a reuse)

**SPEC sections:**

- §13 ADR-013 (sole-writer canonical; ProjectCredential lifecycle authority)
- §13 ADR-015 + addendums (credential transit + Q7 b new addendum target)
- §13 ADR-015 v1 audit-only scope (this task lifts the revocation forward-pin)

**Source authorities (Stage 3 sub-step targets):**

- shieldscan-api `src/app/services/audit.py` (Q6 b `ProjectAction` enum addition target)
- shieldscan-api `src/app/services/orchestrator.py` (Q4 a `find_in_flight_scans_by_project` helper target; default location)
- shieldscan-api `src/app/services/scans.py` OR `services/scan_queue.py` (Q4 a alt helper location; existing `cancel_scan` + `CancelPublisher`)
- shieldscan-api `src/app/routes/projects.py:462-610` (Q3 β DELETE endpoint extension target)
- shieldscan-api `tests/routes/test_projects.py` (test extension target)
- shieldscan-api `src/app/models/projects.py` (ProjectCredential model; consumed unchanged per Q2 γ)

**Pre-verification artifacts:**

- V-FA-V-FJ surface report (this session; 50+ cumulative drift discipline; ZERO new drifts)

**Cumulative drift count:** 50 catches at execution time (unchanged through revocation pre-verification; clean entry).
