# MobSF R2 Pre-Signed URL Pattern: Design

**Status:** Brainstorming chain Y1+Y2+Q1-Q8 complete this session. Forward-pin chain Task 7.4 Q6.4 → ADR-015 Q5 (a) → V10 Q5 (a) → this task settled. Ready for implementation plan landing.

**Date:** 2026-05-23.

**Authority:** Y1 (MobSF-only sole R2 consumer per V-DL) + Y2 (direct Q-chain; no Phase 0 v2 territory) + Q1 (β) + Q2 (a) + Q3 (a) + Q4 (b) + Q5 (a) + Q6 (a) + Q7-refined + Q8 (a) brainstorming chain locks this session; pre-verification surface report V-DA-V-DL (this session); Task 7.4 Q6.4 forward-pin canonical text (`plans/2026-05-09-task-7.4-mobsf-consumer-design.md`); ADR-015 Q5 (a) threat-model-coherence rationale (`plans/2026-05-21-adr-015-enablement-design.md` §3.5); V10 Q5 (a) re-deferral (`plans/2026-05-22-task-7.4-v10-ios-section-dispatch-design.md` §3.5); ADR-015 enablement Stage 3 trio cross-repo precedent (`9a57865` + `742faed` + `b48fef8`); V10 Stage 3 cross-repo precedent (`d4f6ca7` + `ad7cc94` + `06c444c`).

**Related:** Implementation plan landing trigger phrase: ***"Begin MobSF R2 implementation plan landing"*** (after this design doc lands).

---

## 1. Authority + Brainstorming Chain Summary

**Y1 (locked by V-DL empirical reality) MobSF-only scope:** MobSF is sole R2 consumer per engine grep (only `r2.go` + `mobsf.go` reference R2; no other consumer in `internal/tools/`). 1-instance threshold; framework-level Fetch-via-URL pattern NOT warranted at v1 per Task 7.4 future-work *"Framework-level R2 client promotion — when 2nd consumer requires R2/S3 binary fetch"*. Forward-pin: ***"Begin framework-level Fetch-via-URL pattern task"*** at 2nd R2-consumer surfacing.

**Y2 (locked by territory) Direct Q-chain → design doc:** R2 task is architectural-decision-only (orchestrator generates signed URL; engine consumes via plain HTTP; no empirical iOS-style report shape discovery). Mirrors ADR-015 brainstorming order; no Phase 0 v2 cycle.

**Brainstorming Q-chain locks:**

- **Q1 (β) Sibling `SignedFetchURL` field added alongside `UploadRef`:** Backward-compatible migration path; engine prefers signed URL if present, falls back to `r2://` `UploadRef` during transition window. Both fields populated during migration. Wire schema change: +1 new field at `JobMobileConfig`.
- **Q2 (a) Fixed short expiry 600s:** Aligns with ADR-015 short-Redis-queue-TTL discipline + ephemeral-access-token semantic; 10x buffer over expected BullMQ queue-residence (seconds-to-minutes); reduces stolen-URL exposure window.
- **Q3 (a) Prefer `SignedFetchURL`; fallback to `UploadRef`:** Engine consumes both shapes during deploy-window migration. Migration plan: deploy orchestrator first (emits both fields) → deploy engine (consumes `SignedFetchURL` when present) → after stable, orchestrator drops `UploadRef` → after that stable, engine drops `r2.go` fallback. Per Q7-refined: `r2.go` deletion deferred to migration-close task.
- **Q4 (b) Addendums to ADR-013 + ADR-014:** Mirrors ADR-015 enablement multi-ADR-addendum pattern (`9a57865` §13). ADR-013 addendum: payload-contract extension for `SignedFetchURL` field. ADR-014 addendum: transit-medium addendum for signed-URL-in-Redis-transit posture. No new ADR creation (canonical-authority-efficiency).
- **Q5 (a) `SCAN_PRESIGNED_URL_GENERATED` audit:** Mirrors ADR-015 `SCAN_CREDENTIAL_DECRYPTED` pattern (orchestrator-as-sole-writer-of-time-bounded-access-tokens for both credentials + URLs). Architectural symmetry. New `ScanAction` enum value + `audit()` emission at URL generation; existing audit infrastructure consumed.
- **Q6 (a) 3-commit trio:** docs (SPEC §13 ADR-013 + ADR-014 addendums) → api (`r2.py` `generate_presigned_get_url` + orchestrator integration + audit + tests) → engine (`events.go` `SignedFetchURL` field + `mobsf.go` consumer refactor + tests). Mirrors ADR-015 + V10 Stage 3 precedent exactly.
- **Q7-refined Retain `r2.go` this task; forward-pin deletion:** Original Q7 (a) *"delete `r2.go` entirely"* required Q3 (b) hard cutover; Q3 (a) backward-compat fallback requires `r2.go` retention during migration window. Deletion forward-pinned to ***"Begin MobSF R2 migration-close task"*** when `SignedFetchURL` adoption empirically stable in production.
- **Q8 (a) Extend `src/app/services/r2.py`:** Existing `get_r2_client()` helper extended with `generate_presigned_get_url(r2_key, expiry=600)` wrapper. Co-located with existing R2 utilities; bounded ~10-15 LoC; consumes `boto3.client.generate_presigned_url` built-in (zero new dependencies per V-DK).

**Forward-pin chain closure (3 prior tasks → this task → 2 forward-pin successors):**

Task 7.4 Q6.4 (env-var v1 lock; pre-signed URL forward-pin) → ADR-015 Q5 (a) (threat-model coherence deferral) → V10 Q5 (a) (no V10↔R2 coupling re-deferral) → **THIS TASK** (chain settles operationally). Forward-pin successors: ***"Begin MobSF R2 migration-close task"*** (Q7-refined; `r2.go` deletion + `UploadRef` emission removal) + ***"Begin framework-level Fetch-via-URL pattern task"*** (Y1 forward-pin; 2nd R2-consumer trigger).

## 2. Pre-Verification Findings

Pre-verification (this session) grounded R2 actual scope + canonical authority text + engine code shape + orchestrator integration point + SDK availability before brainstorming. Critical findings:

**V-DB Task 7.4 Q6.4 canonical:** *"engine env vars for R2 credentials v1 (Q6.4 lock; pre-signed URL pattern forward-pinned to ADR-015 enablement task)"*.

**V-DC ADR-015 Q5 (a) verbatim:** *"R2 pre-signed URLs operate on shieldscan-internal storage references (different threat model); credentials operate on user-controlled auth secrets (different mitigation surface). Bundling violates threat-model coherence."* Threat-model differentiation preserved as task-boundary discipline.

**V-DD V10 Q5 (a) verbatim:** *"No V10 ↔ R2 architectural coupling surfaced in Phase 0 v2. R2 pre-signed URL pattern forward-pinned to dedicated task."* Chain settled here.

**V-DE engine `r2.go` current shape:** `R2Config{Endpoint, AccessKeyID, SecretAccessKey, Bucket}` + `r2Fetcher` interface + `s3R2Fetcher` production impl with AWS SDK v2 `credentials.NewStaticCredentialsProvider` + `s3.Client{Region:"auto", BaseEndpoint, UsePathStyle:true}` + `GetObject` + temp-file staging. Direct S3 SDK; no pre-signed handling.

**V-DF MobSF binary fetch flow:** `mobsf.go` `Config{APIKey, R2 R2Config, FetcherOverride r2Fetcher}` → `NewBuildScan` closure → `validateMobileTarget` → `fetcher.Fetch(ctx, mt.UploadRef)` → `localPath, cleanup` → `uploadFile` multipart POST to MobSF `/api/v1/upload`. Refactor target: replace `fetcher.Fetch()` with plain HTTP GET against `SignedFetchURL`.

**V-DG wire payload `JobMobileConfig` shape:** Current `{UploadRef "upload_ref" string + Platform + AnalysisType}`; `UploadRef` carries `"r2://<key>"` reference. Q1 (β) extends with `SignedFetchURL string \`json:"signed_fetch_url,omitempty"\``.

**V-DH orchestrator `dispatch()` post-ADR-015 state:** `dispatch()` line 220-280 has ADR-015 credential-decrypt block (`742faed`); line 290-303 mobile_config block emits `r2://{upload.r2_key}`. R2 pre-signed URL generation integration point = parallel to credential-decrypt block (orchestrator-as-sole-writer-of-time-bounded-access-tokens architectural analog).

**V-DI engine config layering + ADR-013 sole-writer:** `SHIELDSCAN_R2_*` env-loaded in `config.go` `Load()` (worker-side credentials currently). ADR-013 sole-writer canonical scope (*"Python is the SOLE writer for Scan + ScanJob status columns"*) extends via ADR-013 addendum to *"ephemeral access tokens (URLs)"* per Q4 (b) pattern.

**V-DJ Go AWS SDK v2 engine state:** `aws-sdk-go-v2 v1.32.2` + `service/s3 v1.66.0` in `go.mod`. Engine consume-only (zero `PresignClient` usage). Post-refactor: engine drops `s3.Client` from MobSF consumer → `net/http` GET against `SignedFetchURL`. Reduces engine dependency footprint at consumer level.

**V-DK Python AWS SDK on api:** `boto3 ^1.35.0` in `pyproject.toml`. `src/app/services/r2.py` already exists with `get_r2_client()` returning boto3 S3 client; only `put_object` exposed. `boto3.client.generate_presigned_url` is built-in (zero new dependency); ~10-15 LoC wrapper at `r2.py` per Q8 (a).

**V-DL Cross-task R2 consumer scope:** MobSF is SOLE R2 consumer (engine grep confirms `r2.go` + `mobsf.go` only). 1-instance threshold; Y1 MobSF-only lock empirically grounded.

## 3. Architectural Decisions

Cross-references Q1-Q8 brainstorming chain locks (§1) + pre-verification findings (§2).

### 3.1 Q1 (β) Sibling SignedFetchURL Wire Field

**Decision:** `JobMobileConfig` extended with `SignedFetchURL` field alongside existing `UploadRef`. Both fields populated during migration window.

**Implementation surface:** `internal/events/events.go` `JobMobileConfig` struct extension. ~3-5 LoC delta. Wire JSON: `{"upload_ref":"r2://...", "signed_fetch_url":"https://...?X-Amz-...", "platform":"...", "analysis_type":"..."}`.

**Migration safety:** Engine deployment timing independent from api deployment timing. Old engine consuming new orchestrator payload: ignores unknown `signed_fetch_url` field; consumes `UploadRef` as today. New engine consuming old orchestrator payload: `signed_fetch_url` empty/absent; falls back to `UploadRef` per Q3 (a).

**Rejected alternatives:** Q1 (α) backward-incompatible `UploadRef` semantics shift; Q1 (γ) field rename complicates migration.

### 3.2 Q2 (a) Fixed 600s URL Expiry

**Decision:** Pre-signed URLs valid for 600 seconds (10 minutes) from generation.

**Rationale:** Aligns with ADR-015 ephemeral-access-token + short-Redis-queue-TTL discipline; BullMQ queue-residence typically seconds-to-minutes; 10x buffer; stolen-URL exposure window minimized.

**Adaptive option deferred:** Q2 (c) per-scan-deadline-derived forward-pinned to v1.1+ enhancement task if empirical queue-delays exceed 600s threshold.

### 3.3 Q3 (a) Engine Prefer-SignedFetchURL + UploadRef Fallback

**Decision:** Engine consumer (MobSF) checks for `SignedFetchURL` first; uses plain HTTP GET if present + non-empty. Falls back to existing `r2://` `UploadRef` path via `r2Fetcher` interface if `SignedFetchURL` absent/empty.

**Implementation surface:** `internal/tools/docker/service/mobsf/mobsf.go` `NewBuildScan` closure refactor — branch on `target.SignedFetchURL` presence; new `httpFetcher` (plain HTTP GET; `net/http` only) for signed-URL path; preserve existing `r2Fetcher` for fallback path. ~30-50 LoC delta.

**Migration window:** Bounded by deployment-stability metric; deletion forward-pinned to Q7-refined *"Begin MobSF R2 migration-close task"*.

### 3.4 Q4 (b) ADR-013 + ADR-014 Addendums

**Decision:** Land 2 SPEC §13 addendums.

**ADR-013 Addendum: Payload Contract Pre-Signed URL Extension.** Orchestrator emits both `UploadRef` + `SignedFetchURL`; sole-writer responsibility preserved per ADR-013 canonical. ~5-10 LoC.

**ADR-014 Addendum: Pre-Signed URL Transit Posture.** Signed URLs at-rest in Redis JobDispatch payloads for queue-residence duration; mitigations (URL expiry policy per Q2 (a) + Redis authenticated access per ADR-014); analog to ADR-015 credential transit. ~5-10 LoC.

**Rejected alternatives:** Q4 (a) new ADR (canonical-authority overhead); Q4 (c) ADR-008 addendum (future-consumer scope-mismatch risk).

### 3.5 Q5 (a) SCAN_PRESIGNED_URL_GENERATED Audit

**Decision:** Add `ScanAction.SCAN_PRESIGNED_URL_GENERATED = "scan.presigned_url_generated"` enum value at `src/app/services/audit.py`. Emit via existing `audit()` helper at orchestrator URL-generation time with `details={"r2_key": <key>, "expiry_seconds": 600, "scan_id": <scan-id>}`.

**Architectural analog:** ADR-015 `SCAN_CREDENTIAL_DECRYPTED` pattern; orchestrator-as-sole-writer-of-time-bounded-access-tokens; symmetric forensics posture.

**Implementation surface:** ~10-15 LoC enum addition + audit emission inline at orchestrator `dispatch()`.

### 3.6 Q6 (a) 3-Commit Cross-Repo Trio

**Decision:** docs (SPEC ADR-013/ADR-014 addendums) → api (`r2.py` + orchestrator + audit + tests) → engine (`events.go` + `mobsf.go` consumer refactor + tests).

**Sequence:** Commit 1 docs lands SPEC §13 ADR-013 + ADR-014 addendums; Commit 2 api lands `r2.py` extension + orchestrator `dispatch()` integration + `ScanAction` enum + positive-path tests; Commit 3 engine lands `SignedFetchURL` field + `mobsf.go` consumer refactor + `httpFetcher` + tests + DRIFT-LOG entry.

**Cross-reference shape:** Commit 1 docs references future api+engine hashes via placeholders; Commit 2 api references docs Commit 1 hash + engine placeholder; Commit 3 engine references both docs + api hashes concretely. Mirrors ADR-015 Stage 3 trio + V10 Stage 3 trio precedent.

### 3.7 Q7-Refined r2.go Retention + Forward-Pin Deletion

**Decision (refined from initial Q7 a lock):** Retain `r2.go` production code in this task to support Q3 (a) `UploadRef` fallback during migration window. Original Q7 (a) *"delete `r2.go` entirely"* assumed clean cutover incompatible with Q3 (a) backward-compat.

**Forward-pin trigger:** ***"Begin MobSF R2 migration-close task"*** — Q3 (a) fallback path retired; `r2.go` deletion + `UploadRef` emission removal + `s3.Client` + `credentials` imports dropped.

**Migration-stability signal:** Production metric *"≥30 days of zero `UploadRef` fallback consumption"* OR explicit migration-close decision by Mahmoud.

### 3.8 Q8 (a) r2.py Helper Extension

**Decision:** Extend `src/app/services/r2.py` with `generate_presigned_get_url(r2_key, expiry=600)` async helper. Reuses existing `get_r2_client()`; calls `boto3.client.generate_presigned_url("get_object", Params={Bucket, Key}, ExpiresIn)`.

**Implementation surface:** ~10-15 LoC at `r2.py`. Zero new dependencies.

## 4. Cross-Repo Implementation Surface

### 4.1 Docs Side (Stage 3 Commit 1)

**`SPECIFICATION.md` §13 additions:**

- ADR-013 Addendum: Payload Contract Pre-Signed URL Extension (~5-10 LoC)
- ADR-014 Addendum: Pre-Signed URL Transit Posture (~5-10 LoC)

**Total docs delta:** ~15-25 LoC. Mirrors ADR-015 Stage 3 Commit 1 `9a57865` shape.

### 4.2 API Side (Stage 3 Commit 2)

- `src/app/services/r2.py`: `generate_presigned_get_url` helper extension (~10-15 LoC)
- `src/app/services/audit.py`: `ScanAction.SCAN_PRESIGNED_URL_GENERATED` enum value (~1-2 LoC)
- `src/app/services/orchestrator.py`: `dispatch()` mobile_config block extension — call `generate_presigned_get_url` + emit `SignedFetchURL` alongside `UploadRef` + `audit()` emission (~15-25 LoC)
- `tests/services/test_orchestrator.py`: Positive-path test (verify `SignedFetchURL` emitted alongside `UploadRef` when mobile scan) + audit-row test (verify `SCAN_PRESIGNED_URL_GENERATED`) + URL expiry verification (~40-60 LoC)

**Total api delta:** ~70-100 LoC.

### 4.3 Engine Side (Stage 3 Commit 3)

- `internal/events/events.go`: `JobMobileConfig` `SignedFetchURL` field extension (~3-5 LoC)
- `internal/tools/runner.go`: `Target.SignedFetchURL` field extension if needed (~2-3 LoC; verify at execution)
- `internal/tools/docker/service/mobsf/mobsf.go`: `NewBuildScan` closure refactor — `httpFetcher` when `SignedFetchURL` present; fallback to `r2Fetcher` otherwise (~30-50 LoC)
- `internal/tools/docker/service/mobsf/http_fetcher.go` (NEW): plain HTTP GET fetcher implementation (~30-50 LoC)
- `internal/tools/docker/service/mobsf/mobsf_test.go`: `SignedFetchURL` preference tests + `UploadRef` fallback tests (~40-60 LoC)
- `DRIFT-LOG.md`: R2 entry inline per Y-DRIFT-LOG-PLACEMENT (a) precedent (~10-20 LoC)

**Total engine delta:** ~115-188 LoC.

### 4.4 Aggregate Stage 3 LoC

Total across 3 commits: **~200-313 LoC**. Smaller than V10 Stage 3 (~500-770 LoC) per scope (no parser refactor; no 5 adaptors); comparable to ADR-015 Stage 3 (~205-340 LoC).

## 5. Phase Structure

Per Q6 (a) 3-commit cross-repo + Y2 direct Q-chain (no Phase 0 v2):

### Stage 1 — Design Doc Landing (THIS COMMIT; this session)

Lands design doc at `plans/2026-05-23-mobsf-r2-presigned-url-design.md`. ~280-320 LoC (similar to V10 280 LoC). No code change.

### Stage 2 — Implementation Plan Landing (~30-45min same session if budget allows OR next session)

Lands plan doc at `plans/2026-05-23-mobsf-r2-presigned-url-implementation.md`. ~250-310 LoC per ADR-015 + V10 plan precedent shapes.

### Stage 3 — 3-Commit Cross-Repo Implementation (~2-3h)

Per §4.

### Stage 4 — Phase 5 Sub-Phases (rolled into Stage 3 Commit 3 OR separate)

Per Task 7.6 + ADR-015 + V10 Phase 5 precedent.

## 6. Out of Scope

1. `r2.go` deletion + `UploadRef` emission removal (Q7-refined forward-pin to ***"Begin MobSF R2 migration-close task"***)
2. Framework-level Fetch-via-URL pattern (Y1 forward-pin to ***"Begin framework-level Fetch-via-URL pattern task"*** when 2nd R2-consumer surfaces)
3. Per-scan-deadline-derived URL expiry (Q2 (c) forward-pin to v1.1+ enhancement task)
4. URL refresh-on-expiry pattern (Q2 (d) forward-pin to scale-up task if empirical queue-delays surface)
5. New ADR creation (Q4 (a) rejected; addendum pattern preserved)
6. R2 client SDK on api side beyond boto3 (Q8 (a) confirmed; no new dependency)
7. Hard cutover deployment (Q3 (b) rejected; backward-compat migration window preserved)
8. Non-MobSF consumer integration (V-DL grounds; sole consumer scope)
9. Migration timeline / production-stability signal definition (out of design scope; operational concern)
10. Audit log retention + analytics (out of v1 scope; existing audit infrastructure default)

## 7. Forward-Pins

**Pre-execution forward-pins (Stage 2 entry):**

1. **Stage 2 plan trigger phrase:** ***"Begin MobSF R2 implementation plan landing"***
2. **Design doc canonical authority:** This commit's hash for Stage 2 plan reference

**Post-Stage-3 forward-pins:**

3. **MobSF R2 migration-close task** — ***"Begin MobSF R2 migration-close task"*** (Q7-refined; `r2.go` deletion + `UploadRef` emission removal when `SignedFetchURL` adoption empirically stable)
4. **Framework-level Fetch-via-URL pattern** — ***"Begin framework-level Fetch-via-URL pattern task"*** (Y1 forward-pin; 2nd R2-consumer trigger; Task 7.4 future-work language)
5. **v1.1+ URL expiry enhancements** — per-scan-deadline-derived (Q2 (c)) + refresh-on-expiry (Q2 (d)) at scale-up motivation

## 8. Cross-References

**Engine commits:**

- `ad7cc94` (V10 Stage 3 C2; latest engine state; `mobsf.go` infrastructure consumed)
- `b48fef8` (ADR-015 Stage 3 C3; SQLMap cookie-injection precedent for fetcher-pattern reference)
- `d31b831` (Task 7.4 V13+V16 engine close)

**Docs commits:**

- `06c444c` (V10 P5.A; latest docs state)
- `0347a79` (V10 design doc; structural precedent + Q5 (a) re-deferral source)
- `7c4fe75` (V10 plan; structural precedent)
- `b344d0c` (ADR-015 design doc; Q5 (a) threat-model-coherence rationale source + structural precedent)
- `00dd2d1` (ADR-015 plan; structural precedent for Stage 2)
- `9a57865` (ADR-015 Stage 3 C1; ADR-013/ADR-014 addendum precedent shape for Stage 3 C1)
- `plans/2026-05-09-task-7.4-mobsf-consumer-design.md` (Task 7.4 Q6.4 forward-pin canonical authority)

**API commits:**

- `742faed` (ADR-015 Stage 3 C2; orchestrator credential-decrypt pattern + `SCAN_CREDENTIAL_DECRYPTED` audit precedent for Q5 (a) analog)

**SPEC sections:**

- §13 ADR-013 (sole-writer canonical; Q4 (b) addendum target)
- §13 ADR-014 (Redis Streams transit; Q4 (b) addendum target)
- §13 ADR-015 (credential-delegation; architectural analog)
- §13 ADR-008 (MobSF service-shape; consumed unchanged)

**Source authorities (Stage 3 sub-step targets):**

- shieldscan-api `src/app/services/r2.py` (Q8 (a) extension target)
- shieldscan-api `src/app/services/audit.py` (Q5 (a) enum addition target)
- shieldscan-api `src/app/services/orchestrator.py` (Q6 (a) integration target)
- shieldscan-engine `internal/events/events.go` (Q1 (β) field addition target)
- shieldscan-engine `internal/tools/docker/service/mobsf/mobsf.go` (Q3 (a) refactor target)
- shieldscan-engine `internal/tools/docker/service/mobsf/http_fetcher.go` (NEW; Q3 (a) `httpFetcher` implementation)
- shieldscan-engine `internal/tools/docker/service/mobsf/r2.go` (Q7-refined retention target)

**DRIFT-LOG:**

- shieldscan-engine `DRIFT-LOG.md` (Stage 3 Commit 3 inline entry per Y-DRIFT-LOG-PLACEMENT (a) precedent)
