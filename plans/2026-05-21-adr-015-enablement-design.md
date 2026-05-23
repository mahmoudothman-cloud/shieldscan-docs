# ADR-015 Enablement Task: Credential Transit Design

**Status:** Brainstorming chain Y1+Y2+Q1-Q7 complete this session. Pre-verification grounded existing infrastructure state. Ready for implementation plan landing.

**Date:** 2026-05-21.

**Authority:** Y1 (b) phased scope + Y2 (β) direct Q-chain bundle lock this session; Q1-Q7 brainstorming chain locks this session; pre-verification surface report V-BA through V-BI this session (existing infrastructure state grounded across api + engine + docs); Task 7.6 Drift #35 forward-pin closure target (architectural-reconciliation drift between Q7 b cookie/session forward-pin + plan §4 P1.11 assertion expectations); Task 7.6 design doc structural precedent (commit `d8e25b5`; 276 LoC); Task 7.5e ADR-026 Mounts Extension addendum precedent shape (commit `15d1ac5`); ADR-013 (sole-writer + payload contract) + ADR-014 (Redis Streams transit) cross-ADR addendum scope per Q4 (a) lock.

**Related:** Implementation plan landing trigger phrase: ***"Begin ADR-015 implementation plan landing"*** (after this design doc lands).

---

## 1. Authority + Brainstorming Chain Summary

**Scope lock Y1 (b) phased:** This task covers axes 1-4 of ADR-015 enablement: (1) ADR-015 SPEC §13 authoring + ADR-013/ADR-014 addendums; (2) api orchestrator decrypt-and-emit code + lift test pin + positive-path tests + audit logging; (3) discriminator rename `auth_type`→`type` via orchestrator-level translation; (4) SQLMap consumer cookie wiring + integration test upgrade from wiring-validation to richer assertions. Out of scope (forward-pinned): ZAP consumer (axis 5; ZAP greenfield); MobSF R2 pre-signed URL pattern (axis 6; deferred to MobSF V10 task per Q5 a); revocation flow (per Q6 a; forward-pinned to separate task).

**Brainstorming Q-chain locks:**

- **Q1 (a) Decrypted-in-Redis-transit security model:** Orchestrator decrypts ProjectCredential at scan-dispatch time; decrypted credentials transit in Redis job payload (`JobDispatch.Auth.Data`); worker consumes pre-decrypted + threads to `target.AuthConfig`. Threat surface bounded via short queue TTL + Redis authenticated access + TLS-in-transit + no-persistence Redis config. Matches pre-built infrastructure (V-BD encrypt/decrypt_credential + V-BE engine AuthConfig + jobDispatchToTarget routing already in place). Industry-standard for job-queue + worker architectures.
- **Q2 (b) All 5 AuthType values v1:** ADR-015 v1 enables all 5 typed values (cookie/bearer/basic/custom_header/form). Orchestrator decrypt+emit code is auth-type-agnostic. Engine + api infrastructure already supports all 5. Consumer-side handling per-consumer concern (SQLMap v1 cookie-only; bearer/basic/custom_header/form added per consumer need).
- **Q3 (a) Orchestrator-level discriminator translation:** ProjectCredential model keeps `auth_type` column; orchestrator translates DB `auth_type` → wire `type` at emit-time. Two contracts (DB schema + engine wire) live legitimately; translation at orchestrator boundary is canonical separation-of-concerns. ADR-013 sole-writer compatibility preserved.
- **Q4 (a) Full cross-ADR addendum coverage:** ADR-013 addendum (payload-contract extension for `auth` field semantics: nullable → typed when present) + ADR-014 addendum (transit-medium addendum for credential-secrets-in-transit posture: Redis Streams encryption-at-rest + TTL + ACL) + new ADR-015 §13 section (security model + threat model + decrypted-in-transit decision + mitigations + audit logging). Mirrors Task 7.5e ADR-026 Mounts Extension addendum precedent.
- **Q5 (a) MobSF R2 pre-signed URL deferred:** R2 pre-signed URL pattern (Task 7.4 MobSF Q6.4 forward-pin) deferred to MobSF V10 task. ADR-015 v1 scope is credential/cookie auth only; R2 binary fetch is orthogonal pattern with different threat model.
- **Q6 (a) v1 audit-only; revocation forward-pinned:** ADR-015 v1 lands credential-access audit logging (orchestrator emits audit row at decrypt-time). Revocation flow (revocation API endpoint + in-flight scan handling + scan-job cancellation + worker-side revocation handling) forward-pinned to separate task per multi-axis decision territory. Trigger phrase preserved: ***"Begin credential revocation flow task"***.
- **Q7 (a) 3-commit cross-repo:** docs FIRST (ADR-015 + ADR-013 addendum + ADR-014 addendum) → api SECOND (orchestrator + tests + pin lift + audit) → engine THIRD (SQLMap consumer cookie wiring + integration test upgrade). Each commit atomically independent + working state. Mirrors Task 7.6 docs+api+engine sequencing extended to 3-commit.

## 2. Pre-Verification Findings (V-BA through V-BI)

Pre-verification (this session) grounded existing infrastructure state before brainstorming. Critical findings inform §3 architectural decisions.

### 2.1 V-BB ADR-015 Reserved Placeholder Status (DOMINANT FINDING)

ADR-015 IS A RESERVED PLACEHOLDER in `SPECIFICATION.md`; NO ADR-015 §13 SECTION EXISTS YET. Single reference at `SPECIFICATION.md` line 1366 (ADR-013 "Open follow-ups" item: *"ADR-015 (decrypted credentials in Redis transit): defer until the orchestrator's `auth` block is enabled in job payloads (currently `null`)."*). `DRIFT-LOG.md` line 175 confirms reservation status; line 581-583 documents Task 4.2 `cf3b30a` deferral.

**Implication:** This task must (1) WRITE ADR-015 as new SPEC §13 section AND (2) implement its decisions. ADR-landing + cross-repo implementation = combined scope.

### 2.2 V-BD api Infrastructure (largely PRE-BUILT)

Existing infrastructure:

- `src/app/services/credentials.py` — `AuthType` enum {cookie, bearer, basic, form, custom_header}; `encrypt_credential` + `decrypt_credential` Fernet primitives
- `src/app/models/projects.py` — `ProjectCredential` model with `auth_type` (indexed plaintext) + `encrypted_data` (Fernet ciphertext)
- `src/app/schemas/credentials.py` — Pydantic discriminated-union schema
- `src/app/services/orchestrator.py` `_build_job_payload` — emits `auth: None` with inline `# TODO(M5+): inject decrypted credential — see ADR-015 (deferred)`
- `tests/services/test_orchestrator.py::test_dispatch_payload_auth_is_null_pending_adr_015` — regression pin

**What's missing:** orchestrator look-up of ProjectCredential per scan + decrypt + pack `{type, data}` + lift pin + positive-path test + audit logging. Scope: ~50-80 LoC api change.

### 2.3 V-BE engine Infrastructure (READY — ZERO REQUIRED CHANGES for cookie path)

- `internal/tools/runner.go` `AuthConfig` struct (5-value Type contract + Data string + Fields map)
- `internal/events/events.go` `JobAuth` struct (`{type, data}` wire shape) + `JobDispatch.Auth *JobAuth` top-level wire field; deserialization-ready
- `internal/worker/processor.go:421` `jobDispatchToTarget` — explicit `if job.Auth != nil` branch
- `internal/worker/processor_test.go:459-472` — end-to-end test proves `JobAuth{Type:"cookie",Data:"session=abc"}` → `target.AuthConfig` round-trip

Engine is READY-TO-CONSUME. Only SQLMap consumer needs cookie wiring (~3-line `buildArgs` change + integration test upgrade).

### 2.4 V-BF Redis Usage (no new primitive needed)

Existing: api side (`redis_client.py` + `scan_queue.py` RPUSH job payloads to `shieldscan:queue:{priority}` per ADR-016 + Streams via `scan_queue.py` per ADR-014) + engine side (`internal/redis/{stream,queue,idem,pubsub}.go`; `JobConsumer.BRPOP` consumes JSON-encoded `JobDispatch` payload carrying `Auth *JobAuth`). ADR-015 introduces NO new Redis primitive; populates existing nullable Auth field on existing queue payload. Security framing: "decrypted credentials in Redis transit at-rest for queue-residence duration" = decision territory of ADR-015 §13 threat model.

### 2.5 V-BG Drift #35 Forward-Pin Closure

Drift #35 (Task 7.6 commit `723426d` body) reframed integration test from "≥4 sql_injection + 1 dbms_fingerprint" to "wiring-validation" because Q7 (b) cookie/session forward-pin prevented authenticated DVWA SQLi scan. ADR-015 enablement task closes this architectural-reconciliation drift: integration test auto-upgrades to richer assertions when SQLMap consumer cookie wiring lands (axis 4).

### 2.6 V-BH SQLMap Consumer Cookie Shape

`bootstrapDVWA` helper at `integration_test.go` captures `PHPSESSID` + `security=low` cookies into `simpleCookieJar`; comment block: *"When ADR-015 enablement task lands, this test gets upgraded to assert ≥1 sql_injection + 1 dbms_fingerprint per V4 baseline"*. Consumer wire shape ADR-015 must satisfy: cookies arrive at `target.AuthConfig.Data` as `"name=value; name2=value2"` semicolon-joined per engine `AuthConfig.Data` docstring contract. SQLMap `buildArgs` ~3-line change: append `--cookie=<data>` to argv when `AuthConfig.Type == "cookie"`.

## 3. Architectural Decisions

Cross-references Q1-Q7 brainstorming chain locks (§1) + pre-verification findings (§2).

### 3.1 Q1 Security Model — Decrypted-in-Redis-Transit

**Decision:** Orchestrator decrypts ProjectCredential at scan-dispatch time; decrypted credentials transit in Redis job payload (`JobDispatch.Auth` field); worker consumes pre-decrypted + threads to `target.AuthConfig`.

**Threat model:** Decrypted credentials at-rest in Redis for queue-residence duration (typically seconds-to-minutes per BullMQ-style queue semantics). Attack surface: anyone with Redis read access during queue-residence window.

**Mitigations (mandatory v1):** (a) Redis authenticated access (existing); (b) TLS-in-transit between api/engine ↔ Redis (existing/required); (c) Short queue TTL (queue-residence-duration bounded at scan-dispatch-latency); (d) Redis no-persistence config (`appendonly no`; no AOF write of payload).

**Mitigations (recommended v1.1+):** (e) Redis ACL per-queue (credential-bearing queues read-restricted to worker-pool identity); (f) Encryption-at-rest at Redis layer (TLS + AOF cipher if persistence enabled).

**Industry alignment:** Standard pattern for job-queue + worker architectures (BullMQ + RabbitMQ + SQS workflows). Alternative fetch-by-reference + signed-token patterns rejected per (b) Vault integration + worker DB ACLs requirement (operational complexity) + (c) new attack surface (token signing/validation) without proportional security gain.

### 3.2 Q2 AuthType Coverage

**Decision:** ADR-015 v1 enables all 5 typed values: cookie + bearer + basic + custom_header + form. Orchestrator decrypt+emit code is auth-type-agnostic. Per-consumer cookie wiring lands per consumer task.

**v1 consumer support:** SQLMap consumer (axis 4) handles cookie type only; bearer/basic/custom_header/form forward-pinned per consumer need.

**Forward extensibility:** SQLMap v1.1+ `buildArgs` extends to bearer (`--header`), basic (`--auth-cred`), custom_header (`--headers`), form (`--data`) as scan-target-need surfaces.

### 3.3 Q3 Discriminator Translation

**Decision:** ProjectCredential model keeps `auth_type` column (DB schema unchanged); orchestrator translates DB `auth_type` → wire `type` at emit-time.

**Two-contract canonical separation:** DB schema (`auth_type`) + engine wire (`type`) are different contracts with different naming conventions. Translation at orchestrator boundary is canonical separation-of-concerns per ADR-013 sole-writer pattern.

**Migration economy:** Zero DB migration risk; zero API contract breakage; bounded ~3-line orchestrator translation logic.

### 3.4 Q4 Cross-ADR Addendum Coverage

**Decision:** Full coverage — ADR-013 addendum + ADR-014 addendum + new ADR-015 §13 section.

**ADR-013 addendum content:** Payload contract extension for `auth` field semantics — nullable (no credential) → typed when present (`{type: <auth_type>, data: <decrypted-blob>, fields?: <form-fields>}`); referenced ADR-015 v1 enablement.

**ADR-014 addendum content:** Transit-medium addendum noting credential-secrets-in-transit posture under decrypted-in-Redis-transit model; mitigations enumerated (short TTL + no-persistence + ACL + TLS); references ADR-015 §13 threat model.

**ADR-015 §13 new section:** Full text drafted in §4.

### 3.5 Q5 R2 Pre-Signed URL — Deferred

**Decision:** MobSF R2 pre-signed URL pattern (Task 7.4 Q6.4 forward-pin) deferred to MobSF V10 task. ADR-015 v1 scope is credential/auth-secrets-in-transit only.

**Rationale:** R2 pre-signed URLs operate on shieldscan-internal storage references (different threat model); credentials operate on user-controlled auth secrets (different mitigation surface). Bundling violates threat-model coherence.

### 3.6 Q6 Audit Logging Scope

**Decision v1:** Audit logging at credential-decrypt time. Orchestrator emits audit row `{timestamp, project_id, credential_id, scan_id, dispatcher_user_id}` per scan-job-dispatch consuming a credential.

**Revocation forward-pinned:** Revocation flow (revocation API endpoint + in-flight scan handling + scan-job cancellation + worker-side revocation handling) is multi-axis territory; reserved for separate task. Trigger phrase preserved: ***"Begin credential revocation flow task"*** (TBD; conditional on production credential-management requirements).

### 3.7 Q7 3-Commit Cross-Repo Sequencing

**Decision:** docs FIRST (ADR-015 §13 + ADR-013 addendum + ADR-014 addendum) → api SECOND (orchestrator decrypt-and-emit + lift test pin + positive-path tests + audit logging) → engine THIRD (SQLMap consumer cookie wiring + integration test upgrade).

**Atomic-independence preserved:** Each commit produces working state. Docs commit lands canonical authority. Api commit enables credential transit + tests + audit (engine still passive). Engine commit lands SQLMap consumer extension + integration test richer assertions.

**Cross-reference shape:** Docs commit references future api+engine hashes via placeholders; api commit cross-references docs hash; engine commit cross-references both docs + api hashes. Mirrors Task 7.6 docs+api+engine sequencing extended to 3-commit.

## 4. ADR-015 SPEC §13 Content Draft

The new ADR-015 section to land at `SPECIFICATION.md` §13 (alphabetically between ADR-014 and ADR-016 per existing §13 numbering convention).

```
### ADR-015: Decrypted Credentials in Redis Transit (2026-05-21)

**Context:** Scan targets often require authentication (cookies, bearer
tokens, basic-auth, custom headers, form-based login) to reach
injectable surfaces. Credentials are stored encrypted (Fernet) in
ProjectCredential.encrypted_data. Workers (engine) execute scans
against authenticated targets and need decrypted credentials at
runtime. The architectural question is: where does decryption happen
— orchestrator-side (transit decrypted via Redis) or worker-side
(transit encrypted; worker decrypts)?

**Decision:** Orchestrator decrypts ProjectCredential at scan-dispatch
time and emits decrypted credentials in the Redis job payload
(JobDispatch.Auth field). Workers consume pre-decrypted credentials
and thread them to target.AuthConfig for consumer-side use.
Alternative "fetch-by-reference" (worker fetches ciphertext +
decrypts) was considered and rejected per (a) inverting current
pre-built infrastructure; (b) requiring worker-side Vault integration
+ DB ACLs; (c) longer attack window (DB + key distribution).

**Architectural pattern:** Decrypted-in-transit with bounded threat
surface.

- Threat surface: decrypted credentials at-rest in Redis for
  queue-residence duration (typically seconds-to-minutes per BullMQ
  semantics).
- Mandatory v1 mitigations: (1) Redis authenticated access; (2)
  TLS-in-transit between api/engine ↔ Redis; (3) short queue TTL
  (bounded queue-residence-duration); (4) Redis no-persistence config
  (appendonly no; no AOF write of credential-bearing payloads).
- Recommended v1.1+ mitigations: (5) Redis ACL per-queue
  (credential-bearing queues read-restricted to worker-pool identity);
  (6) encryption-at-rest at Redis layer (TLS + AOF cipher if
  persistence enabled).

**AuthType coverage:** ADR-015 v1 enables all 5 typed values per
existing AuthType enum (cookie, bearer, basic, custom_header, form).
Orchestrator decrypt+emit code is auth-type-agnostic. Per-consumer
handling is per-consumer concern (SQLMap v1 cookie-only;
bearer/basic/custom_header/form land per consumer need).

**Discriminator translation:** ProjectCredential model keeps
`auth_type` DB column (DB schema unchanged). Orchestrator translates
DB `auth_type` → wire `type` at payload-emit time. Two-contract
separation (DB schema vs engine wire) is canonical
separation-of-concerns; translation is orchestrator's sole-writer
responsibility per ADR-013.

**Audit logging:** Orchestrator emits audit row at decrypt-time:
`{timestamp, project_id, credential_id, scan_id, dispatcher_user_id}`
per scan-job dispatch consuming a credential. Audit table (TBD; can
land as credential_access_audit). v1 includes audit; revocation flow
forward-pinned to separate task.

**Out of scope (forward-pinned):** (a) credential revocation flow —
separate task (trigger: "Begin credential revocation flow task"); (b)
per-credential rate limits; (c) credential rotation (use existing
application-level patterns); (d) R2 pre-signed URL pattern
(forward-pinned to MobSF V10 task; orthogonal threat model).

**Cross-references:** ADR-013 (sole-writer + payload contract;
addendum extends `auth` field semantics) + ADR-014 (Redis Streams
transit; addendum captures credential-transit posture mitigations) +
ADR-024 (RawFinding schema; unaffected) + ADR-027 (Metadata;
unaffected). Task 4.2 cf3b30a (orchestrator deferral; lifted by this
ADR). Task 7.3 (ZAP design forward-pin; consumer-side cookie
pass-through). Task 7.6 (SQLMap Drift #35 architectural-reconciliation;
integration test auto-upgrade target). DRIFT-LOG.md (lines 175 +
581-583; reservation status). engine commits 723426d (Task 7.6
cross-repo pair Commit 2; Drift #35 origin) + ba860a5 (Task 7.5e
Phase 1+2; framework Mounts capability + Entrypoint fix). docs
commits 15d1ac5 (Task 7.5e ADR-026 Mounts Extension addendum
precedent; addendum shape) + c28a5de (Task 7.6 P5.C latest docs
state). api commits 2cd4065 (Task 7.6 SCAN_TYPE_TOOLS; SQLMap
routing; auth-block null currently) + e6fb0a5 (Task 7.1; orchestrator
scaffold).

**Selected.** Decrypted-in-Redis-transit is the canonical pattern for
job-queue + worker architectures; matches existing pre-built
infrastructure; bounded threat surface mitigatable via existing Redis
security practices.
```

## 5. ADR-013 + ADR-014 Addendum Content Drafts

### 5.1 ADR-013 Addendum (Payload Contract Auth-Field Extension)

Lands at `SPECIFICATION.md` ADR-013 section (after existing Open follow-ups; preserves chronological addendum ordering).

```
### ADR-013 Addendum: Payload Contract Auth-Field Extension
(ADR-015 Enablement; 2026-05-21)

Extends payload contract per ADR-015 enablement: `auth` field
semantics — nullable (no credential provided) → typed when present
(`{type: <auth_type>, data: <decrypted-blob>, fields?:
<form-fields-map>}`). Orchestrator is sole writer of credential-
bearing payloads per ADR-013 canonical authority. Cross-reference
ADR-015 §13 for security model + mitigations + threat surface.

**Selected.**
```

### 5.2 ADR-014 Addendum (Credential Transit Posture)

Lands at `SPECIFICATION.md` ADR-014 section (after existing Open follow-ups; preserves chronological addendum ordering).

```
### ADR-014 Addendum: Credential Transit Posture
(ADR-015 Enablement; 2026-05-21)

Extends Redis Streams transit medium per ADR-015 enablement:
credential-bearing payloads carry decrypted credentials at-rest in
Redis for queue-residence duration. Mitigations enumerated in
ADR-015 §13 — Redis authenticated access + TLS-in-transit + short
queue TTL + no-persistence config (appendonly no) for credential-
bearing queues. v1.1+ adds per-queue ACL + AOF cipher if persistence
enabled.

**Selected.**
```

## 6. Phase Structure

Per Y1 (b) phased + Q7 (a) 3-commit lifecycle:

### Stage 1 — Design Doc Landing (THIS COMMIT; this session)

Lands design doc at `plans/2026-05-21-adr-015-enablement-design.md`. ~330-400 LoC (denser than Task 7.6 design doc 276 LoC due to inline SPEC §13 + addendum drafts). No code change; no SPEC change; ADR-015 + addendum content drafted in §4 + §5 for Stage 3 docs commit.

### Stage 2 — Implementation Plan Landing (~30-45min next session OR same session)

Lands plan doc at `plans/2026-05-21-adr-015-enablement-implementation.md` per Task 7.6 plan precedent shape (`40606c5`; 269 LoC). Plan structures Stage 3 sub-steps + cross-repo commit ordering.

### Stage 3 — 3-Commit Cross-Repo Implementation (~2-3h)

**Commit 1 (docs FIRST):** SPEC §13 lands ADR-015 + ADR-013 addendum + ADR-014 addendum. `DRIFT-LOG.md` line 175 reservation entry updated to "landed". Cross-references future api+engine hashes via placeholders. ~80-120 LoC delta.

**Commit 2 (api SECOND):** orchestrator decrypt-and-emit + lift `test_dispatch_payload_auth_is_null_pending_adr_015` pin + positive-path test (verify cookie-type credential reaches payload) + audit logging at decrypt-time + discriminator translation `auth_type`→`type` at emit-time. ~80-120 LoC delta. Cross-references docs hash from Commit 1.

**Commit 3 (engine THIRD):** SQLMap consumer cookie wiring (`buildArgs` append `--cookie=<data>` when `target.AuthConfig.Type=="cookie"`) + integration test upgrade (lift wiring-validation; assert ≥1 sql_injection + 1 dbms_fingerprint per V4 baseline; threads `bootstrapDVWA` cookies through `Target.AuthConfig`). ~50-80 LoC delta. Cross-references docs + api hashes.

### Stage 4 — Phase 5 Sub-Phases (~1-2h conditional)

Mirrors Task 7.6 Phase 5 + Task 7.1 Phase 5 patterns.

- **P5.A** Design doc drift annotations (drifts caught during Stage 3 implementation)
- **P5.B** SPEC further addendum work — LIKELY OUTCOME γ (ADR-013/ADR-014 addendums landed at Stage 3 Commit 1)
- **P5.C** SPEC §3.2 directory annotation — LIKELY OUTCOME γ (no new package directory)
- **P5.D** DRIFT-LOG entry — credential-transit pattern documentation if 2nd-instance pattern surfaces
- **P5.E** §14.1 invocation tracking — LIKELY OUTCOME γ

## 7. Out of Scope (Y1 (b) phased lock + Q5/Q6 deferrals)

1. **ZAP consumer cookie pass-through** (axis 5) — ZAP greenfield (not yet shipped); land in ZAP consumer task
2. **MobSF R2 pre-signed URL pattern** (axis 6; Task 7.4 Q6.4 forward-pin) — deferred to MobSF V10 task
3. **Credential revocation flow** (Q6 a forward-pin) — multi-axis territory deserving own task
4. **Per-credential rate limits** — orthogonal to ADR-015 v1 scope
5. **Credential rotation flow** — use existing application-level patterns; not ADR-015 scope
6. **bearer/basic/custom_header/form SQLMap consumer wiring** — v1 SQLMap consumer cookie-only; other types per consumer need
7. **Per-tenant Redis isolation** — multi-tenant Redis ACL strategy is v1.1+ recommended; v1 single-tenant by default
8. **3rd-instance per-section-adaptor pattern** (Task 7.6 P5.D forward-pin) — unrelated; preserved separately

## 8. Forward-Pins

**Pre-execution forward-pins (Stage 2 entry):**

1. **Stage 2 plan trigger phrase:** ***"Begin ADR-015 implementation plan landing"***

**Post-Stage-3 forward-pins:**

2. **ZAP consumer task** — cookie pass-through enablement (consumer task; not ADR-015 scope)
3. **MobSF V10 task** — R2 pre-signed URL pattern enablement (per Task 7.4 Q6.4 + Q5 a deferral)
4. **Credential revocation flow task** — trigger phrase: ***"Begin credential revocation flow task"***
5. **v1.1+ Redis ACL per-queue** — per-tenant Redis security enhancement
6. **SQLMap consumer non-cookie AuthType support** — bearer/basic/custom_header/form per consumer need
7. **Audit log retention + analytics** — credential-access-pattern analysis (out of ADR-015 v1 scope)

## 9. Cross-References

**Engine commits:**

- `06ab650` (Task 7.6 P5.D; DRIFT-LOG 2nd-instance per-section adaptor; latest engine state)
- `723426d` (Task 7.6 cross-repo pair Commit 2; Drift #35 origin; SQLMap consumer integration test wiring-validation reframing)
- `ba860a5` (Task 7.5e Phase 1+2; framework Mounts capability + Entrypoint fix universal benefit)
- `cf3b30a` (Task 4.2; orchestrator scan-job dispatcher; auth: None deferral origin)

**Docs commits:**

- `c28a5de` (Task 7.6 P5.C; SPEC §3.2 directory annotation; latest docs state)
- `b187d63` (Task 7.6 P5.A; design doc drift annotations)
- `40606c5` (Task 7.6 implementation plan; structural precedent for ADR-015 Stage 2 plan)
- `d8e25b5` (Task 7.6 design doc; structural precedent for this design doc)
- `15d1ac5` (Task 7.5e Commit 1; ADR-026 Mounts Extension addendum precedent for ADR-013/ADR-014 addendum shape)

**API commits:**

- `2cd4065` (Task 7.6 cross-repo pair Commit 1; SCAN_TYPE_TOOLS SQLMap routing; auth:None currently)
- `e6fb0a5` (Task 7.1 cross-repo pair Commit 1; orchestrator scaffold)

**SPEC sections:**

- §13 ADR-013 (sole-writer + payload contract; Q4 a addendum target)
- §13 ADR-014 (Redis Streams transit; Q4 a addendum target)
- §13 ADR-015 (NEW SECTION; Q4 a content draft in §4)
- §13 ADR-024 (RawFinding schema; unaffected)
- §13 ADR-027 (Metadata schema; unaffected)

**Source authorities:**

- shieldscan-api `src/app/services/credentials.py` (AuthType enum + Fernet encrypt/decrypt)
- shieldscan-api `src/app/models/projects.py` (ProjectCredential model)
- shieldscan-api `src/app/schemas/credentials.py` (Pydantic discriminated union)
- shieldscan-api `src/app/services/orchestrator.py` (Q3 a discriminator translation site + Q6 a audit emission site + Q1 a decrypt+emit site)
- shieldscan-api `tests/services/test_orchestrator.py` (`test_dispatch_payload_auth_is_null_pending_adr_015` pin lift target)
- shieldscan-engine `internal/tools/runner.go` (AuthConfig contract; consumer-side `target.AuthConfig.Data` routing)
- shieldscan-engine `internal/events/events.go` (JobAuth wire shape; `JobDispatch.Auth` field)
- shieldscan-engine `internal/worker/processor.go` (`jobDispatchToTarget`; Q1 a consume-pre-decrypted pattern; line 421 if-branch)
- shieldscan-engine `internal/tools/docker/sqlmap/scan.go` (`buildArgs` cookie-injection extension site; Stage 3 Commit 3 scope)
- shieldscan-engine `internal/tools/docker/sqlmap/integration_test.go` (`bootstrapDVWA` cookie shape; Stage 3 Commit 3 integration test upgrade target)

**DRIFT-LOG:**

- shieldscan-engine `DRIFT-LOG.md` line 175 (ADR-015 reservation entry; updated to "landed" at Stage 3 Commit 1)
- shieldscan-engine `DRIFT-LOG.md` line 581-583 (Task 4.2 deferral documentation; cross-reference target)
