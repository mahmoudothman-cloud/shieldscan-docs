# M9.D AI Pipeline: Orchestrator — Stage 1 Design Doc (compressed; absorbs Stage 2 plan per Q5)

**Status:** CLOSED at M9.D Stage 3 + P5.A lifecycle closure (2026-07-02). ADR-033 canonical at SPEC §13 (`1583f27`); gap-closure operational across api C1 (`5c092df`; terminal-metadata wiring; V-EEC 8th averted-prediction catch) + C2 (`2f155f0`; 4 e2e tests green first-run; suite 751 ZERO regressions); P5.A Commit 1 docs this commit + Commits 2 (api DRIFT-LOG) + 3 (engine cross-ref) forthcoming. M9.D closes MILESTONE 9 ENTIRELY. Cumulative drift count preserved at 66. Original compressed design+plan content (8 locks + V-EE §7 grounding) preserved below as historical authority.

**Date:** 2026-07-02.

**Authority:** ADR-033 (forthcoming at Stage 3 C0; "AI Pipeline: Orchestrator (M9.D)" at SPEC §13); ADR-013 "Python is the Sole Writer for Scan State" + ADR-014 "Redis Streams (not Pub/Sub)" (both empirically read at V-AAD per the 2-instance ADR-mislabel lesson) + ADR-029 (AI foundation + Q1 e.ii dispatch) + ADR-030/031/032 (pipeline stages) composition. V-AA pre-verification SMALL classification (orchestration operational since M9.0 C2 / Task 4.2; Task 9.7 run_ai_pipeline superseded; V-AAI no-migration). M9.D brainstorming chain CLOSED this session (Mode 2 compressed; 2 gate-decisions + Q1-Q6). 66 cumulative session-tail framing-drift discipline; 7-instance averted-prediction lineage; 3-instance test-gate-within-lock pattern.

**Related:** Task 9.7 (INFORMATIONAL-not-BINDING per Milestone-9 header; superseded by composition; CLOSED-by-composition annotation at P5.A). M9.D is the FINAL M9 sub-milestone — closing it closes M9 (AI Analysis Pipeline) entirely.

---

## 1. Lock Summary (2 gate-decisions + Q1-Q6)

- **Gate-1 Y-M9D-SCOPE-FRAMING = (A) gap-closure.** M9.D = closure-by-composition of the operational M9.0-C2 orchestration + a hardening delta; no new orchestrator code; Task 9.7's `run_ai_pipeline` pseudo-code superseded per the Milestone-9 INFORMATIONAL-not-BINDING header (its steps 1-6 all live inside `pipeline.run()` since M9.A/B/C).
- **Gate-2 Y-FIX-GEN-THROTTLE = (A) honor the M9.C Gate-1 forward-pin.** Fix-gen stays sequential; throttle (Semaphore/asyncio.gather) deferred to the production-readiness empirical trigger; M9.D does NOT touch fix_generation.py (no M9.C-closed artifact reopened).
- **Q1 Y-COMPLETED-AT-WIRING:** consumer terminal paths (COMPLETED/PARTIAL/FAILED) stamp `Scan.completed_at`; CANCELED completed_at forward-pinned to a routes-touch task (avoids reopening the cancel endpoint). Invariant: *consumer terminal status set ⟺ completed_at stamped*.
- **Q2 Y-ERROR-MESSAGE-WIRING:** FAILED writes `error_message = str(exception)` (Task 9.7 shape; at `_fail_scan`); PARTIAL writes a concise job-failure summary ("N of M scan jobs failed; partial results available") derived from sibling ScanJob terminal-failed states; COMPLETED leaves error_message null; `ai_pipeline_degraded` retains the degradation signal (not overloaded into error_message).
- **Q3 Y-WEBHOOKS-DISPOSITION:** forward-pin `fire_scan_completed_webhooks` to Task 12.5 (the distinct webhook feature: delivery/retry/signing/subscription); M9.D lands no stub, no code — documented forward-pin only.
- **Q4 Y-E2E-TEST-SHAPE:** both happy-path (COMPLETED: completed_at stamped + executive_summary present + error_message null) + failure-path (FAILED or PARTIAL: completed_at stamped + error_message populated); reuses M9.C conftest stubs (stub_anthropic + install_openai/qdrant + fakeredis); ~3-5 tests.
- **Q5 Y-STAGE-STRUCTURE:** compressed — this Stage 1 design absorbs the Stage 2 plan content inline; Stage 3 3-commit (C0 ADR-033 docs + C1 ai_pipeline_consumer.py wiring + C2 e2e test); no migration per V-AAI.
- **Q6 Y-ADR-NUMBER:** ADR-033 "AI Pipeline: Orchestrator (M9.D)"; Task 9.7 CLOSED-by-composition annotation at P5.A.

## 2. Architecture Overview

**Gap-closure framing.** The orchestration chain is OPERATIONAL since M9.0 C2 / Task 4.2: CompletionsConsumer (all-jobs-terminal) → `ScanOrchestrator.dispatch_ai_pipeline` (sets ANALYZING + LPUSH `shieldscan:ai_pipeline` + audit) → AIPipelineConsumer (BRPOP drain) → `pipeline.run()` (6 stages: embed → dedup → correlate → score → fix → summary, landed M9.A/B/C) → `_run_pipeline_and_finalize` re-derives terminal status from sibling ScanJobs (PARTIAL if any_failed else COMPLETED) + SCAN_COMPLETED audit; crash path `_fail_scan` (FAILED + ai_pipeline_degraded, fresh session).

**ADR-033 composition:** ADR-013 (Python sole writer for Scan state — M9.D's completed_at/error_message writes are Python-consumer-side, sole-writer compliant) + ADR-014 (Streams/Pub-Sub signaling unchanged) + ADR-029 (AI foundation + Q1 e.ii dispatch) + ADR-030/031/032 (pipeline stages composed in run()).

**M9.D delta (the entire scope):** (1) completed_at write at consumer terminal transitions per Q1; (2) error_message write on FAILED + PARTIAL per Q2; (3) one e2e both-paths test per Q4; (4) ADR-033 documenting closure-by-composition; (5) Task 9.7 CLOSED annotation.

**What M9.D explicitly does NOT do:** no `run_ai_pipeline` (superseded); no new orchestrator code; no fix-gen throttle activation (M9.C pin honored); no webhooks (Task 12.5); no CANCELED completed_at (routes-touch forward-pin); no migration (fields exist).

## 3. Wiring Surface (ai_pipeline_consumer.py)

- **`_run_pipeline_and_finalize` terminal block (~line 156 per V-AAE):** after `scan.status = PARTIAL/COMPLETED`, add `scan.completed_at = datetime.now(UTC)` (Q1); if PARTIAL, derive + write `scan.error_message = f"{n_failed} of {n_total} scan jobs failed; partial results available"` from sibling ScanJob `_TERMINAL_FAILED` states (Q2).
- **`_fail_scan` block (~line 188 per V-AAE):** after `scan.status = FAILED` + `ai_pipeline_degraded = True`, add `scan.completed_at = datetime.now(UTC)` (Q1) + `scan.error_message = str(exception)` (Q2; exception threaded from caller or captured — see V-EEC).
- **datetime import awareness (V-EEB):** confirm datetime + UTC/timezone import present in the consumer module or add.
- **DQ2/DQ4/DQ5 guard interaction (V-EEE):** the ANALYZING idempotency guard (DQ4, ~line 140) + terminal re-derivation (DQ2) + SCAN_COMPLETED audit relocation (DQ5) must not be disturbed — the wiring is additive within the existing terminal block, no guard logic change.

## 4. Test Surface (e2e both-paths per Q4)

New test file OR extension of test_ai_pipeline_consumer.py (V-EEF decides: new `tests/integration/test_m9d_orchestrator_smoke.py` per arc smoke-file precedent OR extend the existing consumer test).

- **Happy-path:** seed scan + raw_findings + sibling ScanJobs all-completed → run consumer finalize → assert status COMPLETED + completed_at not null + executive_summary present + error_message null.
- **Failure-path PARTIAL:** seed scan + sibling ScanJobs with ≥1 failed → assert status PARTIAL + completed_at not null + error_message matches "N of M ... failed" pattern.
- **Failure-path FAILED:** force pipeline.run() exception (stub raises) → assert `_fail_scan` sets FAILED + completed_at not null + error_message = exception str + ai_pipeline_degraded True.
- **completed_at invariant test:** any terminal status ⟹ completed_at stamped.
- Stub reuse: stub_anthropic + install_openai + install_qdrant + fakeredis per M9.C V-CCG conftest precedent. ~3-5 tests.

## 5. Stage 3 Decomposition (Q5 3-commit; no migration per V-AAI)

- **C0** docs ADR-033 SPEC §13 canonical + Task 9.7 activation cross-reference at IMPLEMENTATION-PLAN.md Milestone 9 (~60-90 LoC docs).
- **C1** api ai_pipeline_consumer.py wiring (completed_at + error_message per Q1+Q2; ~15-40 LoC; strongest-risk commit but small; V-EE pre-C1 cascade grounds the terminal-block insertion).
- **C2** api e2e test (both-paths per Q4; ~80-150 LoC; ~3-5 tests).
- Forward-pin: no split needed (small commits).

## 6. Forward-Pins

- Webhooks (`fire_scan_completed_webhooks`) → Task 12.5 (delivery/retry/signing/subscription feature).
- CANCELED completed_at → routes-touch task (cancel endpoint at routes/scans.py; deferred to avoid reopening the cancel path at M9.D).
- Fix-gen throttle (Semaphore/asyncio.gather) → production-readiness per the M9.C Gate-1 pin (empirical cost/rate-limit trigger; M9.D honors).
- Autouse-anthropic-stub suite-speed cleanup → production-readiness (M9.C C1-introduced full-run() real-retry-sleep ~300s; per M9.C P5.A forward-pin).
- Stream-key cleanup TTL → OPS milestone (per ADR-014 open follow-up).

## 7. V-EE Pre-C1 DEFERRED-EMPIRICAL Grounding

Per the 7-instance averted-prediction lineage (extends to 8-instance at the V-EE cascade). These mark testable assumptions deferred to Stage 3 C1 execution; analogous to M9.C V-BB/V-CC operationalization.

- **V-EEA:** ai_pipeline_consumer.py exact terminal-block structure at `_run_pipeline_and_finalize` (~line 156) + `_fail_scan` (~line 188) — insertion points for completed_at/error_message.
- **V-EEB:** datetime + UTC/timezone import presence in ai_pipeline_consumer.py (add if absent; M9.C summary.py precedent used datetime.now).
- **V-EEC (highest-value pre-C1 catch):** `_fail_scan` exception-threading — V-AAE showed signature `_fail_scan(scan_id, org_uuid)`; the exception object may need threading from the caller (or a signature change) for the Q2 FAILED `str(exception)` write.
- **V-EED:** sibling ScanJob `_TERMINAL_FAILED` set + counts access for the PARTIAL job-failure-summary derivation (Q2).
- **V-EEE:** DQ2/DQ4/DQ5 guard structure confirmation — additive wiring must not disturb the ANALYZING idempotency guard + terminal re-derivation + audit relocation.
- **V-EEF:** test conftest stub availability (stub_anthropic + install_openai/qdrant + fakeredis) for the e2e both-paths tests + new-file-vs-extend decision.
