# Findings-Ingest Task — Design Doc

**Date:** 2026-05-03
**Author:** Mahmoud Hassan (Odyssey Technology) with Claude
**Status:** Approved design; pending implementation
**Related:** Closes ADR-024 §3.2 columns-ready posture; ADR-017 Task 5.5
cross-repo scope; M9 prerequisite work
**Cross-references:** ADR-013 (Python sole writer), ADR-017 (findings
inline in job_completed events with sequencing), ADR-024 (RawFinding
schema extension), SPEC §7.3 (Job Completion wire format), SPEC §5.2
(Row-Level Security)

---

## 0. Executive Summary

Implement the Python-side findings-ingest path that consumes
`event["findings"]` arrays from `job_completed` events and persists
them to the `raw_findings` PostgreSQL table, in the same database
transaction as the `ScanJob.status` update.

**Closes two carry-forward concerns:**
1. **ADR-024 §3.2 "Python ingest scope" columns-ready posture.** Phase 2
   of M6-close-followup landed the 4 new columns
   (References, Tags, CVSSVector, AdditionalCWEs) but no consumer reads
   them. This task lands the consumer.
2. **ADR-017 Task 5.5 cross-repo scope.** ADR-017 line 1452 explicitly
   anticipated this work at "Task 5.5 cross-repo" scope; only the
   engine-side landed at M5 close. This is overdue closure of
   ADR-017's full contract.

**Single architectural artifact: ADR-025.**

**Cross-repo coordination: NONE.** Single-repo task in
`shieldscan-api`. Engine side already emits the canonical wire format
(per SPEC §7.3 + Phase 3 of M6-close-followup); Python side now reads
it.

**Coordination shape: Approach C** — single atomic commit covering
Pydantic schema + CompletionsConsumer extension + accumulator +
sequencing + tests + DRIFT entries.

**Estimated total work: 7-9 hours.**

**Forcing-function test pin** explicitly specified by ADR-017 line
1474-1475: `test_consumer_aggregates_sequenced_findings` with 3
sequenced events + 2500 findings + single-txn commit assertion.
Mandatory; failure to include violates ADR-017 forcing-function
contract.

---

## 1. Brainstorming Output (Locked Decisions)

The brainstorming session that produced this design locked five
decisions through five clarifying questions:

| # | Decision | Lock |
|---|----------|------|
| 1 | Findings-ingest scope | **Spec-complete** (ADR-017 sequencing fully implemented; idempotency + ghost-queued janitor deferred to OPS milestone) |
| 2 | Accumulator implementation | **Service-class instance attribute** on CompletionsConsumer (DI-friendly; testable in isolation; symmetric with existing session_factory DI pattern from M4 Task 4.2) |
| 3 | RLS propagation | **Session-local `app.current_org_id`** before insert (defense-in-depth; matches SPEC §5.2 default posture) |
| 4 | Bulk insert mechanism | **`insert(RawFinding).values([...])`** SQLAlchemy 2.0 Core construct (5-10x faster than ORM `add_all`; clean RLS interaction) |
| 5 | Failure mode | **Strict atomic** (single transaction per event; findings + ScanJob.status commit together; honors ADR-017 "preserves ADR-013 sole-writer atomicity" invariant) |

Plus three additional design-confirmation locks:

| # | Decision | Lock |
|---|----------|------|
| 6 | Approach | **C** — single atomic commit (matches M5/M6 single-repo task pattern) |
| 7 | ADR-025 title | **(a)** — descriptive: "CompletionsConsumer findings-ingest path — Pydantic schema + sequenced accumulator + RLS" |
| 8 | Pydantic strictness | **`extra="forbid"`** — mirrors engine `DisallowUnknownFields` invariant; cross-repo schema-drift regression guard |

**Asymmetric-cost meta-principle invoked:** The 4th project-corpus
invocation (after ADR-022 recon-as-helpers, ADR-023 NativeRunner
OutputFile mode, ADR-024 schema extension). ADR-025 will document the
4th instance + reference the prior three.

---

## 2. Problem Statement

### 2.1 Current State (Post-M6-Close-Followup)

After SPEC §7.3 followup landed (commits 59b0f3d, 8f90531, 938ae80,
8fbd085, 16071fe), the cross-repo state is:

**Engine side (production-ready):**
- `events.RawFinding` Go struct contains canonical schema with 4 new
  fields (References, Tags, CVSSVector, AdditionalCWEs)
- 6 tools (Nuclei, Semgrep, Gitleaks, Dep-Check, Checkov, Wapiti)
  populate new fields where source data exists
- Engine emits `job_completed` events with full RawFinding payloads via
  Redis Pub/Sub channel `shieldscan:completions`
- 1000-finding `MaxFindingsPerEvent` cap enforced (per ADR-017 forcing
  function)
- `event_seq` field present on every event for sequencing
- Wire-format omitempty behavior verified end-to-end (Phase 4)

**Python side (incomplete):**
- `app.models.raw_findings.RawFinding` SQLAlchemy model has all schema
  columns ready (Phase 2 of M6-close-followup; commit 938ae80)
- Alembic migration `49e83eb3587c` populated columns
- **`app.services.completions_consumer.CompletionsConsumer.handle_event`
  does NOT insert RawFinding rows** — currently only updates
  `ScanJob.status` + `ScanJob.finding_count` counter from
  `event["finding_count"]`
- No Pydantic schema for RawFinding (no `app/schemas/raw_findings.py`)
- No accumulator state for sequenced events
- No ingest test fixtures (`tests/fixtures/job_completed_*.json`)
- No ingest test coverage

### 2.2 The Architectural Gap

ADR-013 (Python sole writer) requires: Go workers don't write to
PostgreSQL; Python is the sole writer.

ADR-017 (findings inline in job_completed events) specifies:
*"The Python `CompletionsConsumer` (Task 4.2, lifespan-managed, runs
under the sole-writer DB role) inserts the findings rows in the same
transaction as the `ScanJob.status` update."*

**This contract is currently unmet.** Engine emits findings; Python
discards them (only reads `finding_count`). M9 AI pipeline (which
operates on `raw_findings` table per SPEC §8.1) has nothing to operate
on.

### 2.3 Why This Is Overdue Work

ADR-017 line 1452 explicitly anticipates this work:
*"Existing CompletionsConsumer shape (Task 4.2, lifespan-managed,
session_factory DI) needs minimal extension: parse `findings` field,
parse `event_seq`, accumulate when needed, persist on terminal batch.
**Extension is Task 5.5 scope, not Task 5.1.**"*

**Task 5.5 history:** The engine-side emitter (Worker.Process publishing
findings via completions) landed at M5 close (commit 232e878 region).
The cross-repo Python-side consumer extension was deferred — initially
to "Task 5.5 cross-repo" coordination, then implicitly to a later
milestone.

This is genuinely overdue. ADR-017's "Forcing functions" section
specifies the cross-repo test pin:
*"Python CompletionsConsumer test pin (Task 5.5 cross-repo):
`test_consumer_aggregates_sequenced_findings` synthesizes 3 sequenced
events, asserts single-txn commit on terminal batch with all 2500
findings persisted."*

This test does not exist yet. ADR-017's forcing function is
unsatisfied. Findings-ingest task closes it.

### 2.4 M9 Prerequisite Status

SPEC §8.1 AI Analysis Pipeline starts with: *"Raw findings
(PostgreSQL) ↓ [1] Embed findings → OpenAI text-embedding-3-small"*

M9 reads from the `raw_findings` PostgreSQL table. With findings-ingest
unimplemented, M9 has zero rows to read. M9 implementation requires
findings-ingest to be operational.

### 2.5 Operational Significance

Pre-launch, no production scans run; findings-ingest gap is invisible.

**Post-launch implication:** First customer scan completes → M9 pipeline
fires → empty raw_findings table → M9 produces zero vulnerabilities →
customer dashboard empty → product appears broken.

This is the failure mode that findings-ingest closes.

---

## 3. Solution Architecture

### 3.1 Component Inventory

The solution consists of 4 components, all in `shieldscan-api`:

| # | Component | Status | Purpose |
|---|-----------|--------|---------|
| 1 | `app/schemas/raw_findings.py` | **NEW** | Pydantic schema for ingest-side validation |
| 2 | `CompletionsConsumer.handle_event` | **EXTEND** | Parse `event["findings"]` + `event["event_seq"]`; route to single-event vs sequenced-event paths |
| 3 | `CompletionsConsumer._accumulator` | **NEW** | Service-class instance attribute holding sequenced batches keyed by `(scan_id, job_id)` |
| 4 | `tests/services/test_completions_consumer_findings_ingest.py` | **NEW** | Ingest test coverage + ADR-017 forcing-function test pin |

### 3.2 Pydantic Schema (Component 1)

**File:** `src/app/schemas/raw_findings.py` (new)

```python
"""Pydantic schemas for RawFinding ingest path.

Mirrors the SQLAlchemy model app.models.raw_findings.RawFinding.
Used by CompletionsConsumer to validate event["findings"] entries
before bulk-insert.

Per ADR-025 Pydantic strictness decision: extra="forbid" mirrors
engine DisallowUnknownFields invariant. Cross-repo schema-drift
regression guard — if Engine adds a field, Python rejects until
schema updated; forces explicit cross-repo coordination.

Per ADR-017 + ADR-024:
- All 4 schema-extension fields (References, Tags, CVSSVector,
  AdditionalCWEs) are Optional with None defaults.
- Field aliases match engine wire-format (snake_case JSON tags).
"""

from datetime import datetime
from typing import Literal, Optional
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field


class RawFindingCreate(BaseModel):
    """Pydantic schema for creating a RawFinding row from
    job_completed event findings array."""
    
    model_config = ConfigDict(
        extra="forbid",  # Mirrors engine DisallowUnknownFields
        str_strip_whitespace=True,
    )
    
    # Identity (required)
    tool_name: str
    finding_type: str
    
    # Classification (required + optional CWE)
    severity: Literal["critical", "high", "medium", "low", "info"]
    cwe_id: Optional[str] = None
    
    # SPEC §7.3 schema extension fields (M6-close-followup, ADR-024)
    references: Optional[list[str]] = None
    tags: Optional[list[str]] = None
    cvss_vector: Optional[str] = None
    additional_cwes: Optional[list[str]] = None
    
    # Web evidence (DAST, SAST web)
    target_url: Optional[str] = None
    request: Optional[str] = None
    response: Optional[str] = None
    
    # Source evidence (SAST code)
    code_file: Optional[str] = None
    code_line: Optional[int] = None
    code_snippet: Optional[str] = None
    
    # Mobile evidence (per SPEC §5.3 raw_findings extensions)
    mobile_os: Optional[Literal["android", "ios"]] = None
    mobile_permission: Optional[str] = None
    mobile_component_name: Optional[str] = None
    
    # SSL evidence
    ssl_cert_subject: Optional[str] = None
    ssl_cipher_suite: Optional[str] = None
    
    # Metadata (required)
    engine_category: Literal[
        "dast", "sast", "sca", "mobile", "infrastructure",
        "recon", "ssl", "api", "iac", "secrets", "container"
    ]
    fingerprint: str
    discovered_at: datetime
    
    # Secrets-specific
    secret_verified: Optional[bool] = None
```

**ADAPTATION REQUIRED at implementation time:** The exact field list
must mirror the actual SQLAlchemy model post-Phase-2. The above is
the canonical shape per SPEC §7.3 + ADR-024 + Phase 2 commit; verify
against `src/app/models/raw_findings.py` head state before drafting
the Pydantic schema. Surface deviations.

### 3.3 CompletionsConsumer Extension (Component 2)

**File:** `src/app/services/completions_consumer.py` (extend)

**Current state (M4 Task 4.2):**
```python
class CompletionsConsumer:
    def __init__(self, session_factory: async_sessionmaker[AsyncSession]):
        self._session_factory = session_factory
    
    async def handle_event(self, event: dict) -> None:
        # ... parse event ...
        # Updates ScanJob.status + ScanJob.finding_count counter
        # Does NOT insert findings
```

**Target state (post-findings-ingest):**
```python
class CompletionsConsumer:
    def __init__(self, session_factory: async_sessionmaker[AsyncSession]):
        self._session_factory = session_factory
        # ADR-025: accumulator for sequenced events (event_seq.total > 1)
        # Keyed by (scan_id, job_id); cleared on terminal batch.
        # Per ADR-017 line 1455: accumulator failure-mode is a
        # known limitation; ghost-queued janitor (OPS milestone)
        # handles process-restart-loss case.
        self._accumulator: dict[tuple[UUID, UUID], list[dict]] = {}
        self._accumulator_lock = asyncio.Lock()
    
    async def handle_event(self, event: dict) -> None:
        # Parse event_seq + status
        # Route to single-event vs sequenced-event path
        # Strict atomic txn: findings insert + ScanJob update together
        # Per ADR-017 line 1431 + ADR-025
```

**Three event paths:**

#### 3.3.1 Path A: `event_seq.total == 1` (single-event jobs)

```python
async def _handle_single_event(self, event: dict) -> None:
    """Single-event ingest: ≤1000 findings; insert + update in one txn."""
    findings_data = event.get("findings", [])
    
    # Validate findings via Pydantic
    findings = [RawFindingCreate.model_validate(f) for f in findings_data]
    
    # Fetch scan_job to derive organization_id for RLS
    async with self._session_factory() as session:
        scan_job = await self._fetch_scan_job(session, event["job_id"])
        org_id = scan_job.organization_id
        
        # Set RLS session-local
        await session.execute(
            text("SET LOCAL app.current_org_id = :org_id"),
            {"org_id": str(org_id)}
        )
        
        # Bulk insert findings via SQLAlchemy 2.0 Core construct
        if findings:
            finding_dicts = [
                {
                    **f.model_dump(),
                    "id": uuid4(),
                    "organization_id": org_id,
                    "scan_id": scan_job.scan_id,
                    "job_id": scan_job.id,
                }
                for f in findings
            ]
            await session.execute(insert(RawFinding).values(finding_dicts))
        
        # Update ScanJob.status + finding_count in same txn
        scan_job.status = event["status"]
        scan_job.finding_count = event["finding_count"]
        scan_job.duration_ms = event["duration_ms"]
        scan_job.completed_at = datetime.fromisoformat(event["timestamp"])
        
        await session.commit()
        # Atomic: findings insert + ScanJob update commit together.
        # If insert fails, ScanJob.status stays "running"; event will
        # be retried by Pub/Sub redelivery semantics (per ADR-017).
```

#### 3.3.2 Path B: `event_seq.total > 1` (sequenced events, intermediate)

```python
async def _handle_intermediate_event(self, event: dict) -> None:
    """Intermediate sequenced event: status='partial_findings'.
    Accumulate; do NOT update ScanJob status yet."""
    scan_id = UUID(event["scan_id"])
    job_id = UUID(event["job_id"])
    
    # Validate findings via Pydantic
    findings_data = event.get("findings", [])
    findings = [RawFindingCreate.model_validate(f) for f in findings_data]
    
    # Accumulate atomically
    async with self._accumulator_lock:
        key = (scan_id, job_id)
        accumulated = self._accumulator.setdefault(key, [])
        accumulated.extend(f.model_dump() for f in findings)
    
    # NO database write at intermediate batches.
    # NO ScanJob.status update — waits for terminal event.
    # event_seq.total > 1 means more events coming for this (scan_id, job_id).
```

#### 3.3.3 Path C: `event_seq.total > 1` (sequenced events, terminal)

```python
async def _handle_terminal_event(self, event: dict) -> None:
    """Terminal sequenced event: status='completed' or 'partial'.
    Insert union of all accumulated batches + this event's findings;
    update ScanJob; clear accumulator entry."""
    scan_id = UUID(event["scan_id"])
    job_id = UUID(event["job_id"])
    key = (scan_id, job_id)
    
    # Validate this event's findings
    findings_data = event.get("findings", [])
    findings = [RawFindingCreate.model_validate(f) for f in findings_data]
    
    # Lock accumulator; assemble full batch
    async with self._accumulator_lock:
        accumulated = self._accumulator.get(key, [])
        all_findings_data = accumulated + [f.model_dump() for f in findings]
    
    # Fetch scan_job for RLS + atomic insert+update
    async with self._session_factory() as session:
        scan_job = await self._fetch_scan_job(session, str(job_id))
        org_id = scan_job.organization_id
        
        await session.execute(
            text("SET LOCAL app.current_org_id = :org_id"),
            {"org_id": str(org_id)}
        )
        
        if all_findings_data:
            finding_dicts = [
                {
                    **f,
                    "id": uuid4(),
                    "organization_id": org_id,
                    "scan_id": scan_job.scan_id,
                    "job_id": scan_job.id,
                }
                for f in all_findings_data
            ]
            await session.execute(insert(RawFinding).values(finding_dicts))
        
        scan_job.status = event["status"]
        scan_job.finding_count = event["finding_count"]
        scan_job.duration_ms = event["duration_ms"]
        scan_job.completed_at = datetime.fromisoformat(event["timestamp"])
        
        await session.commit()
    
    # Clear accumulator AFTER successful commit
    # (if commit fails, accumulator stays so next attempt retains data)
    async with self._accumulator_lock:
        self._accumulator.pop(key, None)
```

**Critical detail on accumulator cleanup ordering:** Clear AFTER commit
succeeds. If commit fails (deadlock, constraint violation), accumulator
retains data. Pub/Sub redelivery re-fires the terminal event;
accumulated batches are still available for the retry.

**Trade-off:** If process crashes between commit success and accumulator
cleanup, next process restart sees an entry with "stale" data — but
the retry of the terminal event succeeds idempotently because findings
were already committed (DB has them; insert would conflict on
fingerprint+scan_id unique key OR result in duplicates).

**Honest acknowledgment:** Idempotency at the DB level is not yet
addressed in this task scope (deferred per brainstorming Decision 1).
The current design tolerates the "stale accumulator" edge case by
either:
- Letting duplicates accumulate (acceptable pre-launch; DB-side dedup
  can land later)
- Adding a process-startup accumulator-clear (zero state on restart;
  ghost-queued janitor handles ScanJob.status)

**Lean for implementation:** Process-startup clear (simpler; matches
ADR-017's "accumulator survives only the single API process"
explicit semantics).

### 3.4 Event Routing Logic

```python
async def handle_event(self, event: dict) -> None:
    """Route event to appropriate handler based on event_seq + status."""
    event_seq = event.get("event_seq", {"index": 1, "total": 1})
    total = event_seq["total"]
    status = event["status"]
    
    if status in ("failed", "canceled"):
        # No findings on these events; just update ScanJob status
        await self._handle_failed_or_canceled_event(event)
        return
    
    if total == 1:
        # Single-event job (≤1000 findings)
        await self._handle_single_event(event)
    elif status == "partial_findings":
        # Intermediate sequenced event
        await self._handle_intermediate_event(event)
    elif status in ("completed", "partial"):
        # Terminal sequenced event
        await self._handle_terminal_event(event)
    else:
        # Defensive: unknown status combination — log + skip
        logger.error(
            "completions_consumer.unknown_status_combo",
            status=status,
            event_seq=event_seq,
            scan_id=event.get("scan_id"),
        )
```

### 3.5 RLS Session-Local Pattern

Per Decision 3 (RLS via session-local `app.current_org_id`), every
findings-ingest path applies the same RLS preamble:

```python
await session.execute(
    text("SET LOCAL app.current_org_id = :org_id"),
    {"org_id": str(org_id)}
)
```

**Mechanics:**
- `SET LOCAL` applies to the current transaction only (auto-cleared
  on COMMIT/ROLLBACK)
- `app.current_org_id` is the conventional GUC name per SPEC §5.2
  example: `current_setting('app.current_org_id')::uuid`
- RLS policies on tenant-scoped tables filter rows by
  `organization_id = current_setting('app.current_org_id')::uuid`
- Both INSERT (raw_findings) and UPDATE (scan_jobs) inherit the
  setting

**Defense-in-depth:** If a bug introduces a query without proper
org_id scoping, RLS rejects the operation rather than leaking
cross-tenant data.

### 3.6 Bulk Insert Mechanics

Per Decision 4 (`insert().values([...])` SQLAlchemy 2.0 Core):

```python
from sqlalchemy import insert
from app.models.raw_findings import RawFinding

await session.execute(
    insert(RawFinding).values(finding_dicts)
)
```

**Performance:** ~30-80ms for 1000 findings (vs ~100-500ms for ORM
`add_all`); 5-10x faster.

**RLS interaction:** Bulk insert via Core construct still respects
session-level `SET LOCAL app.current_org_id` because the setting
applies to the entire transaction, not per-statement.

**ORM bypass:** This pattern bypasses ORM event hooks (e.g.,
`before_insert` listeners). Acceptable because raw_findings has no
ORM-level events; if listeners are added later, switch to ORM
`add_all` OR explicitly invoke listeners post-insert.

### 3.7 Strict Atomic Failure Semantics

Per Decision 5 + ADR-017's explicit Consequences invariant:

**On ingest failure (any reason):**
- Transaction rolls back automatically (SQLAlchemy default)
- `ScanJob.status` stays in current state (e.g., `running`)
- Pub/Sub Pubsub redelivery re-fires the event (Pub/Sub at-least-once
  semantics within MVP single-process posture)
- Consumer retries on next event delivery
- If retries exhaust → ScanJob ghost-queued → ghost-queued janitor
  sweeps (OPS milestone scope; carry-forward from M4 Task 4.2)

**Failure modes explicitly handled:**
1. **Pydantic validation error** (Engine emitted unknown field):
   - Validation fails → exception propagates → txn rollback
   - Cross-repo schema-drift indicator; surfaces immediately
2. **Constraint violation** (e.g., duplicate fingerprint):
   - DB rejects → txn rollback → retry-via-redelivery
   - Pre-launch: probably indicates idempotency edge case; surface
3. **Deadlock** (concurrent commits):
   - PG returns deadlock error → txn rollback → retry-via-redelivery
   - MVP scale: rare (single API process); becomes operational
     concern at scale
4. **OOM during accumulator** (sequenced event with massive
   accumulated state):
   - Process likely crashes → ScanJob ghost-queued → janitor sweep
   - Trigger for ADR-017 Option C migration (R2 staging); not
     handled in this task

**ADR-017 line 1442-1445 trigger conditions for migration to R2
staging Option C** are operational signals, not findings-ingest
implementation concerns. Findings-ingest implements the in-memory
Option A path; OPS milestone handles trigger detection +
operational alerting.

---

## 4. Component Detail

### 4.1 Pydantic Schema (`app/schemas/raw_findings.py`)

**Decisions captured:**
- `extra="forbid"` per Decision 8
- All 4 SPEC §7.3 schema-extension fields included (References,
  Tags, CVSSVector, AdditionalCWEs)
- Mobile + SSL evidence fields per SPEC §5.3 + raw_findings table
  extensions
- engine_category enum mirrors SQLAlchemy CHECK constraint
- secret_verified field for secrets-specific findings

**Validation behavior:**
- Required: tool_name, finding_type, severity, engine_category,
  fingerprint, discovered_at
- Optional with None defaults: all evidence fields, all 4 §7.3
  extension fields, mobile-specific fields, secret_verified
- Engine emissions without optional fields validate cleanly
  (omitempty on Engine; Optional on Python; symmetric)
- Engine emissions with unknown fields fail validation
  (extra="forbid"); surfaces cross-repo schema drift

**Test coverage targets:**
- Valid minimal payload (only required fields)
- Valid full payload (all fields populated)
- Valid with new §7.3 fields populated (References, Tags,
  CVSSVector, AdditionalCWEs)
- Invalid: missing required field
- Invalid: extra field (extra="forbid" rejection)
- Invalid: severity enum violation
- Invalid: engine_category enum violation

### 4.2 Accumulator State (`CompletionsConsumer._accumulator`)

**Decisions captured:**
- Service-class instance attribute per Decision 2
- `dict[tuple[UUID, UUID], list[dict]]` keyed by `(scan_id, job_id)`
- `asyncio.Lock` for concurrent safety within single process

**Lifecycle:**
- Created in `__init__`; lives for CompletionsConsumer instance
  lifetime
- Populated in `_handle_intermediate_event` (sequenced events)
- Drained + cleared in `_handle_terminal_event` (after successful
  commit)
- Accumulator entries that never receive terminal event → leak until
  process restart (ghost-queued failure mode; OPS milestone
  janitor handles ScanJob.status cleanup)

**Concurrency:**
- Single asyncio.Lock guards both `setdefault + extend` and
  `pop + drain` operations
- Lock held briefly (in-memory operations only); no I/O during
  lock hold
- Pub/Sub consumer is single-threaded async; concurrent calls to
  handle_event happen only via task scheduler, but lock protects
  against race anyway

**Memory bounds:**
- Per-key entry: up to 1000 findings × ~2KB/finding = ~2MB per
  intermediate batch
- Sequenced job with 5000 findings (5 batches × 1000): ~10MB
  accumulated before terminal
- 100 concurrent sequenced jobs: ~1GB worst case
- ADR-017 line 1453: "Pub/Sub size pressure: 1000 findings × ~2KB/
  finding = ~2MB worst case. Comfortably under the 32MB Redis
  Pub/Sub default."
- MVP scale: <10 concurrent sequenced jobs typical; ~100MB ceiling
  acceptable

### 4.3 Event Routing (`CompletionsConsumer.handle_event`)

**Decisions captured:**
- Strict event_seq.total + status routing per Path A/B/C above
- Defensive logging on unknown status combinations
- failed/canceled events bypass findings ingest entirely (per
  SPEC §7.3 line 817: "findings is REQUIRED on terminal events
  ... and absent on failed/canceled events")

**Forward compatibility:**
- event_seq is REQUIRED per SPEC §7.3 line 824 ("forward-compatibility
  — consumers always read it rather than branching on presence")
- New status values would route to defensive "unknown_status_combo"
  log; surfaces cross-repo drift

### 4.4 Bulk Insert Path (`session.execute(insert(RawFinding).values([...]))`)

**Decisions captured:**
- SQLAlchemy 2.0 Core construct per Decision 4
- finding_dicts assembled with explicit organization_id, scan_id,
  job_id (RLS columns + foreign keys)
- UUID primary key generated client-side (uuid4())
- discovered_at parsed from ISO datetime string

**Insert column population:**
- Pydantic-validated fields: copied via `.model_dump()`
- Identity fields: organization_id (from scan_job RLS scope), scan_id
  + job_id (from scan_job foreign keys)
- Generated: id (uuid4), created_at (DB DEFAULT NOW()), updated_at
  (DB DEFAULT NOW())

---

## 5. Implementation Plan (Approach C)

### 5.1 Single Atomic Commit Shape

**Files in commit:**

```
NEW:
  src/app/schemas/raw_findings.py             (~80 LoC; Pydantic schema)
  tests/services/
    test_completions_consumer_findings_ingest.py (~400 LoC; ~15 tests)
  tests/fixtures/
    job_completed_single_event.json           (single-event fixture)
    job_completed_sequenced_3_events.json     (sequenced fixtures)

EDIT:
  src/app/services/completions_consumer.py    (~150 LoC additions)
  DRIFT-LOG.md                                 (~6 entries)
```

**Estimated total LoC:** ~700-800 lines (src + tests + fixtures +
DRIFT-LOG).

### 5.2 Implementation Order Within Commit

Logical dependency order during development:

1. **Pydantic schema first** — define the ingest contract; tests can
   reference the schema
2. **Schema unit tests** — verify Pydantic behavior (validation,
   strictness, enum handling)
3. **CompletionsConsumer extension** — the three-path routing +
   accumulator + bulk insert
4. **CompletionsConsumer unit tests** — test each path in isolation
   with mocked session
5. **Integration tests** — full pipeline with real DB (test schema +
   RLS + atomic txn)
6. **ADR-017 forcing-function test pin** — `test_consumer_aggregates_sequenced_findings`
   with 3 sequenced events + 2500 findings + single-txn assertion
7. **DRIFT-LOG entries** — Phase 0 verification surfaces + design
   doc reference + ADR-025 reference

### 5.3 Phase Decomposition (For Claude Code Execution)

Approach C is single-commit, but execution decomposes into phases:

**Phase 0: Pre-implementation verification (~30 min)**
- Verify current `app.models.raw_findings.RawFinding` SQLAlchemy
  shape (canonical fields list)
- Verify current `CompletionsConsumer` code (Task 4.2 baseline)
- Verify existing services use RLS via `app.current_org_id`
  session-local pattern (or surface deviation)
- Verify pytest infrastructure for async + DB-backed integration tests
- Surface Phase 0 findings before Phase 1

**Phase 1: Pydantic schema (~1h)**
- Create `src/app/schemas/raw_findings.py`
- Schema unit tests
- Run schema tests in isolation; green

**Phase 2: CompletionsConsumer extension (~3-4h)**
- Extend `CompletionsConsumer.__init__` with accumulator + lock
- Add `_handle_single_event`, `_handle_intermediate_event`,
  `_handle_terminal_event` methods
- Update `handle_event` with three-path routing logic
- Run unit tests for each path

**Phase 3: Integration + forcing-function tests (~2-3h)**
- Integration tests with real DB (RLS + atomic txn verification)
- ADR-017 forcing-function test:
  `test_consumer_aggregates_sequenced_findings` (3 events × ~833
  findings each = 2500 total; single-txn commit assertion)
- Edge case tests: failed/canceled events, malformed events,
  sequenced-event-out-of-order

**Phase 4: DRIFT-LOG + commit (~30 min)**
- DRIFT-LOG entries documenting Phase 0 findings, design choices,
  ADR-025 reference, ADR-017 forcing function closure
- Run full test suite (all tests pass)
- Single atomic commit

### 5.4 Estimated Effort Summary

| Phase | Effort |
|-------|--------|
| Phase 0 verification | ~30 min |
| Phase 1 Pydantic schema + tests | ~1h |
| Phase 2 CompletionsConsumer extension + unit tests | ~3-4h |
| Phase 3 integration + forcing-function tests | ~2-3h |
| Phase 4 DRIFT-LOG + commit | ~30 min |
| **Total** | **~7-9h** |

Matches brainstorming bandwidth confirmation.

---

## 6. ADR-025 Draft

```markdown
### ADR-025: CompletionsConsumer findings-ingest path — Pydantic schema + sequenced accumulator + RLS
**Status:** Accepted (2026-05-XX, Findings-ingest task)

**Context.**
ADR-017 (findings inline in job_completed events) specifies that
the Python CompletionsConsumer inserts findings rows in the same
transaction as ScanJob.status update. ADR-017 line 1452 anticipated
this work at "Task 5.5 cross-repo" scope; only the engine-side
emitter landed at M5 close. Cross-repo Python-side consumer
extension was deferred.

ADR-024 §3.2 documented "Python ingest scope deferred" explicitly
when SPEC §7.3 schema extension landed. Phase 2 of M6-close-followup
landed the 4 new schema columns (References, Tags, CVSSVector,
AdditionalCWEs) but no consumer reads them.

This is overdue work. ADR-017's "Forcing functions" section
specifies a cross-repo test pin (`test_consumer_aggregates_sequenced_findings`)
that does not exist; ADR-017's contract is unsatisfied.

M9 AI Pipeline (SPEC §8.1) operates on `raw_findings` PostgreSQL
table; with findings-ingest unimplemented, M9 has zero rows to
read. M9 implementation requires findings-ingest to be operational.

**Decision.**
Implement the Python-side findings-ingest path with five
architectural components:

1. **Pydantic schema** (`app/schemas/raw_findings.py`):
   `RawFindingCreate` model mirroring SQLAlchemy. `extra="forbid"`
   strictness mirrors engine `DisallowUnknownFields` invariant.

2. **Service-class accumulator**: `CompletionsConsumer._accumulator`
   typed as `dict[tuple[UUID, UUID], list[dict]]`, guarded by
   `asyncio.Lock`. DI-friendly; testable in isolation; matches
   M4 Task 4.2 session_factory DI pattern.

3. **RLS via session-local**: Every ingest path executes
   `SET LOCAL app.current_org_id = :org_id` before insert/update.
   Defense-in-depth tenant isolation per SPEC §5.2.

4. **Bulk insert via SQLAlchemy 2.0 Core**:
   `session.execute(insert(RawFinding).values([...]))`. ~5-10x
   faster than ORM `add_all`; clean RLS interaction.

5. **Strict atomic failure semantics**: Single transaction wraps
   findings insert + ScanJob.status update. Failure → rollback
   (no half-state); event re-fires via Pub/Sub redelivery.
   Honors ADR-013 sole-writer atomicity.

Three event-routing paths (per ADR-017 sequencing contract):

- **Path A** (`event_seq.total == 1`): single-event job; insert +
  update in one txn.
- **Path B** (`event_seq.total > 1, status == "partial_findings"`):
  intermediate event; accumulate findings in `_accumulator`; do NOT
  update ScanJob.
- **Path C** (`event_seq.total > 1, status in ("completed", "partial")`):
  terminal event; insert union of accumulated batches + this event's
  findings; update ScanJob; clear accumulator entry after commit.

**Rationale.**

Three implementation alternatives considered and rejected:

| Alternative | Why rejected |
|---|---|
| **Module-level accumulator dict + asyncio lock** | Architectural inconsistency with M4 Task 4.2 session_factory DI pattern. Module-level state creates fragile test isolation (forgetting reset fixture leaks state across tests). |
| **Redis-backed accumulator (process-restart-safe)** | Over-engineered. ADR-017 explicitly identifies process-restart loss as deferred concern with concrete trigger conditions. Redis-backed accumulator is intermediate work that gets thrown away when ADR-017 Option C (R2-staging) trigger fires. |
| **ORM `session.add_all` instead of Core `insert().values()`** | 5-10x slower; problematic for sequenced batches reaching 5000+ findings. SQLAlchemy 2.0 idiomatic pattern is Core construct for bulk insert. |
| **Best-effort failure mode (insert fails → ScanJob still completes)** | Violates ADR-013 sole-writer atomicity invariant. ADR-017 explicitly specifies single-transaction atomicity. |
| **Degraded-mode failure (`completed_degraded` status)** | Premature; introduces new ScanJob enum value + ops alerting infrastructure. Pre-launch, ingest failures should fail loudly to surface bugs. SPEC §8.6's degraded-mode pattern is for AI pipeline failures, not consumer-layer. |

**Cross-reference: asymmetric-cost meta-principle (4th invocation
in project corpus).** Following ADR-022 (recon-as-pre-scan-helpers),
ADR-023 (NativeRunner OutputFile mode), ADR-024 (RawFinding schema
extension), this is the 4th ADR invoking asymmetric-cost reasoning
to justify architectural commitment.

Pattern shape: implementation cost (~7-9h single-commit work) is
asymmetrically smaller than alternative costs:
- Continuing without findings-ingest: M9 cannot operate; SPEC §8
  pipeline blocked
- Implementing minimum viable (single-event only): violates ADR-017
  sequencing contract; tech debt accumulating from day 1
- Implementing comprehensive (idempotency + ghost-queued janitor):
  premature optimization; ghost-queued janitor naturally lives in
  OPS milestone

The locked design implements ADR-017's full sequencing contract
without preempting OPS-milestone-scope concerns.

**ADR-017 forcing-function compliance.**

ADR-017 line 1474-1475 explicitly specifies a cross-repo test pin:
*"Python CompletionsConsumer test pin (Task 5.5 cross-repo):
`test_consumer_aggregates_sequenced_findings` synthesizes 3 sequenced
events, asserts single-txn commit on terminal batch with all 2500
findings persisted."*

Findings-ingest task implements this exact-shape test. The test
synthesizes:
- Event 1: `event_seq={index: 1, total: 3}`, status=`partial_findings`,
  ~833 findings
- Event 2: `event_seq={index: 2, total: 3}`, status=`partial_findings`,
  ~833 findings
- Event 3: `event_seq={index: 3, total: 3}`, status=`completed`,
  ~833 findings (terminal)

Assertions:
- Events 1 + 2 produce zero database writes (accumulator only)
- Event 3 produces single transaction with 2500 raw_findings rows
  inserted + ScanJob.status updated
- Accumulator entry cleared after successful commit
- Pydantic validates each finding correctly
- RLS session-local set before insert

**Consequences.**

Positive:
- ADR-017's full sequencing contract honored on landing
- ADR-024 §3.2 columns-ready posture closes (consumer reads new
  schema columns)
- M9 AI pipeline prerequisite met (raw_findings table populates)
- Cross-repo schema-drift regression guard via Pydantic
  `extra="forbid"`
- Defense-in-depth tenant isolation via RLS session-local
- Bulk insert performance acceptable for 1000-finding cap

Negative:
- **Process-restart accumulator-loss known limitation.** Per
  ADR-017 line 1455, accumulator survives only the single API
  process; on crash mid-sequence, partial findings lost AND
  ScanJob.status remains "running". OPS milestone janitor handles
  ScanJob.status cleanup; finding-loss is the trigger condition
  for ADR-017 Option C (R2-staging) migration.
- **No idempotency on duplicate findings inserts.** If accumulator
  cleanup fails after commit (process crash between commit + clear),
  next process startup retains stale accumulator state. Mitigation:
  process-startup accumulator clear (zero state on restart). Future
  enhancement: DB-side fingerprint+scan_id unique constraint for
  hard idempotency.
- **Pydantic strictness creates cross-repo coordination burden.**
  If Engine adds a field to RawFinding (e.g., future SPEC §7.3
  iteration), Python ingest fails until schema updated. By design
  (regression guard against schema drift); requires explicit
  cross-repo coordination on field additions.

**Alternatives considered (and rejected).** See Rationale table
above.

**Anti-patterns this prevents.**
- **Findings persistence via Engine-side PG writes.** ADR-013
  violation; ADR-017 explicitly closed this loophole; this ADR
  delivers the consumer-side that ADR-017 anticipated.
- **Module-level accumulator state.** Test isolation fragility +
  architectural inconsistency with DI pattern.
- **Best-effort or degraded-mode failure semantics.** ADR-013
  sole-writer atomicity violation.
- **ORM `add_all` for bulk operations.** 5-10x slower; problematic
  at sequenced batch sizes.
- **Permissive Pydantic schema (`extra="ignore"` or `extra="allow"`).**
  Forfeits cross-repo schema-drift regression guard.

**Triggers to revisit.**

1. **Production occurrence of "ScanJob ghost-queued due to
   mid-sequence crash"** (per ADR-017 line 1444-1445). Trigger
   condition for ADR-017 Option C migration to R2-staging. Once
   is the trigger, not sustained.
2. **Sustained job_completed event payloads exceeding 5MB**
   (per ADR-017 line 1442). Operational metric on Pub/Sub message
   size; alerts at OPS milestone.
3. **M9 AI pipeline ingest-time problems with batch sizes.**
   E.g., deep Trivy or Nuclei scans producing 5000+ findings per
   job, where the Python consumer's per-event txn time exceeds 30s
   and starts blocking subsequent events.
4. **DB-side idempotency required.** When customer reports
   duplicate findings in dashboard (post-launch trigger),
   add fingerprint+scan_id unique constraint + handle conflict
   in bulk insert.
5. **Engine adds a new RawFinding field.** Pydantic `extra="forbid"`
   rejects until Python schema updated. Cross-repo coordination
   workflow: SPEC update → Engine struct extension → Python schema
   extension → Python migration if persistent.

**Forcing functions.**
- ADR-017 cross-repo test pin (`test_consumer_aggregates_sequenced_findings`)
  required in this commit; failure to include violates ADR-017
  forcing-function contract.
- Pydantic `extra="forbid"` config explicitly stated in
  `RawFindingCreate.model_config`; future engineers tempted to
  loosen find this ADR via grep on `extra="forbid"`.
- DRIFT-LOG entry references ADR-025 + ADR-017 forcing function
  + ADR-024 §3.2 closure.
- Process-startup accumulator clear documented in code comment
  citing ADR-017 line 1455 + ADR-025.

**Open follow-ups.**
- DB-side idempotency via unique constraint (deferred; trigger
  on customer report).
- ghost-queued janitor (OPS milestone scope; carry-forward from
  M4 Task 4.2 + ADR-017 line 1455).
- ADR-017 Option C (R2-staging) migration (deferred; trigger on
  operational signals per ADR-017 line 1442-1445).

**Cross-references.**
- ADR-013: Python sole writer constraint (load-bearing for
  atomic transaction discipline).
- ADR-017: Findings inline in job_completed events with
  sequencing (this ADR closes the cross-repo Python-side
  follow-up).
- ADR-024 §3.2: Python ingest scope deferral (this ADR closes
  the columns-ready posture).
- SPEC §7.3: Job Completion wire format (canonical RawFinding
  schema; consumer reads this format).
- SPEC §5.2: Row-Level Security default posture
  (RLS session-local pattern).
- SPEC §5.3: raw_findings table schema (mobile + recon
  extensions; mirrored in Pydantic schema).
- SPEC §8.1: AI Analysis Pipeline (M9 prerequisite consumer of
  raw_findings table).
```

---

## 7. Test Strategy

### 7.1 Test Layers

**Pydantic schema tests** (`tests/schemas/test_raw_findings.py` —
new file):
- `test_minimal_payload_validates` (only required fields)
- `test_full_payload_validates` (all fields)
- `test_spec_73_extension_fields_validate` (References + Tags +
  CVSSVector + AdditionalCWEs)
- `test_unknown_field_rejected` (extra="forbid")
- `test_severity_enum_violation` (invalid severity)
- `test_engine_category_enum_violation` (invalid category)
- `test_missing_required_field` (e.g., no tool_name)

**CompletionsConsumer unit tests** (extend `tests/services/test_completions_consumer.py`):
- `test_handle_event_routes_to_single_event_path`
- `test_handle_event_routes_to_intermediate_path`
- `test_handle_event_routes_to_terminal_path`
- `test_handle_event_skips_failed_status` (no findings ingest)
- `test_handle_event_skips_canceled_status`
- `test_handle_event_logs_unknown_status_combo`
- `test_accumulator_lock_prevents_concurrent_modification`
- `test_accumulator_cleared_after_commit`
- `test_accumulator_retained_on_commit_failure`

**CompletionsConsumer integration tests**
(`tests/services/test_completions_consumer_findings_ingest.py` — new):
- `test_single_event_inserts_findings_and_updates_scanjob` (full
  end-to-end with real DB)
- `test_rls_session_local_set_before_insert` (verify SET LOCAL
  query executes)
- `test_atomic_txn_rollback_on_constraint_violation` (force
  failure; assert no findings + ScanJob.status unchanged)
- `test_org_isolation_via_rls` (cross-tenant isolation; verify
  RLS rejects cross-org access)
- `test_bulk_insert_performance_under_1000_findings_under_500ms`
  (acceptance threshold)

**ADR-017 forcing-function test** (mandatory; new file):
- `test_consumer_aggregates_sequenced_findings` — synthesizes 3
  sequenced events × 833 findings each; asserts:
  - Events 1+2 produce zero DB writes
  - Event 3 produces single txn with 2500 rows + ScanJob.status update
  - Accumulator cleared after successful commit
  - All findings have correct organization_id (RLS-scoped)

### 7.2 Test Fixtures

**File:** `tests/fixtures/job_completed_single_event.json`
```json
{
  "event_type": "job_completed",
  "job_id": "...",
  "scan_id": "...",
  "engine": "nuclei",
  "status": "completed",
  "finding_count": 5,
  "duration_ms": 12000,
  "idempotency_key": "...",
  "timestamp": "2026-05-03T...",
  "findings": [
    { /* RawFinding 1 */ },
    /* ... 4 more findings ... */
  ],
  "event_seq": { "index": 1, "total": 1 }
}
```

**File:** `tests/fixtures/job_completed_sequenced_3_events.json`
- Three event objects in array form (or three separate fixture
  files) for test consumption

### 7.3 Coverage Targets

| Component | Target |
|-----------|--------|
| Pydantic schema | ~7 tests |
| Event routing | ~6 tests |
| Single-event ingest | ~5 tests |
| Intermediate-event accumulator | ~3 tests |
| Terminal-event accumulator drain | ~3 tests |
| RLS session-local + atomicity | ~3 tests |
| ADR-017 forcing-function | ~1 test (the named pin) |
| Edge cases (failed/canceled/unknown) | ~4 tests |
| **Total** | **~32 tests** |

### 7.4 Backward Compatibility

Existing CompletionsConsumer tests must remain green. Phase 0 reads
`tests/services/test_completions_consumer.py` baseline; existing
tests should be augmented (not replaced) with new ingest assertions.

### 7.5 ADR-017 Forcing-Function Test Detail

The named test `test_consumer_aggregates_sequenced_findings` is
explicitly specified by ADR-017 line 1474-1475. Implementation
must include:

```python
async def test_consumer_aggregates_sequenced_findings(
    consumer: CompletionsConsumer,
    db_session: AsyncSession,
    sample_scan_job: ScanJob,
):
    """ADR-017 forcing-function test pin (line 1474-1475).
    
    Synthesizes 3 sequenced events × 833 findings each = 2500 total.
    Asserts: events 1+2 produce zero DB writes; event 3 produces
    single-txn commit with 2500 rows persisted + ScanJob.status
    updated.
    """
    # Construct 3 events
    event1 = build_partial_event(
        scan_id=sample_scan_job.scan_id,
        job_id=sample_scan_job.id,
        seq_index=1, seq_total=3,
        finding_count=833,
    )
    event2 = build_partial_event(
        scan_id=sample_scan_job.scan_id,
        job_id=sample_scan_job.id,
        seq_index=2, seq_total=3,
        finding_count=833,
    )
    event3 = build_terminal_event(
        scan_id=sample_scan_job.scan_id,
        job_id=sample_scan_job.id,
        seq_index=3, seq_total=3,
        finding_count=2500,  # authoritative total per ADR-017 line 822
        finding_count_in_payload=834,  # 2500 - 833 - 833
    )
    
    # Process events
    await consumer.handle_event(event1)
    await consumer.handle_event(event2)
    
    # Assert no DB writes after intermediate events
    rows_after_event2 = await db_session.execute(
        select(func.count(RawFinding.id)).where(
            RawFinding.scan_id == sample_scan_job.scan_id
        )
    )
    assert rows_after_event2.scalar() == 0
    
    # Assert ScanJob status unchanged
    await db_session.refresh(sample_scan_job)
    assert sample_scan_job.status == "running"
    
    # Process terminal event
    await consumer.handle_event(event3)
    
    # Assert all 2500 findings persisted
    rows_after_event3 = await db_session.execute(
        select(func.count(RawFinding.id)).where(
            RawFinding.scan_id == sample_scan_job.scan_id
        )
    )
    assert rows_after_event3.scalar() == 2500
    
    # Assert ScanJob.status updated
    await db_session.refresh(sample_scan_job)
    assert sample_scan_job.status == "completed"
    assert sample_scan_job.finding_count == 2500
    
    # Assert accumulator entry cleared
    key = (sample_scan_job.scan_id, sample_scan_job.id)
    assert key not in consumer._accumulator
```

This test is the load-bearing forcing-function compliance check.
It must pass; failing means ADR-017 contract unmet.

---

## 8. DRIFT-LOG Entries Plan

**Engine-side:** No engine changes; no engine DRIFT entries.

**Python-side (shieldscan-api) DRIFT entries (~6):**

1. **Findings-ingest task closes** — ADR-024 §3.2 columns-ready
   posture resolved; ADR-017 Task 5.5 cross-repo scope resolved
   (overdue work).

2. **ADR-025 lands** — CompletionsConsumer findings-ingest path;
   project corpus's 4th asymmetric-cost ADR.

3. **ADR-017 forcing-function compliance** —
   `test_consumer_aggregates_sequenced_findings` test pin honored;
   ADR-017 line 1474-1475 contract met.

4. **Pydantic schema strictness decision** — `extra="forbid"`
   mirrors engine `DisallowUnknownFields`; cross-repo schema-drift
   regression guard established.

5. **Service-class accumulator pattern (1st instance)** —
   asyncio-locked instance-attribute accumulator on
   CompletionsConsumer. Track for promotion if a 3rd+ instance
   surfaces.

6. **M9 prerequisite met** — raw_findings table now populates;
   SPEC §8.1 AI pipeline can operate. Forward-pin: M9 implementation
   reads the now-populated table.

---

## 9. Implementation Plan Handoff

This design doc is the architectural artifact. Phase-by-phase
implementation plan follows the writing-plans skill workflow:

**Phase 0: Pre-implementation verification**
- Verify SQLAlchemy model post-Phase-2 (canonical fields)
- Verify CompletionsConsumer M4 Task 4.2 baseline shape
- Verify existing services use RLS pattern (or surface deviation)
- Verify pytest async + DB integration test infrastructure

**Phase 1: Pydantic schema (~1h)**
- Create `src/app/schemas/raw_findings.py`
- Schema unit tests
- Run schema tests in isolation; green

**Phase 2: CompletionsConsumer extension (~3-4h)**
- Extend `__init__` with accumulator + lock
- Add three event-path handlers
- Update `handle_event` routing logic
- Run unit tests

**Phase 3: Integration + forcing-function tests (~2-3h)**
- Integration tests with real DB
- ADR-017 forcing-function test
- Edge case coverage

**Phase 4: DRIFT-LOG + commit (~30 min)**
- DRIFT-LOG entries
- Run full test suite
- Single atomic commit

---

## 10. Open Questions / Triggers

### 10.1 Pre-Implementation Decisions Needing Phase 0 Verification

1. **SQLAlchemy model canonical fields.** Verify
   `app.models.raw_findings.RawFinding` head state; the Pydantic
   schema must mirror it exactly. Phase 0 reads model;
   adapt Pydantic schema.

2. **Existing RLS pattern in other services.** Verify how scans/
   projects/etc services apply RLS. Lean: matches `SET LOCAL
   app.current_org_id` pattern; if different, surface as
   architectural deviation requiring decision.

3. **Pytest async + DB integration test infrastructure.** Verify
   test fixtures for async session + DB-backed tests exist.
   If absent, factual deviation; build minimal infrastructure
   in Phase 0.

4. **CompletionsConsumer M4 Task 4.2 baseline shape.** Verify
   exact method signatures, session_factory DI pattern, current
   `handle_event` logic. Adapt extension to match.

### 10.2 Future Triggers (Carried Forward)

1. **ADR-017 Option C migration triggers** (per ADR-017 line
   1442-1445):
   - Sustained job_completed event payloads exceeding 5MB
   - Pub/Sub message size approaching 32MB Redis hard limit
   - M9 AI pipeline ingest-time problems with batch sizes
   - Any production occurrence of ghost-queued ScanJob

2. **DB-side idempotency required** (post-launch trigger):
   When duplicates surface in customer reports, add
   fingerprint+scan_id unique constraint + bulk-insert conflict
   handling.

3. **ghost-queued janitor** (OPS milestone scope):
   Background sweep of stuck ScanJob rows; either re-dispatch
   or mark failed.

4. **Cross-repo schema-drift coordination** (ongoing):
   Engine adding new RawFinding fields requires Python schema
   update. SPEC update → Engine struct extension → Python
   schema extension → Python migration if persistent.

5. **Service-class accumulator pattern promotion** (3rd-instance
   trigger):
   Currently 1st instance (this task). If 3rd+ instance surfaces
   (unlikely; specific to sequenced-batch consumer patterns),
   consider DEVELOPMENT-PATTERNS entry.

---

## 11. Brainstorming Process Acknowledgment

This design emerged from a structured brainstorming session that
locked five architectural decisions through five clarifying
questions. The discipline pattern from M5/M6 + SPEC §7.3 followup
holds: brainstorming → design → ADR → implementation chain.

ADR-025 is the project corpus's 4th ADR invoking asymmetric-cost
meta-principle. The pattern is now established norm; future ADRs
inherit the cross-reference shape.

ADR-017's forcing-function test pin (line 1474-1475) is the load-
bearing compliance check. The test name and shape are explicitly
specified by ADR-017; findings-ingest task delivers the test.

This is the 4th task in the project corpus that brainstorming
produced a comprehensive design doc for (after M5 close
retrospective, SPEC §7.3 followup, [findings-ingest as 4th]).
The discipline pattern is mature.

---

**Status:** Ready for implementation per writing-plans skill
workflow.

**Approval required:** None outstanding; brainstorming locked all
decisions.

**Next action:** Transition to writing-plans skill for the
phase-by-phase implementation plan, OR proceed directly to Phase 0
(verification) implementation with this design doc as the
reference.
