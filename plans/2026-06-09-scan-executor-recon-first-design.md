# M8.1β.2 — Scan-Executor Recon-First Architecture (ADR-028): Design

**Status:** Brainstorming chain Q1-Q10 locks complete + V-W pre-Stage-1 refinements (this session); ready for Stage 2 implementation plan landing + Stage 3 4-commit cross-repo (docs → engine → api orchestrator → api completions_consumer+e2e) + Stage 4 P5.A.

**Date:** 2026-06-09.

**Authority:** M81_PV pre-verification surface report (prior session; Outcome 3 architectural-decision territory); M81B_PV pre-verification surface report (prior session; V-UB-V-UM + 7 Y-decisions surfaced with defaults + Pattern A 2-task decomposition); brainstorming chain CLOSED this session (Q1-Q10 locks per §3 below; Mode 1 conversational deliberation); V-W pre-Stage-1 verification (this session; Q8 ADR-028 locked + Q7.4 refined to Option α audit-log lookup + Drift #61 catalogued); M8.1α lifecycle CLOSED at `fb8cff9` + `2b36d62` + `64b8421` (Drift #60 name-mismatch resolved); M8.1β.1 lifecycle CLOSED at `bb3e75f` + `d773776` + `9ccde1a` (Drift #60 engine-variant resolved); Task 8.3α lifecycle CLOSED at `0030319` + `dba6a7c` + `721ba02` + `fc75a98` + `05023f4` + `0e5249e` (Drift #58 resolved end-to-end; foundation for Q1 hybrid follow-up dispatch composition); SPEC §13 ADR-013 + ADR-014 + ADR-017 + ADR-022 + M8.1α/β.1 addendums; ADR-018 forward-pinned; ADR-019 reserved; ADR-020 reserved-now-closed-empirically (per Q3 + this ADR); 61+ cumulative session-tail framing-drift discipline.

**Related:** Implementation plan landing trigger: ***"Begin M8.1β.2 implementation plan landing"***. Stage 3 cross-repo implementation trigger: ***"Resume M8.1β.2 — Stage 3 cross-repo 4-commit implementation"***.

---

## 1. Authority + Q1-Q10 Locks Summary

Brainstorming chain (Q1-Q10) resolved this session under Mode 1 conversational deliberation; V-W pre-Stage-1 verification refined Q7.4 + Q8 empirically:

- **Q1 Y-EXPAND-LOCATION → (c) hybrid follow-up dispatch.** api dispatches initial RECON ScanJob at phase-1; engine RunRecon emits EventAttackSurface (Task 8.3α infrastructure); api completions_consumer UPSERTs AttackSurface AND triggers re-dispatch of per-(target, tool) ScanJobs at phase-2.
- **Q2 Y-DISPATCH-MODEL → (a) per-(target, tool) ScanJobs + (a.ii) target_hash idempotency + (i) empty-AttackSurface-is-correct.** ScanJob.target_url unchanged; idempotency-key format `{scan_id}:{tool}:{sha256(target_url)[:16]}:{ts}`.
- **Q3 Y-CONCURRENCY-MODEL → (a) per-worker only + ADR-020 promotion-trigger closure documented + per-target rate-limiting forward-pinned.**
- **Q4 Y-MIGRATION-NEEDED → (a) NO migration.** Preserved through V-WD reconsideration via Q7.4 Option α.
- **Q5 Y-AUDIT-TRAIL-CHANGES → (b) two-phase audits + (b.i) always-emit empty-case + richer details shape.** New event type `SCAN_DISPATCHED_PHASE2 = "scan.dispatched_phase2"` (per V-WC convention); details = `{scan_type, priority, phase: "tools", target_count, tool_count, job_count, recon_event_id}`.
- **Q6 Y-RECON-ENGINE-NAME → (a) dedicated engine="recon" + (a.ii) implicit orchestrator dispatch + name "recon".** subfinder + httpx removed from SCAN_TYPE_TOOLS entirely (closes Drift #60 recon-orphan sub-category structurally).
- **Q7 Y-DISPATCH-PHASE-COORDINATION → (c) orchestrator dispatch_phase2 method + (c.ii) sequential sessions + (c.ii.B) fail-loud-audit + query-inline.**
- **Q7.4 [REFINED at V-WD] → Option (α) audit-log lookup.** Original lock "Identity from `Scan.created_by_user_id`" invalidated by V-WD (field absent); Drift #61 catalogued; refinement preserves Q4 NO-migration via audit-as-source-of-truth pattern.
- **Q8 Y-ADR-NUMBER [LOCKED at V-WB] → ADR-028 "Scan-Executor Recon-First Architecture".** Next-available canonical at SPEC §13 (existing ADRs 001-018 + 021/022/023/024/026/027; ADR-019 + 020 reserved; ADR-025 lives at api DRIFT-LOG).
- **Q9 Q-STAGE3-DECOMPOSITION → (b) 4-commit cross-repo + bundled tests per commit.** Sequencing: docs → engine → api orchestrator → api completions_consumer+e2e.
- **Q10 Q-MIGRATION-PATH → (c) bounded-staleness + production-readiness audit forward-pinned.** Pre-launch context dominates.

**Drift #60 resolution chain at this ADR:** 6/6 engines resolved end-to-end (name-mismatch 1/1 at M8.1α + engine-variant 3/3 at M8.1β.1 + recon-orphan 2/2 at M8.1β.2 this ADR). Discipline-level "audit-driven model+spec orphan check" forward-pin preserved (rule-of-three trigger fired at #60).

**Drift #61 catalogued at V-WD:** 4th-instance plan/design-vs-empirical-precision catch-class (after #53/#55/#56/#59); rule-of-three already triggered at #59; this confirms catch-class durability across architectural-decision layer not just plan layer. Discipline-level meta-pattern: Q-decisions referencing concrete database fields, file paths, or API contracts should be marked "DEFERRED-EMPIRICAL" rather than locked-with-named-default unless pre-verification has already grounded empirical state.

## 2. Pre-Verification Findings

**M81_PV (prior session; Outcome 3 architectural-decision territory):** V-RB Task 8.1 plan-literal speculative scaffolding; V-RC api per-(scan, tool) ScanJob dispatch with single target_url; V-RD recon primitives + ReconResult.LiveHosts; V-RE NO partial engine scaffolding; V-RF dispatch lifecycle flow; V-RG cancel infrastructure supports multi-job; V-RH ADR-013 sole-writer constraint; V-RI SPEC §6 attack-surface endpoint context; V-RJ arc-evolution-diverged-path evidence; V-RK forward-pin chain references; V-RL Drift #60 candidate (3rd-instance stored-design-intent catch-class).

**M81B_PV (prior session; architectural decision space):** V-UB speculative scaffolding TOOL-ARCH §8.2+§10.3+§12.3+SPEC §1885+ pre-date ADR-013 (treat as INFORMATIONAL); V-UC ADR-020 reserved territory; V-UD SPEC §1885+ M8 invocation explicit speculative; V-UE engine-variant analysis (M8.1β.1); V-UF recon-orphan invocation Option (c) hybrid follow-up; V-UG ScanJob model fitness NO migration; V-UH cancel-fanout adequate; V-UI worker concurrency status quo per-worker only sufficient; V-UJ SCAN_DISPATCHED two-phase territory; V-UK ADR-013 per-option compliance; V-UL test infrastructure readiness; V-UM forward-pin chain genuinely-new territory.

**V-W pre-Stage-1 verification (this session):** V-WA clean state confirmed; V-WB ADR numbering locks ADR-028 as next-available canonical at SPEC §13; V-WC audit event type registration convention (`"scan.dispatched_phase2"` dotted.snake_case); V-WD identity field empirical drift surfaced (`Scan.created_by_user_id` absent) → Q7.4 refined to Option α; V-WE design doc structural precedent confirmed.

**Brainstorming chain Q1-Q10 (this session):** Mode 1 conversational deliberation; 7 Y-decisions + 3 Q-decisions ratified; sub-decisions per Q; rejected alternatives traces; V-W refinements applied to Q7.4 + Q8.

## 3. Architectural Decisions (Q-Lock Detail)

### 3.1 Y-EXPAND-LOCATION (c) Hybrid Follow-Up Dispatch

**Lock:** Phase-1 RECON ScanJob → engine RunRecon emission via Task 8.3α infrastructure → api completions_consumer UPSERT AND phase-2 dispatch trigger → per-(target, tool) ScanJobs created.

**Grounding:** Task 8.3α (`fc75a98` + `05023f4`) built engine-side EventAttackSurface emission AND api-side UPSERT consumer; Q1 extends with ONE NEW component (api re-dispatch hook via `orchestrator.dispatch_phase2`). ADR-013 sole-writer + ADR-022 recon-as-helper preserved by construction.

**Rejected alternatives:**

- (a) api synchronous pre-dispatch — IMPRACTICAL (FastAPI doesn't own recon binaries; would require embedding Go binaries OR HTTP-fetch from engine OR shell out)
- (b) engine multi-target per ScanJob — VIABLE but architectural debt (target_urls JSONB migration + new wire-shape + per-target completion-event semantics + per-(ScanJob, target) idempotency keys)
- (d) ReconCoordinator — SPECULATIVE (writer responsibility uncertain; no advantage over c)

### 3.2 Y-DISPATCH-MODEL (a) Per-(target, tool) ScanJobs

**Lock:** Keep `ScanJob.target_url String(500)`; api creates N×M ScanJobs at phase-2 (N targets × M non-recon tools).

**Idempotency-key (a.ii):** `{scan_id}:{tool}:{sha256(target_url)[:16]}:{ts}` — deterministic + order-independent + URL-identity-stable.

**Empty-AttackSurface (i):** Zero live subdomains = zero phase-2 ScanJobs. Scan completes with phase-1-only per recon-first semantics.

**Rejected:** (b) engine multi-target — Q1 cascade rejection; (c) hybrid target_index column — Q4 migration cascade rejection.

### 3.3 Y-CONCURRENCY-MODEL (a) Per-Worker Only

**Lock:** Existing worker BRPOP-loop + per-worker semaphore consumes N×M ScanJobs naturally; concurrency at dispatch-fanout layer not in-engine.

**ADR-020 closure:** DRIFT-LOG L349 hypothesized M8.1 as promotion-trigger; empirically promotion-trigger DID NOT fire at M8.1β.2 per Q1+Q2 lock cascade. ADR-020 stays deferred indefinitely unless future tasks surface load-bearing concurrency trade-offs.

**Per-target rate-limiting forward-pinned:** Operational concern at scale (e.g., 261 ScanJobs hitting same target infrastructure simultaneously) but out-of-scope at M8.1β.2.

**Rejected:** (b) per-job tool-fanout — Q2 cascade rejection.

### 3.4 Y-MIGRATION-NEEDED (a) NO Migration

**Lock:** Schema preserved through M8.1β.2; idempotency-key extension is dispatch-logic-only; no alembic migration files.

**V-WD reconsideration result:** Q7.4 refined to Option α audit-log lookup preserves Q4 NO-migration lock; alternative (δ) reopen Q4 + add `Scan.created_by_user_id` rejected per brainstorming-chain coherence.

### 3.5 Y-AUDIT-TRAIL-CHANGES (b) Two-Phase Audits

**Lock:** New event type `SCAN_DISPATCHED_PHASE2 = "scan.dispatched_phase2"` (per V-WC `ScanAction` convention).

**Always-emit empty-case (b.i):** Phase-2 audit emitted even when target_count=0; operator visibility into "scan didn't dispatch tools because recon found no targets."

**Rich details shape:**

```json
{
  "scan_type": "FULL_WEB",
  "priority": "normal",
  "phase": "tools",
  "target_count": 5,
  "tool_count": 6,
  "job_count": 30,
  "recon_event_id": "<uuid of phase-1 SCAN_DISPATCHED audit>"
}
```

**Phase-1 SCAN_DISPATCHED preserved unchanged** for backward-compat with existing consumers.

**Rejected:** (a) status quo single audit (broken — job_count unknown at phase-1); (c) extended details with phase discriminator (less discoverable); (d) progress-events instead of audit (semantic layer conflation).

### 3.6 Y-RECON-ENGINE-NAME (a) Dedicated engine="recon" + (a.ii) Implicit Orchestrator Dispatch

**Lock:** subfinder + httpx removed from SCAN_TYPE_TOOLS entirely. SCAN_TYPE_TOOLS stays semantically pure (tools only). Orchestrator phase-1 dispatches recon ScanJob automatically per web-ScanType category (QUICK + FULL_WEB + FULL_WEB_SOURCE + FULL_SPECTRUM). Non-web ScanTypes (MOBILE + CONTAINER + API) skip phase-1.

**Engine processor.Process new dispatch case:** `engine="recon"` → invoke `RunRecon` (already wired at `fc75a98`).

**Drift #60 recon-orphan sub-category resolved STRUCTURALLY** at this lock (engine names not in SCAN_TYPE_TOOLS = no dispatch mismatch).

**Rejected:** (b) preserve subfinder+httpx with special processor handling — architectural debt for preserving artificial engine names; (c) hybrid is_recon_phase column — Q4 cascade rejection.

### 3.7 Y-DISPATCH-PHASE-COORDINATION (c) Orchestrator dispatch_phase2 Method

**Lock:** `orchestrator.dispatch_phase2(scan_id, db)` method invoked from `completions_consumer._handle_attack_surface` after UPSERT commit.

**Sequential sessions (c.ii):** UPSERT commits in handler session; dispatch_phase2 opens NEW session with RLS GUC; commits independently. Composes cleanly: phase-1 done (AttackSurface persisted) → phase-2 attempted (dispatch new ScanJobs).

**Fail-loud-audit (c.ii.B):** Dispatch-failure marks scan FAILED with audit explaining cause. Retry-logic forward-pinned post-M8.1β.2.

**Query-inline pattern:** orchestrator queries AttackSurface rows for scan; returns created ScanJobs (testable + observable).

**Identity resolution refinement (Q7.4) — V-WD empirical drift catalogued as Drift #61:**

Brainstorming chain Q7.4 ratified "Identity from `Scan.created_by_user_id`" as default. V-WD pre-Stage-1 verification (this stage) revealed: Scan model has NO `created_by_user_id` field. Drift #61 catalogued (4th instance of plan/design-vs-empirical-precision catch-class after #53/#55/#56/#59; rule-of-three already triggered at #59; this confirms pattern durability across architectural-decision layer not just plan layer).

Refinement lock: **Option (α) audit-log lookup.** Phase-2 dispatch queries most recent `SCAN_DISPATCHED` `AuditLog` row for `scan_id`; reuses `actor_id`; reconstructs partial `AuthIdentity` for orchestrator.dispatch internal context.

**Implicit constraint documented:** This refinement creates an implicit ordering dependency — phase-2 dispatch ASSUMES `SCAN_DISPATCHED` audit row exists at phase-2 trigger time. Today this is true (audit emission is synchronous within ScanCreate transaction). If audit emission ever becomes async, phase-2 dispatch path requires revisit.

**Rejected alternatives at V-WD:**

- (β) System identity: Loses original-actor attribution at `SCAN_DISPATCHED_PHASE2` audit row; weaker security audit trail
- (γ) Redis-cached identity snapshot: Adds stateful Redis primitive outside ADR-014 mixed-primitives scope; TTL management overhead
- (δ) Reopen Q4 + add `Scan.created_by_user_id` column: Defensible per pre-launch context; rejected per brainstorming-chain coherence + Q4 lock-preservation + audit-as-source-of-truth pattern

### 3.8 Y-ADR-NUMBER ADR-028 + Title

**Lock at V-WB:** ADR-028 "Scan-Executor Recon-First Architecture" at SPEC §13.

Existing canonical ADRs verified: 001-018 + 021/022/023/024/026/027; ADR-019 + 020 reserved; ADR-025 lives at api DRIFT-LOG consumer-side.

### 3.9 Q-STAGE3-DECOMPOSITION (b) 4-Commit Cross-Repo

**Sequencing:**

- Commit 1 (docs FIRST): ADR-028 + ADR-022 addendum continuation + ADR-020 closure documentation + speculative-scaffolding resolution markers
- Commit 2 (engine SECOND): `processor.go` recon dispatch case + DRIFT-LOG entry
- Commit 3 (api orchestrator THIRD): phase-1 dispatch + `dispatch_phase2` method + `SCAN_DISPATCHED_PHASE2` audit registration + tests
- Commit 4 (api completions_consumer + e2e FOURTH): UPSERT hook + end-to-end integration tests

**Rationale:** Canonical authority before implementation; engine recon dispatch enables phase-1; api orchestrator creates dispatch_phase2; api completions_consumer wires trigger + closes chain with e2e tests.

### 3.10 Q-MIGRATION-PATH (c) Bounded-Staleness

**Lock:** Pre-launch context dominates; no production ScanJobs to migrate; new behavior activates at deploy-time; test fixtures get updated.

**Production-readiness audit forward-pinned:** Revisit M8.1β.2 + earlier task decisions (Q2 idempotency-key dual-format, Q4 migration-paths, Q5 audit consumer coordination, Q10) for production-grade considerations before live launch.

## 4. Cross-Repo Implementation Surface

### 4.1 Stage 3 Commit 1 — Docs (ADR-028 + addendums + speculative resolution)

**Files:** `SPECIFICATION.md` (NEW ADR-028 at §13 + ADR-022 addendum continuation for Drift #60 recon-orphan closure + ADR-020 promotion-trigger closure documentation); `TOOL-ARCHITECTURE.md` (SPECULATIVE markers at §8.2 + §10.3 + §12.3 updated to reference ADR-028 as canonical authority).

**LoC forecast:** ~80-150 LoC.

### 4.2 Stage 3 Commit 2 — Engine (processor.go recon dispatch + DRIFT-LOG)

**Files:** `internal/worker/processor.go` (new dispatch case `engine="recon"` → invoke `RunRecon`); `DRIFT-LOG.md` (Drift #60 6/6 closure entry); tests.

**LoC forecast:** ~30-80 LoC.

### 4.3 Stage 3 Commit 3 — API Orchestrator (phase-1 + dispatch_phase2 + audit registration + tests)

**Files:** `src/app/services/orchestrator.py` (SCAN_TYPE_TOOLS rename: remove subfinder+httpx from web ScanTypes; phase-1 web-ScanType recon ScanJob dispatch; `dispatch_phase2` method per §3.7); `src/app/services/audit.py` (`SCAN_DISPATCHED_PHASE2` enum addition); `tests/services/test_orchestrator.py` (phase-1 dispatch unit + dispatch_phase2 unit + audit emission).

**LoC forecast:** ~200-400 LoC.

### 4.4 Stage 3 Commit 4 — API Completions Consumer + E2E (UPSERT hook + e2e flow tests)

**Files:** `src/app/services/completions_consumer.py` (`_handle_attack_surface` extension: post-UPSERT-commit invokes `orchestrator.dispatch_phase2` in new session; fail-loud audit on failure); `tests/services/test_completions_consumer.py` + `tests/integration/test_recon_first_e2e.py` (end-to-end ScanCreate → recon emission → AttackSurface UPSERT → phase-2 dispatch → tool ScanJobs created).

**LoC forecast:** ~200-400 LoC.

**Stage 3 aggregate LoC forecast:** ~510-1030 LoC. Comparable to Task 8.3α Stage 3 trio (+785 LoC).

## 5. Phase Structure

Stage 1 design doc (THIS COMMIT) → Stage 2 implementation plan → Stage 3 4-commit cross-repo → Stage 4 P5.A.

## 6. Out of Scope

1. Per-target execution rate-limiting (forward-pin from Q3)
2. Phase-2 dispatch retry logic (forward-pin from Q7)
3. ADR-018 Streams+consumer-groups migration
4. Speculative scaffolding deprecation (TOOL-ARCH §8.2 + §10.3 + §12.3 + SPEC §1885+ pre-date ADR-013)
5. Task 8.3β attack-surface endpoint (forward-pinned post-M8.1β.2)
6. Production-readiness audit (forward-pin from Q10)
7. M9 entry (M8 closure at M8.1β.2 + Task 8.3β)
8. ADR-018 + ADR-019 promotions (M8.1β.2 only closes ADR-020 reserve)

## 7. Forward-Pins

**Pre-implementation:**

1. Stage 2 plan landing trigger: ***"Begin M8.1β.2 implementation plan landing"***
2. Stage 3 implementation trigger: ***"Resume M8.1β.2 — Stage 3 cross-repo 4-commit implementation"***

**Post-Stage-3:**

3. ***"Begin Task 8.3β attack-surface endpoint task"*** — mechanical compressed-lifecycle on populated rows
4. ***"Begin per-target execution rate-limiting"*** — operational concern from Q3
5. ***"Begin phase-2 dispatch retry logic"*** — operational hardening from Q7
6. ***"Begin speculative scaffolding deprecation pass"*** — TOOL-ARCH + SPEC text cleanup post-M8.1β.2
7. ***"Begin production-readiness audit"*** — revisit decisions for production-grade considerations
8. ***"Begin M9 entry"*** — after M8 declared CLOSED

**Discipline-level:**

9. ***"Integrate audit-driven model+spec orphan check into pre-verification template"*** — Drift #60 rule-of-three trigger
10. ***"Adopt DEFERRED-EMPIRICAL marking for concrete-empirical-field Q-decisions"*** — Drift #61 discipline-level meta-pattern (NEW from V-WD finding)

## 8. Cross-References

**Engine:** `fc75a98` (Task 8.3α Stage 3 C2; RunRecon emission infrastructure); `9ccde1a` (M8.1β.1 DRIFT-LOG); `a0bff50` (source-ingestion fix Stage 3 C3).

**Docs:** `bb3e75f` (M8.1β.1 ADR-022 addendum continuation); `fb8cff9` (M8.1α ADR-022 addendum); `0e5249e` (Task 8.3α P5.A); `0030319` (Task 8.3α design doc structural precedent); `dba6a7c` (Task 8.3α plan); `90fc933` (source-ingestion fix design doc); SPEC §13 ADR-013 + ADR-014 + ADR-017 + ADR-022 + M8.1α/β.1 addendums + ADR-028 (this design's target); ADR-018 + ADR-019 + ADR-020 forward-pinned/reserved.

**API:** `d773776` (M8.1β.1 SCAN_TYPE_TOOLS rename + variant config); `05023f4` (Task 8.3α Stage 3 C3; completions_consumer + AttackSurface UPSERT); `8dbcbab` (source-ingestion fix Stage 3 C2); `2b36d62` (M8.1α SCAN_TYPE_TOOLS rename).

**Pre-verification artifacts:** M81_PV + M81B_PV + M81B1_PRE + V-W surface reports (prior + this session).

## 9. Drift #60 6/6 Closure + Drift #61 Catalogue

**Drift #60 (stored-design-intent-with-unimplemented-mechanism; 3rd instance after #54 + #58; rule-of-three trigger fired):**

6-engine surface across 3 sub-categories:

- Name-mismatch (1/1): `depcheck` (RESOLVED at M8.1α; commits `fb8cff9` + `2b36d62` + `64b8421`)
- Engine-variant (3/3): `nuclei_fast` + `nuclei_api` + `zap_api` (RESOLVED at M8.1β.1; commits `bb3e75f` + `d773776` + `9ccde1a`)
- Recon-orphan (2/2): `subfinder` + `httpx` (RESOLVED at M8.1β.2 this ADR; structural resolution per Q6 — engines removed from SCAN_TYPE_TOOLS entirely; dedicated `engine="recon"` ScanJob at phase-1)

**Drift #60 catch-class CLOSED at this ADR.** Discipline-level forward-pin "audit-driven model+spec orphan check" preserved.

**Drift #61 catalogued (V-WD finding):** Q7.4 lock specified "Identity from `Scan.created_by_user_id`" before empirical verification of field existence. V-WD pre-Stage-1 verification revealed field absent. 4th-instance plan/design-vs-empirical-precision catch-class (after #53/#55/#56/#59). Rule-of-three already triggered at #59 (discipline-level forward-pin "plan-doc consistency check pre-implementation" established at M8.1α §9.F #8); this drift confirms catch-class durability across architectural-decision layer not just plan layer. Cumulative session-tail framing-drift count: 60 → 61.

**Implication for future brainstorming chains:** Q-decisions referencing concrete database fields, file paths, or API contracts should be marked "DEFERRED-EMPIRICAL" rather than locked-with-named-default unless pre-verification has already grounded the empirical state. Discipline-level meta-pattern.
