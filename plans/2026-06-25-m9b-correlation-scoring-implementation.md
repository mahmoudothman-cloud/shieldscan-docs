# M9.B — AI Pipeline: Correlation + Scoring (ADR-031): Implementation Plan

**Status:** PY1-PY8 plan-level Y-decisions + Stage 3 4-commit sub-step breakdown + verification cascade scope + per-commit D-deviation forecasts (this commit); ready for Stage 3 4-commit lifecycle (C0 docs ADR-031 → C1 schema + modules → C2 pipeline integration → C3 tests + smoke) + Stage 4 P5.A.

**Date:** 2026-06-26.

**Authority:** M9.B Stage 1 design doc CLOSED (commit `d10be51`; +120 LoC; §1-§9 sections; Y-CORRELATION-MERGE-VS-LINK Path B gate + Q1-Q18 + ~45+ sub-decisions + 3 undefined-helper gap closures + V-UUE tool_name DEFERRED-EMPIRICAL pre-grounding); M9.A Stage 2 plan structural precedent (commit `a8ad52c`; 147 LoC §1-§8 + PY1-PY4); M9.A lifecycle CLOSED (8-commit chain; ADR-030 canonical authority operational); SPEC §8.2 + §8.3 canonical M9.B authority; CLAUDE.md Gotcha 5 cost-tracking mandate (no new cost at M9.B per Q14 deterministic computation lock); 66 cumulative session-tail framing-drift discipline (Drift #58-#66; Drift #66 NOVEL ORM-vs-DB layer assumption mismatch operational from M9.A C1; 5-instance averted-prediction lineage).

**Related:** Stage 3 C0 landing trigger: ***"Begin M9.B — Stage 3 C0 ADR-031 docs landing"***. Sub-milestone activation triggers after M9.B closes: ***"Begin M9.C — Fix Generation + Executive Summary"*** → ***"Begin M9.D — Pipeline Orchestrator"***.

---

## 1. Plan-Level Y-Decisions (PY1-PY8)

**PY1 — V-VV pre-Stage-3-C0 verification scope.**
ADR-031 SPEC §13 insertion empirical + ADR-030 boundary at line 2632 (per V-UUB) + §14 Meta-Principles boundary at line 2726; mirrors M9.A V-PP pre-Stage-3-C0 pattern. Bounded ~3-5min read-only verification.

**PY2 — V-WW pre-Stage-3-C1 verification scope.**
alembic head re-verification (`b7e4a1f93c2d` expected; per V-UUE) + Vulnerability model empirical re-verification (correlation_cluster_id + severity_score column absence confirmed at `d10be51` §9; Q16 migration target) + RawFinding model (tool_name confirmed per V-UUE for Q10 PoC-derivation; engine_name absent) + Scan model (project_id + is_public + config available) + cwe_hierarchy.py + correlation.py + scoring.py target paths empirical (NEW files at src/app/services/ai/); test_pipeline.py current shape for C2 extension; novel-implementation territory at C1 = highest D-deviation risk per arc precedent. Bounded ~10-15min read-only verification cascade.

**PY3 — V-XX pre-Stage-3-C2 verification scope.**
pipeline.py current state empirical re-verification (run() + 11 module-private helpers from M9.A C1 `91ec273`; insertion point for _correlate_vulnerabilities + _score_vulnerabilities after _dedup_and_promote); test_pipeline.py extension patterns from M9.A C2 `251960a`; Q13 C defensive skip semantics empirical grounding. Bounded ~5-10min.

**PY4 — V-YY pre-Stage-3-C3 verification scope.**
tests/services/ai/conftest.py current shape (M9.A C2 `251960a` established; stub_ai_clients + RawFinding factories + two_clustering_findings + two_distinct_findings; extension target for cross_engine_vulnerabilities + single_engine_vulnerabilities per Q15); test_m9a_smoke.py structural precedent for test_m9b_smoke.py format-adapt; CWEHierarchy minimal test fixture pattern per Q15 C.b. Bounded ~5-10min.

**PY5 — DEFERRED-EMPIRICAL pre-grounding from V-UUE carried into C1.**
RawFinding.tool_name for Q10 PoC-derivation (NOT engine_name; documented at Stage 1 design doc §9). `POC_PROVEN_TOOL_NAMES = {"nuclei", "sqlmap"}` constant in scoring.py. Mirrors V-NNE project_id pre-grounding pattern from M9.A.

**PY6 — Drift #66 ORM-vs-DB FK ordering awareness at C1.**
Vulnerability.correlation_cluster_id is nullable UUID (no FK constraint to other table); Vulnerability.severity_score is Float standalone; both columns are simple ALTER TABLE additions with no cross-table FK ordering concerns. Lower Drift #66-class risk than M9.A C1 (which had raw_findings.vulnerability_id → vulnerabilities.id FK requiring intermediate flush). Forward-pin: verify alembic migration includes both columns + index on correlation_cluster_id; ensure SQLAlchemy Mapped column types align with migration types.

**PY7 — Stage 3 sequential verification cascade discipline.**
V-VV (pre-C0; bounded ~3-5min) → V-WW (pre-C1; ~10-15min novel-implementation territory) → V-XX (pre-C2; ~5-10min composition layer) → V-YY (pre-C3; ~5-10min test fixture). Each cascade's pre-verification surface report PAUSE before commit landing per arc discipline.

**PY8 — D-deviation forecast per commit + outcome distribution.**
- **C0 docs ADR-031:** 0 drifts expected (mechanical canonical text insertion; mirrors M9.A C0 `d408b2c` precedent). Outcome γ likely.
- **C1 api schema + modules:** 0-2 drifts forecast (novel-implementation territory; cwe_hierarchy.py + correlation.py + scoring.py NEW + alembic migration + Vulnerability schema extension). Per M9.A C1 precedent (1 catalogued drift = Drift #66 NOVEL ORM-vs-DB layer assumption mismatch), expect lower at M9.B C1 due to: (a) simpler schema migration (2 nullable columns, no FK); (b) deterministic computation (no AI client mocking complexity); (c) V-UUE tool_name pre-grounding addresses concrete-field gap pre-execution. Outcome β-γ likely.
- **C2 api pipeline integration:** 0 drifts forecast (composition layer; extends established M9.A C1 `91ec273` run() pattern). Outcome γ likely.
- **C3 api tests + smoke:** 0 drifts forecast (test-coverage territory; mirrors M9.A C2 `251960a` discipline). Outcome γ likely.

**Cumulative Stage 3 D-deviation forecast: 0-2 drifts total** (vs M9.A Stage 3 actual 1 drift at C1). Forecast assumes V-UU/V-VV/V-WW/V-XX/V-YY pre-verification cascade catches design-resolution territory before execution.

## 2. Stage 3 4-Commit Sub-Step Breakdown

### 2.1 Stage 3 Commit 0 — ADR-031 docs at SPEC §13
**Repo:** shieldscan-docs. **File:** SPECIFICATION.md (single file).
**Scope:** ADR-031 canonical text insertion between ADR-030 (line 2632) and §14 Meta-Principles (line 2726). Sections: Status + Context + Decision (Path B gate + Q1-Q18 summary by reference to Stage 1 design doc `d10be51`) + Rationale + Rejected Alternatives + Consequences + Composition (ADR-013/014/022/028/029/030) + Cross-references.
**LoC forecast:** ~80-120 (mirrors M9.A C0 `d408b2c` precedent +94 LoC).
**V-VV pre-C0 verification:** ~3-5min.
**Estimated execution time:** ~30-45min total (V-VV + canonical text insertion + verification + commit).
**Single atomic commit.**

### 2.2 Stage 3 Commit 1 — api schema + modules
**Repo:** shieldscan-api. **Files:**
- `alembic/versions/[hash]_add_m9b_correlation_scoring_columns.py` NEW (Q16 migration: ALTER TABLE vulnerabilities ADD correlation_cluster_id UUID indexed + severity_score Float; explicit downgrade())
- `src/app/services/ai/cwe_hierarchy.py` NEW (Q5: CWE_PARENT_CHILD curated dict + CWEHierarchy class + get_cwe_hierarchy() singleton)
- `src/app/services/ai/correlation.py` NEW (Q1-Q4 + Q6 + Q7: CORRELATION_WEIGHTS + CORRELATION_THRESHOLD + correlation_score sync + build_route_map + extract_params_from_url + extract_params_from_snippet + _extract_params_combined + _iter_cross_engine_pairs + _union_find_clusters helpers)
- `src/app/services/ai/scoring.py` NEW (Q9 + Q10 + Q11: compute_severity_score sync + compute_exploitability_multiplier + _map_score_to_severity_enum + POC_PROVEN_TOOL_NAMES = {"nuclei", "sqlmap"} constant per PY5)
- `src/app/models/vulnerabilities.py` (add correlation_cluster_id + severity_score Mapped declarations matching migration columns)

**LoC forecast:** ~400-550 (cwe_hierarchy.py ~80-120; correlation.py ~150-200; scoring.py ~60-100; migration ~40-60; vulnerabilities.py +20-30; aggregate ~350-510).
**V-WW pre-C1 verification:** ~10-15min cascade (alembic head + Vulnerability + RawFinding + Scan + target file paths + test_pipeline.py current shape).
**Estimated execution time:** ~90-120min total (V-WW + 5 file landings + migration creation + verification + commit).
**Single atomic commit.**

### 2.3 Stage 3 Commit 2 — api pipeline integration
**Repo:** shieldscan-api. **Files:**
- `src/app/services/ai/pipeline.py` (extend run() with _correlate_vulnerabilities + _score_vulnerabilities helpers per Q13; sequential composition after _dedup_and_promote; defensive skip when <2 engine_categories present)
- `tests/services/ai/test_pipeline.py` (extend with _correlate_vulnerabilities + _score_vulnerabilities integration tests + defensive skip case)

**LoC forecast:** ~200-300 (pipeline.py +120-200; test_pipeline.py +80-100).
**V-XX pre-C2 verification:** ~5-10min.
**Estimated execution time:** ~60-90min total.
**Single atomic commit.**

### 2.4 Stage 3 Commit 3 — api tests + smoke
**Repo:** shieldscan-api. **Files:**
- `tests/services/ai/test_correlation.py` NEW (correlation_score + cross-engine pair iteration + union-find clustering + score threshold + symmetric + max-not-sum + route_map + parameter extraction integration; ~150-200 LoC; 12-15 tests)
- `tests/services/ai/test_scoring.py` NEW (compute_severity_score + multipliers + cap at 10.0 + base_cvss null handling + score-to-severity-enum mapping + tool_name PoC-derivation; ~100-150 LoC; 10-12 tests)
- `tests/services/ai/test_cwe_hierarchy.py` NEW (CWEHierarchy class + is_parent_child bidirectional + singleton accessor; ~50-80 LoC; 6-8 tests)
- `tests/services/ai/conftest.py` (extend with cross_engine_vulnerabilities + single_engine_vulnerabilities fixtures per Q15 B.a)
- `tests/integration/test_m9b_smoke.py` NEW (M9.B end-to-end + corroborated_count + cluster_id + severity_score post-pipeline assertions; ~80-150 LoC; 4-6 tests)

**LoC forecast:** ~430-660 (aggregate across 5 test files; mirrors M9.A C2 `251960a` +1097 LoC scope but less mock-dependency complexity = lower upper bound).
**V-YY pre-C3 verification:** ~5-10min.
**Estimated execution time:** ~90-120min total.
**Single atomic commit.**

**Stage 3 aggregate LoC forecast:** ~1110-1630 LoC across 4 commits (per Q18 + Stage 1 `d10be51` forecast).
**Stage 3 aggregate execution time:** ~4.5-6h.

## 3. Verification Cascade Discipline

V-VV (pre-C0) → V-WW (pre-C1) → V-XX (pre-C2) → V-YY (pre-C3). Each pre-verification surfaces empirical findings + PAUSE for direction before commit landing.

**V-WW pre-C1 highest D-deviation risk per arc precedent:**
- alembic head re-confirm (`b7e4a1f93c2d`)
- Vulnerability + RawFinding + Scan model empirical state
- target file paths for NEW modules (cwe_hierarchy.py + correlation.py + scoring.py)
- test_pipeline.py current shape for C2 extension awareness
- PY5 V-UUE tool_name (NOT engine_name) re-confirm at scoring.py PoC-derivation
- PY6 Drift #66-class FK ordering assessment (lower risk than M9.A C1; no cross-table FK in migration)

## 4. Out of Scope (Stage 3 Sub-Milestone)

1. Stage 4 P5.A annotations (post-Stage-3-close; mirrors M9.A P5.A pattern)
2. M9.C sub-milestone (Tasks 9.5+9.6; activation trigger after M9.B closes)
3. M9.D sub-milestone (Task 9.7; activation trigger after M9.C closes)
4. Cross-scan correlation (Q3 forward-pin to M11+)
5. Per-pair correlation_score audit trail (Q8 forward-pin junction table at production-readiness)
6. Auth/public-facing exploitability detection (Q10 forward-pin to M9.C/D)
7. Claude tie-break for near-threshold correlations (Q14 forward-pin at production-readiness)
8. AI-based route_map + customer-uploaded override + framework-specific static analysis (Q6 forward-pin)
9. Embedded MITRE CWE library + AST-based parameter extraction (Q5+Q7 forward-pin)
10. severity_score index for M10 ranking (Q16 forward-pin)
11. Uncapped final_score tracking for ranking precision (Q9 forward-pin)
12. vulnerability_correlations junction table for per-pair audit (Q8 forward-pin)

## 5. Forward-Pins

**Sequential M9.B lifecycle close:**
1. Stage 3 C0 landing trigger: ***"Begin M9.B — Stage 3 C0 ADR-031 docs landing"***
2. Stage 3 C1 landing trigger: ***"Begin M9.B — Stage 3 C1 schema + modules"***
3. Stage 3 C2 landing trigger: ***"Begin M9.B — Stage 3 C2 pipeline integration"***
4. Stage 3 C3 landing trigger: ***"Begin M9.B — Stage 3 C3 tests + smoke"***
5. Stage 4 P5.A trigger: ***"Begin M9.B — Stage 4 P5.A annotations"***

**Sub-milestone activation triggers (after M9.B closes):**
6. ***"Begin M9.C — Fix Generation + Executive Summary"*** (Tasks 9.5+9.6)
7. ***"Begin M9.D — Pipeline Orchestrator"*** (Task 9.7)

**Post-M9:**
8. ***"Begin M10 — Report Architecture"*** (after M9 fully closes)

**Discipline-level (active from prior arc + extensions at M9.B):**
9. Audit-driven model+spec orphan check (4-instance validated)
10. DEFERRED-EMPIRICAL marking (5-instance averted-prediction lineage; V-UUE tool_name 5th-instance)
11. Recon-invocation-seam-extension (pipeline-rewrite seam at C2)
12. Test-impact-surface scope completeness (Stage 3 C3)
13. Plan-template-discipline (this commit anticipates Stage 3 ancillary files)
14. Averted-prediction discipline (V-VV/V-WW/V-XX/V-YY pre-verification cascade preserves)
15. NEW from M9.A: ORM-vs-DB layer assumption mismatch (Drift #66 catch-class; PY6 mitigation at M9.B C1)

## 6. Cross-References

**Docs:** `d10be51` (M9.B Stage 1 design doc; Y-locks + V-UUE pre-grounding); `c9130db` (M9.A P5.A docs annotations); `d408b2c` (ADR-030 SPEC §13 canonical authority M9.A composition; C0 structural precedent); `a8ad52c` (M9.A Stage 2 plan structural precedent); `aaf7ea0` (M9.A Stage 1 design doc); `45dcabe` (ADR-029 SPEC §13 canonical authority M9.0 composition); SPECIFICATION.md §8.2 + §8.3 canonical M9.B authority + SPEC §13 ADR-031 (target at Stage 3 C0); IMPLEMENTATION-PLAN.md Tasks 9.3+9.4 (M9.B P5.A annotation target); CLAUDE.md Gotcha 5 cost-tracking mandate.

**Engine:** None at Stage 3 (M9.B is api-side only).

**API:** `cee75bf` (M9.A P5.A api DRIFT-LOG Drift #66 persistent sync; M9.B C1 awaits Drift #66-class catch); `251960a` (M9.A C2 tests; M9.B C3 mirrors); `91ec273` (M9.A C1 implementation; M9.B C2 extends pipeline.run() composition); src/app/services/ai/pipeline.py (C2 extension target); src/app/services/ai/correlation.py + scoring.py + cwe_hierarchy.py (NEW at C1); src/app/models/vulnerabilities.py (C1 extension target); alembic/versions/ (C1 revision after `b7e4a1f93c2d`).

**Multi-session arc:** 99 commits + 66 framing-drift catalogue + Drift #66 NOVEL ORM-vs-DB layer assumption mismatch + 18 closed task lifecycles + M5/M6/M7/M8/M9.0/M9.A all CLOSED + Y2 Task 8.3β forward-pin CLOSED at M9.A C2.

## 7. Phase Structure
Stage 1 design doc ✅ `d10be51` → Stage 2 plan THIS COMMIT → Stage 3 4-commit api-side → Stage 4 P5.A annotations.

## 8. Outcome Forecast Distribution
Per arc precedent: ~80% γ outcome (mechanical landing); ~15% β outcome (1-2 drifts; per M9.A C1 Drift #66 precedent at novel-implementation territory); ~5% α outcome (3+ drifts; lower at M9.B C1 due to PY5+PY6 mitigations). Forecast assumes V-VV → V-WW → V-XX → V-YY pre-verification cascade discipline operational.
