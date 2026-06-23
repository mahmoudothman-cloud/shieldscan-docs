# M9.A — AI Pipeline: Embedding + Deduplication (ADR-030): Design

**Status:** CLOSED at M9.A Stage 3 + P5.A lifecycle closure (2026-06-23). ADR-030 canonical at SPEC §13 (d408b2c); Path C operational at api C1 (91ec273; Q5 B.c→query_points + Q9 B.a→`AsyncQdrantClient(":memory:")` deferred-resolutions locked at execution; Drift #66 FK-ordering catalogued + resolved); tests + Y2 activation at C2 (251960a; 35 tests, suite 657 ZERO regressions); P5.A lifecycle closure across docs + api/engine DRIFT-LOG #66 sync. Original brainstorming-chain content (Y-PROMOTION-TIMING Path C + Q1-Q12 + 35+ sub-decisions + §9 V-MM/V-NNE pre-grounding) preserved below as historical authority.

**Date:** 2026-06-16.

**Authority:** V-MM pre-verification surface report (this session; M9.A architectural decision space grounded; merge_evidence undefined-helper gap surfaced + averted via Path C lock; Y2 vulnerability_count forward-pin activates at M9.A); M9.A brainstorming chain CLOSED this session (Mode 1 sequential conversational; Y-PROMOTION-TIMING Path C gate-decision + Q1-Q12 + 35+ sub-decisions ratified); V-NN pre-Stage-1 verification (this session); M9.0 Stage 1 design doc structural precedent (commit a46fedd; 240 LoC §1-§9); M9.0 Stage 2 plan + Stage 3 C0 ADR-029 + C1 schema/modules + C2 consumer/dispatch + C3 tests/smoke + P5.A annotations (commits 55dbe32 + 45dcabe + 51b26ea + 1c98330 + 8410df4 + 62499a3 + 4616672 + 6254849); ADR-029 canonical authority at SPEC §13 (M9.0 architectural foundation; M9.A composes); SPEC §8.1 pipeline stages (M9.A = stages [1] embed + [2] dedup); SPEC §8.5 cost targets ($0.02/1M tokens for text-embedding-3-small); SPEC §8.6 error-recovery fallback matrix (rule-based fingerprint fallback for embedding service unavailable per Q3 D.b); CLAUDE.md Gotcha 5 cost-tracking mandate (operational at M9.0 ai_api_calls; activates at M9.A first real-cost logging); 65 cumulative session-tail framing-drift discipline (Drift #58-#65 + #66-averted lineage).

**Related:** Implementation plan landing trigger: ***"Begin M9.A — Stage 2 implementation plan landing"***. Sub-milestone activation triggers after M9.A closes: ***"Begin M9.B — Correlation + Scoring"*** → ***"Begin M9.C — Fix Generation + Executive Summary"*** → ***"Begin M9.D — Pipeline Orchestrator"***.

---

## 1. Authority + Y-PROMOTION-TIMING + Q1-Q12 Locks Summary

**Y-PROMOTION-TIMING gate-decision (ratified upfront before the Mode 1 chain):**

**Path C — Promote-at-M9.A with incremental fields.** The first finding of each dedup cluster creates a Vulnerability row with cluster-representative fields; subsequent matches append `raw_finding_id` to the existing `Vulnerability.raw_finding_ids` via the `_merge_evidence` helper. Matches the M9.0 Q6 (A.c) hybrid clustering + traceability + (B.a) raw_finding promotion-state lock. The Y2 `vulnerability_count` forward-pin from Task 8.3β activates at M9.A. The `merge_evidence` undefined-helper gap from V-MM is resolved: it operates on a Vulnerability instance via `raw_finding_ids` array append.

Brainstorming chain (Q1-Q12) resolved under Mode 1 sequential conversational deliberation (~1.5h):

**Q1 Y-PIPELINE-WIRING → (A') Replace run_no_op with run() + module-private stage helpers + (A'.i) rename run_no_op → run.** `pipeline.py` exports `run(db, scan_id)`; internal helpers `_embed_findings` + `_dedup_and_promote` compose the stages; the AIPipelineConsumer call-site updates; M9.B/C/D extend the `run()` composition.

**Q2 Y-EMBEDDING-INPUT-CONSTRUCTION → (A.b) extended field set + (B.c) field-label structure + (C.c) label-prefixed newline-separator format.** Construction: title + finding_type + cwe_id + target_url + description with `"Field: value/N/A"` labels. target_url inclusion prevents endpoint-level vulnerability conflation.

**Q3 Y-EMBEDDING-BATCH-SHAPE → (A.a) batch=100 + (B.a) sequential chunking + (C.c) hybrid SDK-transient + manual 429 exp-backoff + (D.b) rule-based fingerprint fallback per SPEC §8.6.** Embedding service exhausts retries → fall back to `raw_finding.fingerprint` clustering + `Scan.ai_pipeline_degraded=True`.

**Q4 Y-EMBEDDING-CACHE-STRATEGY → (C') No cache at M9.A.** Cache strategy forward-pinned to the production-readiness audit; pre-launch context discipline (~$0.001/scan embedding cost negligible at pre-launch scale).

**Q5 Y-QDRANT-OPERATIONS-API → (A.c) _ensure_collection_exists helper + (B.c) V-NN-Stage-3 defer search/query_points + (C.a') uuid.uuid5(NAMESPACE, fingerprint) deterministic UUID + (D.b) extended payload (raw_finding_id + scan_id + organization_id + cwe_id + target_url + fingerprint + vulnerability_id).** The (D.b) payload was back-amended at Q6 to include `vulnerability_id` for cluster-hit lookup.

**Q6 Y-DEDUP-ALGORITHM-DETAILS → (A.a) limit=1 + score_threshold=0.92 + (B.a) filter to current scan only + (C.a) first-emitted cluster representative + (D.b) two-phase embed-batch-then-dedup-sequential.** Server-side threshold per SPEC §8.1; Q5-M9.0 per-scan dedup lock taken literally; M9.B may update `Vulnerability.severity` per SPEC §8.3 scoring.

**Q7 Y-MERGE-EVIDENCE-SHAPE → (A.a) append-only + (B.a) atomic-same-flush + (C.a) Python read-modify-write reassignment + (D.b) _merge_evidence module-private receiving a Vulnerability instance + Resolution γ pre-generated UUID.** The reassignment pattern (`vuln.raw_finding_ids = [*existing, new_id]`) forces the SQLAlchemy dirty flag; the pre-generated `vulnerability_id` avoids an intermediate flush.

**Q8 Y-COST-TRACKING-INTEGRATION-POINTS → (A.a') no pre-call at M9.A + (B.a) per-batch logging + (C.a) API-response usage field.** Honors the Q2-M9.0 (B.c) hybrid lock (embedding = low-cost = post-call only); `operation_type="embedding"`; `tokens_in = batch.usage.prompt_tokens`.

**Q9 Y-TEST-FIXTURE-STRATEGY → (A.b) unittest.mock patching client.embeddings.create + (B.a) in-memory Qdrant via :memory: mode + (C.a) function-scoped fixtures + (D.b) shared parametrized fixtures.** V-NN Stage 3 verification dependency for qdrant-client `:memory:` support; real similarity behavior preserved via in-memory mode.

**Q10 Y-MIGRATION-NEEDED → NO new schema at M9.A.** The M9.0 C1 schema (`b7e4a1f93c2d`) fully satisfies all M9.A Y-locks; sub-milestone-specific migrations forward-pinned for M9.B/C/D if needed.

**Q11 Y-ADR-NUMBER → ADR-030 "AI Pipeline: Embedding + Deduplication (M9.A)".** Composes the ADR-029 architectural foundation.

**Q12 Y-STAGE3-DECOMPOSITION-FOR-M9.A → (A.a) 3-commit + (B.a) top-down docs-first.** C0 docs (~80-120 LoC) + C1 api pipeline.py rewrite + call-site + M9.0 test conversions (~300-500 LoC) + C2 api new tests + smoke (~300-500 LoC); aggregate ~680-1120 LoC.

## 2. Pre-Verification Findings

**V-MM (this session):** M9.A architectural decision space grounded; `merge_evidence` undefined-helper gap surfaced from plan-literal pseudo-code; Y2 `vulnerability_count` contingent-timing identified; Y-PROMOTION-TIMING central architectural fork identified. ~10-12 Y-decisions territory preliminary estimate. SPEC §8.1 M9.A scope boundary confirmed (stages [1] embed + [2] dedup; M9.B = correlate + score; M9.C = fix + summary; M9.D = orchestrator). M9.0 foundation operational state empirically re-verified (`AIPipelineConsumer._run_pipeline_and_finalize` calls `run_no_op` — the M9.A integration seam).

**V-NN pre-Stage-1 verification (this session):** V-NNA clean state (docs 62499a3 + engine 6254849 + api 4616672); V-NNB ADR-030 SPEC §13 insertion point (after ADR-029 line 2559, before §14 line 2632); V-NNC §1-§9 structural precedent confirmed; V-NND Task 9.1+9.2 Status annotation context; V-NNE **RawFinding has scan_id + engine_category + severity but NOT project_id** → Path C `_create_vulnerability_from_finding` must derive `project_id` from `Scan.project_id` (the pipeline holds `scan_id`; load the Scan once). DEFERRED-EMPIRICAL grounding (see §9); not a Y-lock drift.

## 3. Architectural Decisions (Y-Lock Detail)

### 3.0 Y-PROMOTION-TIMING (Path C) Gate-Decision
**Lock:** Promote-at-M9.A with incremental fields per Path C; matches the M9.0 Q6 (A.c) hybrid clustering + traceability + (B.a) raw_finding promotion-state lock literally.
**Grounding:** The M9.0 Q6 ratification anticipated cluster-time Vulnerability creation; Path C makes this explicit at M9.A entry and resolves the `merge_evidence` undefined-helper gap from V-MMB.
**Rejected alternatives:** Path A (full Vulnerability creation with all scored fields at clustering; semantically heavier, pulls M9.B scoring forward); Path B (defer promotion to M9.C/D; requires Q6 reinterpretation; Y2 activation deferred past M9.A).
**Cascading implications:** Y2 `vulnerability_count` (Task 8.3β) activates at M9.A; `_merge_evidence` operates on a Vulnerability instance with `raw_finding_ids` append; `_create_vulnerability_from_finding` derives `project_id` from Scan (V-NNE).

### 3.1 Y-PIPELINE-WIRING (A' + A'.i)
**Lock:** Replace `run_no_op` with `run()` + module-private stage helpers (`_embed_findings` + `_dedup_and_promote` + `_merge_evidence` + `_create_vulnerability_from_finding`); rename `run_no_op` → `run`; M9.B/C/D extend the `run()` composition.
**Grounding:** Arc functional-style precedent (`cost_tracking` + `clients` are function-style); `run_no_op` was always scaffold for the M9.A+ real implementation (per its M9.0 docstring).
**Rejected alternatives:** (A) replace without stage helpers (monolithic; less extensible for M9.B/C/D); (B) keep `run_no_op` + add `run` (backward-compatible but bifurcates the test surface + leaves dead code); (C) `Pipeline` class with stage methods (deviates from arc functional-style).
**Implementation surface (Stage 3 C1):** `src/app/services/ai/pipeline.py` + AIPipelineConsumer call-site + M9.0 test conversions.
**Recon-invocation-seam-extension forward-pin (Drift #59 + #62 + #66-averted lineage):** the pipeline-rewrite seam has a test-impact-surface at the M9.0 C3 tests asserting `run_no_op` behavior; V-NN at C1 entry must ground the test-conversion scope precisely.

### 3.2 Y-EMBEDDING-INPUT-CONSTRUCTION (A.b + B.c + C.c)
**Lock:** Extended field set (title + finding_type + cwe_id + target_url + description), each as a labeled line (`"Title: ...\nType: ...\nCWE: ...\nURL: ...\nDescription: ..."`), with `"N/A"` for nullable-absent fields (cwe_id, target_url, description).
**Grounding:** Labels give the embedding model structural signal; target_url inclusion prevents conflating same-type findings at different endpoints; `"N/A"` markers keep dimensionality stable vs bare `None` string interpolation.
**Rejected alternatives:** (A.a) plan-literal `f"{title} {description} {finding_type} {cwe_id}"` (bare concat; `None`→"None" noise; no target_url); (B.a) unlabeled join; (C.a) JSON-serialized input (token-wasteful).

### 3.3 Y-EMBEDDING-BATCH-SHAPE (A.a + B.a + C.c + D.b)
**Lock:** batch=100; sequential chunking; hybrid retry (SDK-transient + manual 429 exponential-backoff); on retry-exhaustion → rule-based `fingerprint` fallback + `Scan.ai_pipeline_degraded=True` per SPEC §8.6.
**Grounding:** 100 is conservative vs the ~2048 input limit (safe headroom; predictable token batches); SPEC §8.6 mandates the embedding-down → rule-based-fingerprint fallback (resilience invariant — a degraded scan still completes).
**Rejected alternatives:** (A.b) larger batch (closer to limit; less headroom); (C.a/b) SDK-only or manual-only retry (one misses transient vs 429 classes); (D.a) hard-fail on embedding outage (violates resilience invariant).

### 3.4 Y-EMBEDDING-CACHE-STRATEGY (C')
**Lock:** No cache at M9.A; cache-by-fingerprint forward-pinned to production-readiness.
**Grounding:** Pre-launch embedding cost is ~$0.001/scan (negligible); a cache adds a cross-scan lookup surface without pre-launch payoff. Consistent with the arc's bounded-staleness / no-premature-optimization posture.
**Rejected alternatives:** (A) Qdrant-point-existence as implicit cache (couples cache to per-scan collection lifecycle); (B) separate cache table (premature infra).

### 3.5 Y-QDRANT-OPERATIONS-API (A.c + B.c + C.a' + D.b)
**Lock:** `_ensure_collection_exists(client, collection)` helper (lazy create with `VectorParams(size=1536, distance=COSINE)`); the exact search vs `query_points` API choice deferred to a V-NN check at Stage 3 C1 (qdrant-client ^1.12.0 surface); deterministic point id = `uuid.uuid5(NAMESPACE_FINGERPRINT, fingerprint)`; extended payload {raw_finding_id, scan_id, organization_id, cwe_id, target_url, fingerprint, vulnerability_id}.
**Grounding:** Collection-per-org per M9.0 Q4; deterministic uuid5 gives idempotent re-runs (a re-dispatched scan re-upserts the same point ids); `vulnerability_id` in payload lets a cluster-hit resolve the existing Vulnerability without a PG round-trip.
**Rejected alternatives:** (C.a) `fingerprint` string as point id (Qdrant prefers uuid/int ids); (D.a) minimal payload (no `vulnerability_id` → extra PG lookup on every cluster hit).

### 3.6 Y-DEDUP-ALGORITHM-DETAILS (A.a + B.a + C.a + D.b)
**Lock:** search limit=1, `score_threshold=0.92` (server-side); filter to current scan only (per-scan dedup, M9.0 Q5); first-emitted finding is the cluster representative; two-phase (embed the whole batch, then dedup sequentially).
**Grounding:** SPEC §8.1 fixes the 0.92 cosine threshold; per-scan filter honors the M9.0 Q5 per-scan-dedup lock (cross-scan forward-pinned to M11+); two-phase keeps the embedding batch efficient while dedup stays order-deterministic.
**Rejected alternatives:** (A.b) limit>1 (multi-cluster ambiguity); (B.b) cross-scan filter now (M9.0 Q5 deferred it); (C.b) score-based representative (needs M9.B scoring); (D.a) interleaved embed+dedup (defeats batch efficiency).

### 3.7 Y-MERGE-EVIDENCE-SHAPE (A.a + B.a + C.a + D.b + Resolution γ)
**Lock:** `_merge_evidence(db, vulnerability, raw_finding) -> None`, module-private; append-only; atomic same-flush; Python read-modify-write reassignment (`vuln.raw_finding_ids = [*(vuln.raw_finding_ids or []), raw_finding.id]`); the cluster's `vulnerability_id` is pre-generated (Resolution γ) so the Qdrant payload + raw_finding state can reference it before any flush.
**Grounding:** ARRAY mutation in SQLAlchemy needs reassignment (in-place `.append` doesn't trip the dirty flag); pre-generated UUID avoids an intermediate flush between Vulnerability creation and Qdrant upsert (which needs the id in payload).
**Rejected alternatives:** (A.b) replace-set semantics (loses traceability); (B.b) per-finding flush (chatty; ordering fragility); (C.b) ORM mutable-list tracking (heavier; reassignment is simpler).

### 3.8 Y-COST-TRACKING-INTEGRATION-POINTS (A.a' + B.a + C.a)
**Lock:** No pre-call budget check at M9.A (embeddings are low-cost per M9.0 Q2 B.c hybrid); per-batch `log_ai_call` after each embeddings request; `tokens_in` from the API response `usage.prompt_tokens`; `operation_type="embedding"`, `provider="openai"`, `model="text-embedding-3-small"`, `cost_usd` computed at $0.02/1M tokens.
**Grounding:** M9.0 Q2 (B.c) hybrid: low-cost ops log post-call only. Per-batch (not per-finding) keeps `ai_api_calls` row count proportional to requests.
**Rejected alternatives:** (A.b) pre-call estimate for embeddings (needs tiktoken; over-engineered for low-cost — forward-pinned to M9.C high-cost ops); (B.b) per-finding logging (row explosion); (C.b) local token estimate (the API returns exact usage).

### 3.9 Y-TEST-FIXTURE-STRATEGY (A.b + B.a + C.a + D.b)
**Lock:** `unittest.mock` patching `client.embeddings.create` (deterministic vectors); in-memory Qdrant via `AsyncQdrantClient(":memory:")` for real cosine behavior; function-scoped fixtures; shared parametrized fixtures for finding-sets.
**Grounding:** Mocking embeddings keeps tests offline + deterministic; in-memory Qdrant exercises the real similarity math (no mock-similarity fiction). V-NN at C1 must confirm qdrant-client `:memory:` async support.
**Rejected alternatives:** (A.a) mock Qdrant too (loses real similarity coverage); (B.b) live Qdrant (network dependence); (C.b) session-scoped (cross-test point leakage).

### 3.10 Y-MIGRATION-NEEDED (NO new schema)
**Lock:** No migration at M9.A. The M9.0 C1 migration (`b7e4a1f93c2d`) already added `Vulnerability.qdrant_point_id` + `Vulnerability.raw_finding_ids` + `raw_findings.promoted_at` + `raw_findings.vulnerability_id` + `ai_api_calls` — all M9.A needs.
**Grounding:** M9.0 Q9 deliberately front-loaded the full M9 schema; M9.A is pure logic.

### 3.11 Y-ADR-NUMBER (ADR-030)
**Lock:** ADR-030 "AI Pipeline: Embedding + Deduplication (M9.A)" at SPEC §13, inserted after ADR-029 (line 2559) before §14 (line 2632) per V-NNB.
**Grounding:** Sequential ADR numbering; ADR-030 explicitly composes ADR-029.

### 3.12 Y-STAGE3-DECOMPOSITION-FOR-M9.A (A.a + B.a)
**Lock:** 3-commit, top-down docs-first. C0 docs ADR-030 (~80-120 LoC); C1 api pipeline.py rewrite + call-site + M9.0 test conversions (~300-500 LoC); C2 api new tests + M9.A smoke (~300-500 LoC).
**Grounding:** Docs-first lands the canonical ADR before code references it (M9.0 §2.4 precedent); C1 makes the behavior change + converts impacted M9.0 tests in the same commit (ZERO-regression-at-each-commit); C2 adds positive new-behavior coverage.
**Rejected alternatives:** (A.b) 2-commit (docs+impl merged; large diff); bottom-up (code before ADR — references non-canonical authority).
**Aggregate Stage 3 LoC forecast:** ~680-1120 LoC.

## 4. M9.A Stage 3 Implementation Surface

### 4.1 Stage 3 Commit 0 — ADR-030 Docs at SPEC §13
**Scope:** ADR-030 canonical text between ADR-029 (line 2559) and §14 Meta-Principles (line 2632). Status (Accepted) + Context (M9.A scope; Path C; ADR-029 composition) + Decision (Y-PROMOTION-TIMING + 12 Q-locks) + Rationale + Rejected Alternatives + Consequences + Composition with ADR-013/014/022/028/029 + Cross-references.
**LoC forecast:** ~80-120 LoC.

### 4.2 Stage 3 Commit 1 — api pipeline.py rewrite + call-site + M9.0 test conversions
**Files:** `src/app/services/ai/pipeline.py` (rewrite: `run_no_op` → `run` + `_embed_findings` + `_dedup_and_promote` + `_merge_evidence` + `_create_vulnerability_from_finding`); `src/app/services/ai_pipeline_consumer.py` (call-site `run_no_op` → `run`); `tests/services/test_ai_pipeline_consumer.py` (`run_no_op` → `run` conversions + monkeypatch target update); `tests/integration/test_m9_smoke.py` (`run_no_op` → `run`; with-findings now produces Vulnerability rows — assertion shape update).
**LoC forecast:** ~300-500 LoC.
**Test-impact-surface (Drift #63 extension; V-NN at C1):** grep ALL tests referencing `run_no_op` + the no-op COMPLETED behavior; the M9.0 smoke happy-path with findings now creates Vulnerabilities (was no-op) → assertions shift.

### 4.3 Stage 3 Commit 2 — api new tests + M9.A smoke
**Files:** `tests/services/ai/test_embeddings.py` NEW (embed input construction + batch + fallback) + `tests/services/ai/test_deduplication.py` NEW (cluster + threshold + per-scan filter + promotion + merge_evidence) + `tests/integration/test_m9a_smoke.py` NEW or extend `test_m9_smoke.py` (e2e: scan + raw_findings → embed → dedup → Vulnerability rows + Y2 vulnerability_count surface).
**LoC forecast:** ~300-500 LoC (~50-65 LoC/test × ~5-10 tests/file).

**Aggregate Stage 3 LoC forecast:** ~680-1120 LoC across 3 commits.

## 5. Phase Structure

Stage 1 design doc (THIS COMMIT) → Stage 2 implementation plan → Stage 3 3-commit api-side (C0 docs ADR-030 → C1 api impl + test conversions → C2 api new tests + smoke) → Stage 4 P5.A annotations.

## 6. Out of Scope

1. M9.B sub-milestone (Tasks 9.3+9.4; correlate + score) — own lifecycle
2. M9.C sub-milestone (Tasks 9.5+9.6; fix + summary) — own lifecycle
3. M9.D sub-milestone (Task 9.7; orchestrator composition) — own lifecycle
4. Cross-scan dedup (M9.0 Q5 forward-pin to M11+)
5. Embedding cache-by-fingerprint (Q4 forward-pin to production-readiness)
6. Tiktoken pre-call estimation (Q8 forward-pin to M9.C high-cost ops)
7. Vulnerability scoring formula per SPEC §8.3 (M9.B scope; M9.A sets severity from cluster-representative raw_finding only)
8. Vulnerability fix-generation per SPEC §8.4 (M9.C scope)
9. Cluster-representative refinement by severity (Q6 forward-pin; M9.B scoring may update)
10. Embedding-dimension tuning beyond text-embedding-3-small 1536 (forward-pin)

## 7. Forward-Pins

**Pre-implementation:**
1. Stage 2 plan landing: ***"Begin M9.A — Stage 2 implementation plan landing"***
2. V-NN Stage 3 C1 entry verification: qdrant-client `:memory:` async support + search/query_points API surface + full `run_no_op` test-conversion scope

**Sub-milestone activation triggers:**
3. ***"Begin M9.B — Correlation + Scoring"*** (after M9.A closes)
4. ***"Begin M9.C — Fix Generation + Executive Summary"*** (after M9.B closes)
5. ***"Begin M9.D — Pipeline Orchestrator"*** (after M9.C closes)

**Discipline-level (active + extended at M9.A):**
6. Audit-driven model+spec orphan check (Drift #60 rule-of-three; validated)
7. DEFERRED-EMPIRICAL marking (Drift #61 + #62; `merge_evidence` gap + RawFinding.project_id absence both pre-grounded)
8. Recon-invocation-seam-extension (Drift #59 + #62 + #66-averted lineage; pipeline-rewrite seam)
9. Test-impact-surface scope completeness (Drift #63 extension; `run_no_op` → `run` test conversions at C1)
10. Plan-template-discipline (M9.0 extension; M9.A plan §4.X anticipates ancillary files)
11. Averted-prediction discipline (V-IIB/V-JJC 2-instance; V-MM `merge_evidence` + V-NNE `project_id` demonstrate 3rd/4th instances)

**Production-readiness audit forward-pin chain (M9.A extensions):**
12. Tune the 0.92 dedup threshold via empirical analysis
13. Implement embedding cache-by-fingerprint when scale demands
14. Add embedding cost telemetry assertions at M9.A C2 smoke
15. Add pre-call budget check at `_embed_findings` for very-large-scan scenarios
16. Each subsequent sub-milestone (M9.B/C/D) owns its schema migration if needed

## 8. Cross-References

**Docs:** 62499a3 (M9.0 P5.A docs annotations); 45dcabe (ADR-029 SPEC §13 canonical authority for M9.A composition); 55dbe32 (M9.0 Stage 2 plan; Stage 3 sub-step breakdown precedent); a46fedd (M9.0 Stage 1 design doc structural precedent §1-§9); SPECIFICATION.md §8 canonical M9 pipeline authority + SPEC §13 ADR-029 (composed); CLAUDE.md Gotcha 5 cost-tracking mandate (activates real cost-logging at M9.A).

**Engine:** None (M9.A is api-side only).

**API:** 4616672 (M9.0 P5.A api DRIFT-LOG sync); 1c98330 (M9.0 C2 AIPipelineConsumer + dispatch_ai_pipeline; M9.A extends pipeline.py + call-site); 51b26ea (M9.0 C1 schema + modules; M9.A consumes operational state); 8410df4 (M9.0 C3 tests; M9.A converts `run_no_op` references in test_ai_pipeline_consumer + test_m9_smoke); src/app/services/ai/pipeline.py (rewrite target); src/app/services/ai_pipeline_consumer.py (call-site target); src/app/services/ai/clients.py (qdrant_client + openai_client singleton access); src/app/services/ai/cost_tracking.py (log_ai_call per-batch integration); src/app/models/raw_findings.py (embedding-input fields; NO project_id — derive from Scan per V-NNE); src/app/models/vulnerabilities.py (cluster representative + raw_finding_ids + qdrant_point_id).

**Multi-session arc:** 91 commits + 65 framing-drift catalogue + #66-averted lineage + 18 closed task lifecycles + M5/M6/M7/M8/M9.0 all CLOSED + Y2 Task 8.3β forward-pin activates at M9.A.

## 9. Path C Resolution + merge_evidence + project_id Gap Closures

**V-MM surfaced (this session):** Plan-literal pseudo-code at IMPLEMENTATION-PLAN.md Task 9.2 references `merge_evidence(existing_vuln, finding)` — no such helper exists; a circular dependency is implied (the existing vuln must already exist for the merge, but Task 9.2 is the promotion stage).

**Path C resolution at the Y-PROMOTION-TIMING gate:** the first finding of each dedup cluster creates a Vulnerability with cluster-representative fields; subsequent matches append `raw_finding_id` to the existing `Vulnerability.raw_finding_ids` via `_merge_evidence`. Cluster identity is established at the Qdrant upsert (deterministic uuid5); Vulnerability identity is established at first-cluster-member creation (pre-generated UUID, Resolution γ).

**merge_evidence shape resolved at Q7:** module-private `_merge_evidence(db, vulnerability, raw_finding) -> None`; operates on the passed Vulnerability instance (no re-load); appends `raw_finding.id` to `Vulnerability.raw_finding_ids` via reassignment; sets `raw_finding.promoted_at` + `raw_finding.vulnerability_id` per M9.0 Q6 (B.a) state-transition; no `db.flush()` inside (atomic same-flush at the `_dedup_and_promote` loop end).

**RawFinding.project_id absence resolved at V-NNE (DEFERRED-EMPIRICAL pre-grounding):** `RawFinding` has no `project_id`, but `Vulnerability.project_id` is NOT NULL. `_create_vulnerability_from_finding` therefore derives `project_id` from the Scan — the pipeline holds `scan_id`, loads the Scan once (it already does, to read org/status), and passes `scan.project_id` into the new Vulnerability. Grounded here so C1 implements it directly rather than discovering it at execution (averts a would-be Drift #61-class at C1, mirroring the M9.0 V-IIB config-field-name averting).

**Averted-prediction discipline applied:** both V-MM-class catches (`merge_evidence` undefined-helper gap; `RawFinding.project_id` absence) are addressed within the brainstorming chain + V-NN pre-verification; neither is catalogued as new drift; mirrors the V-IIB/V-JJC averted-prediction precedent from M9.0 C1/C2. **Cumulative session-tail framing-drift count remains 65.**
