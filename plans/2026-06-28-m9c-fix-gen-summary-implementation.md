# M9.C AI Pipeline: Fix Generation + Executive Summary — Stage 2 Implementation Plan

**Status:** CLOSED at M9.C Stage 3 + P5.A lifecycle closure (2026-07-02). PY1-PY8 operational across the 3-commit Stage 3 (C0 ADR-032 `e8fbd8c` → C1 modules + integration `a0522a1` → C2 tests + smoke `6eff189`; 36 new tests, suite 747 ZERO regressions); PY5 V-AB DEFERRED-EMPIRICAL fully operationalized at V-BB + V-CC cascades (5 averted-prediction catches; 7-instance lineage); PY7 fixture-ordering discipline applied; C1 test-assertion narrowing per Option A (3rd test-gate-within-lock instance; not a catalogue increment); P5.A Commit 1 docs this commit + Commits 2+3 forthcoming. Cumulative drift count preserved at 66. Original plan content (PY1-PY8 + 3-commit breakdown + D-deviation forecasts) preserved below as historical authority.

**Date:** 2026-06-28.

**Authority:** Stage 1 design doc `261cf10` (this session; 18 architectural locks = 2 gate-decisions + Q1-Q16; V-AB §9 DEFERRED-EMPIRICAL pre-grounding); SPEC §8.4 (Mobile-Specific AI Fix Generation) + SPEC §8.5 (Multi-Provider AI Strategy + budget targets) canonical; ADR-029 (foundation + ai_api_calls + Gotcha 5) + ADR-030 (M9.A Path C) + ADR-031 (M9.B Path B) + ADR-013 (Python sole-writer) composition. M9.B Stage 2 plan `a172fd1` structural precedent (~159 LoC PY1-PY8 + Stage 3 sub-step breakdown); M9.A + M9.0 Stage 2 plan precedent. 66 cumulative session-tail framing-drift discipline preserved through M9.A + M9.B + M9.C Stage 1; 6-instance averted-prediction lineage operational; 2-instance test-gate-within-lock pattern operational.

**Related:** ADR-032 (forthcoming at Stage 3 C0). Stage 3 C0 landing trigger: ***"Begin M9.C — Stage 3 C0 ADR-032 docs landing"***. Sub-milestone activation after M9.C closes: ***"Begin M9.D — Pipeline Orchestrator"***.

---

## 1. Status Summary

- Stage 1 design doc landed at `261cf10` with 18 architectural locks (2 gate-decisions + Q1-Q16).
- Stage 2 implementation plan landing this session (Path γ).
- Stage 3 3-commit decomposition per Q15 (C0 docs + C1 modules+integration + C2 tests); forthcoming fresh session.
- Stage 4 P5.A annotations (M9.C lifecycle closure) forthcoming after Stage 3 closes.

## 2. Stage 3 Commit Breakdown (per Q15 A)

### C0 docs (~120-150 LoC; ADR-032 SPEC §13 canonical)
- SPECIFICATION.md ADR-032 insertion between ADR-031 and §14 Meta-Principles.
- Title: "AI Pipeline: Fix Generation + Executive Summary (M9.C)".
- Sections: Context + Decision + Composition + Consequences + Forward-pins.
- Cites ADR-029 + ADR-030 + ADR-031 + ADR-013 + SPEC §8.4 + SPEC §8.5.
- IMPLEMENTATION-PLAN.md M9.C activation cross-reference (Stage 1+2 plan links).
- **D-deviation forecast: 0 drifts** (docs-landing; structural pattern proven).

### C1 api modules + integration (~240-400 LoC)
- `src/app/services/ai/fix_generation.py` NEW (~150-250 LoC):
  - `FIX_GEN_SYSTEM_PROMPT` + `FIX_GEN_USER_PROMPT_TEMPLATE` constants.
  - `_build_target_context` dispatcher (mobile/sast/dast/generic).
  - `_build_mobile_context` + `_build_sast_context` + `_build_dast_context` + `_build_generic_context` (4 sub-context builders per Q1 A.c).
  - `_format_evidence` hybrid structured (per Q4).
  - `_fallback_fix_template` static template (per Q4).
  - `_call_claude_with_retry` 3-attempt exp backoff (per Q8 A.b).
  - `generate_fix` main entry (per Q1+Q4+Q5+Q8).
  - `_generate_fixes` orchestrator (per Q6+Q7+Q8 threshold detection).
  - Constants: `ESTIMATED_FIX_COST_USD = Decimal("0.025")` + `RETRY_MAX_ATTEMPTS = 3` + `RETRY_BACKOFF_SECONDS = [1,2,4]` + `DEGRADED_THRESHOLD_COUNT = 3` + `DEGRADED_THRESHOLD_PCT = 0.30`.
- `src/app/services/ai/summary.py` NEW (~80-130 LoC):
  - `SUMMARY_SYSTEM_PROMPT` + `SUMMARY_USER_PROMPT_TEMPLATE` constants.
  - `_build_summary_user_prompt` (per Q10).
  - `_deterministic_fallback_summary` (per Q11 C.c).
  - `_generate_executive_summary` main entry (per Q9+Q10+Q11+Q12 + Q13 C.b zero-vuln defensive).
  - Constant: `ESTIMATED_SUMMARY_COST_USD = Decimal("0.05")`.
- `src/app/services/ai/pipeline.py` extension (~10-20 LoC):
  - Imports: `from .fix_generation import _generate_fixes`; `from .summary import _generate_executive_summary`.
  - `run()` extension after `_score_vulnerabilities` (per Q13 A.a sequential + Q13 B.a before `db.flush()`).
- **D-deviation forecast: 0-1 drifts** (real Anthropic SDK integration novel territory; PY5 V-AB mitigations active; cost-tracking integration novel; threshold detection coordination across fix-gen + summary may surface within-lock under-implementation analogous to M9.B C2 Q1).

### C2 api tests (~400-500 LoC)
- `tests/services/ai/test_fix_generation.py` NEW (~150-200 LoC; 15-20 tests): `_build_target_context` dispatch (mobile/sast/dast/generic) per Q1; `_build_mobile_context` all mobile_* fields per Q2 B.a; `_format_evidence` hybrid per Q4; `_fallback_fix_template` per Q4; `_call_claude_with_retry` success-on-3rd + exhausted per Q8 A.b; `generate_fix` success per Q1+Q5; budget_exceeded fallback per Q5 B.a; retry_then_success + retry_exhausted_fallback per Q8 A.b + Q4; `_generate_fixes` severity_ordered_top_n per Q6; preserve_existing per Q7; empty_vulnerabilities defensive skip per Q13 C.b; threshold_degraded_count + threshold_degraded_pct per Q8 B.b.
- `tests/services/ai/test_summary.py` NEW (~100-150 LoC; 8-12 tests): `_build_summary_user_prompt` severity_tier per Q10 A.c; mid_detail per Q10 B.b; severity_cwe_breakdown per Q10 C.c; corroboration_aggregate per Q10 D.a; `_deterministic_fallback_summary` structure per Q11 C.c; `_generate_executive_summary` success per Q9+Q10; preserve_existing per Q12; budget_exceeded_deterministic_fallback per Q11 C.c; claude_error_deterministic_fallback per Q11 C.c; zero_vuln_trivial_summary per Q13 C.b.
- `tests/integration/test_m9c_smoke.py` NEW (~80-150 LoC; 5-7 tests): full_pipeline_e2e per Q13 sequential; zero_vuln_defensive per Q13 C.b; budget_exhaustion_graceful_degradation per Q5 B.a + Q11 B.a; claude_failures_threshold_degraded per Q8 B.b; severity_ordered_processing per Q6; idempotency_re_run per Q7+Q12.
- `tests/services/ai/conftest.py` extension (~50-80 LoC): `_StubAnthropicMessages` class with side-effect list (Q14 C.b); `_StubAnthropicClient` wrapping messages; `make_anthropic_response` constructor (Q14 B.b); `stub_anthropic` fixture (Q14 A.a monkeypatch on pipeline_mod).
- **D-deviation forecast: 0 drifts** (test-coverage territory; conftest extension; arc-precedent strong from M9.B C3 `f82c38a`; PY7 fixture-ordering lesson applied).

## 3. PY-Decisions (Plan-Level Y-Decisions)

**PY1 — Y-V-AB-CASCADE-DECOMPOSITION-INTO-V-BB-V-CC.** V-AB §9 Stage 1 markers split into V-BB pre-C1 cascade + V-CC pre-C2 cascade. V-BB scope: empirical verification before C1 modules + integration (V-ABG cost_usd precision + V-ABH Severity enum + V-ABI EngineCategory branching + V-ABF pipeline_mod re-confirmation for C1 import surface). V-CC scope: empirical verification before C2 tests + conftest extension (V-ABE Anthropic SDK Message type + V-ABF monkeypatch attachment re-confirmation). Bounded: V-BB ~5-10min; V-CC ~5-10min. Surface report compile then PROCEED unless drift surfaces.

**PY2 — Y-CROSS-MODULE-IMPORTS-NAMING-CONVENTIONS.** fix_generation.py module-private helpers (underscore prefix): `_build_target_context` + `_build_mobile_context` + `_build_sast_context` + `_build_dast_context` + `_build_generic_context` + `_format_evidence` + `_fallback_fix_template` + `_call_claude_with_retry`. fix_generation.py module-public: `generate_fix` + `_generate_fixes` (pipeline.py imports `_generate_fixes` via cross-module). summary.py module-private: `_build_summary_user_prompt` + `_deterministic_fallback_summary`. summary.py module-public: `_generate_executive_summary` (pipeline.py imports). Naming alignment with M9.B convention (M9.B `5dee684` transparency note; `iter_cross_engine_pairs` + `union_find_clusters` public for cross-module pipeline.py import; M9.C extends pattern).

**PY3 — Y-V-CC-PRE-C2-AWARENESS.** V-CC cascade aware of C1-landed surface (fix_generation.py + summary.py exports; pipeline.py extension shape) at C2 conftest extension + test landing. Empirical re-verification: C1 module exports importable; signature compatibility with stub_anthropic monkeypatch; cost_tracking + AsyncAnthropic operational.

**PY4 — Y-STAGE3-C2-TEST-DECOMPOSITION.** Unit tests in test_fix_generation.py + test_summary.py + conftest fixtures; smoke tests in test_m9c_smoke.py (end-to-end pipeline integration). ~32-45 new tests total (15-20 fix-gen + 8-12 summary + 5-7 smoke). Arc precedent: M9.B C3 `f82c38a` landed 41 tests across 4 NEW files + conftest extension; M9.C C2 mirrors decomposition.

**PY5 — Y-V-AB-DEFERRED-EMPIRICAL-OPERATIONALIZATION.** V-ABE (Anthropic SDK Message + TextBlock + Usage construction) → V-CC pre-C2; expected `from anthropic.types import Message, TextBlock, Usage`; verify constructor signature stable. V-ABF (pipeline_mod.get_anthropic_client monkeypatch attachment) → V-CC; M9.A precedent `monkeypatch.setattr(pipeline_mod, "get_openai_client", ...)`. V-ABG (cost_usd Decimal vs Float at log_ai_call) → V-BB; expected Decimal column type per AIAPICall (V-AAF confirmed at session entry). V-ABH (Severity enum CRITICAL/HIGH/MEDIUM/LOW) → V-BB. V-ABI (EngineCategory MOBILE/SAST/DAST + 10 others per M9.B V-TTG) → V-BB; `_build_generic_context` covers non-MOBILE/SAST/DAST gracefully.

**PY6 — Y-DRIFT-CATCH-CLASS-ASSESSMENT.** No-migration territory (V-AAH/I) eliminates Drift #66 ORM-vs-DB class at M9.C; PY6 mitigation already validated at M9.B C1. Real Anthropic SDK integration: novel; may surface SDK construction errors at C2 conftest stub_anthropic; mitigated by V-ABE pre-grounding + PY5. Cost-tracking integration: ai_api_calls operational at V-AAF; first real-load use; may surface threshold detection coordination at fix-gen + summary aggregate fallback-count tracking. Test-gate within-lock under-implementation (2-instance M9.B C2 + C3): expect possible within-lock corrections at C1 (async semantics for asyncio.gather; threshold counter mutation correctness) or C2 (monkeypatch attachment validity; SDK Message construction); both classified per arc-precedent.

**PY7 — Y-TEST-FIXTURE-ORDERING-DISCIPLINE.** stub_anthropic + make_anthropic_response added to conftest.py BEFORE consumer test files. Cross-directory awareness per M9.B C3 fixture-scoping lesson: tests/services/ai/conftest.py fixtures don't cross to tests/integration/test_m9c_smoke.py; either replicate fixture locally OR import from conftest-helpers. C2 ordering: (1) conftest.py extension first; (2) test_fix_generation.py + test_summary.py with conftest fixtures; (3) test_m9c_smoke.py with locally-replicated OR imported fixtures.

**PY8 — Y-D-DEVIATION-FORECAST-PER-COMMIT.** C0 docs: 0 drifts (structural pattern proven from M9.B + M9.A). C1 api modules+integration: 0-1 drifts (novel real-Anthropic SDK integration + cost-tracking activation + threshold-coordination; PY5 V-AB mitigations active; honest forecast 0-1). C2 api tests: 0 drifts (test-coverage; PY7 fixture-ordering discipline; arc-precedent M9.B C3 `f82c38a` strong). Cumulative session-tail framing-drift count: 66 + N where N is Stage 3 execution-time catches; honest N forecast = 0-1.

## 4. Pre-Verification Cascade Design

**V-BB pre-C1 cascade (~5-10min before Stage 3 C1 modules+integration landing):**
- V-BBA working tree state clean.
- V-BBB Vulnerability schema (ai_fix_text + correlation_cluster_id + severity_score + severity operational at M9.B + M9.0 baseline).
- V-BBC Scan schema (executive_summary + ai_pipeline_degraded + project_id/organization_id operational).
- V-BBD RawFinding mobile-context fields (mobile_os/mobile_permission/mobile_component_name + engine_category enum state).
- V-BBE cost_tracking operational (check_budget + log_ai_call + get_consumed_cost + SCAN_TYPE_BUDGETS_USD + AIAPICall + cost_usd Decimal per V-ABG).
- V-BBF AsyncAnthropic client + get_anthropic_client singleton operational (V-AAG re-confirmed).
- V-BBG anthropic SDK Message + TextBlock + Usage import surface empirical (V-ABE partial pre-verification for C1 imports).
- V-BBH Severity enum completeness (V-ABH operationalized).
- V-BBI EngineCategory completeness (V-ABI operationalized).
- V-BB surface report; PROCEED unless drift.

**V-CC pre-C2 cascade (~5-10min before Stage 3 C2 tests + conftest extension landing):**
- V-CCA working tree state clean (post-C1).
- V-CCB C1 module exports importable (fix_generation: generate_fix + _generate_fixes; summary: _generate_executive_summary).
- V-CCC pipeline.py extension shape (run() composition post-C1).
- V-CCD conftest.py current state for fixture extension (stub_openai + stub_qdrant pattern; extension awareness for stub_anthropic).
- V-CCE Anthropic SDK Message + TextBlock + Usage type construction empirical (V-ABE operationalized; verify constructor signature stable).
- V-CCF pipeline_mod.get_anthropic_client monkeypatch attachment empirical (V-ABF operationalized).
- V-CCG test_m9b_smoke.py structural precedent for test_m9c_smoke.py format-adapt (M9.B C3 `f82c38a` precedent).
- V-CC surface report; PROCEED unless drift.

## 5. Implementation Risk Surface

- **Risk class 1 — Real Anthropic SDK integration novel territory:** first real-load use of `AsyncAnthropic.messages.create` at M9.C; SDK version compatibility verified at V-BBG + V-CCE (anthropic SDK installed at M9.0 C1; version may have evolved); mitigation V-AB §9 (V-ABE Message construction; V-ABF monkeypatch attachment) + PY5 operationalization.
- **Risk class 2 — Cost-tracking activation under load:** ai_api_calls operational since M9.0 C1; first real-load use; per-vuln + per-summary log_ai_call rows (~10-30/scan); mitigation V-AAF empirical confirmation (AIAPICall + log_ai_call + check_budget + get_consumed_cost operational).
- **Risk class 3 — Threshold detection coordination:** `_generate_fixes` orchestrator tracks fallback_count + total_attempted; conditional ai_pipeline_degraded set; coordination with `_generate_executive_summary` (which may also set ai_pipeline_degraded per Q11 C.c); mitigation explicit threshold detection within _generate_fixes; summary failure separately sets degraded; combined OR semantics.
- **Risk class 4 — Within-lock under-implementation per 2-instance test-gate pattern:** M9.B C2 Q1 evidence-loading + M9.B C3 fixture-scoping precedent. M9.C C1 candidates: async semantics for asyncio.gather (Path A parallelism); threshold counter mutation correctness; SDK error class handling (anthropic.APIError vs generic Exception). M9.C C2 candidates: stub_anthropic attachment validity; Message construction signature; conftest cross-directory scoping (test_m9c_smoke.py in tests/integration/). Mitigation: explicit test gate at C1 + C2; PY7 fixture-ordering discipline; honest classification when surfaced.

## 6. Test Surface (~32-45 new tests forecast per PY4)

**Unit tests (~30-37 in test_fix_generation.py + test_summary.py):** `_build_target_context` dispatch (4 branches); `_build_mobile/sast/dast/generic_context` field population; `_format_evidence` truncation + null-handling; `_fallback_fix_template` substitution + degraded flag; `_call_claude_with_retry` 3-attempt scenarios (1st/2nd/3rd/exhausted); `generate_fix` cost-tracking (success + failure log_ai_call paths); `_generate_fixes` severity ordering + idempotency + defensive skip + threshold detection; `_build_summary_user_prompt` severity-tier + mid-detail + breakdown + corroboration; `_deterministic_fallback_summary` structure; `_generate_executive_summary` idempotency + budget exhaustion + claude error fallback + zero-vuln trivial.

**Smoke tests (~5-7 in test_m9c_smoke.py):** end-to-end M9.C pipeline (embed → dedup → correlate → score → fix-gen → summary); zero-vuln defensive skip + trivial summary; budget exhaustion graceful degradation; claude failures threshold degraded; severity-ordered processing; idempotency re-run (preserve existing).

## 7. Forward-Pin Chain Reinforcement (carries Stage 1 §8)

- Path B batched-call optimization at production-readiness (~$0.30+/scan threshold OR Quick scan exceedances).
- Path C cluster-batched optimization at ~30% cluster density.
- Vuln-count-aware fallback to Path B (>50 vulns).
- Circuit-breaker activation criteria (≥5% cascading OR budget exhaustion >10/day OR Anthropic outages >1h/quarter).
- Anthropic prompt caching at production-readiness (system prompt cacheable).
- Dynamic cost estimation at production-readiness (tiktoken-based).
- Admin force-regenerate flag (Q7 + Q12 idempotency override).
- Multi-provider abstraction layer (ADR-013 multi-provider future Phase-2).
- M10 Report layer integration: ai_fix_text + executive_summary surface in customer reports.

## 8. Cumulative Budget Tracking

- Cumulative session-tail framing-drift count at Stage 2 plan landing: **66** (preserved through M9.A + M9.B + M9.C Stage 1; this Stage 2 plan lands at 0 increment).
- 6-instance averted-prediction lineage operational (V-IIB + V-JJC + V-MM + V-NNE + V-UUE + V-VV/V-WW/V-XX/V-YY); extending to 7-instance at V-BB + V-CC pre-C1/C2 cascades operationalization at Stage 3.
- 2-instance test-gate-within-lock pattern operational (M9.B C2 Q1 evidence-loading + M9.B C3 fixture-scoping); honest forecast for M9.C C1: possible 0-1 within-lock catch.
- Forward-pin trigger phrases for Stage 3: "Begin M9.C — Stage 3 C0 ADR-032 docs landing" / "Begin M9.C — Stage 3 C1 modules + integration" / "Begin M9.C — Stage 3 C2 tests + smoke".
