# M9.0 — AI Analysis Pipeline Foundation (ADR-029): Design

**Status:** M9.0 lifecycle CLOSED (2026-06-16 P5.A landing). 8-commit chain: design a46fedd + plan 55dbe32 + Stage 3 C0 docs ADR-029 45dcabe + C1 api schema/modules 51b26ea + C2 api consumer/dispatch 1c98330 + C3 api tests/smoke 8410df4 + Stage 4 P5.A docs annotations + persistent DRIFT-LOG sync. ADR-029 architecture OPERATIONAL (foundation; no-op pipeline per Q11 C.c-lite — real stages at M9.A+). Drift #64 + #65 closed (architectural at ADR-029 + code at C1); Drift #66-averted (V-JJC predicted, DQ3 design averted). Sub-milestone decomposition per Q11: M9.0 (this foundation, CLOSED) → M9.A (embed/dedup) → M9.B (correlate/score) → M9.C (fix/summary) → M9.D (orchestrator); strict linear sequencing.

**Date:** 2026-06-12.

**Authority:** V-EE pre-verification surface report (this session; M9 architectural decision space grounded; 3 empirical gaps surfaced — ai_api_calls absence + Scan.executive_summary absence + pipeline trigger seam undefined; ~11 Y-decisions territory identified preliminarily; SPEC §8 canonical authority confirmed); M9.0 brainstorming chain CLOSED this session (Mode 1 sequential conversational chain ~1-2h; Q1-Q12 + 25+ sub-decisions ratified); V-FF pre-Stage-1 verification (this session; ADR-029 insertion point + Vulnerability.fingerprint confirmed + TenantMixin RLS + Scan column conventions + design doc structural precedents); M8.1β.2 Stage 1 design doc structural precedent (commit `3f07611`; 252 LoC; §1-§9 sections); Task 8.3α Stage 1 design doc structural precedent (commit `0030319`; 281 LoC); source-ingestion fix design doc precedent (commit `90fc933`; 388 LoC); SPEC §8 AI pipeline canonical authority (§8.1 pipeline diagram + §8.2 correlation weights + §8.3 scoring formula + §8.4 mobile fix prompt + §8.5 multi-provider cost targets + §8.6 error-recovery fallback matrix); SPEC §13 ADR-013 sole-writer + ADR-014 mixed-primitives + ADR-022 recon-as-pre-scan-helper + ADR-028 M8.1β.2 canonical authority; CLAUDE.md Gotcha 5 cost-tracking mandate (drives Drift #64 cataloguing); ADR-024 + ADR-025 findings-ingest at api DRIFT-LOG (raw_findings population context); 63 cumulative session-tail framing-drift discipline (Drift #58-#63 catalogued through M8 close).

**Related:** Implementation plan landing trigger: ***"Begin M9.0 — Stage 2 implementation plan landing"***. Sub-milestone activation triggers: ***"Begin M9.0 — Stage 3 implementation"*** (after Stage 2 plan lands) → after M9.0 closes ***"Begin M9.A — Embedding + Deduplication"*** → ***"Begin M9.B — Correlation + Scoring"*** → ***"Begin M9.C — Fix Generation + Executive Summary"*** → ***"Begin M9.D — Pipeline Orchestrator"***.

---

## 1. Authority + Q1-Q12 Locks Summary

Brainstorming chain (Q1-Q12) resolved this session under Mode 1 sequential conversational deliberation (~1-2h):

**Q1 Y-PIPELINE-EXECUTION-MODEL → (e) In-api ai-pipeline-consumer task + (e.ii) orchestrator method dispatch.** completions_consumer detects all-jobs-terminal → ScanOrchestrator.dispatch_ai_pipeline (sequential new session per Q7 c.ii M8.1β.2 precedent) → transitions Scan ANALYZING + LPUSH shieldscan:ai_pipeline queue → in-api ai-pipeline-consumer task drains queue → runs M9 pipeline → transitions Scan COMPLETED/FAILED. Mirrors completions_consumer + M8.1β.2 dispatch_phase2 architectural pattern. Recon-invocation architectural seam forward-pin (Drift #59+#62) activates at Stage 3 implementation.

**Q2 Y-COST-TRACKING-MECHANISM → (A.c) per-call + computed aggregates + (B.c) hybrid pre-call/post-call + (C.a) per-scan circuit breaker + hardcoded ScanType budgets per SPEC §8.5.** ai_api_calls table (scan_id + provider + model + operation_type + tokens_in + tokens_out + cost_usd + created_at; TenantMixin RLS); budget enforcement queries SUM(cost_usd) WHERE scan_id; circuit-breaker tripped per-scan; high-cost operations (Claude fix-gen + executive summary) get pre-call estimation check; low-cost operations (embeddings + correlation) post-call log only. Drift #64 catalogued (Drift #60 catch-class 4th-instance: stored-design-intent-with-unimplemented-mechanism per CLAUDE.md Gotcha 5).

**Q3 Y-SUMMARY-STORAGE → (C) Hybrid — Scan.executive_summary at M9 + Report architecture forward-pinned to M10.** Scan.executive_summary Text nullable=True column added at M9.0 migration; full Report architecture (PDF/HTML rendering + R2 + API surface) deferred to M10 lifecycle. Drift #65 catalogued (Drift #61 catch-class 5th-instance: concrete-empirical-field absence per M9.7 plan pseudo-code). Migration to Report.summary_text forward-pinned at M10 if architecturally warranted.

**Q4 Y-QDRANT-TOPOLOGY → (A) Collection-per-organization.** Each org gets dedicated Qdrant collection `findings_{org_id}` (lazy creation at first AI pipeline call); hard isolation at collection level; aligned with ADR-013 sole-writer philosophy + ShieldScan security positioning for regulated industries. Cross-customer trending opt-in via separate `findings_trending_consented` collection forward-pinned.

**Q5 Y-DEDUP-PERSISTENCE → (A.c) per-scan dedup + cross-scan forward-pinned + (B.c) 90-day TTL retention + (C.b) deterministic fingerprint identity + (D.b) Vulnerability.qdrant_point_id reference.** Per-scan dedup at M9.0; cross-scan dedup with regression detection + fix-verification forward-pinned to M11+; Qdrant points persist with 90-day TTL (cleanup-job forward-pinned); point IDs deterministic from hash(tool + target_url + cwe_id + raw_finding signature); Vulnerability.qdrant_point_id UUID column added at M9.0 migration.

**Q6 Y-VULNERABILITY-PROMOTION-SHAPE → (A.c) hybrid clustering + traceability + (B.a) raw_finding.promoted_at + vulnerability_id state + (C.c) existing Vulnerability.fingerprint.** One Vulnerability per dedup-cluster (≥0.92 similarity); preserves per-raw-finding traceability via Vulnerability.raw_finding_ids UUID[]; raw_finding.promoted_at timestamp + raw_finding.vulnerability_id FK; existing Vulnerability.fingerprint column reused (V-FFC empirically verified — String(64) nullable=False index=True). Algorithm-level details (cluster representative selection + correlation weighting + promotion threshold) deferred to M9.A activation.

**Q7 Y-PROVIDER-CLIENT-LIFECYCLE → (A) singleton at startup + (A.i) FastAPI Depends + (D.a) module-level access for background tasks + hard-fail on config error.** AsyncOpenAI + AsyncAnthropic + AsyncQdrantClient at src/app/services/ai/clients.py initialized at app lifespan startup; FastAPI Depends pattern for endpoint dependency injection; module-level singleton access for ai-pipeline-consumer background task (mirrors completions_consumer + redis_client precedent); hard-fail at startup if API keys missing/invalid (configuration errors loud at deploy time).

**Q8 Y-ERROR-RECOVERY-COMPOSITION → (A.b) graceful degradation per SPEC §8.6 + (B.c) Scan.ai_pipeline_degraded boolean + (C.c-lite) structured logging now + fallback_events table forward-pinned.** SPEC §8.6 fallback matrix honored (embedding/Qdrant down → rule-based fingerprint; Claude rate-limit → retry 3× exp backoff; Claude failure → fallback per matrix); Scan.ai_pipeline_degraded boolean column added at M9.0 migration (default False; flipped True on any stage fallback); COMPLETED status preserved when pipeline produces final outputs via fallback; structured logging via logger.warning at M9; dedicated fallback_events table forward-pinned for production-readiness.

**Q9 Y-MIGRATION-NEEDED → (A.a) single Alembic migration + (B.a) lands at M9.0 Stage 3 first commit + (C.c) forward-only with downgrade forward-pinned.** Single migration revision file containing all 7 schema changes (ai_api_calls table + Scan.executive_summary + Scan.ai_pipeline_degraded + Vulnerability.qdrant_point_id + Vulnerability.raw_finding_ids + raw_finding.promoted_at + raw_finding.vulnerability_id); lands at M9.0 Stage 3 C1; forward-only at M9 per pre-launch Q10 M8.1β.2 bounded-staleness precedent; explicit downgrade() forward-pinned for production-readiness audit.

**Q10 Y-ADR-NUMBER → ADR-029 "AI Analysis Pipeline Foundation (M9.0)".** Next-available canonical at SPEC §13 (existing ADRs 001-028; ADR-019 + 020 reserved; ADR-020 closed at M8.1β.2; ADR-025 lives at api DRIFT-LOG). Explicit M9.0 scope in title leaves ADR-030+ reserved for sub-milestone activation. V-FFB confirmed insertion point: immediately after ADR-028 (SPEC line 2497) + before §14 Meta-Principles (SPEC line 2559).

**Q11 Y-SUB-MILESTONE-DECOMPOSITION → (A.a) 5-sub-milestone + (B.a) strict linear + (C.c-lite) M9.0 closure incl. no-op smoke test.** M9.0 (foundation) → M9.A (embed/dedup; Tasks 9.1+9.2) → M9.B (correlate/score; Tasks 9.3+9.4) → M9.C (fix/summary; Tasks 9.5+9.6) → M9.D (orchestrator; Task 9.7). M9.0 closure includes design+plan+migration+infrastructure+skeleton+no-op-smoke-test exercising architectural pipes without real AI calls (real provider calls deferred to M9.A).

**Q12 Y-STAGE3-DECOMPOSITION-FOR-M9.0 → (A.c) 3-commit + (B.c) schema-first bottom-up.** C1 schema migration + AI clients module + cost-tracking module + model column additions (~250-400 LoC); C2 ai-pipeline-consumer + ScanOrchestrator.dispatch_ai_pipeline + completions_consumer hook + no-op pipeline (~200-350 LoC); C3 tests + end-to-end smoke test (~300-500 LoC); aggregate ~750-1250 LoC.

## 2. Pre-Verification Findings

**V-EE (this session; M9 architectural decision space grounded):** 3 empirical gaps surfaced — (1) ai_api_calls table absence vs CLAUDE.md Gotcha 5 hard mandate; (2) Scan.executive_summary column absence vs M9.7 plan-literal pseudo-code reference; (3) pipeline trigger seam undefined at completions_consumer._maybe_complete_scan. SPEC §8 comprehensively specified (correlation weights + scoring formula + error-recovery fallback matrix). Models locked at VERSIONS § (text-embedding-3-small + claude-sonnet-4-6 + opus + haiku-4-5). Deps installed (qdrant-client + anthropic + openai). Zero client code exists (greenfield territory). ~11 Y-decisions territory identified preliminarily.

**V-FF pre-Stage-1 verification (this session):** V-FFA clean state (docs 9d571b6 + engine 899c5be + api 1b7b314); V-FFB ADR-029 insertion point at SPEC §13 immediately after ADR-028 (line 2497) + before §14 Meta-Principles (line 2559); V-FFC Vulnerability.fingerprint column confirmed (String(64) nullable=False index=True → Q6 C.c lock holds); V-FFD TenantMixin RLS pattern at base.py (organization_id auto-column → ai_api_calls integration via Base + TimestampMixin + TenantMixin); V-FFE Scan model column addition conventions confirmed (Boolean default=False nullable=False pattern for ai_pipeline_degraded; Text nullable=True for executive_summary); V-FFF design doc structural precedents (M8.1β.2 §1-§9 + Task 8.3α format-adapt). No drift from Q1-Q12 locks surfaced.

## 3. Architectural Decisions (Q-Lock Detail)

### 3.1 Y-PIPELINE-EXECUTION-MODEL (e) In-api ai-pipeline-consumer + (e.ii) orchestrator dispatch
**Lock:** completions_consumer → orchestrator.dispatch_ai_pipeline (sequential new session) → Scan ANALYZING + LPUSH shieldscan:ai_pipeline → ai-pipeline-consumer drains queue → pipeline runs → Scan COMPLETED/FAILED.
**Grounding:** completions_consumer arc precedent (in-api long-running task triggered by Redis events); M8.1β.2 dispatch_phase2 architectural pattern (sequential sessions per Q7 c.ii); ADR-014 mixed-primitives extension (new Redis queue shieldscan:ai_pipeline); operational simplicity (same deployment unit) at pre-launch scale.
**Rejected alternatives:** (a) inline-consumer execution — long-running synchronous work in completions handler couples ingest channel to AI latency; (b) asyncio-background — fails reliability invariant (no persistence; no observability); (c) dispatched engine ai-job — language boundary at Go engine → Python AI pipeline architectural awkwardness; (d) dedicated Python ai-worker process — operational overhead (separate deployment unit) at pre-launch scale.
**Implementation surface (Stage 3 C2):** src/app/services/ai_pipeline_consumer.py + ScanOrchestrator.dispatch_ai_pipeline method + completions_consumer hook.
**Recon-invocation seam forward-pin (Drift #59 + #62 + #63 catch-class):** examine adjacent-layer architectural surfaces at Stage 3 pre-implementation for type signature + interface contract + DI pattern precision.

### 3.2 Y-COST-TRACKING-MECHANISM (A.c) + (B.c) + (C.a)
**Lock:** ai_api_calls per-call rows + computed aggregates via SUM query + hybrid pre-call/post-call enforcement + per-scan circuit breaker + hardcoded ScanType budgets per SPEC §8.5.
**Grounding:** CLAUDE.md Gotcha 5 hard mandate ("Un-tracked AI calls are forbidden"); SPEC §8.5 cost targets ($0.08/$0.25/$0.55 per ScanType); arc precedent (per-table over multi-table at pre-launch per Q10 M8.1β.2 bounded-staleness).
**Rejected alternatives:** (A.a) minimal ai_api_calls (pushes complexity to read path); (A.b) hierarchical with separate budget table (over-engineered at pre-launch); (B.a) pre-call only (requires accurate token estimation); (B.b) post-call only (budget overrun possible); (C.b) global circuit breaker only (no per-scan discipline); (C.c) both per-scan + global (over-engineered at pre-launch).
**Implementation surface (Stage 3 C1):** ai_api_calls table migration + src/app/services/ai/cost_tracking.py module (check_budget_and_log + log_ai_call functions).
**Drift #64 catalogued:** ai_api_calls table absence (Drift #60 catch-class 4th-instance: stored-design-intent-with-unimplemented-mechanism).

### 3.3 Y-SUMMARY-STORAGE (C) Hybrid
**Lock:** Scan.executive_summary Text nullable=True column at M9.0; full Report architecture (PDF/HTML + R2 + API) deferred to M10.
**Grounding:** SPEC §8.1 terminal output "Vulnerabilities (PostgreSQL) + Report (R2)"; M10 milestone is "Vulnerability & Report APIs" (report API canonically M10 scope); intermediate summary persistence needed at M9.6 generation without blocking on full Report architecture.
**Rejected alternatives:** (A) Scan.executive_summary only (no path to rich Report); (B) Report model + table now (pulls M10 architecture into M9 prematurely; R2 rendering out of M9 scope); (D) R2 object only (no queryable summary; couples M9.6 to R2 availability).
**Implementation surface (Stage 3 C1):** Scan.executive_summary column in single migration.
**Drift #65 catalogued:** Scan.executive_summary absence (Drift #61 catch-class 5th-instance: concrete-empirical-field absence; sub-category 2nd-instance). Migration to Report.summary_text forward-pinned at M10.

### 3.4 Y-QDRANT-TOPOLOGY (A) Collection-per-organization
**Lock:** Dedicated Qdrant collection `findings_{org_id}` per org; lazy creation at first AI pipeline call; hard collection-level isolation.
**Grounding:** ADR-013 sole-writer isolation philosophy; ShieldScan security positioning (regulated industries — hard tenant isolation preferred over payload-filter soft isolation); plan-literal pseudo-code precedent (`findings_{organization_id}`).
**Rejected alternatives:** (B) single multitenant collection + payload filter org_id — soft isolation; one misfiltered query leaks cross-tenant vectors (unacceptable for security product); (C) collection-per-scan — collection sprawl + no cross-scan dedup path (Q5 forward-pin would be impossible).
**Implementation surface (Stage 3 C2 skeleton; M9.A real):** clients.py qdrant collection helper.
**Forward-pins:** cross-customer trending opt-in via separate `findings_trending_consented` collection; eager collection creation at org onboarding; versioned collection naming at first schema-migration need.

### 3.5 Y-DEDUP-PERSISTENCE (A.c) + (B.c) + (C.b) + (D.b)
**Lock:** per-scan dedup at M9.0; 90-day TTL retention; deterministic point IDs from hash(tool + target_url + cwe_id + raw_finding signature); Vulnerability.qdrant_point_id UUID column.
**Grounding:** per-scan dedup is the M9.A baseline (cross-scan needs regression-detection design); deterministic IDs give idempotent re-runs (CLAUDE.md Gotcha 4 fingerprint discipline analog); explicit point-id reference enables Vulnerability→Qdrant traceability.
**Rejected alternatives:** (A.a) per-scan only no cross-scan path; (A.b) cross-scan now (regression-detection design not ready); (B.a) infinite retention (storage growth); (B.b) per-scan ephemeral (no cross-scan future); (C.a) random point IDs (non-idempotent re-runs); (D.a) no point reference (no traceability).
**Implementation surface (Stage 3 C1):** Vulnerability.qdrant_point_id column.
**Forward-pins:** cross-scan dedup + regression detection + fix-verification (M11+); Qdrant TTL cleanup-job (production-readiness).

### 3.6 Y-VULNERABILITY-PROMOTION-SHAPE (A.c) + (B.a) + (C.c)
**Lock:** one Vulnerability per dedup-cluster (≥0.92); Vulnerability.raw_finding_ids UUID[] traceability; raw_finding.promoted_at + raw_finding.vulnerability_id FK; existing Vulnerability.fingerprint reused.
**Grounding:** hybrid clustering preserves traceability (audit + evidence requirements) while collapsing duplicates; raw_finding state columns enable idempotent re-promotion + orphan detection; V-FFC confirmed fingerprint column exists (no schema gap).
**Rejected alternatives:** (A.a) flat one-vuln-per-raw-finding (no dedup value); (A.b) cluster without traceability (loses per-tool evidence); (B.b) no raw_finding state (no idempotency / orphan detection); (C.a/C.b) new fingerprint scheme (existing column sufficient).
**Implementation surface (Stage 3 C1):** Vulnerability.raw_finding_ids + raw_finding.promoted_at + raw_finding.vulnerability_id columns.
**Deferred to M9.A:** cluster-representative selection + correlation weighting + promotion threshold algorithm.

### 3.7 Y-PROVIDER-CLIENT-LIFECYCLE (A) + (A.i) + (D.a)
**Lock:** AsyncOpenAI + AsyncAnthropic + AsyncQdrantClient singletons at src/app/services/ai/clients.py; lifespan startup init; FastAPI Depends for endpoints; module-level access for background task; hard-fail on missing/invalid config.
**Grounding:** redis_client + completions_consumer precedent (module-level singleton for background tasks); FastAPI lifespan is the canonical startup hook; hard-fail at deploy time surfaces config errors loudly (CLAUDE.md Rule 6 secrets + deploy discipline).
**Rejected alternatives:** (B) per-call client construction (connection overhead; no pooling); (C) lazy first-use init (hides config errors until first scan); (D.b) Depends-only (background task has no request scope).
**Implementation surface (Stage 3 C1):** clients.py + main.py lifespan init_ai_clients.

### 3.8 Y-ERROR-RECOVERY-COMPOSITION (A.b) + (B.c) + (C.c-lite)
**Lock:** SPEC §8.6 fallback matrix honored; Scan.ai_pipeline_degraded boolean (default False; True on any fallback); COMPLETED preserved on fallback success; structured logging now; fallback_events table forward-pinned.
**Grounding:** SPEC §8.6 is canonical fallback authority; degraded flag gives queryable observability without a new table at pre-launch; resilience invariant (partial AI failure must not fail the whole scan).
**Rejected alternatives:** (A.a) hard-fail on any AI error (fails resilience invariant); (B.a) no degraded marker (no observability); (B.b) error_message reuse (conflates degraded-but-complete with failed); (C.a/C.b) fallback_events table now (over-engineered at pre-launch).
**Implementation surface (Stage 3 C1):** Scan.ai_pipeline_degraded column. (Real fallback logic at M9.A-M9.D.)
**Forward-pins:** fallback_events dedicated table (production-readiness).

### 3.9 Y-MIGRATION-NEEDED (A.a) + (B.a) + (C.c)
**Lock:** single Alembic migration with all 7 schema changes; lands at Stage 3 C1; forward-only with explicit downgrade() forward-pinned.
**Grounding:** single atomic migration keeps the schema delta reviewable as one unit; C1 schema-first bottom-up (Q12) requires schema before modules; forward-only matches pre-launch bounded-staleness (Q10 M8.1β.2 precedent — no production data to downgrade-protect yet).
**Rejected alternatives:** (A.b) per-change migrations (7 revisions for one logical change); (B.b) migration at C3 (modules would reference non-existent columns at C1/C2); (C.a/C.b) full downgrade now (no production rollback need pre-launch).
**The 7 schema changes:** ai_api_calls table; Scan.executive_summary; Scan.ai_pipeline_degraded; Vulnerability.qdrant_point_id; Vulnerability.raw_finding_ids; raw_finding.promoted_at; raw_finding.vulnerability_id.
**Forward-pin:** explicit downgrade() at production-readiness audit.

### 3.10 Y-ADR-NUMBER ADR-029
**Lock:** ADR-029 "AI Analysis Pipeline Foundation (M9.0)" at SPEC §13.
**Grounding:** highest existing ADR is ADR-028 (V-FFB confirmed); explicit M9.0 scope in title leaves ADR-030+ for sub-milestone activation if their decisions warrant ADR-level capture.
**Insertion point (Stage 3 C1):** after ADR-028 (SPEC line 2497) + before §14 Meta-Principles (line 2559).

### 3.11 Y-SUB-MILESTONE-DECOMPOSITION (A.a) + (B.a) + (C.c-lite)
**Lock:** 5 sub-milestones (M9.0 → M9.A → M9.B → M9.C → M9.D); strict linear sequencing; M9.0 closure includes a no-op smoke test exercising architectural pipes without real AI calls.
**Grounding:** M9 is too broad for one monolithic lifecycle (7 sub-tasks; ~750-1250 LoC foundation alone); strict linear because each sub-milestone consumes the prior's outputs (embed→dedup→correlate→score→fix→summary→orchestrate); no-op smoke validates the seam + queue + consumer + status transitions before real provider integration.
**Rejected alternatives:** (A.b) one monolithic M9 lifecycle (unreviewably large); (B.b) parallel sub-milestones (dependency chain forbids); (C.a) M9.0 closure without smoke test (seam unvalidated until M9.A).
**Mapping:** M9.A = Tasks 9.1+9.2; M9.B = Tasks 9.3+9.4; M9.C = Tasks 9.5+9.6; M9.D = Task 9.7.

### 3.12 Y-STAGE3-DECOMPOSITION-FOR-M9.0 (A.c) + (B.c)
**Lock:** 3-commit, schema-first bottom-up. C1 schema migration + AI clients + cost-tracking + model columns (~250-400 LoC); C2 ai-pipeline-consumer + dispatch_ai_pipeline + completions_consumer hook + no-op pipeline (~200-350 LoC); C3 tests + e2e smoke (~300-500 LoC).
**Grounding:** schema-first means later commits compile against real columns; consumer/dispatch (C2) depend on C1 modules; tests (C3) depend on C1+C2 behavior. ZERO-regression-at-each-commit discipline preserved.
**Rejected alternatives:** (A.a) single commit (unreviewable; violates per-commit green gate); (A.b) 5+ commits (over-decomposed); (B.a) top-down (consumer-first would reference non-existent schema).
**Aggregate Stage 3 LoC forecast:** ~750-1250 LoC.

## 4. M9.0 Stage 3 Implementation Surface

### 4.1 Stage 3 Commit 1 — Schema + Modules
**Files:** alembic/versions/<rev>_m9_ai_pipeline_foundation.py (NEW; single migration); src/app/models/ai.py (NEW; AIAPICall model); src/app/models/scans.py (column additions: executive_summary + ai_pipeline_degraded); src/app/models/vulnerabilities.py (column additions: qdrant_point_id + raw_finding_ids); src/app/models/raw_findings.py (column additions: promoted_at + vulnerability_id); src/app/services/ai/__init__.py (NEW); src/app/services/ai/clients.py (NEW; AsyncOpenAI + AsyncAnthropic + AsyncQdrantClient singletons + init_ai_clients); src/app/services/ai/cost_tracking.py (NEW; check_budget_and_log + log_ai_call); app lifespan startup integration (init_ai_clients in main.py).
**LoC forecast:** ~250-400 LoC.

### 4.2 Stage 3 Commit 2 — Consumer + Dispatch Hook
**Files:** src/app/services/ai_pipeline_consumer.py (NEW; in-api long-running task; mirrors completions_consumer pattern); src/app/services/orchestrator.py (NEW method dispatch_ai_pipeline); src/app/services/completions_consumer.py (_maybe_complete_scan hook → orchestrator.dispatch_ai_pipeline); src/app/services/ai/pipeline.py (NEW; no-op pipeline skeleton).
**LoC forecast:** ~200-350 LoC.

### 4.3 Stage 3 Commit 3 — Tests + Smoke Test
**Files:** tests/services/ai/__init__.py (NEW); tests/services/ai/test_clients.py (NEW); tests/services/ai/test_cost_tracking.py (NEW); tests/services/test_ai_pipeline_consumer.py (NEW); tests/integration/test_m9_smoke.py (NEW; end-to-end no-op smoke test — note: V-BBF M8.1β.2 contingency observed no tests/integration/ dir; create dir or co-locate per Stage 3 V-check).
**LoC forecast:** ~300-500 LoC.

**Aggregate Stage 3 LoC forecast:** ~750-1250 LoC.

## 5. Phase Structure

Stage 1 design doc (THIS COMMIT) → Stage 2 implementation plan → Stage 3 3-commit api-side (schema+modules → consumer+dispatch → tests+smoke) → Stage 4 P5.A annotations.

Sub-milestone decomposition (Q11 lock):
M9.0 (this foundation) → M9.A (Tasks 9.1+9.2; embed + dedup) → M9.B (Tasks 9.3+9.4; correlate + score) → M9.C (Tasks 9.5+9.6; fix + summary) → M9.D (Task 9.7; orchestrator).

## 6. Out of Scope

1. M9.A sub-milestone scope (Tasks 9.1+9.2; embed + dedup) — own lifecycle activation
2. M9.B sub-milestone scope (Tasks 9.3+9.4; correlate + score) — own lifecycle activation
3. M9.C sub-milestone scope (Tasks 9.5+9.6; fix-gen + summary) — own lifecycle activation
4. M9.D sub-milestone scope (Task 9.7; orchestrator composition) — own lifecycle activation
5. M10 Report architecture (PDF/HTML rendering + R2 + API surface; Q3 forward-pin)
6. Cross-scan dedup with regression detection + fix-verification (Q5 forward-pin to M11+)
7. Production-readiness audit (extensive forward-pin chain; Q2/Q4/Q5/Q8/Q9 all contribute)
8. Cross-customer trending consented collection (Q4 forward-pin to onboarding maturity)
9. Qdrant TTL cleanup-job (Q5 forward-pin to production-readiness)
10. fallback_events dedicated table (Q8 forward-pin to production-readiness)
11. Explicit downgrade() implementation across migrations (Q9 forward-pin to production-readiness)
12. Real provider AI calls (embeddings/Claude/Qdrant operations) — deferred to M9.A+ per Q11 no-op-smoke M9.0 closure

## 7. Forward-Pins

**Pre-implementation:**
1. Stage 2 plan landing: ***"Begin M9.0 — Stage 2 implementation plan landing"***
2. Stage 3 implementation: ***"Begin M9.0 — Stage 3 implementation"***

**Sub-milestone activation triggers:**
3. ***"Begin M9.A — Embedding + Deduplication"*** (after M9.0 closes)
4. ***"Begin M9.B — Correlation + Scoring"*** (after M9.A closes)
5. ***"Begin M9.C — Fix Generation + Executive Summary"*** (after M9.B closes)
6. ***"Begin M9.D — Pipeline Orchestrator"*** (after M9.C closes)
7. ***"Begin M10 — Report Architecture"*** (after M9 lifecycle CLOSED)

**Discipline-level (active from prior arc + extensions this M9.0 entry):**
8. Audit-driven model+spec orphan check into pre-verification template (Drift #60 rule-of-three; activated at V-EE for Drift #64/#65 dual detection)
9. DEFERRED-EMPIRICAL marking for concrete-empirical-references in plan pseudo-code (Drift #61 + #62 catch-class evolution; activated at Q3 + Q5 + Q6)
10. Examine recon-invocation architectural seam at pre-verification for engine-side dispatch additions (Drift #59 + #62 adjacent-layer; analogous M9 dispatch seam: completions_consumer → orchestrator.dispatch_ai_pipeline → ai-pipeline-consumer)
11. Pre-verification scope completeness across test-impact-surface (Drift #63 extension; applies to M9.0 behavior changes — ANALYZING transition + Scan column additions could ripple to route-level tests)

**Production-readiness audit forward-pin chain (extensive):**
12. Promote ai_api_calls to hierarchical ai_scan_budget aggregate table at production scale
13. Add global circuit breaker as production safety net
14. Database-configurable per-org budget tiers at enterprise pricing
15. Implement consented cross-customer trending collection at customer onboarding maturity
16. Eager Qdrant collection creation at org onboarding lifecycle
17. Versioned Qdrant collection naming at first schema migration need
18. Qdrant TTL cleanup-job at production-readiness
19. fallback_events dedicated table at production-readiness
20. Explicit downgrade() implementation across migrations at production-readiness

## 8. Cross-References

**Engine:** None (M9.0 is api-side only; preserves Go/Python language boundary per Q1 (c) rejection rationale).

**Docs:** 9d571b6 (M8.1β.2 P5.A docs); 9507acb (M8.1β.2 Stage 3 C1 ADR-028 canonical authority; M9.0 composes ADR-028 phase-1+phase-2 architecture providing AttackSurface/Vulnerability foundation); 0030319 (Task 8.3α design doc structural precedent); 3f07611 (M8.1β.2 design doc 252 LoC §1-§9 structural precedent); 90fc933 (source-ingestion fix design doc 388 LoC precedent); SPECIFICATION.md §8 (canonical M9 pipeline authority; §8.1 stages + §8.2 correlation weights + §8.3 scoring + §8.4 fix prompts + §8.5 cost targets + §8.6 error recovery); SPECIFICATION.md §13 ADR-013 + ADR-014 + ADR-017 + ADR-022 + ADR-028 + ADR-029 (this design's target); CLAUDE.md Gotcha 5 cost-tracking mandate (Drift #64 driver).

**API:** 1b7b314 (Task 8.3β attack-surface endpoint; latest api state); dc39fd1 (M8.1β.2 Stage 3 C4 completions_consumer phase-2 hook; M9.0 extends with _maybe_complete_scan ai-pipeline-dispatch hook); 04a9b5c (M8.1β.2 Stage 3 C3 orchestrator dispatch_phase2 precedent; M9.0 adds dispatch_ai_pipeline analog); src/app/models/scans.py (Scan column additions target); src/app/models/raw_findings.py (raw_finding column additions target); src/app/models/vulnerabilities.py (Vulnerability column additions target); src/app/services/completions_consumer.py (_maybe_complete_scan hook target); src/app/services/orchestrator.py (dispatch_ai_pipeline method target).

**Multi-session arc:** 78 commits + 63 framing-drift catalogue + 17 closed task lifecycles + M8 declarable CLOSED at last session.

## 9. Drift #64 + #65 Catalog

**Drift #64 (ai_api_calls absence; Drift #60 catch-class 4th-instance: stored-design-intent-with-unimplemented-mechanism) — 2026-06-12:**

CLAUDE.md Gotcha 5 hard mandate states: "Every Claude/OpenAI call logs cost to ai_api_calls table … respects per-scan budget … circuit breaker … Un-tracked AI calls are forbidden." V-EE pre-verification (this session) surfaced zero infrastructure: no ai_api_calls table; no cost-tracking model; no budget enforcement code; no circuit breaker.

**Catch-class lineage (4 instances of stored-design-intent-with-unimplemented-mechanism):**
- #54 (source-ingestion fix; M5 stored intent without ingest mechanism)
- #58 (Task 8.3α AttackSurface; M6 stored intent without consumer mechanism)
- #60 (M8.1 SCAN_TYPE_TOOLS 6-engine surface; multi-engine stored intent without dispatch alignment)
- #64 (M9 ai_api_calls; CLAUDE.md mandate without implementation mechanism)

**Resolution:** Q2 architectural lock + M9.0 Stage 3 C1 schema migration + cost_tracking module implementation.

**Discipline-level forward-pin preserved:** "Audit-driven model+spec orphan check into pre-verification template" (rule-of-three trigger fired at #60; M9.0 V-EE pre-verification successfully detected #64 via this discipline; pattern confirmed durable).

**Drift #65 (Scan.executive_summary absence; Drift #61 catch-class 5th-instance: concrete-empirical-field absence) — 2026-06-12:**

M9.7 plan-literal pseudo-code at IMPLEMENTATION-PLAN.md states `scan.executive_summary = summary`. V-EE pre-verification (this session) confirmed Scan model has no such field. SPEC §8.1 terminal output specifies "Vulnerabilities (PostgreSQL) + Report (R2)" — Report architecture deferred to M10; summary persistence intermediate question.

**Catch-class lineage (plan/design-vs-empirical-precision; sub-category concrete-empirical-field absence; SUB-CATEGORY origin at Drift #61):**
- #53 (parameter precision; earlier arc)
- #55 (test path precision)
- #56 (file naming precision)
- #59 (RunRecon +3 params)
- #61 (Scan.created_by_user_id absence; M8.1β.2 V-WD; concrete-empirical-field-absence SUB-CATEGORY origin)
- #62 (interface-vs-concrete type mismatch)
- #63 (test-scope-incompleteness)
- #65 (Scan.executive_summary absence; concrete-empirical-field-absence SUB-CATEGORY 2nd-instance)

**Resolution:** Q3 architectural lock (C) Hybrid + M9.0 Stage 3 C1 schema migration adds Scan.executive_summary Text nullable column; Report architecture migration forward-pinned to M10.

**Discipline-level meta-pattern operational:** "DEFERRED-EMPIRICAL marking for concrete-empirical-references in plan pseudo-code." Stage 1 Q-decisions referencing concrete database fields, file paths, or API contracts marked DEFERRED-EMPIRICAL unless pre-verification has grounded them. M9.0 V-EE successfully grounded Scan.executive_summary absence via this discipline; pattern confirmed durable across M8.1β.2 + M9.0 entries (concrete-empirical-field-absence sub-category now 2-instance: #61 + #65).

**Cumulative session-tail framing-drift count at M9.0 Stage 1 close: 65** (Drift #58 + #59 + #60 + #61 + #62 + #63 + #64 + #65 catalogued).
