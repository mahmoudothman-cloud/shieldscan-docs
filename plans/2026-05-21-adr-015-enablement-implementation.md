# ADR-015 Enablement: Implementation Plan

**Status:** Ready for Stage 3 cross-repo implementation. Pre-verification grounded existing infrastructure state; Y1+Y2+Q1-Q7 brainstorming chain locked at design doc `b344d0c`. Plan structure grounded against verified existing infrastructure shape.

**Date:** 2026-05-21.

**Authority:** ADR-015 design doc (shieldscan-docs commit `b344d0c`; Y1+Y2+Q1-Q7 architectural-decision canonical authority); pre-verification surface report (this session; V-BA through V-BI; existing infrastructure state); Task 7.6 implementation plan structural precedent (shieldscan-docs commit `40606c5`; 269 LoC); Task 7.5e bounded compressed-lifecycle precedent (`829fe4b`; 217 LoC); 39+ cumulative session-tail framing-drift discipline.

**Related:** Stage 3 cross-repo implementation trigger phrase: ***"Resume ADR-015 — Stage 3 cross-repo implementation"*** (after this plan lands).

---

## 1. Authority + Scope Lock

**Y1 (b) phased scope lock:** This task covers axes 1-4 of ADR-015 enablement:

1. ADR-015 SPEC §13 authoring + ADR-013/ADR-014 addendums (Stage 3 Commit 1 docs scope)
2. api orchestrator decrypt-and-emit code + lift test pin + positive-path tests + audit logging (Stage 3 Commit 2 api scope)
3. Discriminator rename `auth_type`→`type` via orchestrator-level translation (Stage 3 Commit 2 api scope)
4. SQLMap consumer cookie wiring + integration test upgrade from wiring-validation to richer assertions (Stage 3 Commit 3 engine scope)

**Out of scope (forward-pinned):**

- ZAP consumer cookie pass-through (axis 5) — ZAP greenfield; lands in ZAP consumer task
- MobSF R2 pre-signed URL pattern (axis 6) — deferred to MobSF V10 task per Q5 (a)
- Credential revocation flow (Q6 a forward-pin) — multi-axis territory; separate task
- 3rd-instance per-section-adaptor pattern — Task 7.6 P5.D forward-pin; unrelated

**Brainstorming chain Q1-Q7 architectural decisions locked at design doc `b344d0c`:** Q1 (a) decrypted-in-Redis-transit security model; Q2 (b) all 5 AuthType values v1; Q3 (a) orchestrator-level discriminator translation; Q4 (a) full ADR-013 + ADR-014 addendums + ADR-015 §13 section; Q5 (a) R2 pre-signed URL deferred; Q6 (a) v1 audit-only + revocation forward-pinned; Q7 (a) 3-commit cross-repo (docs → api → engine).

## 2. Pre-Implementation State

### 2.1 Infrastructure Ready Matrix (V-BD + V-BE + V-BF Pre-Verification Findings)

**api side ready (V-BD pre-built):**

- `src/app/services/credentials.py` — `AuthType` enum {cookie, bearer, basic, form, custom_header} + `encrypt_credential(payload)` + `decrypt_credential(blob)` Fernet helpers
- `src/app/models/projects.py` — `ProjectCredential` model with `auth_type` indexed column + `encrypted_data` Fernet blob
- `src/app/schemas/credentials.py` — Pydantic discriminated-union schema
- `src/app/services/orchestrator.py` — `_build_job_payload` emits `"auth": None` with inline `# TODO(M5+): inject decrypted credential — see ADR-015 (deferred)`
- `tests/services/test_orchestrator.py::test_dispatch_payload_auth_is_null_pending_adr_015` — regression pin awaiting lift

**api side pending (Stage 3 Commit 2 scope):**

- Orchestrator look-up of ProjectCredential per scan-job + decrypt + emit `{type, data}` payload
- Lift `test_dispatch_payload_auth_is_null_pending_adr_015` regression pin
- Add positive-path test (verify cookie-type credential reaches payload)
- Add audit logging at decrypt-time (~5-10 LoC)
- Discriminator translation `auth_type` → `type` at emit-time

**engine side ready (V-BE infrastructure):**

- `internal/tools/runner.go` — `AuthConfig` struct (5-value Type contract + Data string + Fields map)
- `internal/events/events.go` — `JobAuth` struct (`{type, data}` wire shape) + `JobDispatch.Auth *JobAuth` top-level field
- `internal/worker/processor.go:421` — `jobDispatchToTarget` routing: `if job.Auth != nil { target.AuthConfig = &tools.AuthConfig{...} }`
- `internal/worker/processor_test.go:459-472` — end-to-end test proves `JobAuth{Type:"cookie", Data:"session=abc"}` → `target.AuthConfig` round-trip works

**engine side pending (Stage 3 Commit 3 scope):**

- SQLMap consumer `buildArgs` cookie injection (`--cookie=<data>` when `target.AuthConfig.Type=="cookie"`)
- SQLMap integration test upgrade: lift wiring-validation reframing; thread `bootstrapDVWA` cookies through `Target.AuthConfig`; assert ≥1 sql_injection + ≥1 dbms_fingerprint per V4 baseline

**docs side pending (Stage 3 Commit 1 scope):**

- SPEC §13 new ADR-015 section landing (per design doc §4 verbatim draft)
- SPEC §13 ADR-013 addendum landing (per design doc §5.1)
- SPEC §13 ADR-014 addendum landing (per design doc §5.2)
- `DRIFT-LOG.md` line 175 reservation entry update to "landed" + Task reference

### 2.2 Redis Transit Posture (V-BF)

ADR-015 introduces NO new Redis primitive. Populates existing nullable Auth field on existing JobDispatch payload via existing Redis Streams transit (per ADR-014). Mitigations enumerated in ADR-015 §13 §4 design doc draft (mandatory: authenticated Redis access + TLS-in-transit + short queue TTL + no-persistence config; recommended v1.1+: per-queue ACL + AOF cipher).

### 2.3 Drift #35 Closure Mechanism (V-BG + V-BH)

Task 7.6 Drift #35 forward-pin closes operationally at Stage 3 Commit 3: SQLMap consumer `buildArgs` cookie injection + integration test upgrade auto-upgrades wiring-validation reframing to richer assertions. `bootstrapDVWA` captures `PHPSESSID` + `security=low` cookies → `simpleCookieJar`; cookies flow through `Target.AuthConfig.Data` semicolon-joined per engine `AuthConfig.Data` docstring contract → `buildArgs` appends `--cookie=<data>` → SQLMap scans authenticated DVWA SQLi endpoint → V4 baseline assertions auto-trigger.

## 3. Architectural Decisions (Plan-Level Locks)

Brainstorming chain Q1-Q7 architectural decisions locked at design doc `b344d0c`. Plan-level refinements captured below.

### 3.1 Q1 (a) Decrypted-in-Redis-Transit — Implementation Surface

**Orchestrator decrypt-and-emit shape:**

```python
# orchestrator.py _build_job_payload extension (Stage 3 Commit 2)
credential = (
    await session.get(ProjectCredential, scan.credential_id)
    if scan.credential_id else None
)
if credential:
    decrypted_data = decrypt_credential(credential.encrypted_data)
    emit_credential_access_audit(
        timestamp=now,
        project_id=scan.project_id,
        credential_id=credential.id,
        scan_id=scan.id,
        dispatcher_user_id=scan.dispatched_by,
    )
    payload["auth"] = {
        "type": credential.auth_type,
        "data": decrypted_data["data"],
        **(
            {"fields": decrypted_data["fields"]}
            if "fields" in decrypted_data else {}
        ),
    }
else:
    payload["auth"] = None  # legacy null path; preserved for credential-less scans
```

Discriminator translation Q3 (a) at translation site: DB `credential.auth_type` → wire `"type": credential.auth_type` (same string value; field-name only changes).

### 3.2 Q2 (b) All 5 AuthType Values v1

**Implementation pattern:** Orchestrator code is auth-type-agnostic (single decrypt+emit code path). Per-AuthType handling routed at consumer-side:

- **cookie** (v1 SQLMap support): `--cookie=<data>` argv appended at SQLMap `buildArgs`
- **bearer** (forward-pin per consumer; SQLMap v1.1+ via `--headers="Authorization: Bearer <data>"`)
- **basic** (forward-pin; SQLMap v1.1+ via `--auth-cred="<data>"`)
- **custom_header** (forward-pin; SQLMap v1.1+ via `--headers="<data>"`)
- **form** (forward-pin; SQLMap v1.1+ via `--data="<fields>"`)

### 3.3 Q3 (a) Discriminator Translation Implementation Site

**Translation location:** `orchestrator.py` `_build_job_payload` at emit-time only. DB schema (`ProjectCredential.auth_type`) + Pydantic schema (`auth_type` field) UNCHANGED. Wire payload emits `"type"` key (engine wire contract).

**Bounded scope:** Single dict-key transformation at emit-site (~1-line change in pseudo-code above). No DB migration; no schema change; no API contract break.

### 3.4 Q4 (a) Cross-ADR Addendum Implementation Site

**Stage 3 Commit 1 docs sequencing:**

1. ADR-015 §13 new section insertion (per design doc §4 verbatim)
2. ADR-013 addendum append after existing Open follow-ups (per design doc §5.1; preserves chronological addendum ordering)
3. ADR-014 addendum append after existing Open follow-ups (per design doc §5.2; preserves chronological addendum ordering)
4. `DRIFT-LOG.md` line 175 reservation entry update: *"ADR-015 (decrypted credentials in Redis) — defer until enabled M5+"* → *"ADR-015 LANDED at SPEC §13 (commit `<docs-stage3-commit-1-hash>`); cross-repo enabled per api commit `<api-stage3-commit-2-hash>` + engine commit `<engine-stage3-commit-3-hash>`"*

### 3.5 Q6 (a) Audit Logging Implementation

**Audit row shape (canonical):** `{timestamp, project_id, credential_id, scan_id, dispatcher_user_id}`

**Storage:** New table `credential_access_audit` (recommended) OR existing audit infrastructure (verify at Stage 3 Commit 2 execution; may already exist for ADR-013 sole-writer audit).

**Plan-level Y-decision Y-AUDIT:** Determine at Stage 3 Commit 2 whether to create new table OR extend existing audit infrastructure. Default: new table (clean separation; no foreign-key coupling).

### 3.6 Q7 (a) 3-Commit Cross-Repo Sequencing

**Commit 1 docs FIRST:** SPEC §13 ADR-015 landing + ADR-013 addendum + ADR-014 addendum + `DRIFT-LOG.md` reservation update

**Commit 2 api SECOND:** orchestrator decrypt+emit + audit logging + discriminator translation + pin lift + positive-path test

**Commit 3 engine THIRD:** SQLMap `buildArgs` cookie injection + integration test upgrade

**Cross-reference shape:** Commit 1 docs references future api+engine hashes via placeholders (per historical-authority precedent). Commit 2 api cross-references docs hash from Commit 1 + engine hash placeholder. Commit 3 engine cross-references both docs + api hashes concretely.

## 4. Stage 3 Sub-Step Breakdown

### Stage 3 Commit 1 — Docs (SPEC §13 + Addendums + DRIFT-LOG) (~30-45min)

- **C1.1** Insert ADR-015 §13 section at `SPECIFICATION.md` (alphabetically between ADR-014 and ADR-016 per existing §13 numbering convention). Use design doc §4 verbatim draft. ~40-50 LoC delta.
- **C1.2** Append ADR-013 addendum after existing ADR-013 Open follow-ups. Use design doc §5.1 verbatim draft. ~5-10 LoC delta.
- **C1.3** Append ADR-014 addendum after existing ADR-014 Open follow-ups. Use design doc §5.2 verbatim draft. ~5-10 LoC delta.
- **C1.4** Update `DRIFT-LOG.md` line 175 ADR-015 reservation entry to "landed" with cross-repo references. ~2-3 LoC delta.
- **C1.5** Pre-commit verification: grep ADR-015 in `SPECIFICATION.md` (verify section landed + addendums present); `wc -l` on changed files; visual inspection of insertion points.
- **C1.6** Single atomic commit. Cross-references future api+engine hashes via placeholders.

**Total Stage 3 Commit 1 LoC delta:** ~50-75 LoC across 2 files (`SPECIFICATION.md` + `DRIFT-LOG.md`).

### Stage 3 Commit 2 — API (Orchestrator + Audit + Tests) (~45-90min)

- **C2.1** Extend `orchestrator.py` `_build_job_payload`: `ProjectCredential` lookup + decrypt + emit `{type, data, fields?}` + discriminator translation. ~15-25 LoC delta.
- **C2.2** Add audit logging at decrypt-time: `emit_credential_access_audit()` helper + audit row insertion. ~10-20 LoC delta. Y-AUDIT decision: new `credential_access_audit` table OR extend existing audit.
- **C2.3** Add `credential_access_audit` model + migration (if new table; conditional on Y-AUDIT lock). ~20-30 LoC delta + Alembic migration.
- **C2.4** Lift `test_dispatch_payload_auth_is_null_pending_adr_015` regression pin. ~5-10 LoC delta.
- **C2.5** Add positive-path test: verify cookie-type credential reaches payload with correct shape. ~30-50 LoC delta.
- **C2.6** Add audit-logging test: verify audit row written on credential-decrypt. ~20-30 LoC delta.
- **C2.7** Pre-commit verification: `pytest tests/services/test_orchestrator.py` (lifted pin + positive-path + audit tests all green); `pytest tests/services/` (full module green); `alembic upgrade head` (if migration); `pytest` entire test suite if scope-sensitive.
- **C2.8** Single atomic commit. Cross-references docs hash from Commit 1 + engine hash placeholder.

**Total Stage 3 Commit 2 LoC delta:** ~100-165 LoC across ~3-5 files (`orchestrator.py` + `test_orchestrator.py` + audit model + Alembic migration).

### Stage 3 Commit 3 — Engine (SQLMap Consumer + Integration Test) (~30-45min)

- **C3.1** Extend SQLMap `buildArgs` at `internal/tools/docker/sqlmap/scan.go`: append `--cookie=<data>` to argv when `target.AuthConfig != nil && target.AuthConfig.Type == "cookie"`. Defensive: skip if `AuthConfig.Data` empty. ~5-10 LoC delta.
- **C3.2** Extend `scan_test.go`: add test cases for cookie-injection argv shape (4-5 new test cases). ~30-50 LoC delta.
- **C3.3** Upgrade `integration_test.go`: lift wiring-validation reframing; thread `bootstrapDVWA` cookies into `Target.AuthConfig{Type:"cookie", Data:<semicolon-joined>}`; assert ≥1 sql_injection finding + ≥1 dbms_fingerprint finding per V4 baseline; per-finding spot-checks (`CWEID` + `OWASP` + `Payload` non-empty + `Parameter="id"`). ~20-40 LoC delta.
- **C3.4** Pre-commit verification: `go vet` + `go test -race ./internal/tools/docker/sqlmap/` (unit green); `go test -tags integration -v ./internal/tools/docker/sqlmap/` (real-Docker DVWA cookie-authenticated scan PASS; expected duration ~25-30s; findings emerge per V4 baseline).
- **C3.5** Single atomic commit. Cross-references docs hash + api hash concretely.

**Total Stage 3 Commit 3 LoC delta:** ~55-100 LoC across 3 files (`scan.go` + `scan_test.go` + `integration_test.go`).

### Stage 3 Aggregate LoC Forecast

Total across 3 commits: ~205-340 LoC delta (~50-75 docs + ~100-165 api + ~55-100 engine). Smaller scope than Task 7.6 Phase 1+2+3 (~1,632 LoC) per Y1 (b) phased lock.

## 5. D-Deviation Tracking Framework

Per Task 7.1 + Task 7.6 D-PLAN tracking precedent: Stage 3 implementation D-deviations from this plan get tracked via D-PLAN-N entries at commit-message bodies + Stage 4 P5.A annotation time.

**Expected D-deviation count:** MODERATE. Pre-verification grounded most empirical surfaces (V-BA through V-BI; infrastructure shape verified) but Stage 3 introduces new code surface (orchestrator decrypt+emit pattern; audit logging; cookie wiring) where implementation-refinement drifts likely surface (parameter naming + helper-function decomposition + Y-AUDIT decision context).

**Compare to:** Task 7.6 implementation surfaced 8 drifts (#28-#35); ADR-015 Stage 3 expected drift count ~4-8 (smaller scope; pre-verification stronger; existing patterns inform implementation).

## 6. Out of Scope (per design doc §7 + plan-level refinements)

1. ZAP consumer cookie pass-through (axis 5) — ZAP greenfield
2. MobSF R2 pre-signed URL pattern (axis 6) — deferred to MobSF V10 task
3. Credential revocation flow (Q6 a forward-pin) — separate task
4. Per-credential rate limits — orthogonal
5. Credential rotation flow — use existing application-level patterns
6. bearer/basic/custom_header/form SQLMap consumer wiring — v1.1+ per-type
7. Per-tenant Redis isolation — v1.1+ recommended mitigation
8. 3rd-instance per-section-adaptor pattern — Task 7.6 P5.D unrelated
9. R2 pre-signed URL pattern (Task 7.4 Q6.4) — deferred
10. Audit log retention + analytics — out of ADR-015 v1 scope

## 7. Forward-Pins

**Pre-execution forward-pins (Stage 3 entry):**

1. **Stage 3 trigger phrase:** ***"Resume ADR-015 — Stage 3 cross-repo implementation"***
2. **Y-AUDIT decision context preserved:** new `credential_access_audit` table vs extend existing audit (resolve at Stage 3 Commit 2)
3. **Design doc canonical authority:** `b344d0c` §4 + §5 verbatim drafts

**Post-Stage-3 forward-pins:**

4. **Credential revocation flow task** — ***"Begin credential revocation flow task"***
5. **ZAP consumer task** — cookie pass-through enablement (Q7 b ZAP Task 7.3 forward-pin)
6. **MobSF V10 task** — R2 pre-signed URL pattern (Task 7.4 Q6.4)
7. **v1.1+ Redis ACL per-queue** — per-tenant Redis security enhancement
8. **SQLMap consumer non-cookie AuthType support** — bearer/basic/custom_header/form per scan-target need
9. **Stage 4 Phase 5 sub-phases trigger:** ***"Resume ADR-015 — Phase 5 sub-phases"***

## 8. Cross-References

**Engine commits:**

- `06ab650` (Task 7.6 P5.D; DRIFT-LOG 2nd-instance per-section adaptor; latest engine state)
- `723426d` (Task 7.6 cross-repo pair Commit 2; Drift #35 origin; SQLMap consumer integration test wiring-validation reframing — Stage 3 Commit 3 target for upgrade)
- `ba860a5` (Task 7.5e Phase 1+2; framework Mounts capability + Entrypoint fix universal benefit)
- `cf3b30a` (Task 4.2; orchestrator scan-job dispatcher; auth: None deferral origin)

**Docs commits:**

- `b344d0c` (ADR-015 design doc; this plan's canonical authority + §4 ADR-015 §13 draft + §5.1 ADR-013 addendum draft + §5.2 ADR-014 addendum draft)
- `c28a5de` (Task 7.6 P5.C; latest docs state pre-ADR-015)
- `40606c5` (Task 7.6 implementation plan; structural precedent for this plan)
- `15d1ac5` (Task 7.5e ADR-026 Mounts Extension addendum precedent shape)

**API commits:**

- `2cd4065` (Task 7.6 cross-repo pair Commit 1; SCAN_TYPE_TOOLS SQLMap routing; auth:None currently)
- `e6fb0a5` (Task 7.1 cross-repo pair Commit 1; orchestrator scaffold)

**SPEC sections:**

- §13 ADR-013 (sole-writer + payload contract; Q4 a addendum target)
- §13 ADR-014 (Redis Streams transit; Q4 a addendum target)
- §13 ADR-015 (NEW SECTION; Q4 a content draft at design doc `b344d0c` §4)
- §13 ADR-016 (queue priority; unchanged)

**Source authorities (Stage 3 sub-step targets):**

- shieldscan-api `src/app/services/credentials.py` (AuthType enum + Fernet helpers; consumed not modified)
- shieldscan-api `src/app/models/projects.py` (ProjectCredential model; consumed not modified)
- shieldscan-api `src/app/schemas/credentials.py` (Pydantic schema; consumed not modified)
- shieldscan-api `src/app/services/orchestrator.py` (C2.1 + C2.2 modification site; decrypt+emit + audit)
- shieldscan-api `src/app/models/<credential_access_audit-new-OR-existing-audit-table>` (C2.3 Y-AUDIT decision target)
- shieldscan-api `tests/services/test_orchestrator.py` (C2.4 pin lift + C2.5 positive-path + C2.6 audit-logging test sites)
- shieldscan-engine `internal/tools/docker/sqlmap/scan.go` (C3.1 `buildArgs` cookie injection target)
- shieldscan-engine `internal/tools/docker/sqlmap/scan_test.go` (C3.2 cookie test cases)
- shieldscan-engine `internal/tools/docker/sqlmap/integration_test.go` (C3.3 wiring-validation upgrade target; `bootstrapDVWA` cookies threading)

**DRIFT-LOG:**

- shieldscan-engine `DRIFT-LOG.md` line 175 (ADR-015 reservation entry; Stage 3 Commit 1 C1.4 update target)
