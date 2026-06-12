# M9.0 — AI Analysis Pipeline Foundation (ADR-029): Implementation Plan

**Status:** Ready for Stage 3 cross-commit api-side implementation (3-commit per Q12: schema+modules → consumer+dispatch → tests+smoke; refined to 4-commit with a C0 docs ADR-029 landing per §2.4). Pre-implementation grounded against M9.0 Stage 1 design doc canonical (commit a46fedd) + Q1-Q12 locks + V-EE + V-FF + V-GG pre-verification surfaces + Drift #64/#65 catalogue. Sub-milestone decomposition declared per Q11 (M9.0 → M9.A → M9.B → M9.C → M9.D strict linear sequencing).

**Date:** 2026-06-12.

**Authority:** M9.0 Stage 1 design doc canonical authority (shieldscan-docs commit a46fedd; 240 LoC §1-§9 + Q1-Q12 locks + Drift #64/#65 catalogue + sub-milestone decomposition declaration); V-EE pre-verification surface report (this session; M9 architectural decision space grounded; 3 empirical gaps surfaced; SPEC §8 canonical authority confirmed); V-FF pre-Stage-1 verification (this session; ADR-029 insertion point + Vulnerability.fingerprint + TenantMixin RLS + Scan column conventions); V-GG pre-plan-landing verification (this session; M8.1β.2 plan precedent + completions_consumer lifespan convention + app startup ordering); M8.1β.2 implementation plan structural precedent (commit fb61129; 429 LoC §1-§8 + 4 plan-level Y-decisions + Stage 3 sub-step breakdown); source-ingestion fix plan structural precedent (commit 04f44a9; 426 LoC); Task 8.3α plan structural precedent (commit dba6a7c; 422 LoC); SPEC §8 canonical M9 authority + SPEC §13 ADR-029 (Stage 3 C0 implementation target); CLAUDE.md Gotcha 5 cost-tracking mandate; ADR-028 phase-1+phase-2 composition foundation; 65 cumulative session-tail framing-drift discipline; milestone-completion-constraint locked at M9.0 sub-milestone scope.

**Related:** Stage 3 implementation trigger phrase: ***"Begin M9.0 — Stage 3 implementation"*** (after this plan lands).

---

## 1. Authority + Scope Lock

Q1-Q12 locks (locked at Stage 1 design doc a46fedd §1 + §3): see design doc §1 + §3 for full architectural rationale + rejected alternatives traces.

**In scope (Stage 3 sub-steps; Q12 (A.c) + (B.c) shape; 3-commit schema-first bottom-up + C0 docs per §2.4):**

**Stage 3 Commit 1 (Schema + Modules):**
1. alembic/versions/<rev>_m9_ai_pipeline_foundation.py (NEW; single Alembic migration per Q9 A.a)
   - ai_api_calls table creation (Q2)
   - Scan.executive_summary Text nullable column (Q3 + Drift #65 resolution)
   - Scan.ai_pipeline_degraded boolean default=False column (Q8)
   - Vulnerability.qdrant_point_id UUID nullable column (Q5 + D.b)
   - Vulnerability.raw_finding_ids UUID[] nullable column (Q6 A.c)
   - raw_finding.promoted_at timestamp nullable column (Q6 B.a)
   - raw_finding.vulnerability_id UUID FK Vulnerability nullable column (Q6 B.a)
2. src/app/models/ai.py (NEW; AIAPICall model per Q2 + TenantMixin RLS pattern)
3. src/app/models/scans.py (column additions per migration)
4. src/app/models/vulnerabilities.py (column additions per migration)
5. src/app/models/raw_findings.py (raw_finding column additions per migration)
6. src/app/services/ai/__init__.py (NEW; module marker)
7. src/app/services/ai/clients.py (NEW; AsyncOpenAI + AsyncAnthropic + AsyncQdrantClient singleton management per Q7)
8. src/app/services/ai/cost_tracking.py (NEW; check_budget + log_ai_call functions per Q2 + circuit breaker semantics)
9. src/app/main.py lifespan extension (init_ai_clients + shutdown_ai_clients per Q7 + V-GGD ordering)

**Stage 3 Commit 2 (Consumer + Dispatch Hook):**
10. src/app/services/ai_pipeline_consumer.py (NEW; in-api long-running task per Q1 e + Y-AI-PIPELINE-CONSUMER-LIFECYCLE resolved at execution)
11. src/app/services/orchestrator.py (NEW method dispatch_ai_pipeline per Q1 e.ii; mirrors dispatch_phase2 architectural pattern at M8.1β.2)
12. src/app/services/completions_consumer.py (_maybe_complete_scan hook → orchestrator.dispatch_ai_pipeline; sequential session per Q7 c.ii precedent)
13. src/app/services/ai/pipeline.py (NEW; no-op pipeline skeleton per Y-NO-OP-PIPELINE-SHAPE resolved at execution)
14. src/app/main.py lifespan extension (start_ai_pipeline_consumer + stop_ai_pipeline_consumer per V-GGC convention)

**Stage 3 Commit 3 (Tests + Smoke):**
15. tests/services/ai/__init__.py (NEW)
16. tests/services/ai/test_clients.py (NEW; client singleton lifecycle + Depends + module-level access tests)
17. tests/services/ai/test_cost_tracking.py (NEW; budget enforcement + circuit breaker + per-call logging tests)
18. tests/services/test_ai_pipeline_consumer.py (NEW; consumer task + queue draining + no-op pipeline tests)
19. tests/integration/test_m9_smoke.py (NEW; end-to-end no-op smoke test ScanCreate → all jobs terminal → completions_consumer → orchestrator.dispatch_ai_pipeline → ai-pipeline-consumer → no-op pipeline → Scan ANALYZING → Scan COMPLETED + ai_pipeline_degraded=False + cost-tracking exercises zero-cost path)

**Out of scope (forward-pinned per design doc §6):**
- All Q-lock forward-pin chain extensions (20+ items per design doc §7)
- M9.A/B/C/D sub-milestone scopes (own lifecycles per Q11)
- Real AI provider calls (deferred to M9.A activation)
- M10 Report architecture (Q3 forward-pin)
- Production-readiness audit items (Q2/Q4/Q5/Q8/Q9 forward-pins)

**Brainstorming chain Q1-Q12 locks recap (per Stage 1 design doc a46fedd §1):** Y-PIPELINE-EXECUTION-MODEL (e) + (e.ii); Y-COST-TRACKING-MECHANISM (A.c) + (B.c) + (C.a); Y-SUMMARY-STORAGE (C) Hybrid; Y-QDRANT-TOPOLOGY (A) Collection-per-org; Y-DEDUP-PERSISTENCE (A.c) + (B.c) + (C.b) + (D.b); Y-VULNERABILITY-PROMOTION-SHAPE (A.c) + (B.a) + (C.c); Y-PROVIDER-CLIENT-LIFECYCLE (A) + (A.i) + (D.a); Y-ERROR-RECOVERY-COMPOSITION (A.b) + (B.c) + (C.c-lite); Y-MIGRATION-NEEDED (A.a) + (B.a) + (C.c); Y-ADR-NUMBER ADR-029; Y-SUB-MILESTONE-DECOMPOSITION (A.a) + (B.a) + (C.c-lite); Y-STAGE3-DECOMPOSITION-FOR-M9.0 (A.c) + (B.c).

## 2. Pre-Implementation State

### 2.1 API + Docs Infrastructure Ready Matrix (V-EE + V-FF + V-GG Pre-Verification Findings)

**API side ready:**
- TenantMixin RLS pattern operational at base.py (V-FFD; ai_api_calls inherits)
- Scan model with TimestampMixin + TenantMixin (V-FFE; executive_summary + ai_pipeline_degraded column additions straightforward)
- Vulnerability model with existing fingerprint column String(64) nullable=False indexed (V-FFC confirms Q6 C.c lock)
- raw_finding model exists (M5/M6/M7 origin)
- redis_client singleton operational (Q1 e.ii LPUSH target for shieldscan:ai_pipeline queue)
- completions_consumer pattern operational (Q1 reference architecture for ai-pipeline-consumer; Task 8.3α + M8.1β.2 precedent; V-GGC lifespan start/stop convention)
- ScanOrchestrator.dispatch_phase2 precedent operational (Q1 e.ii reference architecture; M8.1β.2 C3 04a9b5c)
- App lifespan startup/shutdown convention (V-GGD; init_redis → CompletionsConsumer.start → yield → stop → close_redis; ai-clients init + ai-pipeline-consumer start insertion point identified)
- ScanStatus enum has ANALYZING value (V-EED confirmed)

**API side pending (Stage 3 C1 + C2 + C3 scope):** all Q-lock implementation per Stage 3 sub-step list above.

**Docs side ready:** ADR-029 canonical text reserved (Stage 1 design doc §3 expanded; SPEC §13 insertion point confirmed at V-FFB line 2497→2559). ADR-029 SPEC §13 landing decision resolved at §2.4 below.

### 2.2 Architectural Analog Strength

**Strong dual-side analog:** ai-pipeline-consumer mirrors completions_consumer pattern (in-api long-running task triggered by Redis events; lifespan start/stop). Orchestrator.dispatch_ai_pipeline mirrors orchestrator.dispatch_phase2 (sequential new session per Q7 c.ii precedent). cost_tracking module is greenfield but follows arc service-layer function conventions.

**D-deviation forecast implication:** MODERATE bound expected per partial-novel-territory hypothesis (~1-3 drifts at Stage 3 execution per M9.0 entry novelty + Drift #62 catch-class precedent at adjacent-layer recon-invocation-seam analog ai-pipeline-seam).

### 2.3 V-FF Refinement Implications for Stage 3

V-FFC confirmed Vulnerability.fingerprint existence (Q6 C.c lock operational). V-FFD confirmed TenantMixin RLS pattern for ai_api_calls integration. V-FFE confirmed Scan column addition conventions. V-FFF confirmed design doc structural precedents.

### 2.4 ADR-029 Canonical Text Landing Decision

**Substantive sub-decision at this plan:** where does ADR-029 canonical text land in SPEC §13?

- Option (i) Stage 3 C1 docs commit (separate from api implementation; mirrors M8.1β.2 C1 docs)
- Option (ii) Stage 4 P5.A annotations (consolidate canonical authority docs with lifecycle close)
- Option (iii) Pre-Stage-3 separate docs commit (lands ADR-029 before any implementation)

Q12 lock specified 3-commit api-side Stage 3. ADR-029 SPEC §13 landing was not explicitly scoped to those commits. **Lock: Option (iii) — a dedicated C0 docs commit landing ADR-029 at SPEC §13 before api implementation.** This matches M8.1β.2 precedent (ADR-028 landed at a docs commit C1 separate from engine/api implementation, establishing canonical authority FIRST). Keeps the api commits (C1-C3) pure-implementation and the canonical ADR authoritative before code references it.

**Refined Stage 3 commit chain:**
- C0 (docs): ADR-029 canonical text at SPEC §13 (separate pre-implementation docs commit; ~80-150 LoC)
- C1 (api): schema migration + modules (per Q12)
- C2 (api): consumer + dispatch hook (per Q12)
- C3 (api): tests + smoke (per Q12)

Aggregate Stage 3 lifecycle: 4 commits total (1 docs + 3 api).

## 3. Architectural Decisions (Plan-Level Locks)

Brainstorming chain Q1-Q12 architectural decisions locked at Stage 1 design doc a46fedd §1 + §3. Plan-level refinements (4 plan-level Y-decisions) captured below.

### 3.1 Y-AI-CLIENTS-MODULE-SHAPE
**Plan-level Y-decision:** (a) single clients.py with all 3 providers / (b) per-provider modules + aggregator.
Default (a) per minimal-module-count + arc precedent (single redis_client at api repo); resolve at Stage 3 C1 execution. DEFERRED-EMPIRICAL: client constructor signatures + Settings field names below are illustrative pending C1.4 grounding against config.py.

```python
# src/app/services/ai/clients.py  (DEFERRED-EMPIRICAL pseudo-code)
_openai_client: AsyncOpenAI | None = None
_anthropic_client: AsyncAnthropic | None = None
_qdrant_client: AsyncQdrantClient | None = None

async def init_ai_clients(settings: Settings) -> None:
    global _openai_client, _anthropic_client, _qdrant_client
    if not settings.OPENAI_API_KEY:
        raise RuntimeError("OPENAI_API_KEY required for AI pipeline")  # Q7 hard-fail
    _openai_client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)
    _anthropic_client = AsyncAnthropic(api_key=settings.ANTHROPIC_API_KEY)
    _qdrant_client = AsyncQdrantClient(url=settings.QDRANT_URL)

async def shutdown_ai_clients() -> None: ...
def get_openai_client() -> AsyncOpenAI: ...  # FastAPI Depends-compatible (Q7 A.i)
def openai_client() -> AsyncOpenAI: ...      # Module-level for background tasks (Q7 D.a)
```

### 3.2 Y-COST-TRACKING-MODULE-SHAPE
**Plan-level Y-decision:** (a) function-style API / (b) class-style / (c) hybrid.
Default (a) per stateless functional discipline + arc precedent (audit.py function-style); resolve at Stage 3 C1 execution. DEFERRED-EMPIRICAL: AIAPICall field names + ScanType enum members below are illustrative pending C1.5 grounding against the migration + scans.py.

```python
# src/app/services/ai/cost_tracking.py  (DEFERRED-EMPIRICAL pseudo-code)
SCAN_TYPE_BUDGETS_USD: dict[ScanType, Decimal] = {
    ScanType.QUICK: Decimal("0.08"),
    ScanType.FULL_WEB: Decimal("0.25"),
    ScanType.FULL_WEB_SOURCE: Decimal("0.40"),
    ScanType.FULL_SPECTRUM: Decimal("0.55"),
    # remaining ScanType members grounded at C1.5 per SPEC §8.5 + scans.py
}

async def check_budget(*, db, scan_id, scan_type, estimated_cost_usd=None) -> bool:
    """True if call permitted; False if budget exceeded. Pre-call per Q2 B.c (high-cost ops)."""
    consumed = await db.scalar(
        select(func.coalesce(func.sum(AIAPICall.cost_usd), Decimal("0")))
        .where(AIAPICall.scan_id == scan_id)
    )
    budget = SCAN_TYPE_BUDGETS_USD[scan_type]
    return (consumed + (estimated_cost_usd or Decimal("0"))) <= budget

async def log_ai_call(*, db, scan_id, provider, model, operation_type, tokens_in, tokens_out, cost_usd) -> AIAPICall:
    """Post-call logging per Q2 B.c."""
    call = AIAPICall(scan_id=scan_id, provider=provider, model=model,
                     operation_type=operation_type, tokens_in=tokens_in,
                     tokens_out=tokens_out, cost_usd=cost_usd)
    db.add(call)
    return call
```

### 3.3 Y-AI-PIPELINE-CONSUMER-LIFECYCLE
**Plan-level Y-decision:** (a) FastAPI lifespan start/stop / (b) module-level import-time / (c) explicit start/stop functions called by lifespan.
Default (a) matches completions_consumer precedent (V-GGC: `.start()` / `.stop()` around `yield`); resolve at Stage 3 C2 execution.

### 3.4 Y-NO-OP-PIPELINE-SHAPE
**Plan-level Y-decision:** (a) immediate-return-no-op (transition ANALYZING→COMPLETED, zero stages) / (b) simulated-stages-zero-cost / (c) NotImplementedError-skip.
Default (a) per simplest-architectural-pipe-exercise + Q11 (C.c-lite) intent; resolve at Stage 3 C2 execution. The no-op must still exercise: queue drain → status ANALYZING→COMPLETED → ai_pipeline_degraded stays False → cost-tracking zero-cost path (no AIAPICall rows when no calls made).

## 4. Stage 3 Sub-Step Breakdown

### Stage 3 Commit 0 — Docs ADR-029 at SPEC §13 (~30-45min; ~80-150 LoC) [PRE-IMPLEMENTATION DOCS]

**C0.1** Locate SPEC §13 ADR-028 + identify ADR-029 insertion point (V-FFB confirmed: line 2497→2559 between ADR-028 and §14 Meta-Principles).

**C0.2** Author ADR-029 canonical text per Stage 1 design doc a46fedd §3.1-§3.12 + §1 summary (lock + rejected alternatives + cross-references; ~60-120 LoC).

**C0.3** Pre-commit verification + single atomic docs commit.

### Stage 3 Commit 1 — Schema + Modules (~60-90min; ~250-400 LoC)

**C1.1** Generate Alembic revision file (per Q9 A.a single migration; B.a M9.0 first-implementation-commit). All 7 schema changes in one revision. ~50-80 LoC.

**C1.2** Add AIAPICall model at src/app/models/ai.py (NEW). TenantMixin RLS integration per V-FFD. ~30-50 LoC.

**C1.3** Add column declarations to Scan + Vulnerability + raw_finding models. Format-adapt per V-FFE Scan column conventions. ~10-20 LoC.

**C1.4** Create src/app/services/ai/__init__.py + src/app/services/ai/clients.py (Q7 + Y-AI-CLIENTS-MODULE-SHAPE resolution). ~50-80 LoC.

**C1.5** Create src/app/services/ai/cost_tracking.py (Q2 + Y-COST-TRACKING-MODULE-SHAPE resolution). ~60-100 LoC.

**C1.6** Update src/app/main.py lifespan with init_ai_clients + shutdown_ai_clients. Ordering: init_redis → init_ai_clients → CompletionsConsumer.start → (ai-pipeline-consumer.start post-C2) → yield → reverse teardown. ~10-20 LoC.

**C1.7** Pre-commit verification + single atomic api commit. Run alembic upgrade head; verify migration applies cleanly. Run pytest tests/models/ (verify model definitions don't break existing tests).

### Stage 3 Commit 2 — Consumer + Dispatch Hook (~60-90min; ~200-350 LoC)

**C2.1** Create src/app/services/ai_pipeline_consumer.py with in-api long-running BRPOP-loop task. Mirrors completions_consumer pattern (start/stop). ~80-120 LoC.

**C2.2** Add ScanOrchestrator.dispatch_ai_pipeline method per Q1 e.ii (sequential session per Q7 c.ii; mirrors dispatch_phase2; transitions Scan ANALYZING + LPUSH shieldscan:ai_pipeline). ~30-50 LoC.

**C2.3** Modify src/app/services/completions_consumer.py to hook _maybe_complete_scan → orchestrator.dispatch_ai_pipeline (in the all-jobs-terminal branch, BEFORE the direct COMPLETED transition — pipeline now owns the terminal transition). ~20-40 LoC.

**C2.4** Create src/app/services/ai/pipeline.py no-op pipeline skeleton per Y-NO-OP-PIPELINE-SHAPE (a) immediate-return. ~30-50 LoC.

**C2.5** Update src/app/main.py lifespan with ai-pipeline-consumer start/stop per Y-AI-PIPELINE-CONSUMER-LIFECYCLE (a). ~10-20 LoC.

**C2.6** Pre-commit verification + single atomic api commit. Run pytest tests/services/ (verify completions_consumer hook does not break existing 18 consumer tests — Drift #63 test-impact-surface forward-pin: grep tests for _maybe_complete_scan COMPLETED-transition assertions that the new ANALYZING interposition may shift).

### Stage 3 Commit 3 — Tests + Smoke (~60-90min; ~300-500 LoC)

**C3.1** Create tests/services/ai/__init__.py + tests/services/ai/test_clients.py (singleton lifecycle + Depends + module-level access; ~80-120 LoC).

**C3.2** Create tests/services/ai/test_cost_tracking.py (budget enforcement + circuit breaker + per-call logging; ~80-120 LoC).

**C3.3** Create tests/services/test_ai_pipeline_consumer.py (consumer task + queue draining + no-op pipeline; ~80-120 LoC).

**C3.4** Create tests/integration/test_m9_smoke.py (end-to-end no-op smoke test per Q11 C.c-lite; ~80-150 LoC). V-check: M8.1β.2 V-BBF noted no tests/integration/ dir — create dir (with __init__.py) or co-locate per C3 V-check (mirror M8.1β.2 e2e-in-test_completions_consumer.py contingency if dir-creation is out of preferred scope).

**C3.5** Pre-commit verification + single atomic api commit. Full suite green; ZERO-regressions-at-each-commit gate.

### Stage 3 Aggregate LoC Forecast
Total across 4 commits (C0 docs + C1 + C2 + C3 api): ~830-1400 LoC. Comparable to M8.1β.2 Stage 3 aggregate (+1356 LoC; ADR-style architectural-decision implementations at ~1300-1400 LoC density level).

## 5. D-Deviation Tracking Framework

Per Task 8.3α + M8.1α + M8.1β.1 + M8.1β.2 D-PLAN tracking precedent.

**Pre-execution drifts catalogued (cumulative count 65):** Drift #58 + #59 + #60 + #61 + #62 + #63 (M8 lifecycle) + Drift #64 + #65 (M9.0 Stage 1 catalogue per V-EE; resolution at Stage 3 C1 schema migration + cost_tracking module).

**Expected Stage 3 D-deviation count:** MODERATE bound (~1-3 drifts at execution) per partial-novel-territory hypothesis. Honest forecast per-commit:
- C0 docs (annotation-heavy): 0 drifts
- C1 schema + modules (multi-Y-decision novel territory; client + cost-tracking greenfield): 1-2 drifts likely (recon-invocation-seam catch-class precedent for ai-pipeline-seam adjacent territory; DEFERRED-EMPIRICAL pseudo-code at §3.1/§3.2 will ground against real config.py + ScanType + AIAPICall)
- C2 consumer + dispatch (mirrors completions_consumer pattern): 0-1 drifts
- C3 tests + smoke (test-impact-surface scope per Drift #63 forward-pin): 0-1 drifts (ANALYZING interposition at _maybe_complete_scan may surface existing consumer/route test impact)

**Plan-level Y-decisions to resolve at execution (4 total):**
- Y-AI-CLIENTS-MODULE-SHAPE: (a) default single clients.py vs (b) per-provider modules; resolve at C1.4
- Y-COST-TRACKING-MODULE-SHAPE: (a) default function-style vs (b) class-style vs (c) hybrid; resolve at C1.5
- Y-AI-PIPELINE-CONSUMER-LIFECYCLE: (a) default FastAPI lifespan vs (b) module-level vs (c) explicit start/stop; resolve at C2.5
- Y-NO-OP-PIPELINE-SHAPE: (a) default immediate-return vs (b) simulated-stages vs (c) NotImplementedError-skip; resolve at C2.4

## 6. Out of Scope (per design doc §6 + plan-level refinements)

All Q-lock forward-pin chain extensions (20+ items per design doc §7) + M9.A/B/C/D sub-milestone scopes + Real AI provider calls (deferred to M9.A) + M10 Report architecture + Production-readiness audit items.

Additionally out-of-scope at M9.0 plan:
- AI pipeline algorithm implementation (embed/dedup/correlate/score/fix-gen/summary; all M9.A/B/C/D sub-milestone scopes)
- Vulnerability promotion logic (M9.A activation)
- raw_finding promoted_at + vulnerability_id state-transition logic (Q6 B.a; M9.A activation populates these — M9.0 only adds the columns)
- Qdrant collection lifecycle management beyond lazy-creation primitive (Q4 forward-pin)

## 7. Forward-Pins

**Pre-execution (Stage 3 entry):**
1. Stage 3 implementation trigger: ***"Begin M9.0 — Stage 3 implementation"***
2. Y-AI-CLIENTS-MODULE-SHAPE decision context preserved (a default; resolve C1.4)
3. Y-COST-TRACKING-MODULE-SHAPE decision context preserved (a default; resolve C1.5)
4. Y-AI-PIPELINE-CONSUMER-LIFECYCLE decision context preserved (a default; resolve C2.5)
5. Y-NO-OP-PIPELINE-SHAPE decision context preserved (a default; resolve C2.4)
6. Stage 1 design doc canonical authority: a46fedd §3 + §4 verbatim drafts; ADR-029 C0 text source

**Post-Stage-3:**
7. ***"Begin M9.A — Embedding + Deduplication"*** (Tasks 9.1+9.2 sub-milestone; after M9.0 P5.A closes)
8. ***"Begin M9.B — Correlation + Scoring"*** (Tasks 9.3+9.4 sub-milestone)
9. ***"Begin M9.C — Fix Generation + Executive Summary"*** (Tasks 9.5+9.6 sub-milestone)
10. ***"Begin M9.D — Pipeline Orchestrator"*** (Task 9.7 sub-milestone)

**Discipline-level (active + extended at M9.0 entry):**
11. Audit-driven model+spec orphan check (Drift #60 rule-of-three; M9.0 V-EE successfully detected #64/#65)
12. DEFERRED-EMPIRICAL marking for concrete-empirical-references (Drift #61 + #62 catch-class evolution; applied at §3.1/§3.2 pseudo-code)
13. Examine recon-invocation architectural seam (Drift #59 + #62 adjacent-layer; M9.0 ai-pipeline-dispatch-seam analog at C2.2/C2.3)
14. Pre-verification scope completeness across test-impact-surface (Drift #63 extension; applied at C2.6 + C3)

**Production-readiness audit forward-pin chain:** 9 items preserved per Stage 1 design doc §7.

**Persistent DRIFT-LOG sync forward-pin (M8.1β.2 V-CC precedent):** Drift #64 + #65 catalogued at design doc §9 + this plan §5; persistent api/engine DRIFT-LOG entries land at Stage 3 (C1 schema-migration timing for #64/#65 resolution canonical) OR Stage 4 P5.A. Track at lifecycle close.

## 8. Cross-References

**Docs:** a46fedd (M9.0 Stage 1 design doc canonical authority for this plan); 9d571b6 (M8.1β.2 P5.A docs); 9507acb (M8.1β.2 Stage 3 C1 ADR-028 canonical authority; M9.0 composes ADR-028 phase-1+phase-2 architecture providing AttackSurface/Vulnerability foundation); 3f07611 (M8.1β.2 design doc structural precedent); fb61129 (M8.1β.2 plan precedent at 429 LoC; closest 3-instance analog); 0030319 (Task 8.3α design doc structural precedent); dba6a7c (Task 8.3α plan precedent at 422 LoC); 04f44a9 (source-ingestion fix plan precedent at 426 LoC); SPECIFICATION.md §8 canonical M9 pipeline authority + SPEC §13 ADR-029 (Stage 3 C0 implementation target); CLAUDE.md Gotcha 5 cost-tracking mandate.

**API:** 1b7b314 (Task 8.3β attack-surface endpoint; latest api state); dc39fd1 (M8.1β.2 completions_consumer phase-2 hook; M9.0 extends with _maybe_complete_scan ai-pipeline-dispatch hook + Q1 e.ii); 04a9b5c (M8.1β.2 orchestrator dispatch_phase2 precedent; M9.0 adds dispatch_ai_pipeline analog); src/app/models/scans.py + src/app/models/vulnerabilities.py + src/app/models/raw_findings.py + src/app/services/completions_consumer.py + src/app/services/orchestrator.py + src/app/main.py (all Stage 3 modification targets); src/app/services/ai/* (Stage 3 NEW files).

**Pre-verification artifacts:** V-EE + V-FF + V-GG surface reports.

**Cumulative drift count:** 65 catches at execution time (Drift #58-#65 catalogued); #64/#65 catalogued at M9.0 Stage 1 design doc §9 + this plan §5 + persistent DRIFT-LOG sync forward-pinned to Stage 3/P5.A.

**Milestone-completion-constraint context:** locked at M9.0 sub-milestone scope; M9.0 lifecycle CLOSED required before M9.A activation; full M9 closure requires M9.0 + M9.A + M9.B + M9.C + M9.D + each sub-milestone Stage 4 P5.A.
