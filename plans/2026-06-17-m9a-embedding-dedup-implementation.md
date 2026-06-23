# M9.A — AI Pipeline: Embedding + Deduplication (ADR-030): Implementation Plan

**Status:** CLOSED at M9.A Stage 3 + P5.A lifecycle closure (2026-06-23). PY1-PY4 plan-level Y-decisions operational at api C1 (91ec273); Stage 3 3-commit chain landed (C0 docs ADR-030 d408b2c → C1 api pipeline.py rewrite 91ec273 → C2 api tests + smoke 251960a; 35 tests, suite 657 ZERO regressions); Drift #66 FK-ordering catalogued + resolved + regression-guarded; Y2 vulnerability_count forward-pin activated; P5.A lifecycle closure across docs + api/engine DRIFT-LOG #66 sync. Original plan content (PY1-PY4 + Stage 3 sub-step breakdown + D-deviation forecasts) preserved below as historical authority.

**Date:** 2026-06-17.

**Authority:** M9.A Stage 1 design doc (commit aaf7ea0; ADR-030; §1-§9 + Y-PROMOTION-TIMING Path C gate + Q1-Q12 + §9 merge_evidence + project_id gap closures); M9.0 Stage 2 plan structural precedent (commit 55dbe32; 293 LoC §1-§8 + 4 plan-level Y-decisions + Stage 3 sub-step breakdown); ADR-029 canonical authority (commit 45dcabe); M9.0 lifecycle CLOSED (9-commit chain); SPEC §8 canonical M9 authority; 65 cumulative session-tail framing-drift discipline (Drift #58-#65 + #66-averted lineage + V-MM/V-NNE averted at Stage 1 §9).

**Related:** Stage 3 C0 trigger: ***"Begin M9.A — Stage 3 C0 ADR-030 docs landing"***.

---

## 1. Authority + Stage 2 Scope

Q1-Q12 + Y-PROMOTION-TIMING Path C locked at Stage 1 design doc aaf7ea0 §1 + §3; see that doc for full architectural rationale + rejected alternatives. This plan carries the locks forward into an executable 3-commit Stage 3 decomposition (per Q12 (A.a) 3-commit + (B.a) top-down docs-first) and ratifies 4 plan-level Y-decisions (PY1-PY4 §3) for execution-time precision.

**Path C carry-through (Stage 1 §3.0 + §9):** the first finding of each dedup cluster creates a Vulnerability (cluster-representative fields; `project_id` derived from Scan per V-NNE); subsequent matches append `raw_finding_id` via `_merge_evidence`. Y2 (Task 8.3β `vulnerability_count`) activates at M9.A. The `merge_evidence` undefined-helper gap + `RawFinding.project_id` absence are pre-grounded at Stage 1 §9 (averted-prediction discipline).

**In scope:** stages [1] embed + [2] dedup (SPEC §8.1) + Path C promotion. **Out of scope:** M9.B correlate/score, M9.C fix/summary, M9.D orchestrator (§6).

## 2. Implementation Surface (Stage 3 Sub-Step Decomposition per Q12)

### 2.1 Stage 3 Commit 0 — ADR-030 Docs at SPEC §13
**Scope:** Canonical ADR-030 text between ADR-029 (line 2559) and §14 Meta-Principles (line 2632). Structure: Status (Accepted) + Context (M9.A scope; Path C; ADR-029 composition) + Decision (Y-PROMOTION-TIMING + 12 Q-locks summary) + Rationale + Rejected Alternatives + Consequences + Composition with ADR-013/014/022/028/029 + Cross-references.
**Files:** SPECIFICATION.md §13.
**LoC forecast:** ~80-120 LoC.
**Sub-steps:** C0.1 V-PP boundary re-confirm (ADR-029/§14 lines) + ADR-029 format-adapt template; C0.2 craft ADR-030 text from Stage 1 §3; C0.3 insert; C0.4 verify boundary intact + existing text preserved; C0.5 commit + hash.

### 2.2 Stage 3 Commit 1 — api pipeline.py rewrite + call-site + M9.0 test conversions
**Scope:** Replace `run_no_op` with `run()` + module-private stage helpers (`_embed_findings` + `_dedup_and_promote` + `_merge_evidence` + `_create_vulnerability_from_finding`); AIPipelineConsumer call-site update; M9.0 C3 test conversions for `run_no_op` → `run`.
**Files:**
- src/app/services/ai/pipeline.py (rewrite)
- src/app/services/ai_pipeline_consumer.py (call-site update at line 148)
- tests/services/test_ai_pipeline_consumer.py (`run_no_op` → `run` conversions + monkeypatch target update)
- tests/integration/test_m9_smoke.py (`run_no_op` → `run`; with-findings now produces Vulnerability rows → assertion shape)
**LoC forecast:** ~300-500 LoC.
**Sub-steps:** C1.1 V-PP pre-verification (qdrant-client `:memory:` + search/query_points API surface + RawFinding/Scan/Vulnerability fields + Pydantic-not-needed + full `run_no_op` test-conversion scope grep); C1.2 pipeline.py rewrite (~200-300 LoC); C1.3 call-site update (~5 LoC); C1.4 M9.0 test conversions (~80-150 LoC); C1.5 import sanity + ZERO-regression gate (pytest + ruff); C1.6 commit + hash.

### 2.3 Stage 3 Commit 2 — api new tests + M9.A smoke
**Scope:** New test coverage for embed/dedup/promote stage helpers + M9.A end-to-end smoke (Y2 forward-pin activation verification).
**Files:**
- tests/services/ai/test_embeddings.py NEW (embed input construction + batch + 429-fallback)
- tests/services/ai/test_deduplication.py NEW (cluster + threshold + per-scan filter + promotion + merge_evidence)
- tests/services/ai/test_pipeline.py NEW (run() composition + load-query + error propagation)
- tests/integration/test_m9a_smoke.py NEW or extend test_m9_smoke.py (e2e real-pipeline + Y2 vulnerability_count assertion)
**LoC forecast:** ~300-500 LoC.
**Sub-steps:** C2.1 V-QQ pre-verification (test fixture scope + in-memory Qdrant + RawFinding fixture refinements); C2.2 test_embeddings.py; C2.3 test_deduplication.py; C2.4 test_pipeline.py; C2.5 test_m9a_smoke.py; C2.6 full suite ZERO-regression gate + ruff; C2.7 commit + hash.

**Aggregate Stage 3 LoC forecast:** ~680-1120 LoC across 3 commits.

## 3. Plan-Level Y-Decisions

### 3.1 PY1 Y-RAW-FINDINGS-LOAD-QUERY
**Lock:** Eager-load all raw_findings for the scan at `run()` entry via `select(RawFinding).where(RawFinding.scan_id == scan_id)`; sequential processing per Q6 (D.b) two-phase pattern (embed the whole batch, then dedup sequentially).
**Grounding:** per-scan finding counts (~50-200 typical) fit in memory comfortably; streaming complexity is unwarranted at pre-launch scale.
**Rejected alternatives:** (b) streaming `yield_per` (premature optimization); (c) `Scan.raw_findings` relationship lazy-load (async lazy-load complexity vs an explicit query).

### 3.2 PY2 Y-VULNERABILITY-FIELDS-AT-CREATION (per V-NNE pre-grounding)
**Lock:** `_create_vulnerability_from_finding` populates Path C cluster-representative fields: `organization_id` + `project_id` (derived from `Scan.project_id`; V-NNE) + `scan_id` + `title` + `finding_type` + `severity` + `engine_category` + `target_url` + `cwe_id` + `fingerprint` + `raw_finding_ids=[first_id]` + `qdrant_point_id`.
**Grounding:** Vulnerability NOT NULL fields per M9.0 C1 schema (b7e4a1f93c2d); `project_id` derived per V-NNE (RawFinding lacks `project_id`; the pipeline loads the Scan, which has it).
**Rejected alternatives:** including M9.B+ scored fields (cvss_score refinement, correlation) at M9.A — out of scope per Y-PROMOTION-TIMING Path C incremental-field lock (M9.B scoring may update later).

### 3.3 PY3 Y-PIPELINE-RUN-RETURN-SHAPE
**Lock:** `run(*, db, scan_id) -> None` per Q1 (A'.i) rename pattern; side-effects only (Vulnerability rows + raw_finding state + ai_api_calls + Qdrant points); the consumer reads DB state for the terminal transition.
**Grounding:** the AIPipelineConsumer already re-derives terminal status from DB (M9.0 DQ2); a structured return would be unused.
**Rejected alternatives:** structured result dict (over-engineering; consumer uses DB state).

### 3.4 PY4 Y-PIPELINE-ERROR-PROPAGATION
**Lock:** `run()` raises on unrecoverable errors (Qdrant unavailable, OpenAI 429 retries exhausted beyond the Q3 fallback, DB errors); the AIPipelineConsumer catches per its M9.0 C2 (1c98330) handler (`Scan.ai_pipeline_degraded=True` + FAILED transition). The Q3 (D.b) rule-based-fingerprint fallback handles embedding-service-down *within* `run()` (degraded-but-complete) before any propagation.
**Grounding:** the M9.0 C2 consumer fail-path already exists; `run()` propagating aligns with that contract — no duplicate mechanism.
**Rejected alternatives:** (b) catch internally + return a degraded flag (duplicates the M9.0 consumer mechanism; weaker contract).

## 4. Stage 3 Sub-Step Breakdown (Per Commit)

Per Q12 (A.a) 3-commit + (B.a) top-down docs-first. **ZERO-regression-at-each-commit discipline** preserved at every commit gate (full suite green before commit).

- **C0 (docs):** C0.1 V-PP boundary; C0.2-C0.3 craft + insert ADR-030; C0.4 verify; C0.5 commit. Gate: SPEC §13 boundary intact, ADR-029/§14 preserved verbatim.
- **C1 (api impl + conversions):** C1.1 V-PP pre-verification (the highest-grounding step — qdrant API surface + fields + test-conversion scope per Drift #63 extension); C1.2-C1.4 rewrite + call-site + conversions; C1.5 gate (pytest full suite + ruff ZERO net-new F401); C1.6 commit. The behavior change (no-op → real pipeline) + the impacted M9.0 test conversions land in the SAME commit (atomic; no red gate).
- **C2 (api new tests):** C2.1 V-QQ fixture pre-verification; C2.2-C2.5 four test files; C2.6 gate (full suite + ruff); C2.7 commit. Positive new-behavior coverage (embed/dedup/promote + Y2 smoke).

Stage 4 P5.A (separate, post-Stage-3): IMPLEMENTATION-PLAN.md M9.A closure annotation + design/plan Status frontmatter + (if warranted) persistent DRIFT-LOG sync — though M9.A forecasts 0 new drifts, so DRIFT-LOG sync may be unnecessary (determine at P5.A).

## 5. D-Deviation Framework (Per-Commit Drift Forecast)

Pre-execution cumulative: **65** (Drift #58-#65 + #66-averted lineage; V-MM merge_evidence + V-NNE project_id pre-grounded at Stage 1 §9, not catalogued).

### 5.1 Stage 3 C0 ADR-030 Docs
**Forecast: 0 drifts.** Mechanical canonical-text landing; ADR-030 content crafted at Stage 1 §3 + §3.0-§3.12.

### 5.2 Stage 3 C1 api pipeline.py rewrite
**Forecast: 0-2 drifts.** Novel implementation territory; C1.1 V-PP covers qdrant-client `:memory:` + search/query_points API surface + RawFinding/Scan/Vulnerability fields + full `run_no_op` test-conversion scope (Drift #63 test-impact-surface extension; recon-invocation-seam-extension catch-class active). DEFERRED-EMPIRICAL pre-grounding (merge_evidence + project_id at Stage 1 §9) should hold the lower bound — the M9.0 C1 precedent came in at 0 after equivalent pre-grounding.

### 5.3 Stage 3 C2 api new tests + smoke
**Forecast: 0-1 drifts.** Test coverage territory; C2.1 V-QQ covers fixture scope + in-memory Qdrant + RawFinding fixture shape; strong arc test-precedent (M9.0 C3 came in at 0 framing-drift, 2 mechanical refinements absorbed).

**Aggregate Stage 3 forecast: 0-3 drifts** (MODERATE-low bound; pre-grounding-heavy).

## 6. Out of Scope (per design doc §6 + plan-level refinements)

1. M9.B sub-milestone (Tasks 9.3+9.4; correlate + score)
2. M9.C sub-milestone (Tasks 9.5+9.6; fix + summary)
3. M9.D sub-milestone (Task 9.7; orchestrator composition)
4. Cross-scan dedup (M9.0 Q5 forward-pin to M11+)
5. Embedding cache-by-fingerprint (Q4 forward-pin to production-readiness)
6. Tiktoken pre-call estimation (Q8 forward-pin to M9.C high-cost ops)
7. Vulnerability scoring formula per SPEC §8.3 (M9.B; M9.A sets severity from cluster-representative raw_finding only)
8. Vulnerability fix-generation per SPEC §8.4 (M9.C)
9. Cluster-representative refinement by severity (Q6 forward-pin; M9.B scoring)
10. Embedding-dimension tuning beyond text-embedding-3-small 1536 (forward-pin)
11. New schema migration (Q10 — none needed; M9.0 C1 b7e4a1f93c2d satisfies all M9.A locks)

## 7. Forward-Pins

**Pre-execution:**
1. Stage 3 C0 trigger: ***"Begin M9.A — Stage 3 C0 ADR-030 docs landing"***
2. Stage 3 C1 trigger: ***"Begin M9.A — Stage 3 C1 api implementation"*** (carries V-PP qdrant-client `:memory:` + API-surface + test-conversion-scope verification)
3. Stage 3 C2 trigger: ***"Begin M9.A — Stage 3 C2 api tests + smoke"***
4. Stage 4 P5.A trigger: ***"Begin M9.A — Stage 4 P5.A annotations"***

**Sub-milestone activation (post-M9.A close, Q11 strict linear):**
5. ***"Begin M9.B — Correlation + Scoring"***
6. ***"Begin M9.C — Fix Generation + Executive Summary"***
7. ***"Begin M9.D — Pipeline Orchestrator"***

**Discipline-level (active + extended at M9.A):**
8. Audit-driven model+spec orphan check (validated)
9. DEFERRED-EMPIRICAL marking (merge_evidence + RawFinding.project_id pre-grounded at Stage 1 §9)
10. Recon-invocation-seam-extension (pipeline-rewrite seam at C1)
11. Test-impact-surface scope completeness (`run_no_op` → `run` conversions at C1; Drift #63 extension)
12. Plan-template-discipline (this plan §2 anticipates ancillary files — call-site + test conversions enumerated)
13. Averted-prediction discipline (V-MM + V-NNE = 3rd/4th instances)

**Production-readiness audit forward-pin chain (M9.A extensions):**
14. Tune the 0.92 dedup threshold empirically
15. Implement embedding cache-by-fingerprint at scale
16. Add embedding cost telemetry assertions at C2 smoke
17. Add pre-call budget check at `_embed_findings` for very-large-scan scenarios

## 8. Cross-References

**Docs:** aaf7ea0 (Stage 1 design doc canonical authority for this plan) + 62499a3 (M9.0 P5.A docs annotations) + 55dbe32 (M9.0 Stage 2 plan structural precedent) + 45dcabe (ADR-029 SPEC §13 canonical; M9.A composition foundation) + a46fedd (M9.0 Stage 1 design doc); SPEC §8 canonical M9 authority + SPEC §13 ADR-029 (composed) + ADR-030 (Stage 3 C0 target).

**API:** 4616672 (M9.0 P5.A api DRIFT-LOG) + 1c98330 (M9.0 C2 AIPipelineConsumer + dispatch_ai_pipeline; M9.A extends pipeline.py + call-site at line 148) + 51b26ea (M9.0 C1 schema + modules; M9.A consumes operational state) + 8410df4 (M9.0 C3 tests; M9.A converts `run_no_op` references); src/app/services/ai/pipeline.py (rewrite) + ai_pipeline_consumer.py (call-site) + ai/clients.py (qdrant_client + openai_client) + ai/cost_tracking.py (log_ai_call per-batch) + models/raw_findings.py (embedding-input fields; no project_id) + models/vulnerabilities.py (cluster representative + raw_finding_ids + qdrant_point_id).

**Engine:** None (M9.A is api-side only); 6254849 (M9.0 P5.A engine DRIFT-LOG cross-ref) for arc continuity.

**Multi-session arc:** 92 commits + 65 framing-drift catalogue + #66-averted lineage + 18 closed task lifecycles + M5/M6/M7/M8/M9.0 all CLOSED + Y2 Task 8.3β forward-pin activates at M9.A.
