# M9.B — AI Pipeline: Correlation + Scoring (ADR-031): Design

**Status:** Brainstorming chain Y-CORRELATION-MERGE-VS-LINK Path B gate-decision + Q1-Q18 locks complete + V-TT pre-verification + V-UU pre-Stage-1 refinements (this session); ready for Stage 2 implementation plan landing + Stage 3 4-commit implementation (C0 docs ADR-031 → C1 schema + modules → C2 pipeline integration → C3 tests + smoke) + Stage 4 P5.A.

**Date:** 2026-06-25.

**Authority:** V-TT pre-verification surface report (this session; M9.B architectural decision space grounded; SPEC §8.2 rule-based weighted scoring distinct from M9.A Qdrant cosine dedup; exploitability inputs thinnest empirical area; CWE hierarchy + route_map + parameter extraction undefined-helper gaps surfaced); M9.B brainstorming chain CLOSED this session (Mode 1 sequential conversational; Y-CORRELATION-MERGE-VS-LINK Path B gate-decision + Q1-Q18 + ~45+ sub-decisions ratified); V-UU pre-Stage-1 verification (this session); M9.A Stage 1 design doc structural precedent (commit `aaf7ea0`; 200 LoC §1-§9); M9.A lifecycle CLOSED (8-commit chain; ADR-030 canonical authority operational; Path C cluster→Vulnerability promotion operational); ADR-029 SPEC §13 canonical authority (M9.0 foundation composed); ADR-030 SPEC §13 canonical authority (M9.A composed); SPEC §8.2 correlation weights canonical (cwe_exact 0.40 + cwe_parent 0.25 + url_path 0.20 + finding_type 0.30 + parameter_name 0.15; threshold ≥0.75); SPEC §8.3 scoring formula canonical (final_score = base_cvss × corroboration × exploitability); CLAUDE.md Gotcha 5 cost-tracking mandate (operational at M9.A first real-cost logging; no new cost at M9.B per Q14 deterministic computation lock); 66 cumulative session-tail framing-drift discipline (Drift #58-#66 + #66-averted lineage; Drift #66 NOVEL ORM-vs-DB layer assumption mismatch operational from M9.A C1).

**Related:** Implementation plan landing trigger: ***"Begin M9.B — Stage 2 implementation plan landing"***. Sub-milestone activation triggers after M9.B closes: ***"Begin M9.C — Fix Generation + Executive Summary"*** → ***"Begin M9.D — Pipeline Orchestrator"***.

---

## 1. Authority + Y-CORRELATION-MERGE-VS-LINK + Q1-Q18 Locks Summary

**Y-CORRELATION-MERGE-VS-LINK gate-decision (ratified upfront before the Mode 1 chain): Path B — Link + Corroborate.** Two Vulnerabilities scoring ≥0.75 per SPEC §8.2 → both rows preserved (M9.A Path C lock honored literally); correlation creates link relationships via `Vulnerability.correlation_cluster_id` (UUID; union-find clustering); `corroborated_count` updated from cross-engine evidence per SPEC §8.3; `severity`/`severity_score` refined per SPEC §8.3; no row deletion at M9.B; SPEC §8.2 "merged into corroborated vulnerability" read as "linked into corroboration relationship." Path B composes + extends M9.A Path C, not replace.

Brainstorming chain (Q1-Q18) resolved under Mode 1 sequential conversational deliberation:

- **Q1 Y-CORRELATION-ALGORITHM-IMPLEMENTATION (A.c + SPEC weights + C.b + D.a):** `correlation_score` takes Vulnerability instances + loads raw_finding fields via `raw_finding_ids`; SPEC §8.2 weights canonical; ≥0.75 strict threshold + score stored for tunability; sync function (no DB inside); cwe_hierarchy pre-loaded at pass entry.
- **Q2 Y-CORRELATION-QUERY-PATTERN (A.a + B.b + C.a):** SQL pairwise; N² acceptable at pre-launch scale (~10-50 vulns/scan → ~25-625 pairs); cross-engine_category generalized; skip same-engine pairs (M9.A clustering scope).
- **Q3 Y-CORRELATION-SCOPE (A):** per-scan primary (honors Q5-M9.0 lock); cross-scan forward-pinned to M11+.
- **Q4 Y-CORRELATION-PAIRWISE-COMPLEXITY (A.a + symmetric + C.a):** `itertools.combinations` + engine_category filter; neutral naming (vuln_a/vuln_b); asymmetric url_path tried both directions with max-not-sum (avoids 0.40 double-count).
- **Q5 Y-CWE-PARENT-CHILD-SOURCE (A):** hardcoded `CWE_PARENT_CHILD` dict + top-~50 coverage; `CWEHierarchy` class (bidirectional lookup) + module-level singleton; MITRE CWE-1000 Research View; library forward-pinned.
- **Q6 Y-ROUTE-MAP-CONSTRUCTION (B):** heuristic 1-to-1 (`build_route_map`; last URL-path segment ↔ code_file basename; "api" filtered; ambiguous skipped); AI/uploaded/static-analysis forward-pinned.
- **Q7 Y-PARAMETER-EXTRACTION-HELPERS (A.c)' + (B.b)' + (C.b):** `raw_finding.parameter` primary + URL query fallback; multi-language regex (Flask/Django/FastAPI/Express/Spring) + len≥3; normalized (lowercase + alphanumeric); AST forward-pinned.
- **Q8 Y-CORRELATION-STORAGE-SHAPE (A):** `Vulnerability.correlation_cluster_id` nullable UUID, indexed; transitive A↔B↔C merged via union-find with path compression; junction table forward-pinned.
- **Q9 Y-SCORING-FORMULA-IMPLEMENTATION:** `final_score = base_cvss × corroboration × exploitability` (SPEC §8.3 literal); cap 10.0; sync `compute_severity_score`; default base_cvss=5.0 when null; uncapped-tracking forward-pinned.
- **Q10 Y-EXPLOITABILITY-INPUTS-SOURCE (D):** hybrid — PoC-proven derivation from `RawFinding.tool_name` ∈ {"nuclei","sqlmap"} → 1.2; auth/public default 1.0 at M9.B (URL heuristics insufficient); detection forward-pinned to M9.C/D + production-readiness. *(V-UUE: `engine_name` absent; `tool_name` is the field — see §9.)*
- **Q11 Y-VULNERABILITY-SEVERITY-UPDATE-SHAPE (A.a + CVSS mapping + C.c):** add `Vulnerability.severity_score` Float (M9.B migration); standard CVSS mapping (≥9.0 CRITICAL / 7.0-8.9 HIGH / 4.0-6.9 MEDIUM / <4.0 LOW); unconditional update; cvss_score + severity + severity_score kept distinct.
- **Q12 Y-CORROBORATED-COUNT-COMPUTATION (A):** `corroborated_count = len({v.engine_category for v in cluster})`; aligns with SPEC §8.3 "2+ engines"; unconditional update for all cluster members.
- **Q13 Y-PIPELINE-RUN-EXTENSION-FOR-M9.B (A + C):** add `_correlate_vulnerabilities` + `_score_vulnerabilities` after `_dedup_and_promote` in `run()`; defensive early-exit when <2 distinct engine_categories.
- **Q14 Y-COST-TRACKING-AT-M9.B (A):** none — correlation + scoring purely deterministic; zero ai_api_calls impact; Claude tie-break (0.65-0.85) forward-pinned.
- **Q15 Y-TEST-FIXTURE-STRATEGY-EXTENSION (A.a + B.a + C.a + C.b):** per-domain test files (test_correlation/test_scoring/test_cwe_hierarchy + extend test_pipeline + test_m9b_smoke); `cross_engine_vulnerabilities` + `single_engine_vulnerabilities` fixtures; real + minimal CWE dicts; ~430-660 LoC.
- **Q16 Y-MIGRATION-NEEDED-AT-M9.B:** single alembic migration (2 columns + 1 index); revision after `b7e4a1f93c2d`; correlation_cluster_id indexed; severity_score not-indexed (M10 forward-pin); explicit `downgrade()`.
- **Q17 Y-ADR-NUMBER:** ADR-031 "AI Pipeline: Correlation + Scoring (M9.B)"; composes ADR-029 + ADR-030.
- **Q18 Y-STAGE3-DECOMPOSITION-FOR-M9.B:** 4-commit + top-down docs-first (C0 docs ~80-120 + C1 schema/modules ~400-550 + C2 pipeline ~200-300 + C3 tests ~430-660; aggregate ~1110-1630).

## 2. Pre-Verification Findings

**V-TT (last session):** M9.B decision space grounded; SPEC §8.2 rule-based weighted scoring is a distinct mechanism from M9.A Qdrant cosine dedup; §8.3 final_score deterministic; M9.A foundation operational re-verified; exploitability inputs (PoC/auth/public) thinnest empirical area; CWE parent-child + route_map + parameter extraction undefined in codebase (V-MM-class undefined-helper gaps; addressed in Q5/Q6/Q7); 13 EngineCategory members; Y2 forward-pin CLOSED at M9.A C2.

**V-UU pre-Stage-1 (this session):** V-UUA clean; V-UUB ADR-031 insertion point (between ADR-030 line 2632 + §14 line 2726); V-UUC §1-§9 precedent confirmed; V-UUD Tasks 9.3/9.4 Status (no CLOSED annotation; awaits M9.B P5.A); V-UUE empirical fields (Vulnerability lacks correlation_cluster_id + severity_score → Q16 migration; RawFinding has `tool_name` not `engine_name` → Q10 grounding; Scan has is_public/config/project_id; alembic head `b7e4a1f93c2d`); V-UUF disambiguation context preserved.

## 3. Architectural Decisions (Y-Lock Detail)

### 3.0 Y-CORRELATION-MERGE-VS-LINK (Path B) Gate-Decision
**Lock:** Link + Corroborate; M9.A Path C lock honored literally; correlation creates `correlation_cluster_id` link relationships; no row deletion at M9.B.
**Grounding:** Q5-M9.0 + Q6-M9.A preserve Vulnerability rows per cluster; Path B composes + extends; SPEC §8.2 "merged" read as "linked into corroboration relationship."
**Rejected:** Path A (Merge — contradicts M9.A Path C; deletes rows M9.A created); Path C-hybrid (Link@M9.B + Merge@M9.D — surface without M9.B value); Mode-2 single-doc (heavier than needed).
**Cascade:** migration adds correlation_cluster_id + severity_score; Y2 query → DISTINCT(correlation_cluster_id); corroborated_count engine-distinct within cluster; severity update preserves rows.

### 3.1–3.18 Y-Lock detail
Each Y resolves as locked in §1 (lock + grounding + rejected-alternatives + sub-decisions). Highlights with non-obvious rejected alternatives:
- **§3.1 (Q1):** rejected async function (no DB inside → sync; cwe_hierarchy pre-loaded).
- **§3.2 (Q2):** rejected Qdrant cross-search (§8.2 is rule-based, not cosine) + same-engine inclusion (M9.A owns that).
- **§3.5 (Q5):** rejected runtime MITRE API call (offline determinism) + full CWE-1000 import (over-scoped for pre-launch).
- **§3.6 (Q6):** rejected exact-path matching (too brittle) + AI route_map now (cost + ~$0.10/scan threshold not met).
- **§3.7 (Q7):** rejected AST extraction now (dependency burden) + single-language regex (misses cross-stack).
- **§3.8 (Q8):** rejected junction-table-now (audit surface without M9.B need) + pairwise flags (no transitive clustering).
- **§3.10 (Q10):** rejected URL-heuristic auth/public detection (~30-70% accuracy insufficient) → default 1.0 + forward-pin.
- **§3.11 (Q11):** rejected overwriting `severity` enum only (loses numeric ranking) → add `severity_score`.
- **§3.16 (Q16):** rejected no-migration (Q11/Q8 need columns) + indexing severity_score now (M10 concern).

## 4. M9.B Stage 3 Implementation Surface

### 4.1 C0 — ADR-031 Docs at SPEC §13
ADR-031 canonical text between ADR-030 + §14: Status + Context + Decision (Path B + Q1-Q18) + Rationale + Rejected Alternatives + Consequences + Composition (ADR-013/014/022/028/029/030) + Cross-references. **~80-120 LoC.**

### 4.2 C1 — api schema + modules
`alembic/versions/[hash]_add_m9b_correlation_scoring_columns.py` (correlation_cluster_id UUID indexed + severity_score Float; explicit downgrade()); `cwe_hierarchy.py` NEW (CWE_PARENT_CHILD + CWEHierarchy + singleton); `correlation.py` NEW (CORRELATION_WEIGHTS + correlation_score + build_route_map + extract_params_* + _union_find_clusters); `scoring.py` NEW (compute_severity_score + compute_exploitability_multiplier + _map_score_to_severity_enum); `models/vulnerabilities.py` (2 Mapped declarations). **~400-550 LoC.**

### 4.3 C2 — api pipeline integration
`pipeline.py` `run()` extended with `_correlate_vulnerabilities` + `_score_vulnerabilities` + defensive skip <2 engines. **~200-300 LoC.**

### 4.4 C3 — api tests + smoke
`test_correlation.py` + `test_scoring.py` + `test_cwe_hierarchy.py` NEW + `test_pipeline.py` extensions + `test_m9b_smoke.py` NEW. **~430-660 LoC.**

**Aggregate Stage 3 forecast: ~1110-1630 LoC across 4 commits.**

## 5. Phase Structure
Stage 1 design doc (THIS COMMIT) → Stage 2 implementation plan → Stage 3 4-commit api-side → Stage 4 P5.A annotations.

## 6. Out of Scope
1. M9.C (Tasks 9.5+9.6; fix-gen + summary) — own lifecycle.
2. M9.D (Task 9.7; orchestrator) — own lifecycle.
3. Cross-scan correlation (Q3 → M11+).
4. Per-pair correlation_score audit junction table (Q8 → production-readiness).
5. Auth-path + public-facing exploitability detection (Q10 → M9.C/D + production-readiness).
6. Claude tie-break for near-threshold correlations (Q14 → production-readiness).
7. AI route_map + uploaded override + framework-specific static analysis (Q6 forward-pin).
8. Embedded MITRE CWE library + AST parameter extraction (Q5+Q7 forward-pin).
9. severity_score index for M10 ranking (Q16 forward-pin).
10. Uncapped final_score tracking (Q9 forward-pin).

## 7. Forward-Pins
**Pre-implementation:** (1) Stage 2 plan landing trigger; (2) V-UU PROCEED confirmed; V-UUE `engine_name` absence → Q10 uses `tool_name` (DEFERRED-EMPIRICAL, §9).
**Sub-milestone triggers:** (3) M9.C trigger; (4) M9.D trigger.
**Discipline-level (carried + extended):** (5) audit-driven model+spec orphan check; (6) DEFERRED-EMPIRICAL (Q5/Q6/Q7 undefined-helper gaps + V-UUE tool_name); (7) recon-invocation-seam-extension (run() composition at C2); (8) test-impact-surface (Drift #63; C3 scope); (9) plan-template-discipline (ancillary files anticipated); (10) averted-prediction (V-IIB/V-JJC/V-MM/V-NNE + V-TT/V-UU = 5th-instance); (11) NEW Drift #66 ORM-vs-DB layer assumption mismatch (C1 migration + ORM mapping awareness).
**Production-readiness audit chain (M9.B):** (12) cwe_id-pre-filter at >500 vulns/scan; (13) AI route_map + uploaded override + framework static analysis; (14) embedded MITRE CWE library; (15) AST parameter extraction; (16) vulnerability_correlations junction table; (17) over-clustering edge-case investigation; (18) auth/public detection at M9.C/D; (19) is_proof_of_concept/is_authenticated_path/is_publicly_accessible columns if persistence needed; (20) uncapped severity_score tracking; (21) severity_score index at M10; (22) Claude tie-break (0.65-0.85).

## 8. Cross-References
**Docs:** `c9130db` (M9.A P5.A docs) + `d408b2c` (ADR-030 SPEC §13) + `aaf7ea0` (M9.A Stage 1 precedent) + `a8ad52c` (M9.A Stage 2 precedent) + `45dcabe` (ADR-029 SPEC §13); SPECIFICATION.md §8.2 + §8.3 canonical + SPEC §13 ADR-029/030 (composed) + ADR-031 (target); CLAUDE.md Gotcha 5.
**Engine:** None (M9.B is api-side only).
**API:** `cee75bf` (M9.A P5.A api DRIFT-LOG Drift #66) + `251960a` (M9.A C2 tests; M9.B mirrors) + `91ec273` (M9.A C1; M9.B extends run()); `pipeline.py` (C2 target); `correlation.py` + `scoring.py` + `cwe_hierarchy.py` (NEW at C1); `models/vulnerabilities.py` (2 new columns); `alembic/versions/` (revision after `b7e4a1f93c2d`).
**Multi-session arc:** 99 commits + 66 framing-drift catalogue + Drift #66 NOVEL ORM-vs-DB layer assumption mismatch + 18 closed task lifecycles + M5/M6/M7/M8/M9.0/M9.A CLOSED + Y2 Task 8.3β CLOSED at M9.A C2.

## 9. Path B Resolution + Undefined-Helper Gap Closures + Empirical Pre-Grounding

**V-TT surfaced (multi-helper gap analogous to M9.A V-MM merge_evidence):** plan-literal IMPLEMENTATION-PLAN Task 9.3 references undefined helpers — `is_cwe_parent_child(cwe_a, cwe_b)`, `route_map` construction, `extract_params(url)` + `extract_code_params(snippet)`. None exist in codebase.

**Resolution within brainstorming chain (no catalogued drift; averted-prediction):**
- Q5 → `is_cwe_parent_child` via curated `CWE_PARENT_CHILD` dict + `CWEHierarchy` class (`cwe_hierarchy.py`).
- Q6 → `route_map` via heuristic 1-to-1 URL-path↔code_file matching.
- Q7 → `extract_params` (URL + `raw_finding.parameter` hybrid) + `extract_code_params` (multi-language regex).

**Path B resolution at the gate-decision:** two Vulnerabilities ≥0.75 → both rows preserved (M9.A Path C honored); `correlation_cluster_id` (UUID) assigned via union-find over threshold-crossing pairs; transitive A↔B + B↔C → all share cluster_id; Y2 vulnerability_count adapts via DISTINCT(correlation_cluster_id) to preserve accurate distinct-vulnerability counts.

**V-UUE empirical pre-grounding (DEFERRED-EMPIRICAL; mirrors V-NNE project_id at M9.A):** `RawFinding` has **no `engine_name`** column — it has `tool_name` (String 50) + `engine_category` (enum). The plan-literal/Q10 "engine name match against {nuclei, sqlmap}" therefore resolves against **`RawFinding.tool_name`** at C1. Pre-grounded here to avert a Drift #61-class concrete-field catch at C1 execution. Vulnerability confirmed lacking `correlation_cluster_id` + `severity_score` (Q16 migration adds both); alembic head `b7e4a1f93c2d` (Q16 revisions after).

**Averted discipline applied:** V-TT/V-UU catches (3 undefined-helper gaps + exploitability empirical thinness + `engine_name`→`tool_name`) addressed within the brainstorming chain + pre-grounding, not catalogued as new drifts; mirrors M9.A V-MM/V-NNE averted-prediction. Cumulative drift count remains 66.
