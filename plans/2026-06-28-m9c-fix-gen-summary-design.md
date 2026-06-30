# M9.C AI Pipeline: Fix Generation + Executive Summary — Stage 1 Design Doc

**Status:** Stage 1 design ratified; Stage 2 plan + Stage 3 implementation forthcoming.

**Date:** 2026-06-28.

**Authority:** ADR-032 (forthcoming at Stage 3 C0; "AI Pipeline: Fix Generation + Executive Summary (M9.C)" at SPEC §13); SPEC §8.4 (Mobile-Specific AI Fix Generation) + SPEC §8.5 (Multi-Provider AI Strategy + budget targets) canonical; ADR-029 (foundation + ai_api_calls + Gotcha 5) + ADR-030 (M9.A Path C promotion) + ADR-031 (M9.B Path B link-and-corroborate) + ADR-013 (Python sole-writer for scan state) composition. M9.C brainstorming chain CLOSED this session (Mode 1 sequential; 2 gate-decisions + Q1-Q16 ratified). V-AA pre-verification cascade grounded the decision space; 6-instance averted-prediction lineage + Drift #66 ORM-vs-DB catch-class + 2-instance test-gate-within-lock pattern operational; cumulative session-tail framing-drift count 66 (preserved through M9.A + M9.B lifecycles).

**Related:** Stage 2 plan landing trigger: ***"Begin M9.C — Stage 2 implementation plan landing"***. Sub-milestone activation after M9.C closes: ***"Begin M9.D — Pipeline Orchestrator"***.

---

## 1. Y-Lock Summary (2 gate-decisions + Q1-Q16)

**Gate-1 Y-FIX-GEN-PER-VULN-VS-BATCH = Path A (per-vuln).** One Claude call per Vulnerability; `asyncio.gather` parallelism; per-vuln `check_budget` pre-call; isolated failures + `_fallback_fix_template`; per-vuln `ai_api_calls` granularity; idempotency via `WHERE ai_fix_text IS NULL`; Path B (batched) + Path C (cluster-batched) forward-pinned to production-readiness.

**Gate-2 Y-CIRCUIT-BREAKER-ACTIVATION = (B) forward-pin to production-readiness.** M9.C ships per-vuln graceful degradation + `check_budget` pre-call + `Scan.ai_pipeline_degraded` threshold; circuit-breaker activation criteria forward-pinned (≥5% scans cascading failures OR budget exhaustion >10/day OR Anthropic outages >1h/quarter).

- **Q1 Y-FIX-GEN-PROMPT-TEMPLATE (A.c + B.b + C.b):** single template + dynamic `_build_target_context` + structured prose sections (`## Explanation` / `## Fix Code` / `## Remediation Steps`); system + user prompt split; `FIX_GEN_SYSTEM_PROMPT` carries role + output format + constraints; max 500 words.
- **Q2 Y-FIX-GEN-PROMPT-CONSTRUCTION (A.a + B.a + C.b):** representative raw_finding (first by `raw_finding_ids`; per M9.B C2 `6c9e270` Q1 evidence-loading precedent) + all `mobile_*` fields when MOBILE (SPEC §8.4 N/A treatment) + cluster summary one-liner in TARGET CONTEXT.
- **Q3 Y-FIX-GEN-MOBILE-CONTEXT-DETECTION (A):** `engine_category == MOBILE` only.
- **Q4 Y-2-UNDEFINED-HELPER-RESOLUTION:** `_format_evidence` hybrid structured (tool_name + engine + description-trunc 500 + code_snippet-trunc 1000); `_fallback_fix_template` static template + variable substitution (severity + cwe_id + finding_type + standard 3-section structure; sets `ai_pipeline_degraded` at first invocation).
- **Q5 Y-FIX-GEN-COST-BUDGET-INTEGRATION (A.c + B.a + C.b + D.a):** `ESTIMATED_FIX_COST_USD` $0.025 conservative + per-vuln `check_budget` pre-call + `log_ai_call` success+failure (failure: cost_usd=0 + tokens=0) + single `operation_type="fix_generation"`.
- **Q6 Y-FIX-GEN-WHICH-VULNS (C):** severity-ordered top-N by `severity_score` DESC; budget enforces via per-vuln graceful fallback.
- **Q7 Y-FIX-GEN-IDEMPOTENCY (A):** preserve existing (skip `WHERE ai_fix_text IS NOT NULL`); admin force-regenerate flag forward-pinned.
- **Q8 Y-FIX-GEN-FAILURE-HANDLING (A.b + B.b + C.b):** 3-attempt exp backoff (1s/2s/4s) + threshold-based `ai_pipeline_degraded` (≥3 OR ≥30%) + explicit "AI fix unavailable" header (via Q4 `_fallback_fix_template`).
- **Q9 Y-EXEC-SUMMARY-PROMPT-TEMPLATE (A.a + B.b + C.b):** mirror Q1 system+user split + 4-section structure (`## Executive Summary` / `## Key Findings` / `## Risk Posture` / `## Recommended Actions`) + C-level executive framing; max 600 words.
- **Q10 Y-EXEC-SUMMARY-INPUT-CONSTRUCTION (A.c + B.b + C.c + D.a):** severity-tier (all CRITICAL + all HIGH + top-5 MEDIUM; skip LOW) + mid-detail per finding (title + severity + cwe_id + location + corroborated_count) + breakdown by severity + top-5 CWE families + aggregate corroboration signal.
- **Q11 Y-EXEC-SUMMARY-COST-BUDGET (A.a + B.a + C.c):** `ESTIMATED_SUMMARY_COST_USD` $0.05 + pre-call `check_budget` + deterministic minimal fallback summary on failure (severity counts + 4-section structure per Task 9.6 hint).
- **Q12 Y-EXEC-SUMMARY-IDEMPOTENCY (A):** preserve existing (skip if `Scan.executive_summary IS NOT NULL`); admin force-regenerate flag forward-pinned.
- **Q13 Y-PIPELINE-RUN-EXTENSION-FOR-M9.C (A.a + B.a + C.b):** sequential `_generate_fixes` → `_generate_executive_summary` + before `db.flush()` (M9.B C2 `6c9e270` atomic-scan pattern) + defensive skip when no vulnerabilities + trivial deterministic `executive_summary` for zero-vuln scans.
- **Q14 Y-TEST-FIXTURE-STRATEGY (A.a + B.b + C.b):** `stub_anthropic` monkeypatch on `pipeline_mod` + dynamic `make_anthropic_response` constructor + side-effect list with exceptions interleaved (retry-then-success + retry-exhausted-fallback scenarios).
- **Q15 Y-STAGE3-DECOMPOSITION (A):** 3-commit — C0 ADR-032 docs canonical + C1 api modules + integration + C2 api tests; forward-pin (B) 4-commit splitting C1 IF empirical LoC >500.
- **Q16 Y-ADR-NUMBER:** ADR-032 "AI Pipeline: Fix Generation + Executive Summary (M9.C)" at SPEC §13.

## 2. Architecture Overview

ADR-032 composition: ADR-029 (foundation + ai_api_calls + Gotcha 5) + ADR-030 (M9.A Path C cluster→Vulnerability promotion) + ADR-031 (M9.B Path B link-and-corroborate) + ADR-013 (sole-writer: api writes `ai_fix_text` + `executive_summary`) + SPEC §8.4 (Mobile-Specific AI Fix Generation) + SPEC §8.5 (Multi-Provider AI Strategy + budget targets).

M9.C is the THIRD real-AI sub-milestone (Tasks 9.5 fix-generation + 9.6 executive summary) and the FIRST sub-milestone making real Anthropic Claude API calls; it activates the `ai_api_calls` cost-tracking infrastructure scaffolded since M9.0. Per-vulnerability fix generation (Path A); single-call executive summary (Q9); sequential composition after M9.B `_score_vulnerabilities`. Deterministic CWE-keyed fallback discipline: Q4 `_fallback_fix_template` + Q11 `_deterministic_fallback_summary` cover all failure scenarios (Claude API errors + budget exhaustion + retry-exhausted). V-AAH/I no-migration: `ai_fix_text` (line 102) + `executive_summary` (line 95) already scaffolded at M9.0 C1 → 3-commit Stage 3 (vs M9.B 4-commit). Cost discipline: SPEC §8.5 budget targets (Quick $0.08 / Full web $0.25 / Full spectrum $0.55) enforced via `check_budget` pre-call per-vuln + per-scan-summary; graceful degradation when budget exhausted.

## 3. Module Surface

**`fix_generation.py` NEW (~150-250 LoC):**
- `FIX_GEN_SYSTEM_PROMPT` constant (role + output format + constraints + max 500 words).
- `FIX_GEN_USER_PROMPT_TEMPLATE` constant (finding + severity + cwe + description + target_context + evidence sections).
- `_build_target_context(vuln, raw_findings)` dispatches by `engine_category` (MOBILE/SAST/DAST/generic); `_build_mobile_context` / `_build_sast_context` / `_build_dast_context` / `_build_generic_context` helpers.
- `_format_evidence(vuln, raw_finding)` hybrid structured (Q4).
- `_fallback_fix_template(vuln, raw_findings)` static template + substitution (Q4); sets `Scan.ai_pipeline_degraded` at first invocation (Q8 B.b coordinator).
- `_call_claude_with_retry(client, system, user_prompt)` 3-attempt exp backoff (1s/2s/4s) (Q8 A.b).
- `generate_fix(*, db, vulnerability, raw_findings, scan)` main entry (Q1+Q4+Q5+Q8).
- `_generate_fixes(*, db, scan_id)` orchestrator (Q6+Q7+Q8 B.b threshold); called from `pipeline.run()` (Q13).
- Constants: `ESTIMATED_FIX_COST_USD = Decimal("0.025")` + `RETRY_MAX_ATTEMPTS = 3` + `RETRY_BACKOFF_SECONDS = [1,2,4]` + `DEGRADED_THRESHOLD_COUNT = 3` + `DEGRADED_THRESHOLD_PCT = 0.30`.

**`summary.py` NEW (~80-130 LoC):**
- `SUMMARY_SYSTEM_PROMPT` constant (role + 4-section structure + C-level framing + max 600 words).
- `SUMMARY_USER_PROMPT_TEMPLATE` constant (scan context + vuln breakdown + top findings + corroboration summary).
- `_build_summary_user_prompt(scan, vulnerabilities)` (Q10).
- `_deterministic_fallback_summary(vulns)` (Q11 C.c).
- `_generate_executive_summary(*, db, scan_id)` main entry (Q9+Q10+Q11+Q12 idempotency + Q13 C.b zero-vuln defensive).
- Constant: `ESTIMATED_SUMMARY_COST_USD = Decimal("0.05")`.

**`pipeline.py` extension (~10-20 LoC):** imports `_generate_fixes` + `_generate_executive_summary`; `run()` extension after `await _score_vulnerabilities`: `await _generate_fixes(db=db, scan_id=scan.id)` then `await _generate_executive_summary(db=db, scan_id=scan.id)`.

## 4. Cost-Budget Integration

`check_budget` signature per V-AAF: `(*, db, scan_id, scan_type, estimated_cost_usd)` — raises `BudgetExceeded`. `log_ai_call` signature per V-AAF: `(provider, model, operation_type, tokens_in, tokens_out, cost_usd)`. Per-vuln `check_budget` pre-call (Q5 B.a); per-summary `check_budget` pre-call (Q11 B.a). Cost estimates: `ESTIMATED_FIX_COST_USD` $0.025 + `ESTIMATED_SUMMARY_COST_USD` $0.05. SPEC §8.5 alignment: Quick $0.08 budget allows ~3 fix-gens + 1 summary; Full web $0.25 ~8 fix-gens + 1 summary; Full spectrum $0.55 ~20 fix-gens + 1 summary. `log_ai_call` success: cost_usd from Sonnet pricing (input_tokens × $3/M + output_tokens × $15/M); `operation_type="fix_generation"` OR `"executive_summary"`. `log_ai_call` failure: cost_usd=0 + tokens=0 marker (Q5 C.b + D.a).

## 5. Failure Handling + Degradation

`_call_claude_with_retry`: 3-attempt exp backoff (1s/2s/4s) per Q8 A.b + M9.A Q3 C.c precedent. `BudgetExceeded` → graceful per-vuln fallback (Q5 B.a) + `Scan.ai_pipeline_degraded` set. Claude API error (after retry exhausted) → `_fallback_fix_template` (Q4) + `log_ai_call` failure + `fallback_count++` tracking. `Scan.ai_pipeline_degraded` activation per Q8 B.b: ≥3 fallbacks OR ≥30% of scan vulns hit fallback (cumulative across fix-gen + summary). Circuit-breaker forward-pinned per Gate-2 B; activation criteria: ≥5% scans cascading failures OR budget exhaustion >10/day OR Anthropic outages >1h/quarter. Executive summary failure → `_deterministic_fallback_summary` (Q11 C.c) preserves customer-facing content.

## 6. Test Fixture Strategy

`stub_anthropic` fixture in conftest.py (Q14 A.a) monkeypatches `pipeline_mod.get_anthropic_client`; mirrors `stub_openai_client` + `stub_qdrant_client` (M9.A C2 `251960a` + M9.B C3 `f82c38a` precedent). `make_anthropic_response(text, input_tokens, output_tokens)` constructor (Q14 B.b) returns `anthropic.types.Message` with `content=[TextBlock(text)]` + `usage=Usage(input_tokens, output_tokens)`. `_StubAnthropicMessages.create` accepts side-effect list with exceptions interleaved (Q14 C.b); supports retry-then-success + retry-exhausted-fallback scenarios. Test surface: `test_fix_generation.py` NEW (~15-20 tests) + `test_summary.py` NEW (~8-12 tests) + `test_m9c_smoke.py` NEW (end-to-end; ~5-7 tests) + conftest.py extension.

## 7. Stage 3 Decomposition (Q15 A — 3-commit)

- **C0** docs ADR-032 SPEC §13 canonical (~120-150 LoC docs delta).
- **C1** api `fix_generation.py` + `summary.py` NEW modules + `pipeline.py` integration + imports (~240-400 LoC).
- **C2** api tests (`test_fix_generation.py` + `test_summary.py` + `test_m9c_smoke.py` + conftest `stub_anthropic` + `make_anthropic_response`; ~400-500 LoC).
- Forward-pin: split C1 to (B) 4-commit modules + integration IF empirical LoC >500.

## 8. Forward-Pin Chain (M9.C → production-readiness)

- Path B batched-call optimization when cost pressure ~$0.30+/scan threshold OR Quick scan budget exceedances.
- Path C cluster-batched optimization when M9.B cluster density exceeds ~30% of scan vulns (correlation-density empirical analysis).
- Vuln-count-aware fallback to Path B for high-vuln-count scans (>50 vulns).
- Circuit-breaker activation criteria: ≥5% scans cascading failures OR budget exhaustion >10/day OR Anthropic outages >1h/quarter.
- Anthropic prompt caching at production-readiness (system prompt cacheable; reduces per-call cost ~50%).
- Dynamic cost estimation (replace fixed $0.025 + $0.05 with tiktoken-based prompt sizing).
- Admin force-regenerate flag (Q7 + Q12 idempotency override).
- Q12 (C) conditional smart-regenerate via `Scan.executive_summary_at` timestamp (requires schema addition; deferred to post-launch).
- Multi-provider abstraction layer (SPEC §8.5 future Phase-2 per single-provider Phase-1 lock).

## 9. V-AB DEFERRED-EMPIRICAL Pre-Grounding

Per 6-instance averted-prediction lineage (operational) + arc-precedent V-UUE (M9.B). These mark testable empirical assumptions deferred to Stage 2 plan or Stage 3 execution; analogous to M9.B V-UUE `tool_name` DEFERRED-EMPIRICAL pre-grounding that prevented a Drift #61-class concrete-field catch at M9.B C1 (extends lineage 6→7-instance).

- **V-ABE:** Anthropic SDK `Message` type construction shape (`anthropic.types.Message` + `TextBlock` + `Usage` import surface + parameter signatures); verify at Stage 2 plan + Stage 3 C2 conftest extension; SDK evolution may shift type construction.
- **V-ABF:** `pipeline_mod.get_anthropic_client` current state for monkeypatch attachment (per V-AAG `get_anthropic_client` singleton ready; re-verify at Stage 3 C2 conftest time).
- **V-ABG:** `cost_usd` precision (Decimal vs Float) at `log_ai_call` argument type + `AIAPICall.cost_usd` column type; re-verify at Stage 3 C1.
- **V-ABH:** Severity enum reference completeness (CRITICAL/HIGH/MEDIUM/LOW members + `severity_score` → severity mapping at Q6/Q10 reference).
- **V-ABI:** EngineCategory branching completeness (MOBILE/SAST/DAST + 10 other categories per M9.B V-TTG 13-member enum; `_build_generic_context` fallback covers non-MOBILE/SAST/DAST gracefully).
