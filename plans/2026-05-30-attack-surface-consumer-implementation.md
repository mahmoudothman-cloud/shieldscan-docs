# Task 8.3α — AttackSurface Persistence Consumer: Implementation Plan

**Status:** Ready for Stage 3 cross-repo (docs → engine → api) implementation. Pre-implementation grounded against Task 8.3α design doc `0030319` 12-Q-chain locks + T83_PV + T83A_PV pre-verification + V-LB-V-LI + V-MA-V-MI empirical state. Plan structure grounded against verified `completions_consumer.py` architecture + `AttackSurface` ORM + `uq_scan_subdomain` UPSERT precedent + ADR-014 mixed-primitives canonical authority.

**Date:** 2026-05-30.

**Authority:** Task 8.3α design doc canonical authority (shieldscan-docs commit `0030319`; 12-Q-chain architectural-decision authority + Drift #58 two-layer manifestation cataloging); T83_PV + T83A_PV pre-verification surface reports (this session; V-LB-V-LI + V-MA-V-MI grounding); source-ingestion fix implementation plan structural precedent (commit `04f44a9`; 426 LoC); revocation implementation plan structural precedent (commit `fdad021`; 338 LoC); R2 implementation plan structural precedent (commit `721f788`; 357 LoC); V10 implementation plan structural precedent (commit `7c4fe75`; 304 LoC); ADR-015 implementation plan structural precedent (commit `00dd2d1`; 278 LoC); Task 7.6 implementation plan structural precedent (commit `40606c5`; 269 LoC); 58+ cumulative session-tail framing-drift discipline; milestone-completion-constraint locked this session.

**Related:** Stage 3 cross-repo implementation trigger phrase: ***"Resume Task 8.3α — Stage 3 cross-repo implementation"*** (after this plan lands).

---

## 1. Authority + Scope Lock

Y-CONSUMER-LOCATION + Y-WIRE-SHAPE + 10 Q-locks (locked at design doc `0030319` §1): see design doc §1 + §3 for full architectural rationale.

**In scope (Stage 3 sub-steps; Q-COMMITS 3-commit shape):**

1. shieldscan-docs `TOOL-ARCHITECTURE.md` §8 addendum + `SPEC.md` §7 addendum: dual canonical authority for `EventAttackSurface` (Stage 3 Commit 1; ~30-55 LoC)
2. shieldscan-engine `internal/events/events.go`: NEW `EventAttackSurface` struct + JSON tag conventions (Stage 3 Commit 2; ~10-15 LoC)
3. shieldscan-engine `internal/tools/recon/recon.go`: `RunRecon`-internal callsite publishing `EventAttackSurface` after httpx phase (Stage 3 Commit 2; ~30-50 LoC)
4. shieldscan-engine `internal/redis/pubsub.go`: `CompletionsPublisher` extension per Y-EVENT-PUBLISH-PATH resolution (Stage 3 Commit 2; ~5-15 LoC)
5. shieldscan-engine tests: emission unit + integration extension (Stage 3 Commit 2; ~20-30 LoC)
6. shieldscan-api `src/app/services/completions_consumer.py`: `_handle()` routing extension + NEW `_handle_attack_surface(event)` handler (Stage 3 Commit 3; ~80-150 LoC)
7. shieldscan-api `tests/services/test_completions_consumer.py`: 5-7 attack_surface event scenario tests (Stage 3 Commit 3; ~80-150 LoC)

**Out of scope (forward-pinned per design doc §6):** Task 8.3β GET endpoint; M8.1 recon-invocation; ADR-018 consumer-groups; per-scan-stream subscription consumers; `EventReconCompleted` enrichment; `AttackSurface` schema migrations; engine recon-pipeline retries; findings-table join for vulnerability_count; multi-tenant isolation testing beyond existing RLS pattern; ADR-013 modifications; `processor.go` modifications; operational deployment posture changes.

**Brainstorming chain 12-Q-locks recap:** Y-CONSUMER-LOCATION (α) extend completions Pub/Sub channel; Y-WIRE-SHAPE (a) new `EventAttackSurface`; Q-ENGINE-EMIT-CALLSITE (a) `RunRecon`-internal; Q-EVENT-PAYLOAD-FIELDS rich subdomain rows; Q-UPSERT-PATTERN `ON CONFLICT (scan_id, subdomain) DO UPDATE`; Q-LIFECYCLE `completions_consumer.py` extension; Q-RLS-GUC org-scoped `SET`; Q-TEST-PATTERN fakeredis + direct `_handle`; Q-MULTI-PROCESS-POSTURE (b) UPSERT-idempotency-by-construction; Q-DOCS-LOCATION (c) dual TOOL-ARCH §8 + SPEC §7 addendums; Q-PHASE-0-V2 NO; Q-COMMITS 3-commit cross-repo (docs → engine → api).

## 2. Pre-Implementation State

### 2.1 Engine + API + Docs Infrastructure Ready Matrix (V-LB-V-LI + V-MA-V-MI Pre-Verification Findings)

**Engine side ready:**

- `internal/events/events.go`: existing `EventLivenessProbed` + `EventReconCompleted` emission pattern + `JobTarget.SourceRepoURL` recent precedent for new event-type additions
- `internal/tools/recon/recon.go`: 4 emission sites for `EventSubdomainsDiscovered` + `EventLivenessProbed` + `EventReconCompleted`; `RunRecon` returns `ReconResult.LiveHosts` rich struct (V-MD); rich data EXISTS in-memory but never published
- `internal/redis/pubsub.go`: `CompletionsPublisher` at L132-161; `Publish(ctx, ev events.JobCompletedEvent) error` typed-to-completion-event today; channel `shieldscan:completions` per ADR-014 (V-NB confirmed)
- `internal/worker/processor.go`: `completionsPub` field at L65; injection at L90 via `NewCompletionsPublisher(client)`; 3 callsites at L260/L371/L406 publish `JobCompletedEvent` only
- `go.mod`: existing infrastructure sufficient; no new dep

**Engine side pending (Stage 3 Commit 2 scope):**

- `internal/events/events.go`: NEW `EventAttackSurface` event type + `SubdomainRow` struct with `scan_id` + `organization_id` + `root_domain` + `subdomains` array (~10-15 LoC)
- `internal/redis/pubsub.go`: `CompletionsPublisher` extension per Y-EVENT-PUBLISH-PATH resolution — either (b1) new `PublishAttackSurface(ctx, ev EventAttackSurface) error` second typed method OR (b2) generalize via event-typed dispatch (~5-15 LoC)
- `internal/tools/recon/recon.go`: `RunRecon`-internal callsite — after httpx phase aggregates `ReconResult.LiveHosts` → publishes `EventAttackSurface` to completions channel BEFORE `EventReconCompleted` emission per Q-ENGINE-EMIT-CALLSITE (a) (~30-50 LoC; depends on V-NB-confirmed publisher being plumbed into recon — verify at C2.1 / resolve via Y-RECON-PUBLISHER-WIRING below)
- tests: emission unit test (mock publisher) + integration test extension with real completions assertion (~20-30 LoC)

**API side ready (V-MB):**

- `src/app/services/completions_consumer.py`: 601 LoC architecture; channel `shieldscan:completions` Pub/Sub; lifespan-integrated; `_handle(event)` dispatcher; per-event session via `session_factory`; `SET app.current_org_id` GUC pre-write; ADR-013 sole-writer + ADR-017 sequencing-locked
- `src/app/models/recon.py`: `AttackSurface` ORM + `SubdomainStatus` enum (V-LC; `uq_scan_subdomain` unique constraint)
- `src/app/routes/projects.py:519` UPSERT precedent (V-ME; `ON CONFLICT DO UPDATE` pattern)

**API side pending (Stage 3 Commit 3 scope):**

- `src/app/services/completions_consumer.py`: `_handle()` routing extension to dispatch `event_type=="attack_surface"` → NEW `_handle_attack_surface(event)` handler with UPSERT logic + RLS GUC `SET` pre-write (~80-150 LoC)
- `tests/services/test_completions_consumer.py`: 5-7 attack_surface scenarios — per-subdomain UPSERT + duplicate-idempotency + RLS isolation + malformed-event-handling + cross-org-rejection (~80-150 LoC)

**Docs side pending (Stage 3 Commit 1 scope):**

- `TOOL-ARCHITECTURE.md` §8 addendum: `EventAttackSurface` engine emission canonical authority (~15-25 LoC)
- `SPEC.md` §7 addendum: `EventAttackSurface` wire shape canonical (JSON schema) + ADR-014 channel reference + ADR-013 sole-writer continuity + Drift #58 root-cause repair note (~15-30 LoC)

### 2.2 Architectural Analog: AttackSurface Consumer Pattern Origin

**Direct analog: `completions_consumer.py` V-MB architecture.** Unlike source-ingestion fix (net-new pattern at engine layer with limited analog), Task 8.3α has STRONG api-side analog: existing `completions_consumer` drains `job_completed` + `partial_findings` events with identical lifecycle pattern (Pub/Sub subscribe + `_handle` dispatch + per-event session + RLS GUC + commit). 8.3α extends this canonical pattern with new event type. Engine-side analog: existing `EventReconCompleted` emission shape provides template for new `EventAttackSurface` emission.

**Implication for D-deviation forecast:** likely LOW (~0-2 drifts at execution) per strong dual-side analog; pattern recognition: source-ingestion fix engine-side novelty drove ~3 drifts; 8.3α extends established patterns on both sides; pre-verification pre-locked Q-chain comprehensively.

### 2.3 No Phase 0 v2 Territory

Per Q-PHASE-0-V2 NO lock at brainstorming chain: pre-verification fully grounded all empirical concerns (`completions_consumer` architecture V-MB + event wire shapes V-MD + UPSERT pattern V-ME + test pattern V-MH + `CompletionsPublisher` V-NB). No execution-time-unknown empirical gaps. Architectural-decision-only territory for Q-chain; no Phase 0 v2 confirmation pass needed.

## 3. Architectural Decisions (Plan-Level Locks)

Brainstorming chain 12-Q-locks architectural decisions locked at design doc `0030319` §1 + §3.1-§3.12. Plan-level refinements captured below.

### 3.1 Y-CONSUMER-LOCATION (α) Extend Completions Pub/Sub Channel

Implementation: shieldscan-api `src/app/services/completions_consumer.py` `_handle()` routing extension; new event type `"attack_surface"` dispatches to `_handle_attack_surface(event)` per design doc §3.1 architectural rationale (ADR-014 compliance + per-scan-stream-consumer-deferred-to-ADR-018).

### 3.2 Y-WIRE-SHAPE (a) New `EventAttackSurface` Event Type

Implementation surface: shieldscan-engine `internal/events/events.go`:

```go
// EventAttackSurface carries rich per-subdomain attack-surface data
// emitted after recon's httpx phase. Persistence-targeted; consumed
// by api completions_consumer to UPSERT AttackSurface rows.
type EventAttackSurface struct {
    EventType      string         `json:"event_type"` // "attack_surface"
    ScanID         string         `json:"scan_id"`
    OrganizationID string         `json:"organization_id"`
    RootDomain     string         `json:"root_domain"`
    Subdomains     []SubdomainRow `json:"subdomains"`
    Timestamp      string         `json:"timestamp"` // RFC3339
}

type SubdomainRow struct {
    URL          string   `json:"url"`
    Status       string   `json:"status"` // "live" / "dead" / "timeout"
    StatusCode   int      `json:"status_code,omitempty"`
    TechStack    []string `json:"tech_stack,omitempty"`
    LastProbedAt string   `json:"last_probed_at,omitempty"` // RFC3339
}
```

~10-15 LoC at `events.go` + supporting consumer-side Pydantic schema in `completions_consumer.py`.

**Plan-level Y-decision Y-EVENT-PUBLISH-PATH:** (a) extend existing `CompletionsPublisher.Publish` to accept either `JobCompletedEvent` or `EventAttackSurface` via interface generalization; (b1) add a second typed method `PublishAttackSurface(ctx, ev EventAttackSurface) error` to existing publisher; (b2) introduce sibling `AttackSurfacePublisher` with identical channel binding; (c) `recon.go` accepts caller-provided publisher dependency directly. Default (b1) per V-NB finding — minimal disruption + same-channel-same-publisher; resolve at Stage 3 Commit 2 C2.2 execution.

**Plan-level Y-decision Y-RECON-PUBLISHER-WIRING:** V-NB confirmed `completionsPub` lives at `processor.go` not at `recon.go`. Recon currently has no `CompletionsPublisher` dependency. Options at C2.3: (a) plumb `*CompletionsPublisher` into `recon.RunRecon` signature OR recon-struct field via `cmd/worker/` wiring; (b) have processor invoke recon then publish at processor layer (loses Q-ENGINE-EMIT-CALLSITE (a) `RunRecon`-internal lock); (c) recon returns rich payload to caller for caller-side publish (same trade-off as (b)). Default (a) preserves Q-ENGINE-EMIT-CALLSITE (a); resolve at C2.3.

### 3.3 Q-ENGINE-EMIT-CALLSITE (a) `RunRecon`-Internal

Implementation: shieldscan-engine `internal/tools/recon/recon.go`:

```go
// After httpx phase completes; before EventReconCompleted publishes
liveHosts := result.LiveHosts // existing internal aggregation
subdomainRows := make([]events.SubdomainRow, 0, len(liveHosts))
for _, host := range liveHosts {
    subdomainRows = append(subdomainRows, events.SubdomainRow{
        URL:          host.URL,
        Status:       statusFromStatusCode(host.StatusCode),
        StatusCode:   host.StatusCode,
        TechStack:    host.Tech,
        LastProbedAt: time.Now().UTC().Format(time.RFC3339),
    })
}
publisher.PublishAttackSurface(ctx, events.EventAttackSurface{
    EventType:      "attack_surface",
    ScanID:         scanID,
    OrganizationID: orgID,
    RootDomain:     domain,
    Subdomains:     subdomainRows,
    Timestamp:      time.Now().UTC().Format(time.RFC3339),
})
// existing EventReconCompleted emission preserved unchanged
```

~30-50 LoC at `recon.go`. Format-adapt per V-NB `CompletionsPublisher` exact pattern; account for Y-RECON-PUBLISHER-WIRING (a) signature plumbing.

### 3.4 Q-EVENT-PAYLOAD-FIELDS — Full ORM Mapping

Wire shape (per §3.2) maps to `AttackSurface` ORM columns: `scan_id` → `scan_id` FK; `organization_id` → TenantMixin RLS; `root_domain` → `root_domain`; `subdomains[i].url` → `full_url`; `subdomains[i].status` → `status` `SubdomainStatus` enum; `subdomains[i].status_code` → `status_code`; `subdomains[i].tech_stack` → `tech_stack` JSONB; `subdomains[i].last_probed_at` → `last_probed_at`. `subdomain` field derived from URL parsing at consumer side.

### 3.5 Q-UPSERT-PATTERN — `ON CONFLICT DO UPDATE`

**API consumer UPSERT logic (`src/app/services/completions_consumer.py` NEW `_handle_attack_surface(event)`):**

```python
async def _handle_attack_surface(self, event: dict[str, Any]) -> None:
    async with self._session_factory() as session:
        await session.execute(
            text("SET LOCAL app.current_org_id = :org"),
            {"org": event["organization_id"]},
        )
        scan_id = UUID(event["scan_id"])
        org_id = UUID(event["organization_id"])
        root_domain = event["root_domain"]
        subdomains = event["subdomains"]

        # Per Y-UPSERT-ATOMICITY (c): atomic-per-event transaction;
        # all subdomains in single commit.
        from sqlalchemy.dialects.postgresql import insert

        stmt_values = []
        for sub in subdomains:
            subdomain_label = urlparse(sub["url"]).hostname
            stmt_values.append({
                "scan_id": scan_id,
                "organization_id": org_id,
                "root_domain": root_domain,
                "subdomain": subdomain_label,
                "full_url": sub["url"],
                "status": SubdomainStatus(sub["status"]),
                "status_code": sub.get("status_code"),
                "tech_stack": sub.get("tech_stack"),
                "last_probed_at": parse_rfc3339(sub.get("last_probed_at")),
            })

        if stmt_values:
            stmt = insert(AttackSurface).values(stmt_values)
            stmt = stmt.on_conflict_do_update(
                index_elements=["scan_id", "subdomain"],
                set_={
                    "full_url": stmt.excluded.full_url,
                    "status": stmt.excluded.status,
                    "status_code": stmt.excluded.status_code,
                    "tech_stack": stmt.excluded.tech_stack,
                    "last_probed_at": stmt.excluded.last_probed_at,
                },
            )
            await session.execute(stmt)
            await session.commit()
```

~50-80 LoC. Format-adapt per V-MB exact session-handling + V-ME UPSERT precedent shape.

**Plan-level Y-decision Y-UPSERT-ATOMICITY:** (a) per-subdomain UPSERT in loop (N statements; simple); (b) bulk `INSERT ... VALUES ON CONFLICT` (single statement; performance-optimal); (c) atomic-per-event transaction (all subdomains in single commit). Default (c); resolve at Stage 3 Commit 3 C3.2 per V-ME UPSERT pattern shape.

### 3.6 Q-LIFECYCLE — `completions_consumer.py` Extension

No new service; extends existing `CompletionsConsumer.start()/stop()` FastAPI-lifespan-managed task per design doc §3.6. No `main.py` changes.

### 3.7 Q-RLS-GUC — Org-Scoped `SET`

`SET LOCAL app.current_org_id = :org` precedes any AttackSurface INSERT inside per-event session per ADR-013 sole-writer + existing `_handle_*` pattern at V-MB.

### 3.8 Q-TEST-PATTERN — Fakeredis + Direct `_handle`

API test pattern per V-MH: fakeredis subscribe + direct `_handle_attack_surface(event)` invocation; 5-7 scenarios enumerated at §3.14.

### 3.9 Q-MULTI-PROCESS-POSTURE (b) UPSERT-Idempotency-By-Construction

`uq_scan_subdomain` UNIQUE + `ON CONFLICT DO UPDATE` makes duplicate-delivery safe by construction. No additional dedup layer. Honest-acknowledgment-inherited posture from `completions_consumer` baseline; ADR-018 consumer-groups remains forward-pinned for once-only semantics.

### 3.10 Q-PHASE-0-V2 NO

Per §2.3 — no Phase 0 v2 territory.

### 3.11 Q-COMMITS — 3-Commit Cross-Repo

See §3.13 for full sequencing.

### 3.12 Q-DOCS-LOCATION (c) Dual TOOL-ARCH §8 + SPEC §7 Addendums — Implementation Site (Stage 3 Commit 1)

**`TOOL-ARCHITECTURE.md` §8 addendum content (~15-25 LoC):**

- Engine emission canonical authority for `EventAttackSurface`
- `RunRecon`-internal callsite specification
- ADR-014 completions Pub/Sub channel reference
- Drift #58 Layer A root-cause repair note

**`SPEC.md` §7 addendum content (~15-30 LoC):**

- `EventAttackSurface` wire shape canonical (JSON schema + field documentation)
- ADR-014 channel routing reference
- ADR-013 sole-writer continuity (api consumer is sole writer of `AttackSurface` rows)
- Drift #58 Layer B root-cause repair note
- Cross-reference to TOOL-ARCH §8 addendum (engine emission side)

**Mirrors source-ingestion fix Stage 3 Commit 1 multi-location-addendum precedent shape (`9d6ab25` TOOL-ARCH §3.2 addendum) extended to dual-location.**

### 3.13 Q-COMMITS (a) 3-Commit Cross-Repo Sequencing (docs → engine → api)

**Commit 1 (docs FIRST; ~30-55 LoC):** `TOOL-ARCHITECTURE.md` §8 + `SPEC.md` §7 dual addendums.

**Commit 2 (engine SECOND; ~65-110 LoC):** `events.go` `EventAttackSurface` + `pubsub.go` publisher extension + `recon.go` `RunRecon`-internal callsite + tests.

**Commit 3 (api THIRD; ~160-300 LoC):** `completions_consumer.py` `_handle` routing + `_handle_attack_surface` handler + tests.

**Cross-reference shape:** Commit 1 references future engine+api hashes via placeholders; Commit 2 references docs Commit 1 hash + api placeholder; Commit 3 references both docs + engine hashes concretely. Mirrors source-ingestion fix + R2 + V10 + revocation Stage 3 trio precedent.

**Sequencing rationale:** engine emission lands SECOND (not last) per Q-COMMITS — api consumer tests can verify against real `EventAttackSurface` payload shape from engine; this is event-emits-before-consumer-test ordering. Different from source-ingestion fix sequencing (docs → api → engine) where api validator + orchestrator-threading needed engine wire field land later to consume.

### 3.14 Engine + API Test Strategy

**Engine unit test:** `internal/tools/recon/recon_test.go` extension OR new test file: mock `CompletionsPublisher`; verify `EventAttackSurface` published with rich payload from `ReconResult.LiveHosts`.

**Engine integration test:** existing recon flow + httptest-served fake completions endpoint OR fakeredis; verify wire-shape correctness.

**API unit tests:** fakeredis + direct `_handle_attack_surface(event)` invocation per V-MH pattern. 5-7 scenarios — per-subdomain UPSERT happy-path + duplicate-event-idempotency (rerun same event, no duplicate rows) + RLS isolation (cross-org rejection) + malformed-event-rejection + empty-subdomains-list-handling + missing-optional-fields (`status_code`, `tech_stack`, `last_probed_at`).

## 4. Stage 3 Sub-Step Breakdown

### Stage 3 Commit 1 — Docs (TOOL-ARCH §8 + SPEC §7 Dual Addendums) (~20-30min)

**C1.1** Locate `TOOL-ARCHITECTURE.md` §8 section; identify existing §8 content + insertion point for AttackSurface Emission Lock addendum. grep §8 + adjacent sections.

**C1.2** Locate `SPEC.md` §7 section; identify existing §7 content + insertion point for `EventAttackSurface` wire-shape addendum. grep §7 + adjacent sections.

**C1.3** Append TOOL-ARCH §8 addendum content per §3.12. ~15-25 LoC.

**C1.4** Append SPEC §7 addendum content per §3.12. ~15-30 LoC.

**C1.5** Pre-commit verification: grep `"AttackSurface Emission Lock\|EventAttackSurface\|Drift #58"` `TOOL-ARCHITECTURE.md` `SPEC.md` (verify addendums landed + existing canonical text preserved); wc -l `TOOL-ARCHITECTURE.md` `SPEC.md` (delta within bound).

**C1.6** Single atomic commit. Cross-references future engine+api hashes via placeholders.

**Total Stage 3 Commit 1 LoC delta:** ~30-55 LoC.

### Stage 3 Commit 2 — Engine (events.go + pubsub.go + recon.go + tests) (~30-60min)

**C2.1** `events.go`: NEW `EventAttackSurface` + `SubdomainRow` structs with JSON tags per §3.2. ~10-15 LoC.

**C2.2** `pubsub.go`: Y-EVENT-PUBLISH-PATH resolution — default (b1) add `PublishAttackSurface(ctx, ev events.EventAttackSurface) error` method on existing `*CompletionsPublisher`; same channel `shieldscan:completions`. ~5-15 LoC.

**C2.3** `recon.go`: Y-RECON-PUBLISHER-WIRING resolution — default (a) plumb `*CompletionsPublisher` into recon dependency; `RunRecon`-internal callsite aggregates `ReconResult.LiveHosts` → publish `EventAttackSurface` BEFORE `EventReconCompleted` per §3.3. ~30-50 LoC.

**C2.4** Test extension: emission unit test (mock publisher captures `EventAttackSurface`) + integration test where applicable. ~20-30 LoC.

**C2.5** Pre-commit verification: `go vet ./...` clean; `go test -race ./internal/events/ ./internal/redis/ ./internal/tools/recon/` green; `go test -race ./...` green (25 packages baseline preserved).

**C2.6** Single atomic commit. Cross-references docs Commit 1 hash + api placeholder.

**Total Stage 3 Commit 2 LoC delta:** ~65-110 LoC.

### Stage 3 Commit 3 — API (completions_consumer.py + tests) (~45-90min)

**C3.1** `services/completions_consumer.py`: `_handle()` routing extension — add `event_type=="attack_surface"` dispatch to `_handle_attack_surface(event)`. ~10-15 LoC.

**C3.2** `services/completions_consumer.py`: NEW `_handle_attack_surface(event)` per §3.5 with UPSERT logic + RLS GUC `SET` + Y-UPSERT-ATOMICITY (c) atomic-per-event transaction. ~50-80 LoC.

**C3.3** Optional schemas extension: Pydantic event schema for type-safe validation per Y-EVENT-PAYLOAD-VALIDATION (c) hybrid default. ~20-40 LoC at `schemas/` OR inline.

**C3.4** `tests/services/test_completions_consumer.py`: 5-7 attack_surface scenarios per §3.14 strategy. ~80-150 LoC.

**C3.5** Pre-commit verification: `pytest tests/services/test_completions_consumer.py` green (existing + 5-7 new tests); `pytest` full suite green (564 baseline preserved per source-ingestion fix Stage 3 C2 reference).

**C3.6** Single atomic commit. Cross-references docs Commit 1 + engine Commit 2 hashes concretely.

**Total Stage 3 Commit 3 LoC delta:** ~160-285 LoC.

### Stage 3 Aggregate LoC Forecast

Total across 3 commits: ~255-450 LoC (docs ~30-55 + engine ~65-110 + api ~160-285). Comparable to source-ingestion fix Stage 3 (~193-358 forecast; +1035 actual due to novel-pattern + framework-drift) + revocation Stage 3 (~88-145) + R2 Stage 3 (~200-313). Calibration update from source-ingestion fix: comprehensive-tests + novel-pattern may push upper to ~400-700 LoC; this task has STRONG dual-side analog (mitigates novel-pattern inflation); realistic landing range ~280-500 LoC.

## 5. D-Deviation Tracking Framework

Per Task 7.6 + ADR-015 + V10 + R2 + revocation + source-ingestion fix D-PLAN tracking precedent.

**Pre-execution drifts catalogued:** Drift #58 (two-layer manifestation; catalogued at T83_PV + T83A_PV pre-verification this session); cumulative count stays at 58 pending Stage 3 execution.

**Expected Stage 3 D-deviation count:** LOW (~0-2 drifts at execution) per §2.2 strong dual-side analog observation. Lower than source-ingestion fix (3 drifts at Stage 3 execution per novel-pattern engine territory) given `completions_consumer.py` + `EventReconCompleted` patterns are well-established analogs. Honest forecast: 0 drifts at docs Commit 1; 0-1 at engine Commit 2 (Y-EVENT-PUBLISH-PATH + Y-RECON-PUBLISHER-WIRING precision at V-NB execution); 0-1 at api Commit 3 (Y-UPSERT-ATOMICITY precision at V-ME existing UPSERT-shape vs new bulk-insert convention).

**Plan-level Y-decisions to resolve at execution:**

- **Y-EVENT-PUBLISH-PATH:** (b1) second typed method on existing `CompletionsPublisher` (default per V-NB + ADR-014 separation) vs (b2) sibling publisher vs (a) interface generalization vs (c) caller-publishes; resolve at Stage 3 Commit 2 C2.2 per V-NB finding
- **Y-RECON-PUBLISHER-WIRING:** (a) plumb publisher into recon dependency (default; preserves Q-ENGINE-EMIT-CALLSITE (a)) vs (b) processor-layer publish vs (c) caller-publishes via returned payload; resolve at Stage 3 Commit 2 C2.3
- **Y-EVENT-PAYLOAD-VALIDATION:** hybrid Pydantic+permissive (c default per V-MB convention) vs strict (a) vs permissive (b); resolve at Stage 3 Commit 3 C3.3 per existing schema convention
- **Y-UPSERT-ATOMICITY:** atomic-per-event transaction (c default; event-as-unit-of-work invariant) vs per-subdomain-loop (a) vs bulk-INSERT (b); resolve at Stage 3 Commit 3 C3.2 per V-ME UPSERT pattern shape

## 6. Out of Scope (per design doc §6 + plan-level refinements)

1. Task 8.3β GET endpoint (next-task; mechanical compressed-lifecycle on populated `AttackSurface` rows)
2. M8.1 recon-invocation production wiring (arc-evolution-pivot; brainstorming forward-pinned fresh session)
3. ADR-018 Streams+consumer-groups migration
4. Per-scan progress-Stream subscription consumers (Y-CONSUMER-LOCATION β; deferred until ADR-018)
5. `EventReconCompleted` enrichment (Y-WIRE-SHAPE b; rejected separation-of-concerns)
6. `AttackSurface` schema migrations
7. Engine recon-pipeline retries / fault-tolerance
8. Findings-table join for `vulnerability_count` aggregation (Task 8.3β concern)
9. Multi-tenant isolation testing beyond existing RLS pattern
10. ADR-013 sole-writer modifications
11. Engine `processor.go` modifications
12. Operational deployment posture changes
13. shieldscan-api modifications outside `services/completions_consumer.py` + `tests/services/`
14. shieldscan-engine modifications outside `events.go` + `pubsub.go` + `recon.go` + tests + wiring required for Y-RECON-PUBLISHER-WIRING
15. shieldscan-docs modifications outside `TOOL-ARCHITECTURE.md` §8 + `SPEC.md` §7 (no SPEC §13 ADR changes; addendums at sub-locations)

## 7. Forward-Pins

**Pre-execution forward-pins (Stage 3 entry):**

1. **Stage 3 trigger phrase:** ***"Resume Task 8.3α — Stage 3 cross-repo implementation"***
2. **Y-EVENT-PUBLISH-PATH decision context preserved:** (b1) second typed method (default) vs alternatives; resolve at C2.2
3. **Y-RECON-PUBLISHER-WIRING decision context preserved:** (a) plumb publisher into recon (default) vs alternatives; resolve at C2.3
4. **Y-EVENT-PAYLOAD-VALIDATION decision context preserved:** hybrid (c default) vs alternatives; resolve at C3.3
5. **Y-UPSERT-ATOMICITY decision context preserved:** atomic-per-event (c default) vs alternatives; resolve at C3.2
6. **Design doc canonical authority:** `0030319` §3 + §4 verbatim drafts

**Post-Stage-3 forward-pins:**

7. **Task 8.3β GET endpoint task** — mechanical compressed-lifecycle on populated `AttackSurface` rows; ready for fresh session post-Stage-3-close
8. **M8.1 scan-executor brainstorming task** — arc-evolution-pivot territory; recon-invocation production wiring; fresh session
9. **ADR-018 Streams+consumer-groups migration task** — multi-process-posture upgrade; Y-CONSUMER-LOCATION (β) trigger
10. **`EventReconCompleted` enrichment task** — Y-WIRE-SHAPE (b) trigger if dual-purpose-payload need surfaces
11. **`AttackSurface` `vulnerability_count` join task** — Task 8.3β endpoint scope
12. **Recon retry/fault-tolerance task** — operational hardening
13. **Per-scan-stream-persistence-consumer migration task** — Y-CONSUMER-LOCATION (β) + ADR-018
14. **M8/M9 transition trigger** — post Task 8.3α + 8.3β + M8.1 close; M8 declarable CLOSED + M9 entry ready

**In-scope forward-pin closures at Stage 3:** Drift #58 two-layer-manifestation operationally settled (Layer A engine wire-shape repair at Commit 2 + Layer B api consumer repair at Commit 3).

## 8. Cross-References

**Engine commits:**

- `a0bff50` (source-ingestion fix Stage 3 C3; latest engine state; `recon.go` callsite precedent for `EventReconCompleted` preservation pattern)
- `3ccf5b8` (R2 Stage 3 C3; `events.go` field addition precedent)
- `ad7cc94` (V10 Stage 3 C2; engine pattern)
- Source authorities: `internal/events/events.go` (C2.1 modification target) + `internal/redis/pubsub.go` (C2.2 modification target; `CompletionsPublisher` V-NB) + `internal/tools/recon/recon.go` (C2.3 modification target) + `recon_test.go` (C2.4 extension)

**Docs commits:**

- `0030319` (Task 8.3α Stage 1 design doc; this plan's canonical authority)
- `dacf5bb` (Task 8.2 retirement; latest docs state pre-this-plan)
- `ac82d48` (source-ingestion fix Stage 4 P5.A close)
- `9d6ab25` (source-ingestion fix Stage 3 C1; multi-addendum precedent for TOOL-ARCH §3.2; analog for dual-addendum at §8 + SPEC §7)
- `90fc933` (source-ingestion fix Stage 1 design doc; structural precedent + Drift #54 catch-class analog)
- `04f44a9` (source-ingestion fix Stage 2 plan; closest plan precedent at 426 LoC)
- `0e55a4f` (revocation design doc)
- `fdad021` (revocation plan precedent at 338 LoC)
- `b25e9ba` (R2 design doc)
- `721f788` (R2 plan precedent at 357 LoC)

**API commits:**

- `8dbcbab` (source-ingestion fix Stage 3 C2; latest api state; orchestrator + validator pattern precedent)
- Source authorities: `src/app/services/completions_consumer.py` (C3.1 + C3.2 modification targets; V-MB canonical analog) + `tests/services/test_completions_consumer.py` (C3.4 extension target) + `src/app/models/recon.py` (V-LC `AttackSurface` ORM; unchanged) + `src/app/routes/projects.py:519` (V-ME UPSERT precedent; reference)

**SPEC sections:**

- `TOOL-ARCHITECTURE.md` §8 (engine emission canonical authority target)
- `SPEC.md` §6 line 633-654 (Task 8.3β endpoint wire shape; future consumer of populated rows)
- `SPEC.md` §7 (API/wire canonical authority target)
- §13 ADR-013 (sole-writer; api consumer remains sole `AttackSurface` writer)
- §13 ADR-014 (mixed-primitives Pub/Sub completions = right channel; per-scan Streams reserved)
- §13 ADR-017 (sequencing — not invoked for `AttackSurface`)
- §13 ADR-018 forward-pinned (Streams+consumer-groups)

**Phase 0 v2 artifacts:** None (Q-PHASE-0-V2 NO; pre-verification fully grounded).

**Pre-verification artifacts:** T83_PV + T83A_PV surface reports (this session).

**Cumulative drift count:** 58 catches at execution time (Drift #58 catalogued at T83_PV + T83A_PV; clean Stage 2 plan-landing entry).

**Milestone-completion-constraint context:** locked this session; Task 8.3α + 8.3β multi-stage close required before M8 declarable CLOSED + M9 entry; M8.1 brainstorming forward-pinned fresh session as parallel arc-evolution-pivot territory.
