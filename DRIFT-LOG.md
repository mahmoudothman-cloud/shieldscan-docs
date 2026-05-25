# ShieldScan — Version Drift Log

**Purpose:** Track versions that have drifted past VERSIONS.md pins, so deliberate upgrade sessions can be scheduled.

**Rule:** Per CONSTITUTION.md §11.1 Commandment #1, pinned versions win. Entries here are *candidates* for a future upgrade session, not silent upgrades.

---

## Open drift (as of 2026-04-20)

Captured from `scripts/verify-versions.sh` run at project start. Re-run before every upgrade session to refresh.

### Python libraries (shieldscan-api)

| Library | Pinned | Latest observed | Classification | Target milestone to address |
|---|---|---|---|---|
| fastapi | 0.135.0 | 0.136.0 | patch — caret range handles at `poetry install` | No action; verify at M11 |
| sqlalchemy | 2.0.36 | 2.0.49 | patch — caret range handles | No action; verify at M11 |
| pydantic | 2.9.0 | 2.13.3 | minor drift, 4 minor versions | **Review before M9** (AI pipeline uses pydantic schemas heavily) |
| **anthropic** | 0.40.0 | 0.96.0 | **MAJOR SDK rewrite**, 56 minor versions | **Dedicated upgrade session required before M9** — breaking API changes expected |

### Security scanning tools (shieldscan-engine)

Per VERSIONS.md §4.1, security tools should "always upgrade to latest stable." These are not blockers for M1–M4 (API + scaffolding) but must be addressed before M5 (Go Worker) and M6 (Native Tool Runners).

| Tool | Pinned | Latest observed | Target milestone |
|---|---|---|---|
| Nuclei | v3.7.1 | v3.8.0 | M6 |
| Subfinder | v2.6.7 | v2.13.0 | M6 (used in recon, M8) |
| httpx | v1.6.10 | v1.9.0 | M6 / M8 |
| Gitleaks | v8.21.2 | v8.30.1 | M6 |
| Trivy | v0.58.0 | v0.70.0 | M7 (persistent Docker service) |
| ZAP | 2.16.0 | v2.17.0 | M7 |
| sslyze | 6.1.0 | 6.3.1 | M6 |
| semgrep | 1.95.0 | 1.160.0 | M6 — large jump, review changelog |
| checkov | 3.2.340 | 3.2.524 | M6 — patch range |
| wapiti3 | 3.2.4 | 3.2.10 | M6 — patch range |

### verify-versions.sh script issues (resolved)

| Issue | Status |
|---|---|
| Python parser returned oldest 3.13 release instead of newest; no pre-release filter | **Fixed** in commit `9929bfe` (shieldscan-api) |
| PostgreSQL parser returned minor integer from oldest major (v6.3) instead of supported 18.x | **Fixed** in commit `9929bfe` (shieldscan-api) |
| Trivy GitHub API returned "unknown" on first run | Self-resolved on re-run (intermittent rate limit) |

---

## Upgrade session protocol

When addressing an entry:

1. Re-run `scripts/verify-versions.sh` to confirm drift still exists.
2. Read the changelog for the pinned→latest range.
3. Update `VERSIONS.md` with the new pin + date.
4. Update `pyproject.toml` / `go.mod` / `provision-worker.sh` per VERSIONS.md §5.
5. Run full test suite.
6. Remove the entry from this log (or move to "Closed" section with commit SHA).

---

## Technical decisions (non-drift)

### 2026-04-20 — `continuous_clean_days` uses ORM default only

`Organization.continuous_clean_days` has a SQLAlchemy `default=0` (applied by the ORM on INSERT) but **no** PostgreSQL `server_default`. All inserts go through SQLAlchemy today. If raw SQL inserts become a pattern (data migrations, admin scripts, direct DB writes from another service), revisit and add `server_default=text("0")` to avoid a `NOT NULL violation`.

### 2026-04-20 — JSONB over JSON for all PostgreSQL JSON columns

**Convention:** use `JSONB` (not `JSON`) for every PostgreSQL JSON column project-wide. **Reason:** indexable (GIN), binary storage, identical ORM usage, no downside on PG16+. **How to apply:** when adding any JSON-typed column, import `from sqlalchemy.dialects.postgresql import JSONB` and use `mapped_column(JSONB, ...)`. Already applied to `scans.config` and `attack_surface.tech_stack`; applies forward to all new JSON columns. The IMPLEMENTATION-PLAN snippets that say `JSON` (e.g., Task 1.4 §Scan `config`) should be read as JSONB.

### 2026-04-20 — pytest-asyncio 0.24 forces function-scoped test engines

`tests/conftest.py` recreates the SQLAlchemy async engine + schema per test. Reason: pytest-asyncio **0.24** supports `asyncio_default_fixture_loop_scope` but *not* `asyncio_default_test_loop_scope`, so a session-scoped engine's asyncpg connection gets "attached to a different loop" when touched by a function-scoped test. **How to apply:** when pytest-asyncio bumps to ≥0.25 (currently pinned `^0.24.0` in `pyproject.toml`), re-evaluate session-scoped engines + savepoint rollback — 13 tests in 0.62s is fine now, will matter at 500+ tests.

### 2026-04-23 — Task 3.2: idempotent re-verify emits a fresh audit row

**Decision pin (no code change).** A successful `POST /v1/orgs/{org_id}/projects/{id}/verify` on a project where `domain_verified` was already `True` emits a fresh `PROJECT_DOMAIN_VERIFIED` audit row regardless. The DB-side state transition is a no-op (column was True, stays True), but the audit log gets a new entry on every attempt.

**Rationale:** audit logs track *events*, not *state*. A re-verify action IS an event — the user took an action, the system performed a check, the check succeeded. Suppressing the audit because the end state is identical would lose information:

- *When* did the customer re-verify? (Sequence of timestamps tells us.)
- *How often* are they re-verifying? (A sustained re-verify cadence often correlates with troubleshooting DNS issues elsewhere — operationally useful intel.)
- *Did they verify, lose access, then re-verify successfully?* (The audit trail makes this visible; deduplication would mask it.)

Consistent with the `PROJECT_DOMAIN_VERIFICATION_FAILED` discipline, which fires on every failure attempt regardless of prior state — symmetric treatment of attempts on both success and failure paths.

**For future engineers:** if you're tempted to add `if not was_already_verified: audit(...)` to the success branch — don't. Audit-row dedup adds complexity for questionable benefit (the only "noise" suppressed is intentional event records). The pattern as shipped is the desired behavior.

### 2026-04-30 — Task 4.4: SSE generator extracted to module-level function for testability

**Implementation pivot.** The SSE handler's event generator was originally inline as a closure inside `stream_scan_progress`. End-to-end SSE testing through `httpx.AsyncClient.stream()` + `ASGITransport` + `sse-starlette.EventSourceResponse` hangs on `aiter_lines()` — known integration weakness where streaming chunks buffer indefinitely until the response generator completes (which an SSE generator never does).

**Fix:** extract `progress_generator(redis, scan_id_str, last_event_id)` as a module-level async function. The route handler wraps it with instrumentation (connect/disconnect INFO logs) and `EventSourceResponse(..., ping=15)`. Tests drive `progress_generator` directly via `async for` iteration on fakeredis; HTTP-level tests cover only the connect-time auth/404 paths.

**Test split:**
- 4 HTTP-level tests (status codes only — no body iteration): unauthenticated 401, cross-tenant 404, nonexistent 404, `_parse_last_event_id` parser pin.
- 8 generator-level tests (full event-flow contract): live delivery, multiple events in order, concurrent readers, `Last-Event-ID` skip-history, replay-since-id, replay-then-live seamless, format compliance, terminal-scan replay.

**Why this is the right shape:** the SSE-over-test-client hang is a transport-level integration artifact, not a behavior bug. The behavior under test is the event-flow logic in `progress_generator`. Testing it directly is more accurate AND more reliable than fighting the transport. Future SSE-equivalent endpoints (M5+ live worker logs, etc.) should follow the same extract-and-test-directly pattern.

**Pinned by:** all 12 tests in `tests/routes/test_scan_progress.py`.

### 2026-04-30 — Task 4.4: DB session held for SSE connection lifetime (Option B deferred)

**Decision (deferred Option B).** Initial scope-proposal called for closing the request-scoped DB session after the scan-existence lookup, freeing the connection-pool slot for the streaming phase. **Implementation reverted** to holding the session open for the connection's lifetime via FastAPI's standard `Depends(get_db)` teardown.

**Why reverted:** explicit `await db.close()` in the handler interferes with the test fixture's persistent-connection model. The test's `db_session` shares one `AsyncConnection` with the route handler; closing it mid-test orphans subsequent assertions on `db_session`. The fix would be to refactor the test fixture to use independent per-request connections (closer to production), which is non-trivial and out of scope for 4.4.

**Production impact:** at MVP viewer counts (<20 concurrent SSE viewers per uvicorn worker), connection-pool exhaustion is not a concern. At larger scale, the right fix is a per-request session checkout pattern that the SSE handler explicitly releases — that's an ops-milestone refactor when the test fixture's persistent-connection assumption is also addressed.

**Trigger for action (concrete):** sustained **15+ concurrent SSE connections per uvicorn worker** observed in production metrics (Sentry / Prometheus pool-saturation gauges), **OR** addition of a new long-lived streaming endpoint (e.g. M11 WebSocket dashboard, M5+ live worker logs) that compounds pool pressure on the same uvicorn process. Either condition flips this from "acceptable MVP shape" to "ship Option B + test fixture refactor in the next ops touch."

**Pinned by inline comment** at the SSE handler's "Step 3" block, explicitly calling out the deferred optimization + the test-fixture interaction. Future engineer reading the code will see why `await db.close()` is NOT called.

### 2026-04-30 — Task 4.4: SSE block_ms = 200ms (vs proposal's 10s)

**Tuning refinement.** The scope proposal pinned `_SSE_BLOCK_MS = 10_000` (10s). Implementation reduced to `200ms` because:

- 10s blocks made test-side assertions wait up to 10s for disconnect-detection cycles, blowing test timeouts.
- 200ms = 5 iterations/sec. Loop body is empty when no events arrive — CPU cost negligible.
- Production responsiveness improves: client disconnect detection at 200ms granularity instead of 10s.
- Heartbeat (15s) still protects against proxy idle-timeouts; block_ms is the inner-loop iteration interval, not the heartbeat.

**Trade-off:** ~4.5x more redis round-trips per second per active SSE connection vs the 10s proposal. At MVP viewer counts (<20), redis can handle thousands of XREAD/sec trivially. Revisit only if profiling shows fakeredis-style XREAD looping is hot in production (it won't be).

### 2026-04-30 — Task 4.4: Pub/Sub → Streams reversal in plan §4.4

**Plan deviation.** `IMPLEMENTATION-PLAN.md` §4.4 sketched a Pub/Sub-based progress streaming design (`redis_client.pubsub() + pubsub.listen()`). ADR-014 (Task 4.1) reversed that decision; Task 4.4 implementation uses the `ProgressSubscriber` wrapper from `app.services.scan_queue` (Streams via XADD/XRANGE/XREAD BLOCK).

**Plan §4.4 code snippet should be ignored.** The `routes/scan_progress.py` filename in plan §4.4 was also superseded — handler lives in `routes/scans.py` on the `scan_router` (Task 4.5 router-split precedent), colocated with cancel + compare.

**SPEC §7.2** was patched in ADR-014's docs commit (Channel → Stream wording).

### 2026-04-30 — Task 4.4: no audit on SSE connect; INFO logs instead

**Pin.** `GET /scans/:id/progress` is read-only. Following the pattern that M10 GET endpoints will inherit: read-only operations don't audit. Every dashboard refresh would emit a `SCAN_PROGRESS_VIEWED` audit row → noise that drowns security-relevant events.

**Operational visibility via INFO logs:**

```python
logger.info("sse_progress_connected scan_id=%s identity_kind=%s last_event_id=%s", ...)
# ... (connection lifetime)
logger.info("sse_progress_disconnected scan_id=%s duration_seconds=%.1f events_sent=%d", ...)
```

Greppable from app logs for "is SSE being used? are clients reconnecting often? are connections being held too long?" without polluting `audit_logs`. Audit log stays clean for security/compliance focus.

### 2026-04-30 — Task 4.4: Last-Event-ID graceful degradation (malformed → "$")

**Decision.** `_parse_last_event_id` accepts:
- Empty/None → `"$"` (live-only)
- `"-"` → replay all from beginning (test/debug clients)
- Well-formed `<ms>-<seq>` Redis Stream id → echoed (replay since)
- **Malformed → degrade gracefully to `"$"`**

**Why graceful degradation:** rejecting malformed ids with 400 is correct HTTP semantic but breaks reconnect-recovery for clients with stale state. A browser that crashed mid-stream may have a corrupted `Last-Event-ID` cookie; we'd rather they get the live-events-only fallback than a hard error.

**Pinned by** `test_parse_last_event_id_handles_all_cases` — covers empty, whitespace-only, malformed, well-formed, and the special `-` value.

### M4 milestone-boundary close (2026-04-30)

**M4 — Scan Orchestration & Redis Contracts — closes after Task 4.4.**

| Item | Detail |
|---|---|
| Tasks shipped | 4.1 Redis primitives · 4.2 Orchestrator + completions consumer · 4.3 POST /scans · 4.5 DELETE cancel · 4.6 POST compare · 4.4 SSE progress |
| Task ordering followed | 4.1 → 4.2 → 4.3 → 4.5 → 4.6 → 4.4 (deferred 4.4 to last per its known SSE testing complexity) |
| API commits | 6 (one per task) |
| Docs commits | 7 (one per task + the ADR-013/014 inline updates) |
| Test count | 416 (M3 close) → 506 (M4 close) — +90 total (+82 task tests + 8 module-reachability auto-pickups) |
| Per-task test growth | 4.1: 12 · 4.2: 14 · 4.3: 19 · 4.5: 12 · 4.6: 13 · 4.4: 12 = 82 task tests; +8 module-reachability auto-pickups → +90 total |
| Migrations | 0 (M4 is read-mostly + Redis primitives; no schema changes) |
| ADRs added | ADR-013 (Python sole writer for scan state) · ADR-014 (Streams over Pub/Sub for progress) |
| ADRs reserved | ~~ADR-015 (decrypted credentials in Redis) — defer until enabled M5+~~ → **LANDED 2026-05-21** at SPEC §13 per shieldscan-docs commit `9a57865`; cross-repo enablement complete per shieldscan-api `742faed` + shieldscan-engine `b48fef8`. Engine-side `DRIFT-LOG.md` carries full Drifts #40-#44 catalog. |
| DEVELOPMENT-PATTERNS.md sections added | +2 (M4 delta) — `session_factory` DI for long-lived background tasks (4.2) · API-key audit attribution (4.6, triple-pin promotion). **Total now 3** including section 1 (M3 select_fresh helper). |
| §6.2 endpoints status | 4 of 9 scans-domain endpoints shipped in M4 (POST create, DELETE cancel, GET progress SSE, POST compare). The other 5 (GET list, GET single, GET jobs, GET jobs/:jid, GET attack-surface) **deferred to M10** read-side cluster per Task 4.1's batched DRIFT-LOG decision. _(Counts reflect Checkpoint 2 §6.2 recount: scans subsection 7→9.)_ |
| Carry-forwards to M5+ | Worker-side cancel consumption (4.5) · "Ghost queued" retry janitor (4.2 commit-then-dispatch) · ADR-015 (auth-block decryption in Redis transit) · Stream-key cleanup TTL (4.1 ops carry-forward) |
| Carry-forwards to OPS milestone | Per-request DB session checkout for SSE (4.4 deferred Option B) · SSE per-org/per-scan connection limits · Server-side SSE timeout · HMAC-signed completion events |
| Carry-forwards to M10 | Pagination for read-side endpoints (referenced from 4.6's truncation cap) · GET /scans + GET /scans/:id/jobs |
| Patterns established | Commit-then-dispatch for visible-failure-preferred semantics · Per-task scope-proposal A-H pattern with architectural-commitments section · Triple-pin → DEVELOPMENT-PATTERNS.md promotion criterion · Cancel-wins at scan-level, completion-wins at job-level (M5+) |

**M4 health:** clean execution. The SSE testing complexity surfaced exactly where the M4 landscape pass predicted (4.4 deserved a clean session due to test infrastructure) — generator-extraction pivot delivered the right testability with no behavior compromise.

**Ready for M5.** Worker-side foundation in `shieldscan-engine` Go repo. ADR-013 forcing-function: Go workers must NOT have PostgreSQL credentials in their config (M5 task 5.6).

### 2026-04-30 — Checkpoint 2 (M4 → M5 transition consolidation): SPEC §6.2 endpoint inventory full recount

**Recount executed.** SPEC §6.2 endpoint inventory underwent a full subsection-by-subsection recount during Checkpoint 2. **Total Phase 1 endpoints corrected from 55 to 68 (+13 net).** Six subsection headers updated:

| Subsection | Header before | Header after | Δ |
|---|---:|---:|---:|
| Auth & Users | 8 | 10 | +2 |
| Organizations & Teams | 6 | 7 | +1 |
| Projects | 7 | 8 | +1 |
| Scans | 7 | 9 | +2 |
| Vulnerabilities | 6 | 7 | +1 |
| Integrations | 5 | 6 | +1 |

API Keys (3), Mobile Scanning (1), Reports (5), Compliance (3), Billing (6), Tool Health (1), Health & Meta (2), and Marketplace Phase 2 (4) verified correct.

**Drift anatomy (two patterns):**

1. **"New" annotations not folded into headers.** Auth & Users gained `verify-email` + `resend-verification` (each annotated `← new (v3 gap #9)`); Mobile Scanning gained an upload endpoint (`(1 new)`); Tool Health was added wholesale (`(1 new)`); Scans gained `attack-surface` and `compare` (`← NEW`). The annotations were added but the parenthetical subsection counts were never updated.
2. **Silent additions over time.** Organizations gained PATCH-member; Projects gained `/stats`; Vulnerabilities gained `/history`; Integrations gained `/webhooks`. No annotations and no count updates — pure drift.

**Forcing function added** at the head of §6.2 to prevent recurrence: "_Subsection headers carry endpoint counts. When adding endpoints, update both the line count and the subsection header. Total Phase 1 count must equal sum of subsection counts. Last full recount: 2026-04-30 (Checkpoint 2, M4→M5 transition)._" The "last full recount" date provides a hygiene marker; future engineers see the comment when adding endpoints, and the recount discipline becomes part of the endpoint-add review.

**Grep sweep** for "55 endpoint" references across `shieldscan-docs/` returned only the two SPECIFICATION.md hits (§6.2 header line + total-line). No external/customer-facing materials reference the old 55 figure.

**Historical entries unchanged.** DRIFT-LOG entries that quote the old subsection counts (lines 175, 267, 494–496, 636) are historical records and remain as written; they document state-at-time, not current state.

### 2026-05-03 — M6-close-followup Phase 1: ADR-024 lands (M6's 3rd ADR; canonical schema-extension artifact)

**ADR-024 (RawFinding schema extension — References, Tags, CVSSVector, AdditionalCWEs) accepted at M6-close-followup.** This is the third ADR in the M6 corpus (after ADR-022 recon-as-pre-scan-helpers and ADR-023 NativeRunner OutputFile mode) and the third consecutive invocation of the asymmetric-cost meta-principle. The principle is now project corpus norm: **architectural commitments are made when the alternative is operationally worse, not when a generic threshold is met.**

**Brainstorming preceded the ADR.** A structured brainstorming session locked six decisions including the moderate scope (4 fields over conservative 2 / comprehensive 6) and the load-bearing M9 §8.2 multi-CWE forward-pin. Design doc archived at `plans/2026-05-03-spec73-schema-extension-design.md` for the per-tool retrofit checklist and scope-rationale audit trail. ADR-024 is the load-bearing architectural artifact; the design doc is the supporting brainstorming output.

**Three subsection adjustments to the design-doc-draft ADR text** were folded into the final ADR-024 in this commit per the user's locked Path A direction:

1. **Python ingest scope subsection** documents the Phase 0 verification deviation: the Python ingest path (Pydantic schema + `CompletionsConsumer.handle_findings()` insert path) the design doc assumed does NOT exist; ADR-024 adopts Path A (extend SQLAlchemy model + Alembic migration in Phase 2; defer Pydantic + ingest to a separate findings-ingest task).
2. **Consequences/Negative** lists "Python ingest deferred" as a known intermediate state with a SPEC §7.3 implementation-status callout pointing future readers to the rationale.
3. **Triggers to revisit** adds "Findings-ingest task lands" as the explicit trigger to retrofit ingest tests for the four ADR-024 fields and remove the implementation-status callout.

The Path A reasoning is preserved in ADR-024 itself (Path B and Path C explicitly rejected with rationale) so future engineers reading the corpus a year+ later understand both the chosen path and the rejected alternatives.

### 2026-05-03 — M6-close-followup Phase 1: SPEC §7.3 explicit per-field listing + Go-struct cross-reference

**SPEC §7.3 updated** with an explicit per-field listing for the `findings[]` array entries (RawFinding wire shape). Replaces the prior literal placeholder (`"...": "..."`) with a 30-row table covering identity / classification / 4 evidence groups (web/source/mobile/SSL) / categorical (ADR-024) / metadata.

**Canonical source of truth pinned.** SPEC §7.3 explicitly identifies the Go struct `events.RawFinding` (in `shieldscan-engine/internal/events/events.go`) and its Python mirror `app.models.raw_findings.RawFinding` (in `shieldscan-api`) as the canonical sources; SPEC §7.3 is the cross-repo agreement doc per ADR-017's "Schema versioning of `RawFinding`" follow-up. ADR-024 establishes the workflow for synchronized cross-repo extensions.

**Strict-schema discipline cross-referenced.** SPEC §7.3 update notes that both sides enforce strict-schema validation (Go: `DisallowUnknownFields`; Python: `extra="forbid"` per ADR-017's M4 Pydantic discipline transfer). Adding a new field requires synchronized extension on both sides per ADR-024's coordination workflow — discoverable at the SPEC §7.3 reading point, not buried in an ADR.

**Implementation-status callout** below the per-field table acknowledges the Python ingest deferral honestly: as of M6-close-followup the SQLAlchemy model has all columns required to persist the wire shape (post-Phase 2), but `CompletionsConsumer` does not yet insert `findings[]` rows. Pointing readers to ADR-024's "Triggers to revisit" closes the loop.

### 2026-05-03 — M6-close-followup Phase 0: Path A adoption (Python ingest deferred; columns-ready posture)

**Phase 0 verification surfaced a load-bearing deviation from the design doc.** The design doc assumed the Python ingest path (Pydantic schema + `CompletionsConsumer.handle_findings()` insert path) existed; verification confirmed it does NOT:

- ✅ `app.models.raw_findings.RawFinding` SQLAlchemy model exists (141 lines).
- ❌ `app.schemas.raw_findings` Pydantic schema does NOT exist (the `app/schemas/` directory has 8 files: api_keys, auth, credentials, errors, mobile, projects, scan_compare, scans — no raw_findings).
- ❌ `CompletionsConsumer.handle_event` does NOT insert RawFinding rows (the 287-line consumer handles `ScanJob.status` + `ScanJob.finding_count` updates only; never references `RawFinding` or the `findings` array).

**Path A adopted (over Path B fold-ingest-in and Path C defer-entire-task).** The user's locked decision: extend the SQLAlchemy model + Alembic migration in Phase 2 (this commit's sibling); defer the Pydantic schema + ingest path to a separate findings-ingest task (likely M4-completion or pulled forward from M9). The columns-ready posture is honest about state — the Engine ships emissions for consumers that don't yet exist (the same architectural pattern as M5/M6 wire-format work that preceded ADR-024).

**Why Path A wins (recorded for future readers):**
1. Honest about state; columns-ready posture is architecturally sound (Engine ships things consumers don't yet exist for — same pattern as M5/M6 wire-format work).
2. Path B (fold ingest into this task) conflates SPEC §7.3 followup with a findings-ingest M4-completion deliverable; scope discipline matters, and the M4-completion work is cross-cutting (RLS via `app.current_org_id` + transaction discipline + bulk insert + idempotency).
3. Path C (defer entire task until ingest exists) wastes real value: the Engine retrofit + Docs commit deliver immediate value even without an ingest path (32% of M6 folds rescued, M9 §8.2 forward-pin documented earlier in the corpus).

**Task shape post-Path-A:** ~4.5–5h (down from ~6.5–7h). Phase 2 reduced from ~2.5h to ~30–45 min (SQLAlchemy + Alembic + extending existing `tests/models/test_findings.py`; no Pydantic, no consumer changes, no ingest fixtures). Phase 3 unchanged (~3h Engine work).

### 2026-05-03 — M6-close-followup Phase 0: design-doc inaccuracy honestly acknowledged

**The original design doc was wrong about Python ingest state.** The doc was authored before the Phase 0 verification pass and assumed (a) a Pydantic schema exists at `app.schemas.raw_findings`, and (b) `CompletionsConsumer` ingests `findings[]` rows. Neither assumption holds in the current `shieldscan-api` codebase.

This entry exists to make the inaccuracy discoverable in the project corpus rather than silently corrected. The same forcing-function discipline that catches code drift catches design-doc drift: surfacing the deviation BEFORE Phase 1 implementation prevents the deviation from compounding into Phase 2 / Phase 3 implementation churn.

**Pattern reinforced (3rd self-catch instance in M6):**
- M6.3 — empirical re-eval reversed inline-tempfile lean (httpx stdin pipe is correct shape).
- M6.6 — empirical re-eval surfaced Nikto XML support (sidestepped fragile text parser).
- M6.7 — Wapiti `-o /dev/stdout` corruption empirically verified, motivated ADR-023.
- **M6-close-followup Phase 0 — design-doc Python-ingest assumption empirically refuted before Phase 1.**

The pre-prep / Phase 0 verification habit is now load-bearing project corpus discipline. Future task scope proposals should explicitly include verification items for any cross-repo concern that the proposal's lean depends on.

### 2026-05-01 — Task 5.1 Commit 2: engine repo bootstrapped, all forcing functions wired

**Engine bootstrap shipped under TDD discipline.** Following Commit 1 (docs sweep, `95d04fe`), the user installed Go 1.26.2 + golangci-lint v2.11.4 between sessions; this commit bootstraps `shieldscan-engine/` with all forcing-function infrastructure for ADR-013/016/017/018/021.

**16 tests pass** at engine bootstrap (4 buildguard + 4 config + 8 events). All race-clean. golangci-lint v2 reports 0 issues. Worker binary builds clean.

**Two version adjustments surfaced during bootstrap:**
- **golangci-lint v2.11.4** (not v1.62.0 as referenced in scope proposal). v2.x is current upstream stable; v1.62.0 would be a stale pin. v2 has breaking config-schema changes from v1 (the `version: "2"` directive + restructured linter blocks). Pinned in VERSIONS.md §2.4 with a dedicated lint-toolchain row. Engine `.golangci.yml` authored against v2 schema. See engine `DRIFT-LOG.md` for full rationale.
- **aws-sdk-go-v2 v1.32.0 → v1.32.2** (patch bump). aws-sdk-go-v2/service/s3 v1.66.0 has a hard transitive requirement on aws-sdk-go-v2 v1.32.2; resolver refused v1.32.0. Patch bump within the same minor — auto-upgrade per VERSIONS.md §4.4. VERSIONS.md §2.4 updated.

**Forcing-function self-catch.** Linter caught `exec.Command` (instead of `exec.CommandContext`) in the buildguard tests on the first lint pass. Refactored to `exec.CommandContext(t.Context(), ...)` before commit. ADR-021 working as intended — the noctx linter caught a violation in the very test code that pins ADR-021's forcing function. Pattern validation: the discipline is enforceable, not aspirational.

**Engine repo structure** matches the M5 landscape pass §10 proposal:
```
shieldscan-engine/
├── cmd/worker/main.go
├── internal/{buildguard,config,events}/
├── testdata/README.md
├── .github/workflows/engine.yml
├── .golangci.yml
├── README.md, CLAUDE.md, DRIFT-LOG.md
├── .env.example, .gitignore
└── go.mod, go.sum
```

Future packages (lazy creation in 5.2-5.6): `internal/tools/`, `internal/redis/{stream,pubsub}.go`, `internal/worker/`, `deploy/docker-compose.services.yml`. Engine-side `DEVELOPMENT-PATTERNS.md` deferred per triple-pin precedent.

**Two-repo docs split (Q3 Option B) implemented:**
- Engine `CLAUDE.md` — delta from parent, ~150 lines, with cross-reference block, ctx-discipline gotchas, build commands, PR checklist subsuming CONTRIBUTING.md (Q4 decision).
- Engine `DRIFT-LOG.md` — engine-only decisions, newest-first.
- Cross-cutting docs (CONSTITUTION/SPEC/VERSIONS/PLAN/TOOL-ARCH/OPS) remain in `shieldscan-docs/`.

**M5 milestone-boundary opens** with this commit. Tasks 5.2-5.6 follow the M4 cadence at smaller scope; M5 close estimated at 1.5-2.5 weeks per the landscape pass.

### 2026-05-01 — Task 5.1 docs sweep (Commit 1 of 2): four ADRs land + SPEC patches

**Architectural foundation for M5.** Task 5.1's docs commit lands the four ADRs (016/017/018/021) drafted during the M4→M5 transition Checkpoint 3 landscape pass + scope-proposal review. SPEC §13 grows from ADR-014 → ADR-021 with intentional 019/020 gap (see numbering note below).

**Refinements folded into ADR text (vs the scope-proposal drafts):**
- **ADR-017 Refinement 1.** Accumulator failure-mode promoted from "open follow-up" to **explicit Consequence** — Python CompletionsConsumer's in-memory accumulator for sequenced events is a load-bearing M5+M6 concern, not a deferred concern. Recovery path: M5+ ghost-queued janitor (Task 4.2 carry-forward). Loss-rate threshold for triggering Option-C migration: **any production occurrence** of "ScanJob ghost-queued due to mid-sequence crash" — once is the trigger, not sustained. The "once is the trigger" framing changes operational posture.
- **ADR-021 Refinement 2.** Rule 3 carves out **legitimate `context.Background()` locations**: `main()`, top-level test functions, AND worker-lifetime services that intentionally outlive request-scoped ctxs (heartbeat, idempotency-reaper). These derive worker-root ctx in `main()` and pass it down — they do NOT construct `context.Background()` at the goroutine spawn site. Without this precision, Task 5.6's heartbeat logic would be misread as Rule 3 violation.

**SPEC patches in this commit:**
- §3.3 Tech Stack line: drop "asynq" from Scan Engine row (ADR-016).
- §7.3 `job_completed` schema: add `findings` array + `event_seq` object with full sequencing semantics documented inline (ADR-017).
- §13: append ADR-016, ADR-017, ADR-018, ADR-021 in numerical order. ADR-numbering note inline: §13 jumps from ADR-018 to ADR-021; ADR-019 (cancel Pub/Sub confirmation) and ADR-020 (worker concurrency model) numbers are reserved for promotion-on-trigger from DRIFT-LOG entries — not missing.

**Plan patches in this commit:**
- IMPLEMENTATION-PLAN.md preamble line 9: drop "asynq" from stack list. Plan §5.1 lines 1510-1525 stale go.mod literal (go 1.22, asynq v0.24.1) **left as written** per state-at-time discipline; Task 5.1 implementation derives go.mod from VERSIONS.md per CLAUDE.md hierarchy.

**VERSIONS.md patch in this commit (H.5 reversal — see separate entry below).**

**Commit 2 (engine bootstrap) deferred to next session.** Go toolchain not installed in current session. Per CLAUDE.md Rule 1 (TDD mandatory: failing test → confirm fail → implement → confirm pass → commit), forcing-function tests cannot be honestly committed without compile + test verification. Honest engineering judgment over "ship and hope" — Task 4.4 SSE precedent applies. User installs Go 1.26.2 + golangci-lint v1.62.0 between sessions; Commit 2 lands next session with full TDD per scope-proposal §C.2 substep list.

**Forcing-function commitments pinned in SPEC §13** (carry into Commit 2 for enforcement):
1. ADR-013: `cmd/worker/...` build target excludes `lib/pq` / `jackc/pgx` / `database/sql`. Buildguard test `TestWorkerBinary_DoesNotImportPostgresDriver`.
2. ADR-016: `go.mod` excludes `hibiken/asynq`. Buildguard test `TestGoMod_ExcludesAsynq`.
3. ADR-017: `internal/events/` package owns `MaxFindingsPerEvent = 1000` constant + `EventSeq` struct. Tests verify constant value + DisallowUnknownFields invariant.
4. ADR-018: progress publisher in `internal/redis/stream.go` uses `XAdd`, NOT `Publish`. (Lands at Task 5.4; 5.1 establishes file-layout convention.)
5. ADR-021: `goleak.VerifyTestMain(m)` template in buildguard package; `go vet` lostcancel + golangci-lint `containedctx`/`noctx` in CI.

### 2026-05-01 — Task 5.1: lib/pq removal (Checkpoint 4 reversal)

**Reversal of Checkpoint 4 scope-comment approach.** Checkpoint 4 commit `39e1e5e` added a scope-only comment for `lib/pq v1.10.9` reserving it for future `cmd/admin/` use, with the rationale that ADR-013's buildguard test would prevent `cmd/worker/` misuse regardless of what `cmd/admin/` did. **Task 5.1 reverses this:** remove `lib/pq` from VERSIONS.md §2.4 and from the engine's go.mod entirely.

**Reason:** `go mod tidy` strips unused dependencies from `go.mod`. With no current binary (`cmd/worker/` excludes per ADR-013, `cmd/admin/` doesn't exist yet) importing `lib/pq`, the next `go mod tidy` run after engine bootstrap would remove the line — making the scope-comment approach unstable. Either we add a `// +build never`-tagged stub import to keep tidy from stripping (anti-pattern), or we accept that `lib/pq` doesn't belong in `go.mod` until a binary actually needs it.

**The buildguard test is the durable forcing function.** `TestWorkerBinary_DoesNotImportPostgresDriver` excludes `lib/pq` / `jackc/pgx` / `database/sql` from `cmd/worker/...`'s deps regardless of whether those drivers exist anywhere else in the module. When `cmd/admin/` is created (future task — likely M9+ or OPS milestone), it adds `lib/pq` (or `pgx`) back at that time. The buildguard test continues to enforce the worker exclusion; the test does not assert the driver is absent globally, only that `cmd/worker/` doesn't transitively pull it.

**Why the change of mind is fine.** Checkpoint 4's decision was reasonable given what was known at the time (we hadn't yet considered the `go mod tidy` interaction). Scope-proposal-time reflection surfaced the simpler approach. This is the kind of principled reversal the architectural-commitments preamble pattern is designed to surface — better to catch at scope time than after committing the engine repo with an unstable convention.

**Action items closed in this commit:**
- VERSIONS.md §2.4: `lib/pq v1.10.9` line removed; replaced with a comment block explaining the intentional absence + buildguard-test reasoning + reservation policy.
- Engine `go.mod` (Commit 2 next session): authored without `lib/pq`.

### 2026-05-01 — ADR numbering reservation: 019 + 020 intentional gap

**Pin.** SPECIFICATION §13 jumps from ADR-018 to ADR-021. ADR-019 and ADR-020 numbers are reserved for future promotion-on-trigger from DRIFT-LOG entries — not missing.

**Reservations:**
- **ADR-019 (cancel Pub/Sub confirmation).** SPEC §7.4 + plan agree on cancel channel `shieldscan:cancel:{scan_id}` Pub/Sub primitive. M5 landscape pass surfaced this as ADR-candidate but it's confirmation, not novel decision. Stays as DRIFT-LOG-only unless M5+ work surfaces a need to revisit (e.g., if cancel propagation latency becomes a load-bearing concern requiring Streams-with-replay semantics).
- **ADR-020 (worker concurrency model).** TOOL-ARCHITECTURE.md §10.3 specifies per-job tool-fanout (5 tools concurrent for one scan); SPEC §11 specifies per-worker job concurrency (5 jobs concurrent on one worker). M5.5 ships per-worker BRPOP-loop concurrency only (env-var configurable via `SHIELDSCAN_WORKER_CONCURRENCY`, default 5). M8's recon-first executor ships per-job tool-fanout. The model decision is currently captured implicitly in those task scopes; promote to ADR-020 if M8.1 surfaces a load-bearing trade-off (e.g., total goroutine accounting on 8GB RAM workers).

**Reservation discipline:** when a DRIFT-LOG entry's decision becomes load-bearing across multiple tasks, promote to full ADR with the reserved number. This keeps SPEC §13 ADR numbering meaningful (each ADR represents a load-bearing architectural commitment) while DRIFT-LOG captures the broader stream of design decisions including confirmations and deferrals.

### 2026-05-01 — Checkpoint 4 (VERSIONS sync before M5): asynq dropped + test deps added

**Pre-M5 docs sweep.** Ahead of Task 5.1 scaffolding the `shieldscan-engine` Go repo, VERSIONS.md §2.4 was reconciled with the four ADRs landing at Task 5.1's architectural-commitments preamble (ADR-016 asynq drop · ADR-017 findings-path · ADR-018 Streams correction · ADR-021 ctx discipline).

**Version-bumps portion was a no-op confirmation.** §2.1 + §2.4 had been kept current with the Go ecosystem independently of the plan literal:

| Component | VERSIONS.md (already pinned) | Plan §5.1 lines 1510-1525 (stale) |
|---|---|---|
| Go | 1.26.2 | 1.22 |
| go-redis/v9 | 9.7.0 | 9.4.0 |
| docker | 27.3.1+incompatible | 25.0.0+incompatible |
| aws-sdk-go-v2 | 1.32.0 | 1.24.0 |
| testify | 1.9.0 | 1.8.4 |
| zerolog | 1.33.0 | 1.31.0 |
| cobra | 1.8.1 | 1.8.0 |

**Net commit deltas:**
- **REMOVED:** `github.com/hibiken/asynq v0.25.1` (per ADR-016, finalized at Task 5.1). Inline comment at the head of the require block pins the rationale: "raw Redis matches M4 dispatch contract."
- **ADDED:** `github.com/alicebob/miniredis/v2 v2.33.0` — in-process Redis for tests (fakeredis-equivalent for Go). Compat with go-redis/v9.7.0 documented as confidence, not test-verified (rework risk if surfaced at Task 5.4 implementation).
- **ADDED:** `github.com/jarcoal/httpmock v1.3.1` — transport-level HTTP mock for DockerServiceRunner tests (Task 5.3).
- **ADDED:** `go.uber.org/goleak v1.3.0` — goroutine-leak detection (ADR-021 ctx discipline forcing function).
- `lib/pq v1.10.9`: scope comment added — repo-scope only; `cmd/worker/` MUST NOT import (ADR-013 forcing function reservation for future admin tooling under `cmd/admin/`).

**Plan-staleness flag (state-at-time discipline).** IMPLEMENTATION-PLAN.md §5.1 lines 1510-1525 carry a stale `go.mod` block (go 1.22, asynq v0.24.1, redis 9.4.0, docker 25.0.0, etc.). The plan stays as written; per CLAUDE.md document hierarchy (VERSIONS wins), Task 5.1's actual `go.mod` will be authored from VERSIONS.md, not from the plan literal. **This is plan-staleness, not VERSIONS-staleness** — VERSIONS.md was kept current; the plan was authored before the version refresh and reflects state-at-time. Future eyes opening §5.1 should treat the dependency block as illustrative, not authoritative.

**Forcing function for Task 5.1 kickoff.** ADR-016/017/018/021 cross-references in this DRIFT-LOG (and in §13 ADR additions when 5.1 ships) ensure no rediscovery cycle for the asynq question, the findings-path question, the Pub/Sub-vs-Streams question, or the goroutine-discipline question.

### 2026-04-30 — Task 4.6: cross-project comparison rejected (409 scans_project_mismatch)

**Decision.** Compare endpoint requires both scans to belong to the same project. Cross-project comparison (project A baseline vs project B current) is conceptually valid for orgs running the same codebase across multiple environments (staging/prod), but at MVP it's a niche use case that complicates the contract.

**409 `scans_project_mismatch`** when `baseline.project_id != current.project_id` (both org-scoped, both visible — semantic conflict, not existence-hiding). Rejecting now keeps the contract narrow; **removing later is an additive change** — drop the guard, document the new behavior. The forcing function (`test_compare_cross_project_returns_409`) is symmetric: the guard's removal would be a deliberate decision, not an accident.

### 2026-04-30 — Task 4.6: non-terminal scan handling — 409 scan_not_terminal for FAILED + CANCELED + active states

**Decision.** Compare endpoint requires both scans in `{COMPLETED, PARTIAL}` state. `FAILED`, `CANCELED`, and active states (`QUEUED`, `RECONNING`, `RUNNING`, `ANALYZING`) all return 409 `scan_not_terminal`.

**Failure mode prevented:** comparing against a non-terminal-final scan reports fake "resolved" findings — vulnerabilities that simply weren't discovered yet (or were aborted), not actually resolved. A customer running a regression check would see "12 vulnerabilities resolved!" when nothing was resolved — the new scan just hasn't found them yet.

**`PARTIAL` is the right boundary:** some jobs ran successfully, providing meaningful baseline data even if not all jobs completed. `FAILED` (whole-scan failure, no useful data) and `CANCELED` (user-aborted, incomplete by definition) are explicitly rejected.

**Pinned by** `test_compare_non_terminal_scan_returns_409` (active state) + `test_compare_canceled_scan_returns_409` (explicit CANCELED pin).

### 2026-04-30 — Task 4.6: response size cap with truncated flag + logger.warning

**Decision.** Each diff category capped at `MAX_FINDINGS_PER_CATEGORY = 5000` (in `services/scan_compare.py`). When any category hits the cap, `summary.truncated: true` flag fires + the route handler emits `logger.warning("scan_compare hit category cap of 5000 ...")`.

**Why a logger.warning, not an error:** the cap is by design, not a bug. The warning is an **operational signal for capacity planning** — if it fires regularly, raise the cap or ship pagination earlier than M10. Greppable log line for ops dashboards.

**At ~300 bytes/entry × 5000 × 3 categories** ≈ **4.5 MB max response.** Hardly common at MVP scale (typical scans have 10–500 findings). Pagination → M10 read-side cluster, where it can be designed alongside `GET /vulnerabilities` and friends rather than retrofit per-endpoint.

### 2026-04-30 — Task 4.6: read-only fingerprint algorithm — M9 pipeline populates, 4.6 reads

**Pin.** `Vulnerability.fingerprint` column is canonical (CLAUDE.md gotcha 4):
```
SHA-256(tool_name|finding_type|target_url|parameter|code_file|code_line)
```

**M9 AI pipeline computes and populates the column** for each deduplicated finding. **4.6 only READS** — no schema changes, no algorithm definitions in the compare service. `compute_diff` partitions by reading `fingerprint` directly off `Vulnerability` rows.

**Defensive on duplicates within one scan:** `setdefault` in the partition logic means accidentally-duplicated fingerprints (shouldn't happen post-M9 dedup but defensive) yield first-write-wins, not a crash.

### 2026-04-30 — Task 4.6: "changed" category deferred (severity/details delta on persisting findings)

**Carry-forward.** A vulnerability whose fingerprint matches across scans but whose `severity` or `description` shifted (e.g., CVE was rescored) is currently in the **persisting** bucket. A future "changed" sub-category could surface these to draw customer attention to severity drift.

**Why defer:** fingerprint match is the strong "same vulnerability" signal. Severity/title drift is softer. Modeling the second cleanly requires (a) a canonical comparison shape — which fields' delta is meaningful? (b) UI work to surface a distinct bucket, (c) customer feedback on whether the granularity is useful. None in scope for 4.6's MVP.

**Promotion criterion:** ship "changed" if a paying customer asks for severity-drift surfacing or if M9 AI pipeline adds reliable severity-shift signals.

### 2026-04-30 — Task 4.5: cancel-wins at scan-level (reversal of M4 landscape framing)

**Reversal + clarification.** The M4 landscape pass said *"completion wins, cancel becomes a no-op"* for cancel-vs-completion races. That framing conflated two different races:

- **Job-level race:** A worker tool finishes as the cancel signal arrives. The worker decides what status to emit (generally completion if results were written, cancel otherwise). M5+ worker concern.
- **Scan-level race:** The cancel endpoint sets `Scan.status = CANCELED` while the last `job_completed` event is in flight to the consumer. **Different question.**

**Task 4.5 decision (scan-level):** **cancel wins.** User intent is sticky. `CompletionsConsumer._maybe_complete_scan` adds `ScanStatus.CANCELED` to its terminal-state short-circuit set so a completion event arriving after the cancel does not overwrite the scan-level CANCELED state.

Job-level state still reflects what each worker reported — some `completed`, some `canceled`. Power users can drill into `scan_jobs` rows even when scan-level says CANCELED.

**Forcing function:** `test_consumer_does_not_overwrite_canceled_scan` in `tests/services/test_completions_consumer.py`. Test cancels at scan-level, drives completion events for all jobs, asserts `Scan.status` stays CANCELED + zero `SCAN_COMPLETED` audit rows. Future "helpful" change that drops `CANCELED` from the short-circuit breaks this test immediately. Test docstring explicitly references this DRIFT-LOG entry + ADR-013.

**Cross-reference:** ADR-013 anti-patterns section ("Periodic reconciliation jobs that 'verify Redis matches PG'") — same single-writer discipline.

**Commit:** `shieldscan-api` `97bbba6`.

### 2026-04-30 — Task 4.5: terminal-state distinction (409 vs 204 idempotent)

**Pin.** Cancel state-transition policy:
- Active states (queued/reconning/running/analyzing) → CANCELED + audit + 204.
- Already-CANCELED → 204 + fresh audit row (idempotent state, audit-on-event discipline).
- Terminal-final (completed/partial/failed) → **409 `scan_already_terminal`**. The work concluded; cancel is semantically meaningless. 409 (not 410) because the scan isn't in the deleted state — it's in a final-success/partial/failure state that conflicts with the cancel operation. Same distinction-logic as Task 3.X PATCH-on-archived-project 409.

**Audit-on-event discipline matches Task 3.2 reverify pin:** the cancel ACTION is the event. Even when state is unchanged (already-CANCELED), each cancel attempt emits a fresh audit row. Pinned by `test_cancel_already_canceled_returns_204_with_fresh_audit`.

### 2026-04-30 — Task 4.5: worker-side cancel consumption is M5+ work — signal goes into the void today

**Operational pin.** The cancel endpoint publishes to `shieldscan:cancel:{scan_id}` per SPEC §7.4 + ADR-014. **No subscriber exists yet.** Go workers (M5) will subscribe in Task 5.5 (worker processing loop) and call `ctx.Cancel()` on in-flight tools.

Until M5 ships:
- Cancel endpoint works from the API perspective: PG state flips, audit row emitted, Pub/Sub publish succeeds.
- No actual abort happens — there are no Go workers yet.
- Future engineer running the cancel endpoint and not seeing scans abort should NOT think the system is broken; the worker side just isn't built.

**M5 task 5.5 acceptance criterion:** worker subscribes to cancel channel for each job it picks up; abort propagates through tool runner. The cancel pattern is end-to-end testable only after M5.5 lands.

### 2026-04-30 — Task 4.5: DELETE verb + scan_id-scoped router pattern

**Pattern pin.** SPEC §6.2 line 491: `DELETE /orgs/:org_id/scans/:scan_id`. Note: **scan_id-scoped path, not project-scoped.** Different shape from Task 4.3's POST under `/orgs/{org_id}/projects/{project_id}/scans`.

`routes/scans.py` now exports two `APIRouter` instances:
- `project_scans_router` — Task 4.3 POST/(future GET) project-scoped.
- `scan_router` — Task 4.5 DELETE + future M4 (4.4 SSE) + M10 (single GET, jobs GET) scan_id-scoped operations.

Two routers in one file matches the `routes/projects.py` precedent (CRUD + nested verify + nested credentials all colocated). Both registered in `main.py`.

**DELETE verb-discipline:** matches existing project DELETE (soft-delete via `archived_at`) + api-key DELETE. Cancel = soft-delete-of-active-work. Row remains; status flips.

### 2026-04-30 — Task 4.5: any-member auth (symmetric with scan-create)

**Decision.** Cancel uses `require_org_membership()` (default — JWT or API key, no admin gate). Symmetric with Task 4.3 scan-create.

**Cost-protection counter-argument considered + rejected:** cancel terminates work that may have spent compute + AI cost. But the same is true of scan-create (also any-member, also costs money). Asymmetric gating would require admin to cancel scans they themselves created via CI/API key. Symmetry wins: anyone who can create a scan can cancel one.

**Pinned by** `test_cancel_via_api_key_succeeds` + `test_cancel_via_api_key_audits_with_key_prefix` (the second also pins the audit details API-key-prefix shape).

### 2026-04-30 — Task 4.5: cancel endpoint ships without rate limiting

**Decision.** No rate limit on the cancel endpoint in 4.5. Cancel-spam abuse vectors exist:
- Single-scan audit-log fill (the audit-on-event discipline emits a row per attempt).
- Multi-scan CPU load (each cancel hits PG + Redis + audit_logs).

Both real but not serious at MVP scale. Revisit if abuse patterns emerge in production.

**Adding rate-limit later is mechanical:** existing `rate_limit` dep + new scope name (e.g. `scan_cancel`). Cross-ref: scan-create rate-limit pattern (also not yet shipped — same deferral).

### 2026-04-30 — Task 4.3: route + orchestrator share a single transaction (H.1)

**Pin.** `POST /v1/orgs/{org}/projects/{pid}/scans` flushes the new `Scan` row and passes the same `AsyncSession` to `ScanOrchestrator.dispatch()`. The orchestrator INSERTs ScanJob rows + emits `SCAN_DISPATCHED` audit + COMMITs **one** transaction covering Scan + ScanJobs + audit atomically.

**Failure modes:**
- Pydantic / domain-verified / mobile-upload validation fails → no DB writes (request rolls back at end).
- INSERT failures (FK violation, etc.) → entire tx rolls back; customer never sees a Scan with zero ScanJobs.
- Commit succeeds + Redis dispatch fails → "ghost queued" rows present, no queue entries. Visible-failure mode preferred over silent (DRIFT-LOG Task 4.2 H.3 entry). M5+ retry janitor recovers.

The route deliberately does NOT commit after `dispatch()` returns — orchestrator owns commit per ADR-013.

### 2026-04-30 — Task 4.3: `config` field ships with shape-bounds validation only

**Decision.** `ScanCreateRequest.config` is `dict[str, Any]` with three DoS guards:
- max **64 KB** serialized (json.dumps length)
- max depth **5** nested levels
- max **50** keys at the root

Per-scan-type semantic schemas (discriminated union à la 3.X credentials) deferred until worker-side schemas firm up — likely M5–M6 when Go runners land. Shape-bounds are sufficient for now because workers are first-party (not handling customer-controlled downstream input); the DoS surface is the only real concern at the route boundary.

**Pinned by:** `test_config_oversized_returns_422`, `test_config_too_deep_returns_422`, `test_config_too_many_keys_returns_422`. All three use real `json.dumps`-derived payloads to ensure rejection triggers on serialization, not character-count heuristics.

### 2026-04-30 — Task 4.3: `callback_url` deliberately rejected via `extra="forbid"`

**Decision.** SPEC §6.3 example body carries `callback_url`. Webhook delivery infrastructure (HMAC signing, retry, DLQ) is M12.5+ work. Accept-and-ignore would create a silent "webhook never fires" bug class.

`ScanCreateRequest` has `model_config = ConfigDict(extra="forbid")` — sending `callback_url` (or any other unknown field) returns 422 with a clear message that the field isn't accepted. When M12.5 ships webhook delivery, the field is added explicitly. Loud-rejection > silent-acceptance for this class of feature.

**Pinned by** `test_extra_fields_rejected_422`.

### 2026-04-30 — Task 4.3: `mobile_upload_id: UUID` request shape (deviation from PLAN/SPEC `mobile_config.upload_ref`)

**Plan/SPEC deviation.** IMPLEMENTATION-PLAN.md §4.3 + SPECIFICATION.md §6.3 example bodies carry `mobile_config: {upload_ref: "r2://..."}` — the request body would force API consumers to construct/parse `r2://` URIs.

Shipped instead: `mobile_upload_id: UUID` referencing the `mobile_uploads.id` PK directly. The `r2://` URI representation is internal — built by `orchestrator._build_job_payload` from the `MobileUpload` row's `r2_key` column. Cleaner client API, no internal-detail leakage.

**Implementation note:** the route validates the `mobile_upload_id` exists, belongs to the same org (404 on cross-tenant — existence-hiding), and isn't soft-deleted (`deleted_at IS NULL`) BEFORE creating the Scan row. Pinned by `test_mobile_upload_from_different_org_returns_404` + `test_mobile_scan_without_upload_id_returns_422`.

### 2026-04-30 — Task 4.3: orchestrator commits the session — caller MUST NOT commit again (explicit pin)

**Pin (re-stating Task 4.2's contract for clarity at the first caller).** `ScanOrchestrator.dispatch()` commits the session passed to it. The 4.3 route handler:

1. Validates request + flushes a new `Scan` row (no commit).
2. Eager-attaches `scan.project = project` in memory.
3. Calls `await orchestrator.dispatch(scan, identity, ...)` — orchestrator commits.
4. Returns 201 directly from the in-memory ORM objects (post-commit, no `select_fresh` needed).

**Asymmetric** with `audit()` (never commits) and the credentials handler (route commits). Verified the orchestrator docstring is explicit. Future endpoints calling `dispatch()` should follow the same caller-flushes-orchestrator-commits pattern.

### 2026-04-30 — Pattern: long-lived background tasks need `session_factory` DI

**Carry-forward from Task 4.2 (asyncpg pool race).** `CompletionsConsumer` opens fresh `AsyncSession` instances per event. Production binds the `session_factory` parameter to the global `AsyncSessionLocal`. **Tests must inject a factory bound to the test fixture's connection** — the global `AsyncSessionLocal` is bound to the engine's first event loop, which becomes stale across pytest-asyncio's per-test loop cycle, surfacing as `asyncpg.InterfaceError: cannot perform operation: another operation is in progress`.

Test pattern (from `tests/services/test_completions_consumer.py`):

```python
@pytest_asyncio.fixture
async def session_factory(db_session):
    """Bind each "fresh" session to the test's connection — same RLS
    context, no cross-loop pool contention."""
    from sqlalchemy.ext.asyncio import AsyncSession as _S
    connection = db_session.bind

    class _Factory:
        async def __aenter__(self):
            self._session = _S(bind=connection, expire_on_commit=False)
            return self._session
        async def __aexit__(self, *exc):
            await self._session.close()

    return lambda: _Factory()
```

**Pattern reusable for any future long-lived background task** (M5 worker dispatch listeners, scheduled scan jobs, the M5+ ghost-queued retry janitor). DEVELOPMENT-PATTERNS.md gets a dedicated section in this docs commit.

### 2026-04-30 — Task 4.2: Python is sole writer for scan state (ADR-013)

**ADR-013 ships.** Captures the state-machine ownership decision: Python writes; Go signals via Redis. The companion `CompletionsConsumer` consumes `shieldscan:completions` Pub/Sub events and applies them to `scan_jobs`/`scans` rows.

**Forcing functions baked in:** Go worker config (M5 task 5.6) excludes PG credentials. `test_consumer_aggregates_scan_status_when_all_jobs_terminal` asserts aggregation happens via Python event consumption, not external trigger. ADR-013's "Anti-patterns this prevents" section pre-emptively addresses three temptations (Go writing partial state, Python reading state from Redis, periodic reconciliation jobs) so a future engineer doesn't quietly violate the discipline.

**Cross-reference forward:** ADR-013 referenced in M5 task 5.6 task definition as the reason worker config must omit PG credentials.

**Commit:** `shieldscan-api` `cf3b30a` (orchestrator + consumer + ScanAction enum), `shieldscan-docs` `a8a024d` (ADR-013 + entries).

### 2026-04-30 — Task 4.2: ScanAction enum split from ProjectAction

**Refactor.** Audit-domain enum split discipline (Task 3.2 precedent): `AuthAction` for auth events, `ProjectAction` for project events, **`ScanAction` (new)** for scan events. `ALL_ACTION_ENUMS` registry auto-grows; `test_action_enum_values_are_globally_disjoint` covers it without manual update.

Five values shipped (`SCAN_DISPATCHED`, `SCAN_CANCELED`, `SCAN_COMPLETED`, `SCAN_FAILED`, `SCAN_COMPARED`); one reserved (`SCAN_JOB_RETRIED`) for M5+ retry policy. `audit()` signature widened to `AuthAction | ProjectAction | ScanAction`.

### 2026-04-30 — Task 4.2: payload.auth = None pending ADR-015

**Pin.** Job payloads dispatched to Redis carry `auth: None`. Decrypted-credential transit through Redis (ADR-015 — deferred) ships when the Go worker side can consume it (M5+). Inline TODO in `_build_job_payload` references ADR-015. Pinned by `test_dispatch_payload_auth_is_null_pending_adr_015` — a future "helpful" enablement without writing the ADR breaks the test.

> **PIN LIFTED — 2026-05-21 ADR-015 LANDED.** Cross-repo trio complete per shieldscan-docs `9a57865` (SPEC §13 ADR-015 + ADR-013/ADR-014 addendums) + shieldscan-api `742faed` (orchestrator decrypt+emit + `SCAN_CREDENTIAL_DECRYPTED` audit + positive-path tests; `test_dispatch_payload_auth_is_null_pending_adr_015` regression pin DELETED per C2.4 option a; replaced with positive-path coverage including credential-less `auth=None` case) + shieldscan-engine `b48fef8` (SQLMap consumer cookie wiring + integration test V4 baseline upgrade; Task 7.6 Drift #35 architectural-reconciliation closed). Engine-side `DRIFT-LOG.md` top entry carries full Drifts #40-#44 catalog (Y-AUDIT favorable infrastructure-discovery + critical architectural-correctness + emission-site refactor + latent pre-existing regression fix + critical integration-test network-topology mismatch).

### 2026-04-30 — Task 4.2: completions consumer is FastAPI-lifespan-managed (single task per API process)

**Pin.** `CompletionsConsumer.start()` is invoked from `main.py`'s `lifespan` after Redis init; `.stop()` runs before Redis close. One asyncio task per uvicorn worker process. With N>1 uvicorn processes, every worker subscribes to the same Pub/Sub channel — every completion event is handled N times. Second-and-later handlers see the job already in the terminal state and short-circuit with a DEBUG log:

```python
if job.status == new_status and job.status in _TERMINAL_JOB_STATUSES:
    logger.debug(
        "completion event for already-terminal job_id=%s; "
        "expected at MVP with N>1 uvicorn workers, not at scale",
        job_id,
    )
```

The DEBUG line is greppable so a future ops view can quantify the actual rate of duplicate handling. Acceptable double-work at MVP scale; consumer-groups (which require Streams, not Pub/Sub for completions) deferred to scale milestone.

### 2026-04-30 — Task 4.2: per-completion audit deliberately skipped — terminal aggregate only

**Audit emission policy for scan lifecycle.** Per-job completion is captured in `scan_jobs.status` column, NOT in `audit_logs`. Only scan-level terminal transitions (`SCAN_COMPLETED`, `SCAN_FAILED`) emit audit rows.

**Don't 'helpfully' add per-job audit rows** — they're noise that would dilute the audit log for security-relevant events. Job-level traceability lives in `scan_jobs` row.

**Forcing function:** `test_consumer_does_not_audit_per_job` emits 2 of 3 completion events (non-terminal at scan level), asserts zero scan-domain audit rows, then emits the final event and asserts exactly ONE `SCAN_COMPLETED` row. Future engineer adding per-job audit emission breaks this test immediately with a clear failure message.

### 2026-04-30 — Task 4.2: commit-then-dispatch ordering — visible failure preferred

**Decision.** `ScanOrchestrator.dispatch()` order is:
1. INSERT `ScanJob` rows + emit `SCAN_DISPATCHED` audit
2. **COMMIT the transaction**
3. Push job payloads to Redis queue

If step 3 fails, DB rows are already committed — customer sees a "ghost queued" scan they can triage (visible failure). The rejected alternative (dispatch-then-commit) buries failures in worker logs as orphan queue entries with no DB row to explain them (silent failure).

**M5+ requirement:** retry/re-dispatch janitor for scans whose `ScanJob.status = 'queued'` longer than a threshold (suggested: 5 minutes). Without it, ghost-queued scans stay forever queued. Acceptable in M4 because visible-failure recovery is easier than silent-failure detection. Cross-ref: ADR-013 forcing function for state ownership.

**Pinned by:** `test_dispatch_commits_before_redis_dispatch` — mocks `queue.dispatch` to raise, asserts `ScanJob` rows persist + the audit row is committed.

### 2026-04-30 — Task 4.2: orchestrator commits its own transaction (not the caller's)

**Pin.** Initial scope-proposal had orchestrator follow the `audit()` "caller owns the transaction" discipline. Reversed at H.3 to support commit-then-dispatch ordering (entry above). Now: `ScanOrchestrator.dispatch()` calls `self.db.commit()` itself before pushing to Redis. The 4.3 route handler MUST NOT commit again after `dispatch()` returns — pinned in the orchestrator docstring.

This is asymmetric with `audit()` (which never commits) and the credentials handler (which commits at the end of the route). Documented loud so future patterns don't inherit the wrong shape.

### 2026-04-30 — Task 4.2: RLS context for non-request paths — completions consumer SETs GUC per event

**New pattern.** The completions consumer runs as a long-lived background task with no request context. Each event opens a fresh `AsyncSession` from `AsyncSessionLocal` and SETs `app.current_org_id` from the event's `organization_id` field before any RLS-scoped read or write.

**Defense-in-depth concern:** the GUC value comes from the consumed event. The Go worker is the only legitimate emitter, but the event could in principle be spoofed by anyone with Redis access. Defense: Redis runs inside our trust boundary (private VPC). Future hardening: HMAC-sign completion events at Go side, verify in consumer. Defer to ops-hardening milestone.

**Carry-forward to DEVELOPMENT-PATTERNS.md** at the next docs cleanup pass: this is the second non-request RLS pattern (after the completions consumer itself). Worth a dedicated entry alongside `select_fresh`.

### 2026-04-30 — Task 4.2: M5+ retry/re-dispatch janitor (commit-then-dispatch follow-up)

**Carry-forward.** Task 4.2 ships commit-then-dispatch ordering. The "ghost queued" failure mode (DB commit succeeds + Redis dispatch fails) requires a janitor process to recover. M5+ worker milestone must include:

- A periodic task that finds `ScanJob` rows with `status = 'queued'` older than threshold (suggested: 5 minutes) AND no corresponding queue entry.
- Either re-dispatch them (build payload from the persisted row + push to Redis) or transition them to `failed` with a clear `error_message`.
- Audit emission either way (a new `ScanAction.SCAN_JOB_RETRIED` is already declared/reserved).

Without the janitor, ghost-queued scans stay forever queued and customer support would have to manually intervene. Tracking cost: half-day in M5+ when worker startup lands.

### 2026-04-30 — Task 4.1: Redis Streams over Pub/Sub for progress events (ADR-014)

**ADR-014 ships.** Plan literal §4.1 + §4.4 sketched a Pub/Sub-only design for `shieldscan:progress:{scan_id}`. SPEC §6.4 requires `Last-Event-ID` 60s replay — Pub/Sub cannot satisfy this on a multi-process uvicorn worker. ADR-014 captures the decision: Streams (`XADD` + `XREAD BLOCK` for live, `XRANGE` for replay) as a single primitive; multi-process reconnects work correctly because all workers read the same Stream.

**Hybrid (Streams + Pub/Sub) considered and rejected** for MVP scope: two writes per event, two consumer codepaths, two failure modes — justifiable at 100k+ concurrent connection scale (Discord, Slack pattern), not at MENA-SMB MVP scale (1–3 concurrent scans × <10 viewers each). `XREAD BLOCK` is Redis's Streams-equivalent of `SUBSCRIBE` and handles the fan-out trivially. Wrapper isolation makes a future hybrid swap mechanical if sustained-load profiling shows fan-out hot.

**Forcing function:** `test_subscriber_replay_returns_history` requires `XRANGE` semantics. Test docstring explicitly references ADR-014 so a future "simplification" reverting to Pub/Sub fails immediately with a self-documenting message.

**Commit:** `shieldscan-api` `349fc5e` (api), `shieldscan-docs` `60d71f8` (ADR + SPEC patch).

### 2026-04-30 — Task 4.1: SPEC §7.2 wording patched (Channel → Stream)

**Cosmetic + semantic.** SPEC §7.2 line 769 originally read "Channel: `shieldscan:progress:{scan_id}`" — Pub/Sub vocabulary inconsistent with the ADR-014 Streams decision. Patched to "Stream:" + a paragraph noting the producer/consumer primitives (`XADD ... MAXLEN ~ 1000` produce, `XRANGE`/`XREAD BLOCK` consume) and a final paragraph noting that cancel + completions remain Pub/Sub.

Same docs commit as ADR-014 — patching the spec without the ADR would be confusing in isolation.

### 2026-04-30 — Task 4.1: Mixed-primitive use (Streams for progress, Pub/Sub for cancel + completions)

**Pin.** Streams used ONLY for `shieldscan:progress:{scan_id}` because §6.4 requires replay. Pub/Sub stays for:
- `shieldscan:cancel:{scan_id}` — one-shot live-only signal. A worker not subscribed when cancel emits cannot usefully consume a stale cancel.
- `shieldscan:completions` — one-shot completion broadcast. The orchestrator's completions consumer either receives live or rebuilds state from the DB row at startup. Replay adds no value.

This is **intentional, not transitional**. A future "let's unify on Streams everywhere" refactor should NOT happen without a fresh ADR — the asymmetry is the right answer for these specific channels.

### 2026-04-30 — Task 4.1: Cancel-vs-completion race — completion wins

**Pin (no code in 4.1).** If a job finishes at the same instant a cancel arrives, both `job_completed` (to `shieldscan:completions`) and the cancel signal (to `shieldscan:cancel:{scan_id}`) may publish. **Resolution at the worker side (M5 task 5.5):** if the job is already in a terminal state (completed/failed) when the cancel is consumed, cancel becomes a no-op. The DB row reflects whichever terminal state actually landed first. Documented here so 4.5 (cancel endpoint) doesn't accidentally try to override completion server-side.

### 2026-04-30 — Task 4.1: Completions subscriber explicitly punted to 4.2

**Pin.** SPEC §7.3 defines `shieldscan:completions` as a Pub/Sub channel that publishes job-completion events back to the orchestrator. M4 needs a consumer that subscribes and updates `ScanJob.status`/`finding_count`/`duration_ms` rows. **Not a 4.1 primitive** — it's orchestrator-internal coordination, surfaces in Task 4.2's scope proposal as part of the ScanOrchestrator design (likely a background task lifecycle the orchestrator owns, not a standalone wrapper class).

### 2026-04-30 — Task 4.1: Idempotency-key handling deferred to Go worker (M5.5), not Python

**Pin.** SPEC §7.5 specifies `idempotency_key` format `{scan_id}:{tool}:{unix_ts}` with 24h Redis TTL. Python orchestrator (4.2) GENERATES the key inside the dispatched job payload but does NOT touch Redis idempotency state. The SETNX-with-TTL claim happens worker-side at job-pickup time (M5 task 5.5). `test_dispatch_does_not_dedupe_by_idempotency_key` is a NEGATIVE pin — it asserts that duplicate `dispatch()` pushes duplicate items, so a future engineer adding Python-side dedupe breaks this test loudly.

ADP-5 from M4 landscape pass: option 2 (worker-side claim) confirmed.

### 2026-04-30 — Task 4.1: Stream-key cleanup TTL — OPS milestone carry-forward

**Carry-forward.** `XADD ... MAXLEN ~ 1000` bounds each stream's *length*, but the **stream keys themselves** persist after scan completion. At MVP scale this is fine (handful of concurrent scans, tiny memory footprint). At enterprise scale, accumulating thousands of inactive stream keys becomes housekeeping overhead.

**OPS-milestone scope:** janitor that purges `shieldscan:progress:*` keys 24–48h after `Scan.completed_at`. Possibly fold into a single Redis-cleanup job that also handles `shieldscan:cancel:*` channel-key residue. Tracking cost: ~half-day in OPS milestone. Don't try to add cleanup logic in 4.1 — wrong domain.

### 2026-04-30 — Task 4.1: Scan list/detail SPEC §6.2 endpoints deferred from M4 to M10

**Pin.** SPEC §6.2 lists `GET /scans`, `GET /scans/:id`, `GET /jobs[/:jid]` under the Scans domain (7 endpoints). M4 ships 4 of those (POST create, DELETE cancel, GET progress SSE, POST compare). The other 3 are read-side endpoints.

**Decision (M4 boundary):** ship them in M10 (Vulnerability & Report APIs) — the read-side cluster — rather than tacking them onto M4 as a 4.X gap-closer. Pattern variant of the SPEC-gap-closure pattern: 2.Y / 3.X closed gaps where the work fit the milestone's domain (small composition); scan read-side is read-side and benefits from consistent pattern design across the read-side cluster (scans + findings + reports). Building one per write-side milestone risks read-side pattern inconsistency.

The other M4 endpoints are write-side scan operations + the SSE stream; M4 closes after Task 4.6 (compare).

### 2026-04-30 — Task 3.X: AuthType enum lifted from M1 inline comment

**Refactor.** M1's `ProjectCredential.auth_type` carried only an inline comment (`# cookie | bearer | basic | form | custom_header`) describing legal values. Task 3.X needs the value set programmatically (Pydantic discriminated union, route-handler dispatch), so the comment is lifted into a proper `AuthType(str, PyEnum)` in `app/services/credentials.py`.

**Five canonical values** (frozen): `basic`, `bearer`, `cookie`, `form`, `custom_header`. Pinned by `CANONICAL_AUTH_TYPES` constant in the service module + the schema's `Literal[...]` discriminators. Adding a new value requires updating both surfaces and the test that asserts the canonical set.

**Disjointness from action enums:** `AuthType` is in a different namespace from `AuthAction`/`ProjectAction` — not registered in `ALL_ACTION_ENUMS`. Action enums encode audit events; `AuthType` encodes credential payload shape. Kept separate to avoid a cross-domain registry that would invite future confusion.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-30 — Task 3.X: PATCH + DELETE for credentials (vs single PATCH with `none` sentinel)

**Decision.** Two endpoints — `PATCH /credentials` (set/replace) + `DELETE /credentials` (clear) — instead of a single PATCH with `auth_type: "none"` clearing semantics.

**Rationale:** Verb discipline matches the rest of the codebase (DELETE on `/projects`, DELETE on `/api-keys`); cleaner audit boundary (`PROJECT_CREDENTIAL_SET` vs `PROJECT_CREDENTIAL_DELETED` cannot collide); no awkward `"none"` sentinel that would require a sixth schema variant carrying nothing. Trade-off accepted: one extra route, ~30 LoC and one extra test (`test_delete_credentials_204_and_audit`).

**DELETE idempotency:** calling DELETE on a project with no credential is a no-op 204 with no audit row. Audit rows fire only for events that actually happened.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-30 — Task 3.X: form-auth credential exposes five fields; `additional_fields` deferred

**Decision.** The `form` credential variant accepts exactly: `login_url`, `username_field`, `password_field`, `username`, `password`. No `additional_fields` map for arbitrary form inputs.

**Known limitation.** Real-world login forms sometimes carry hidden CSRF tokens, captcha challenges, or arbitrary submitted-state fields. The MVP form-credential ships without that flexibility — a customer whose login form needs an extra hidden field cannot configure it through this endpoint today.

**Why ship the limited shape now:** the five-field set covers the MENA SMB common case (per market research feeding milestone planning). Adding `additional_fields` later is an additive Pydantic change (no schema-level breaking change for existing clients). Until a paying customer hits this limit we defer the complexity.

**Forcing function:** `test_patch_form_happy_path` asserts the exact field set; adding a sixth field will require updating that test, surfacing the contract change to review.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-30 — Task 3.X: PATCH/DELETE on archived project → 409 Conflict

**Decision.** Both PATCH and DELETE on `/credentials` return 409 `project_archived` when the parent project is archived (`archived_at IS NOT NULL`). Symmetric on the two endpoints.

**Why 409, not 410:** 410 already means *"this resource is in the deleted state"* (used by `DELETE /projects` on already-archived). 409 = *"the operation conflicts with the resource's current state"* — credentials are not themselves deleted/archived; the project's state simply forbids credential mutation. Distinct semantics, distinct status codes.

**Why a hard error, not silent-allow:** archived projects shouldn't accept new state. Allowing credential mutation on archived projects creates a silent-data-leak path (re-activate an archived project with stale credentials the customer thought were cleared). Hard 409 forces the customer to un-archive first.

**Pinned by** `test_patch_archived_project_returns_409` + `test_delete_archived_project_returns_409`.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-30 — Task 3.X: single-key Fernet for MVP; multi-key `MultiFernet` rotation deferred to OPS milestone

**Carry-forward.** SPEC §10.1 promises *"key rotation support for encryption keys."* Task 3.X ships single-key Fernet only (`Fernet(settings.FERNET_KEY)`), no `MultiFernet` rotation primitive.

**Why MVP is sufficient:** zero production credentials exist pre-launch. Key rotation is a pre-launch checklist item: generate fresh key, re-encrypt seed credentials (none yet), update settings.

**OPS-milestone scope:** `MultiFernet` with primary + retiring keys, a re-encryption job that walks `project_credentials` rolling rows from old → new key, ops runbook entry for the rotation procedure. Tracking cost: ~1 day in OPS milestone.

**No SPEC update needed** — §10.1 isn't wrong, just early. The promise stands; this entry pins when it's delivered.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-30 — Task 3.X: Vault-stored secrets deferred from SPEC §10.1 to OPS milestone

**Carry-forward.** SPEC §10.1 says *"All secrets in Vault (not env vars in production)."* Task 3.X reads `FERNET_KEY` from `settings` (which loads from env) — Vault integration is OPS-milestone work.

**Pre-launch posture:** env-var secrets are acceptable when (a) the env file is `chmod 600` on the host, (b) no shared infra, (c) operational discipline of not committing the file. All true for MVP.

**Vault scope when it lands:** `app/config.py` swap from env to a Vault-backed loader, secret-rotation hooks tied to deployment, audit trail for secret reads. Substantial enough to be its own milestone task, not a 3.X gap-closer.

### 2026-04-30 — Task 3.X: FERNET_KEY conftest fixture corrected

**Bookkeeping.** The previous fixture in `tests/conftest.py` used the literal string `"test-fernet-key-base64-urlsafe-44-bytes="`, which is NOT a valid Fernet key (not 32 bytes of urlsafe-base64). The malformed key never crashed because no test instantiated Fernet — Task 3.X is the first code path to do so.

**Fix:** replaced with `"c2hpZWxkc2Nhbi10ZXN0LWZlcm5ldC1rZXktMzJieSE="` — valid, deterministic, base64 encoding of `b"shieldscan-test-fernet-key-32by!"`. Determinism matters because Test 14 asserts byte-level plaintext absence in the encrypted blob.

**Forcing function:** `test_fernet_key_in_settings_is_valid_at_import` instantiates Fernet at import time. Future broken keys fail at test collection, not mid-run.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-30 — Task 3.X: `project_credentials.project_id` UNIQUE constraint added (M1 schema correction)

**Schema correction.** M1 schema declared `project_credentials` with a non-unique `project_id` index. Task 3.X's PATCH semantics require exactly one credential per project — without UNIQUE, the application could (under a SELECT-then-INSERT race) end up with two credential rows for one project.

**Resolution.** Migration `d4f6b1e9a527` adds `UNIQUE (project_id)` constraint. Application-layer enforcement uses PostgreSQL UPSERT (`INSERT ... ON CONFLICT (project_id) DO UPDATE`) for atomicity, with the new constraint as the conflict target.

**Downgrade safety:** dropping the constraint is harmless. Upgrading on a dev DB with duplicate `project_id` rows fails loudly (unique-violation) — operator dedupes manually. No data migration needed pre-launch.

**Commit:** `shieldscan-api` `c7a12e9`.

### 2026-04-26 — Task 3.3: `.zip` dropped from mobile-upload allowed extensions

**Plan deviation.** IMPLEMENTATION-PLAN.md §3.3 lists the allow-list as `{".apk", ".ipa", ".zip"}`. Shipped as `{".apk", ".ipa"}`.

**Three reasons:**

1. **Adversarial input surface.** A `.zip` upload would force content-sniffing (or trust the filename) to decide platform. Either path is fragile — filenames lie, and content-sniffing on attacker-controlled input is a known footgun pattern.
2. **No clear MVP use case.** APKs and IPAs are themselves ZIP archives; a separate `.zip` extension only matters if customers send arbitrary bundles, which isn't the mobile-scan flow.
3. **Virus scanning deferred.** Accepting unscanned `.zip` while malware analysis is still scan-time-only (entry below) compounds risk for no MVP benefit.

**Reversibility.** Adding `.zip` back is a one-line change to `ALLOWED_EXTENSIONS` plus a platform-detection rule. Do it if a real workflow demands it; the test pinning `.zip → 400 invalid_extension` (`test_upload_zip_extension_rejected`) is the forcing function for that follow-up.

**Commit:** `shieldscan-api` `d9b0fbf`.

### 2026-04-26 — Task 3.3: error envelope is `{"detail": <code>}`, not `{"error": {"message": ...}}`

**Plan deviation, codebase-consistent.** IMPLEMENTATION-PLAN.md §3.3 test snippets assert `r.json()["error"]["message"]`. Shipped as `{"detail": <code>}` per the codebase-wide `ErrorResponse` standard established in Task 2.3.4.

The plan literal predates 2.3.4's standardization. Reading the plan literally would diverge every Task 3.3 error path from every other endpoint in the codebase — strictly worse than ignoring the literal.

**Pattern for future tasks:** plan literals whose error-shape predates 2.3.4 use `{"detail": ...}` without ceremony. No DRIFT-LOG entry needed per occurrence; this entry establishes the pattern.

**Commit:** `shieldscan-api` `d9b0fbf`.

### 2026-04-26 — Task 3.3: virus / malware scanning deferred to scan-time

**Decision.** The mobile-upload endpoint accepts `.apk` / `.ipa` files as content-addressable blobs. ZIP magic-bytes is the only content check. There is **no virus scanning at upload time**.

Static + dynamic analysis (including malware indicators surfaced by MobSF) happens later in the mobile-scan worker (M5+). The upload endpoint is intentionally **not a security gate** — its job is to land the artifact safely in R2 and trigger downstream scanning.

**Why this is OK at MVP:** R2 storage is private (no public read URLs). Files don't execute on our infrastructure between upload and scan. Customers can only retrieve their own org's uploads. The risk surface is "customer uploads malware their own scanner will flag" — exactly what the product is designed to do.

**Revisit when:** we add public download URLs, when uploads become a customer-facing distribution mechanism, or when free-tier abuse (e.g. uploading malware as a free file-host bypass) shows up.

**Commit:** `shieldscan-api` `d9b0fbf`.

### 2026-04-26 — Task 3.3: `MobileUpload.uploaded_by` changed from NOT NULL to nullable

**Schema correction.** M1 declared `mobile_uploads.uploaded_by` as `NOT NULL` referencing `users.id`. Task 3.3 enables API-key authentication on the upload endpoint per decision #5. API keys belong to an org, not a user — so the `NOT NULL` constraint was unsatisfiable on the API-key path.

**Resolution.** Migration `c8e3d2a4f9b1` drops `NOT NULL`. JWT upload writes `uploaded_by = identity.user.id`; API-key upload writes `uploaded_by = NULL` and surfaces the actor identity via the audit row's `details.api_key_prefix` instead.

**This is correction, not invention.** The audit-logs row already follows this exact pattern: `audit_logs.actor_id` has been nullable for API-key actions since M2 (Task 2.3.2). `mobile_uploads.uploaded_by` deviated from that established convention; this entry brings it back into line.

**API contract:** responses report `uploaded_by_user_id: UUID | None`. UI renders "uploaded via API key" when null. Pinned by `test_upload_via_api_key_writes_null_uploaded_by`.

**Downgrade safety:** the migration's downgrade restores `NOT NULL`. If any rows have `uploaded_by IS NULL` at downgrade time (i.e. an API-key upload landed under this revision), the ALTER fails loudly with `column "uploaded_by" contains null values` — operator must delete or backfill those rows before retrying. Silent data loss is the worse failure mode.

**Commit:** `shieldscan-api` `d9b0fbf`.

### 2026-04-26 — SPECIFICATION.md §6.2 header "Projects (7)" actually enumerates 8 endpoints

**Cosmetic typo, bookkeeping.** The §6.2 header reads "Projects (7)" but the body lists 8 endpoints (CRUD 5 + verify + the future 2 credential endpoints). Surfaced during Task 3.2 review; flagged here to bundle into a future docs cleanup pass rather than touch SPECIFICATION.md inside an unrelated commit.

**Not a 3.3 issue** — included in the 3.3 batch only because the entry batch is the natural place to track docs-bookkeeping items as they're spotted.

### 2026-04-26 — Task 3.3: new dep `python-multipart` 0.0.20

**New runtime dependency.** Added `python-multipart = "^0.0.20"` to `pyproject.toml` for FastAPI's `UploadFile` support — required mechanically by the file-upload endpoint, not a discretionary library choice.

**VERSIONS.md trace:** not yet listed in VERSIONS.md (it's a transitive-style dep that FastAPI doesn't pin itself but requires for multipart/form-data parsing). Add to VERSIONS.md in the next docs cleanup pass; pin range `^0.0.20` is conservative (0.0.x semver discipline).

**Commit:** `shieldscan-api` `d9b0fbf`.

### 2026-04-23 — Task 3.2: per-domain audit-action enum split

**Refactor.** The single `AuthAction` enum was accreting non-auth values (`EMAIL_VERIFICATION_SENT`, `PASSWORD_RESET_*`, etc.) — adding M3's project events (`PROJECT_CREATED`, `PROJECT_DOMAIN_VERIFIED`, …) would sprawl it further. Split now at the natural M3 boundary (the longer the split is deferred, the more painful it becomes).

**New shape:**
- `AuthAction` (unchanged — preserves every existing audit-row value).
- `ProjectAction` (new) for project-domain events.
- `audit()` widened to accept `AuthAction | ProjectAction`.
- `ALL_ACTION_ENUMS` registry tuple — single source of truth. Future enums (`ScanAction`, `FindingAction`, etc.) extend by appending.

**Load-bearing invariant:** string values must be globally unique across all action enums. Pinned by `test_action_enum_values_are_globally_disjoint` — verified to fail when given a duplicate value (manual sanity check during commit prep). Plus `test_action_enum_registry_lists_all_known_enums` catches a future engineer adding a new enum but forgetting to register it.

**Back-fill in same commit:** Task 3.1's create/patch/delete handlers shipped without audit (small gap). Closed here so the project-event audit trail is complete from M3 day one.

**Commit:** `shieldscan-api` `b84b7a3`.

### 2026-04-23 — Task 3.2: domain verification ships both DNS TXT + meta tag (plan literal)

**Decision.** `POST /v1/orgs/{org_id}/projects/{id}/verify` supports two methods: `dns_txt` and `meta_tag`. Plan literal IMPLEMENTATION-PLAN.md §3.2 specified both; my pre-task lean was DNS-only; user reversed that lean after weighing three arguments (MENA SMB shared-hosting reality, mitigable spoofing concern, plan-author judgment).

**Two security tightenings layered on top of the plan's literal code:**

1. **`follow_redirects=False`** on the meta-tag fetch. Plan snippet didn't specify either way; without this, an attacker who controls a redirect handler at the customer's `target_url` could redirect us to an attacker page that carries the right meta tag — verification would succeed for a domain the customer doesn't actually control. Pinned by `test_meta_tag_redirect_does_not_follow`.

2. **`html.parser` (stdlib) instead of regex** for meta-tag detection. Plan snippet used a regex (`re.search(r'<meta[^>]+name=...')`) that fails on common HTML variations (single-quoted attrs, attribute-order changes, self-closing tags). Structured parsing handles all four pinned-by-test variants without false negatives. No new dependency — the parser is in the stdlib.

**Failure-mode UX:** the 400 body includes `expected_record` showing the customer the EXACT TXT record / meta tag they need to add. Substantial UX win at zero security cost (the verification_token is already in `ProjectResponse` per Task 3.1 §H decision #2 — it's an org-internal value, not a credential).

**Discrete `failure_reason` codes** for ops differentiation: `dns_nxdomain`, `dns_no_answer`, `dns_timeout`, `dns_token_not_found`, `dns_error`, `meta_tag_not_found`, `meta_tag_token_mismatch`, `http_status_<n>`, `http_timeout`, `http_connect_error`, `html_parse_error`. Customer-facing — leaks DNS state about the customer's OWN domain, which is fine.

**Re-verify failure semantics:** a re-verify whose record has been removed returns 400 but does NOT downgrade `domain_verified` from True to False. DNS is occasionally flaky; one failure must not invalidate historical state. Per Task 3.2 user refinement on decision #3.

**No new ADR** — composition on plan literal with documented tightenings.

**Commit:** `shieldscan-api` `5f74cc5`. Cross-ref `dnspython ^2.7` pinned in VERSIONS.md.

### 2026-04-23 — Task 3.1: PATCH target_url auto-resets domain verification when root_domain changes

**Decision.** `PATCH /v1/orgs/{org_id}/projects/{id}` with a `target_url` field that produces a different `root_domain` automatically:
- Updates `project.root_domain` to the new derived value
- Sets `project.domain_verified = False`
- Mints a fresh `project.verification_token = uuid4().hex`

Path/port-only changes within the same `root_domain` do NOT reset verification — those are legitimate same-domain target updates.

**Rationale:** verifying a domain is a security claim ("this org controls example.com"). Allowing a verified `target_url` to silently move to `different.org` while keeping `domain_verified=True` would let any project member silently re-aim a verified target at a domain they don't actually control. Auto-resetting forces a re-verification round at Task 3.2's `/verify` endpoint.

**Pinned by tests:**
- `test_patch_target_url_same_domain_keeps_verification` (no reset on same-domain change)
- `test_patch_target_url_different_domain_resets_verification` (reset + token rotation on cross-domain)

**Commits:** `shieldscan-api` `fdcb0db`.

### 2026-04-23 — Task 3.1: GET on archived project returns 200, not 404

**Decision.** `GET /v1/orgs/{org_id}/projects/{id}` for an archived project returns **200 with `archived_at` populated** in the response body, not 404.

**Rationale:** soft-delete is a reversible state, not absent state. Archived projects retain scan history that users may legitimately reference. List endpoints filter archived out by default (`?archived=true` includes them), but single-project GET always returns the row regardless of archive status.

**This is also part of the broader pattern** ADR-012 establishes for app-layer scoping: cross-tenant lookups return 404 (existence leak avoidance), but in-tenant lookups return 200 even when the row is "soft-deleted." 410 Gone is reserved for the **DELETE→DELETE** double-action signal.

**Pinned by `test_get_archived_project_returns_200`.**

### 2026-04-23 — Task 3.1: tldextract PSL vendoring carry-forward to M12.5

`tldextract` (added in Task 3.1 for `extract_root_domain`) consults the Mozilla Public Suffix List. Default behavior fetches+caches the PSL on first use. For air-gapped on-prem deployments (M12.5+) we'll need to **ship a vendored snapshot** of the PSL alongside the application — the runtime fetch is not available in restricted-network environments.

**Action carried forward to M12.5:** as part of the on-prem deployment build, vendor a current PSL snapshot into the app distribution and configure `tldextract.TLDExtract(suffix_list_urls=(), cache_dir=<bundled>)` to use it. PSL refresh policy: re-vendor on each release; PSL changes glacially so manual refresh is fine.

Not blocking M3 / SaaS deployment (default PSL fetch works there).

### 2026-04-23 — Task 3.1: PATCH project — `extra="forbid"` blocks immutable fields at schema level

**Pattern note (not a deviation).** `ProjectUpdateRequest` uses `model_config = ConfigDict(extra="forbid")` so that any client sending `organization_id`, `root_domain`, `domain_verified`, `verification_token`, `id`, or `archived_at` in the PATCH body gets a 422 — these fields aren't on the schema at all.

**Why this matters as a pattern for M3+:** every PATCH endpoint shipped after this should use the same `extra="forbid"` posture. Without it, Pydantic v2's default (`extra="ignore"`) silently drops unknown fields, which means a client trying to write to an immutable column gets a 200 success response with no field change — confusing, and a security footgun (think: a client trying to PATCH `is_admin` on a member resource and getting "success" back).

**Convention going forward:** every UpdateRequest schema in M3+ MUST set `extra="forbid"`. Pinned in 3.1 by `test_patch_project_immutable_fields_rejected`.

### 2026-04-23 — Task 3.1: API-key URL spec misalignment carry-forward

**Carry-forward from M2.** `SPECIFICATION.md` §6.2 (line 466–467) lists API-key endpoints under `/orgs/:org_id/api-keys`. Task 2.4 shipped them at `/v1/auth/api-keys` (no `/orgs/:org_id/` prefix). The shipped paths are reasonable for an account-scoped resource where the org is implicit from the JWT, but they diverge from the spec literal.

**Resolution direction (refined 2026-04-30, Checkpoint 2):** rename code to match SPEC (Outcome B). Org-scoped form (`/v1/orgs/{org_id}/api-keys`) is consistent with every other tenant resource (`/orgs/:org_id/projects`, `/orgs/:org_id/scans`, `/orgs/:org_id/vulnerabilities`, etc.) and makes the tenant boundary explicit in the URL. Updating SPEC to match code (Outcome A) was considered but rejected — the org-scoped pattern is the load-bearing convention across §6.2 and is worth preserving.

**Why deferred from Checkpoint 2:** Checkpoint 2 is a docs-only commit (M4 → M5 transition consolidation pass). The rename requires a route prefix change in `src/app/routes/api_keys.py`, addition of the `org_id` path parameter (with mismatch-vs-JWT validation matching other org-scoped endpoints), test path updates across `tests/routes/test_api_keys.py`, and grep for any client/integration-test references. Bundling code changes into a docs commit was explicitly scoped out.

**Trigger for action:** next-touch on `routes/api_keys.py`, OR pre-launch consistency sweep, OR explicit task to align. Acceptable to defer because API-key paths are pre-launch (no customer integrations to break by renaming).

**Estimated effort:** ~2 hours — route prefix rename + `org_id` path-param addition with require-org-membership validation + test path updates (~12 tests touch the prefix) + grep-and-fix for any cross-references.

### 2026-04-23 — Task 2.Y closes SPECIFICATION/IMPLEMENTATION-PLAN gap

**Plan/spec misalignment, resolved.** SPECIFICATION.md §6 (lines 447–448) enumerated `/auth/forgot-password` + `/auth/reset-password` as part of the auth API surface alongside register/login/refresh/logout/verify-email. IMPLEMENTATION-PLAN.md never numbered these as a task — its M2 section stops at Task 2.5 (email verification flow).

**Per the CLAUDE.md document hierarchy** (SPECIFICATION wins over IMPLEMENTATION-PLAN on product truth), this is a contract gap to close, not scope creep to defer. M2 ships these endpoints as Task 2.Y to leave the spec and the implementation in agreement.

**Composition cost was small** because Task 2.X had just shipped the user-level revocation primitive (`set_user_revoked_before`) — reset is functionally a forced password change and reuses every existing pattern (token-issuance from 2.3.3, GETDEL atomic consumption from 2.5, no-enumeration policy from resend-verification, audit reserved enums from 2.3.2, rate-limit dual scope from login).

**Commit:** `shieldscan-api` `74cc9ee`.

### 2026-04-23 — Task 2.Y: reset token TTL = 1 hour

**Decision.** Password-reset tokens expire in 1 hour, set as `TOKEN_TTL_SECONDS = 3600` in `src/app/services/password_reset.py`.

**Reasoning:**
- Reset is the highest-value short-lived credential in the system — full account takeover on consumption.
- 24-hour TTL (the email-verification choice) was right for verification (verifying your own already-accessible email is a low-value target). It is wrong for reset.
- **Email-prefetch attack surface is real.** Outlook/Office365 and several spam scanners fetch URLs on email receipt; mail-relay logs may capture link contents. A 24h TTL means a compromised relay sees a usable token for 24h. 1h cuts that window 24×.
- **Industry norms cluster at 1h.** GitHub, Stripe, Auth0 default to 1 hour. Most banks shorten further (15–30 min); we don't go shorter because legitimate users may take longer to read mail on a slow client.
- Users who exceed 1h re-request. The cost of a fresh request is one more email; the cost of a 24h-window token in a compromised relay is account takeover.

### 2026-04-23 — Task 2.Y: new password may equal current on reset (NOT on change)

**Asymmetric policy, deliberate.** The two password-write endpoints differ on the "new equals current" check:

- `POST /auth/password/change`: REJECTS new == current with 400 `invalid_current_password` (merged with wrong-current per Task 2.X scope decision #2).
- `POST /auth/reset-password`: ALLOWS new == current. No check.

**Rationale:** the two endpoints prove different things.
- **Change** is a knowledge-of-current-password flow. The user demonstrably remembers their current password (they typed it in `current_password`). Forcing them to choose a different new password is reasonable: they know what they're avoiding.
- **Reset** is a knowledge-of-mailbox flow. By definition, the user does NOT remember their current password — that's why they're resetting. Asking them to type something different from a hash they can't produce would be theatre. Internally we'd have to call `verify_password(new, user.hashed_password)`; if the user happens to type their current password (which they don't remember anyway), we'd reject — pointlessly.

**Industry alignment:** GitHub, Google, AWS Cognito do not block "new password matches current" on reset. They DO on change. Same posture.

**Audit trail unaffected:** `auth.password.reset.completed` fires regardless. If the user resets to the same string, it's a noop in security terms but a deliberate event in the audit.

### 2026-04-23 — Task 2.Y: password reset does NOT auto-verify email

**Pinned by `tests/routes/test_password_reset.py::test_18_reset_does_not_change_email_verified_status` with explicit failure-message comment.**

Reset proves mailbox ownership (same mechanism as email verification). The two could be coupled — "if you can reset, you've proven mailbox; therefore mark email as verified." We deliberately do NOT.

**Reasons:**
- Verification is an affirmative, semantic action separate from reset. A user clicking "I confirm this is my email" is different from a user clicking "let me reset because I'm locked out."
- Auto-verifying on reset silently changes account state. Silent state changes are surprising — and the test docstring explicitly says: changing this is a policy decision, not a refactor.
- **Future email-change flows would be complicated.** If the user changes their primary email, reset tokens for the OLD email could end up auto-verifying the NEW one without consent. Cleaner to keep verification an explicit user choice.
- The cost of NOT auto-verifying is one extra click ("now click the verify link"). Tiny.

**If anyone reverses this in the future:** add an ADR documenting the reasoning, OR a DRIFT-LOG entry with the new design + migration plan for any users affected. Don't quietly swap the behavior.

### 2026-04-23 — Task 2.X: reuse-detection auto-trigger of user-level revocation deliberately deferred

**Decision at Task 2.X close:** the `user_revoked_before:{user_id}` mechanism shipped, but the `/auth/refresh` reuse-detection path does NOT auto-trigger it. Reuse-detection remains **targeted-jti-only** — only the reused token is revoked; the user's other outstanding tokens stay valid.

**Considerations against auto-trigger:**
- **False positives from network blips:** a client retrying a request after a transient timeout might legitimately replay the same refresh token. Auto-revoking all the user's sessions on every blip-induced retry would be hostile UX.
- **Observability loss:** once we auto-blow-away all sessions, we lose the per-jti reuse signal that tells us *which* token was replayed. The audit row stays, but the pattern of "this user has had 3 reuse events in 10 minutes" is the kind of intel that would get muddied.
- **No undo path:** `set_user_revoked_before` is one-shot. If we trigger on a false positive, the user has to log in again on every device. We can't "un-set" the revocation timestamp meaningfully.
- **Revocation storms:** a buggy client library that mishandles refresh-rotation could create cascading revocation events across all of its users — self-amplifying, hard to diagnose.

**Future ADR required** to revisit. Considerations for that ADR: per-source rate-limiting (only auto-trigger on >N reuses within M minutes), a "soft" revocation that still allows the current valid token to keep working (would need a different mechanism), or sticking with targeted-jti for the foreseeable future.

**Approved:** user, 2026-04-23.
**Commits:** `shieldscan-api` `07d0189`, `1bb0b66`.

### 2026-04-23 — Task 2.X: no caching layer for `user_revoked_before` reads

**Decision at Task 2.X close:** the auth path's `get_user_revoked_before` Redis GET runs uncached on every authenticated request. Approved short-circuit (Option C) — no further optimization for M2.

**Performance characterization:** Redis GET on a nonexistent key is sub-millisecond and O(1). The 99.9% no-revocation path costs roughly the same as the existing jti-revocation `EXISTS` check we already accepted. The set-revocation path runs once per password-change event (rare).

**If profiling ever shows latency matters,** consider in this order:
1. **Per-request memoization** — if multiple deps in the same request resolve the auth path, cache the result on the request object. Cheapest, no invalidation surface.
2. **Pipeline jti+user GETs** — batch `is_revoked` and `get_user_revoked_before` into a single Redis round-trip. Saves the second TCP round-trip; no invalidation issues.
3. **DO NOT** add a TTL'd in-memory cache behind the DB / pre-Redis layer. Cache invalidation on password change would double the invalidation surface for zero gain — the password-change handler would have to invalidate every API pod's local cache, which is exactly the problem we just solved by going to Redis in the first place.

**Approved:** user, 2026-04-23.

### 2026-04-21 — Task 2.5: email normalization (latent bug fix)

**Pre-empted by Task 2.5 design review.** Pydantic EmailStr lowercases the domain but leaves the local part case-sensitive (verified: `Alice@Example.COM` → `Alice@example.com` — `Alice` preserved). Without app-layer normalization, register + login were case-sensitive at the local part, enabling:

- Duplicate user creation: `Alice@x.com` and `alice@x.com` hit the `UNIQUE` index as different bytes and both registered successfully.
- Login lock-out: user registers as `Alice@x.com`, types `alice@x.com` at login, `User.email == req.email` returns 0 rows → 401.

**Three-layer fix (shieldscan-api `d4fdc96`):**

1. `normalize_email()` helper + Pydantic `@field_validator` on every `EmailStr` field at request boundary.
2. Defensive `.strip().lower()` in `register_identity` for non-HTTP callers (CLI, background jobs).
3. Alembic rev `b7f2a9c1e8d4`: `CHECK (email = lower(email))` on `users` — catches ORM bypass (raw SQL, SECURITY DEFINER).

Three regression tests cover each layer. No data migration (pre-launch, zero production users).

### 2026-04-21 — Test harness artifact: shared-session aborted-tx visibility

**Bookkeeping entry — not a deviation.** `tests/conftest.py`'s shared `db_session` pattern causes an `IntegrityError` in one client request to abort the outer SQLAlchemy transaction on the shared session. Post-failure `SELECT`s from the same session then see either zero rows OR require a compensating `rollback()` that also unwinds the prior successful request's data.

**Isolated to test observability, not behavior.** Production code paths use a fresh `AsyncSessionLocal()` per request, so aborted-tx state never crosses request boundaries.

**When a test needs to verify state AFTER an expected-failure request**, either:
- Rely on the HTTP response shape as proof (UserResponse body serializes committed ORM rows; if the body contains the expected email, the row exists), OR
- Open a fresh `AsyncSession` bound to the test engine (not the shared fixture session) for the verification query.

Encountered during Task 2.5 Commit 1a while building `test_register_rejects_mixed_case_duplicate`. Worked around by asserting the response-body shape instead of a post-register DB query.

### 2026-04-21 — Task 2.5: plan commit-message deviation (SendGrid → ConsoleEmailService)

IMPLEMENTATION-PLAN §2.5 says `feat(auth): add email verification via SendGrid`. Actual commit message: `feat(auth): add email verification consumption endpoints`.

**Why deviated:** M2 envelope item 5 explicitly scopes email to the `ConsoleEmailService` stub for M2; real SMTP (SendGrid or Resend) is deferred to M12.5 (see DRIFT-LOG 2026-04-21 M12.5-prerequisite entry from Task 2.4). Using the plan's literal message would misrepresent what shipped — no SendGrid code exists in the commit.

**Approved:** user, 2026-04-21.
**Commit:** `b3b7ec0` (shieldscan-api).

### 2026-04-21 — M1 schema tension: `audit_logs.actor_id ON DELETE SET NULL` conflicts with append-only trigger

Encountered during Task 2.5 test development. Deleting a `User` row cascades via `audit_logs.actor_id` (`ON DELETE SET NULL` per M1.5) — which issues an `UPDATE audit_logs SET actor_id = NULL`, which the append-only trigger from M1.6 rejects with `"audit_logs is append-only: UPDATE operations are forbidden"`.

**Net effect:** a User cannot be deleted as long as any `audit_logs` row references them as `actor_id`. Which is every auth event. Which is every User who has ever registered.

**Not a Task 2.5 issue — M1 design tension.** Out of scope for 2.5. Logged for future attention. Candidate resolutions when it becomes a real problem (user-deletion feature, GDPR data purge, admin offboarding):

- Option A: NULL-out `actor_id` via a bypass function (SECURITY DEFINER path that skips the trigger for `actor_id` column only). Clean.
- Option B: Drop the `ON DELETE SET NULL` and retain `actor_id` as a historical pointer even if the user is gone (no FK). Ugly data model.
- Option C: Soft-delete users (`deleted_at` column, no row removal). Standard practice for audit-heavy systems.

**Option C is likely the right answer** when we get there. Log in the GDPR-compliance milestone.

### 2026-04-21 — Task 2.4: intentional breaking change to `/auth/me` response shape

**Pre-launch breaking change, documented for the audit trail.**

Task 2.3 shipped `/v1/auth/me` returning a flat `UserResponse`:

```json
{"id": "...", "email": "...", "full_name": "...", "email_verified": true}
```

Task 2.4 reshapes it to `AuthIdentityResponse` — a discriminated-union on `kind` — to accommodate API-key-credentialed callers (per Task 2.4 endpoint policy: API keys are allowed on `/auth/me` for CI-verify use cases). The new shape:

```json
{
  "kind": "jwt" | "api_key",
  "organization": {"id": "...", "name": "...", "slug": "..."},
  "user":    { ...UserResponse... } | null,
  "api_key": { ...APIKeySummary... } | null
}
```

**Why it's the right call now:**
- Pre-launch — no external clients yet. Reshape cost: zero.
- Alternative A (flat UserResponse with synthetic user for API-key callers) creates the "optional-field soup" problem we avoided with scopes (deferred in 2.4).
- Alternative C (new `/auth/whoami` endpoint, leave `/auth/me` JWT-only) splits the client-side branch into "check credential kind first, then call the right endpoint." Worse UX; the discriminated-union shape collapses the branch to one endpoint + one `switch (body.kind)`.
- Alternative B (chosen) is a discriminated union at the schema level rather than optional-field soup — clients branch on `kind`, and the field absent at serialization is `null`, not missing.

**What changed in code:**
- `/v1/auth/me` dep swapped from `get_current_user` (JWT-only) to `get_auth_identity` (either kind).
- New response schemas: `AuthIdentityResponse`, `OrganizationSummary` in `src/app/schemas/api_keys.py`.
- Existing test `test_me_returns_user` renamed to `test_me_returns_identity_for_jwt` and updated for the new shape; new `test_me_returns_identity_for_api_key` added.

**Post-launch discipline:** once external clients exist, changes to `/auth/me` (or any public response shape) go through versioned endpoints (`/v2/auth/me`) or additive-only field additions. This Task 2.4 reshape used the pre-launch window deliberately; future reshapes won't have that option.

**Commits:** `shieldscan-api` `af4c1d9`.

### 2026-04-21 — RLS UUID-cast behavior is a security positive (not a deviation)

Discovery during Task 2.4 Commit 0 while writing the CREDENTIAL_INDEXED_TABLES regression guard: the `tenant_isolation` RLS policy uses `current_setting('app.current_org_id')::uuid` in its `USING` clause, which raises `invalid input syntax for type uuid: ""` when the GUC is unset (empty string) rather than silently returning zero rows.

This is a **stronger security mode** than silent-zero-rows:
- Silent zero rows: ambiguous — "no data exists" vs "not authorized" look the same to the caller.
- Loud UUID cast error: unambiguous — "you tried to query tenant data without tenant context" is distinguishable from "you queried data that doesn't exist."

Legitimate code paths always set the GUC (orchestrator, endpoints, tests via `use_org`); accidental bypass attempts produce loud `DBAPIError`s that fail noisily rather than silently succeed with empty results.

**Code at risk:** any caller that swallows `DBAPIError` and returns `[]` would mask a tenant-context bug. Reviewed during Task 2.4 Commit 0 — our error-handling paths all propagate `DBAPIError` upward.

**Not a deviation — a discovery.** Logging here so when someone asks in 12 months "what happens if `current_org_id` isn't set in production?" the answer is recorded.

### 2026-04-21 — Dummy-bcrypt timing test: security-test-suite candidate

**Decision at Task 2.3 close:** skip the timing-distribution test for `login_nonexistent_user` vs `login_wrong_password` in the unit-test suite. CI-flaky; the mechanism is already locked by `test_login_runs_verify_password_on_nonexistent_user` which asserts the code path is taken.

**Follow-up:** actual timing-distribution testing belongs in a separate **performance/security test suite** (tentatively `tests/security/` with `@pytest.mark.security` marker, runs in a dedicated CI job with a quiet machine). Candidate tests to land there when the suite is created:
- `test_login_timing_distribution_no_enumeration` — 100 logins of each kind (wrong-password, no-such-user), assert p95 times overlap within 50ms.
- Similar distribution tests for other constant-time-critical paths (`verify_api_key`, future HMAC webhook verification).

**Not a blocker.** Not a Task-level item. File for the ops-hardening milestone.

### 2026-04-21 — M12.5 prerequisite: email send failure retry + DLQ

**Deferred from Task 2.3 with explicit M12.5 tie.** `ConsoleEmailService` stub cannot fail; real SMTP (M12.5) can. First task in M12.5 MUST add retry + audit-correction-on-failure for the email send path in `identity.py` (`register_identity` post-commit email dispatch) and any future email-sending callsite.

**Why audit-correction matters:** Task 2.3 audits `EMAIL_VERIFICATION_SENT` inside the register transaction, before the email actually fires. A silent email-delivery failure post-commit would leave the audit log saying "sent" when nothing was sent. M12.5 must close this by either (a) moving the audit to post-send-success, or (b) adding an `EMAIL_DELIVERY_FAILED` compensating audit action and firing it on SMTP failure.

**Log this in OPERATIONS-RUNBOOK §11.6** next time that file is edited — no standalone docs commit per user direction 2026-04-21.

### 2026-04-21 — M1 latent shadowing bug: `src/app/db.py` unreachable

**Discovered during Sub-task 2.3.4 Commit 2** when `get_db` needed `AsyncSessionLocal`: `src/app/db.py` was shadowed by the `src/app/db/` package (empty `__init__.py` took precedence). The file had been dead since M1 — no imports, no test coverage, no runtime reach. All 107 M1-era tests passed because nothing depended on the orphaned symbols.

**Fix:** consolidated `engine` + `AsyncSessionLocal` into `src/app/db/__init__.py`; deleted `db.py`. No behavior change (nothing previously used the orphan). `shieldscan-api` commit `f0e9d68`.

**Lesson:** *"tests pass" does not imply "all code is reachable."* Type-checkers (mypy) *might* catch this for typed imports, but files that are never imported anywhere are invisible to every tool in the default toolchain.

**Follow-up shipped:** `tests/test_module_reachability.py` (in `shieldscan-api` commit `711dfbc`) — a parametrized test that imports every `src/app/**/*.py` by dotted name. Catches the "shadowed orphan" class of bug immediately on any future occurrence. 31 cases at commit time; auto-grows with the codebase.

### 2026-04-21 — Sub-task 2.3.4: /logout revokes refresh only (not access)

**Policy pin.** `/auth/logout` revokes the refresh token's `jti` but leaves the access token alive until its natural 15-minute expiry. Rationale: revoking both would require the client to supply the access jti in the logout body (it's in the Authorization header, but header values are not body payload), AND an access token surviving 15 minutes after logout is a bounded window that the refresh-token revocation has already closed the door on (no new access tokens can be minted). Clients needing immediate full invalidation use the (forthcoming) password-change endpoint.

Regression guard: `test_logout_does_not_revoke_access_token` in `shieldscan-api` commit `711dfbc`. If someone changes the policy, this test fails loudly and forces the decision to be explicit.

### 2026-04-21 — Sub-task 2.3.3: slug-collision retry policy

**Decision.** Organization slug collisions during register are handled by retry with an integer suffix: `acme` → `acme-2` → `acme-3` → `acme-4` → `acme-5`. Cap at **5 attempts** (`MAX_SLUG_ATTEMPTS`); on exhaustion the orchestrator raises `SlugGenerationFailed`, which the endpoint layer translates to a generic HTTP 400 (no information leak per M2 envelope item 6a).

**Mechanism:** each attempt runs inside `async with db.begin_nested()` (PostgreSQL SAVEPOINT). A `UNIQUE` violation on `organizations.slug` rolls back the savepoint without poisoning the surrounding transaction, so the next candidate gets a clean slate. Approved by user 2026-04-21.

**Failure semantics:** exhaustion is legitimately vanishingly rare in production (6 distinct orgs choosing names that slugify to the same base within one namespace). On the miss, the caller retries with a modified organization name. No need for randomized suffixes or a dedicated collision table.

**Commit:** `3260561` (shieldscan-api).

### 2026-04-21 — Sub-task 2.3.3: email verification uses Redis-backed tokens (plan §2.5 literal)

**Not a deviation.** IMPLEMENTATION-PLAN §2.5 specifies "Token stored in Redis with 24h TTL." Sub-task 2.3.3 pre-implements the token issuance half of that design (orchestrator mints + emails the token) so `register` can emit a real verification flow from day one. Task 2.5 later adds the consumption endpoint (`POST /auth/verify-email`).

**Key shape:** `email_verify:{sha256(token)}` → `user_id` with `EX 86400`. Chosen to sit alongside `ratelimit:*` for consistent ops `SCAN MATCH` patterns (RUNBOOK §11.6).

**Audit entry scope (DELIBERATE omission):** `auth.email.verification_sent` audit rows carry `details = {"method": "email"}` only — **no token, no hash, no prefix**. Rationale: audit retention is 7 years (CONSTITUTION §14.6); verification tokens live 24 hours. Storing any portion of the hash creates a permanent cross-reference to Redis keys that are already gone — pure noise for SIEM consumers, potential footgun for future forensics ("why is this hash prefix showing up in audit_logs?").

**Commit:** `3260561` (shieldscan-api).

### 2026-04-21 — Secure comparison of security-sensitive values (pattern)

**Pattern, not a one-off.** Any plan literal using `==` (or equivalent) to compare security-sensitive values — tokens, hashes, HMACs, API keys, session IDs, signing-key-derived values — must be replaced with `hmac.compare_digest` (Python stdlib). Applies retroactively to code review and forward to all future tasks.

**Rationale:** `==` on strings/bytes short-circuits on the first mismatching byte. Timing differences across requests let an attacker recover the secret byte-by-byte. `hmac.compare_digest` runs in time dependent only on the shorter operand's length, not its contents.

**Applies to:** `verify_api_key` (Task 2.2, applied), any future token validators (Task 2.3+ refresh-token rotation, email verification tokens, password reset tokens), HMAC webhook signatures (Stripe in M10, any inbound signed callback).

**Does NOT apply to:** comparing non-secret public values (email addresses, usernames, URLs, organization slugs). `==` is correct there — constant-time comparison of public strings costs performance without adding security.

### 2026-04-21 — Task 2.2: verify_api_key uses hmac.compare_digest (not ==)

**Minor deviation.** IMPLEMENTATION-PLAN.md §2.2 literal:

```python
def verify_api_key(plain: str, stored_hash: str) -> bool:
    return hashlib.sha256(plain.encode()).hexdigest() == stored_hash
```

**Actual implementation:** `hmac.compare_digest(candidate_hash, stored_hash)`.

**Reason:** `==` on the candidate hash vs stored hash is vulnerable to timing analysis on the API-key auth path. `compare_digest` is the correct stdlib primitive for comparing secrets. Not ADR-worthy — this is implementation-of-existing-requirement (secure comparison), not new architecture. Falls under the "secure comparison of security-sensitive values" pattern logged above.

**Applied:** `shieldscan-api/src/app/services/api_keys.py`.
**Approved:** user, 2026-04-21.
**Commit:** `ace008e` (shieldscan-api).

### 2026-04-21 — Task 2.1: direct bcrypt (replacing passlib)

**Plan deviation.** IMPLEMENTATION-PLAN.md §2.1 imports `passlib.context.CryptContext`. Actual implementation uses the `bcrypt` library directly.

**Reason:** passlib 1.7.4 (its last release, Oct 2020) probes `bcrypt.__about__.__version__` — an attribute removed in bcrypt 4.1. With the installed bcrypt 5.0.0 the probe fails silently and passlib raises a misleading `ValueError: password cannot be longer than 72 bytes` on any input. passlib is effectively unmaintained (5+ years no release).

**Options considered at Task 2.1:**
- **A.** Pin `bcrypt < 4.1` — rejected, pins to an EOL bcrypt line and creates future CVE exposure.
- **B.** Monkey-patch `bcrypt.__about__` — rejected, encodes a workaround in product code.
- **C.** Drop passlib, use `bcrypt` directly — **chosen.** 8 LoC, PyPA-maintained, future-proof. The spec-level requirement ("bcrypt cost=12 hashing") is satisfied identically.

**Applied:**
- `shieldscan-api/pyproject.toml`: `passlib` removed, `bcrypt = "^5.0"` added.
- `VERSIONS.md §2.3`: same swap (docs commit `d75ff51`).
- `SPECIFICATION.md §13`: ADR-010 added (docs commit `4e6099e`).
- `shieldscan-api/src/app/services/auth.py`: calls `bcrypt.hashpw` / `bcrypt.checkpw` directly.

**Consequence:** the bcrypt 72-byte password-length limit is now an API-layer concern — Task 2.2 register endpoint must reject passwords >72 bytes with HTTP 400. `hash_password`/`verify_password` do not enforce.

**Approved:** user, 2026-04-21.
**ADR:** 010 (SPECIFICATION.md §13).
**Commit:** `f6b9bbf` (shieldscan-api).

---

## Closed

### 2026-04-20 — M1.5 absorbs secret_verified column

M1.5 absorbs ADDENDUM-TOOLS-5 §8 `secret_verified` column (was scheduled for M6.5.0). Rationale: nullable column, zero cost, eliminates forward reference. Approved by user 2026-04-20. Partial index `idx_raw_findings_verified_secret ON raw_findings(scan_id) WHERE secret_verified = TRUE` ships in the same M1.5 migration. M6.5.0 task is reduced to a verification step.
