# MobSF R2 Pre-Signed URL: Implementation Plan

**Status:** Ready for Stage 3 cross-repo implementation. Pre-implementation grounded against R2 design doc `b25e9ba` Y/Q-chain + pre-verification V-DA-V-DL. Plan structure grounded against verified engine `r2.go` + `mobsf.go` + orchestrator.py + r2.py + audit.py code shapes.

**Date:** 2026-05-23.

**Authority:** R2 design doc canonical authority (shieldscan-docs commit `b25e9ba`; Y1+Y2+Q1-Q8 + Q7-refined architectural-decision authority); pre-verification surface report (this session; V-DA-V-DL grounding); ADR-015 implementation plan structural precedent (commit `00dd2d1`; 278 LoC); V10 implementation plan structural precedent (commit `7c4fe75`; 304 LoC); Task 7.6 implementation plan structural precedent (commit `40606c5`; 269 LoC); 49+ cumulative session-tail framing-drift discipline.

**Related:** Stage 3 cross-repo implementation trigger phrase: ***"Resume MobSF R2 — Stage 3 cross-repo implementation"*** (after this plan lands).

---

## 1. Authority + Scope Lock

**Y1 (MobSF-only sole consumer per V-DL):** R2 task scope is engine-side MobSF consumer refactor + orchestrator pre-signed URL generation + ADR-013/ADR-014 addendums. Framework-level Fetch-via-URL pattern NOT in scope (1-instance threshold; forward-pinned to 2nd R2-consumer trigger).

**In scope (Stage 3 sub-steps):**

1. shieldscan-docs `SPECIFICATION.md` §13 ADR-013 + ADR-014 addendums landing (Stage 3 Commit 1)
2. shieldscan-api `src/app/services/r2.py` `generate_presigned_get_url()` helper extension (Stage 3 Commit 2)
3. shieldscan-api `src/app/services/audit.py` `SCAN_PRESIGNED_URL_GENERATED` ScanAction enum (Stage 3 Commit 2)
4. shieldscan-api `src/app/services/orchestrator.py` `dispatch()` mobile_config integration + audit emission (Stage 3 Commit 2)
5. shieldscan-api `tests/services/test_orchestrator.py` positive-path + audit-row tests (Stage 3 Commit 2)
6. shieldscan-engine `internal/events/events.go` `JobMobileConfig.SignedFetchURL` field (Stage 3 Commit 3)
7. shieldscan-engine `internal/tools/docker/service/mobsf/http_fetcher.go` (NEW; plain HTTP GET fetcher) (Stage 3 Commit 3)
8. shieldscan-engine `internal/tools/docker/service/mobsf/mobsf.go` `NewBuildScan` refactor (SignedFetchURL preference + UploadRef fallback per Q3 (a)) (Stage 3 Commit 3)
9. shieldscan-engine `internal/tools/docker/service/mobsf/mobsf_test.go` SignedFetchURL + UploadRef fallback tests (Stage 3 Commit 3)
10. shieldscan-engine `DRIFT-LOG.md` R2 entry inline (Stage 3 Commit 3)

**Out of scope (forward-pinned per design doc §6):** `r2.go` deletion (Q7-refined; *"Begin MobSF R2 migration-close task"*); framework-level Fetch-via-URL pattern (Y1 forward-pin); per-scan-deadline-derived URL expiry (Q2 (c) v1.1+); URL refresh-on-expiry (Q2 (d) scale-up); new ADR creation (Q4 (a) rejected); new boto3-alternative SDK on api (Q8 (a) confirmed); hard cutover deployment (Q3 (b) rejected); non-MobSF consumer integration (V-DL grounds); migration timeline definition; audit log retention.

**Brainstorming chain Q-locks recap:** Y1 (MobSF-only) + Y2 (direct Q-chain) + Q1 (β sibling field) + Q2 (a 600s expiry) + Q3 (a SignedFetchURL prefer + UploadRef fallback) + Q4 (b ADR-013/ADR-014 addendums) + Q5 (a SCAN_PRESIGNED_URL_GENERATED audit) + Q6 (a 3-commit trio) + Q7-refined (retain `r2.go` this task; deletion forward-pinned) + Q8 (a `r2.py` helper extension) per design doc `b25e9ba` §1 + §3.

## 2. Pre-Implementation State

### 2.1 Engine Infrastructure Ready Matrix (V-DE + V-DF + V-DJ Pre-Verification Findings)

**Engine side ready (V-DE + V-DF pre-built):**

- `internal/tools/docker/service/mobsf/r2.go` — `R2Config{Endpoint, AccessKeyID, SecretAccessKey, Bucket}` + `r2Fetcher` interface + `s3R2Fetcher` production impl with AWS SDK v2; PRESERVED in this task per Q7-refined (deletion forward-pinned)
- `internal/tools/docker/service/mobsf/mobsf.go` — `Config{APIKey, R2, FetcherOverride}`; `NewBuildScan` closure with `fetcher.Fetch(ctx, mt.UploadRef)` → `uploadFile` flow (V-DF empirical)
- `internal/tools/runner.go` — `Target.MobileUploadRef` field (verify extension needed for `SignedFetchURL` field at execution)
- `internal/events/events.go` — `JobMobileConfig` struct `{UploadRef, Platform, AnalysisType}`; `SignedFetchURL` field addition target (V-DG)
- `go.mod` `aws-sdk-go-v2 v1.32.2` + `service/s3 v1.66.0` PRESERVED (per Q7-refined `r2.go` retention)

**Engine side pending (Stage 3 Commit 3 scope):**

- `internal/events/events.go`: `JobMobileConfig.SignedFetchURL` field extension
- `internal/tools/runner.go`: `Target.SignedFetchURL` field extension (verify at execution per V-DG plumbing)
- `internal/tools/docker/service/mobsf/http_fetcher.go`: NEW plain HTTP GET fetcher implementation
- `internal/tools/docker/service/mobsf/mobsf.go`: `NewBuildScan` refactor — branch on `target.SignedFetchURL` presence per Q3 (a); `httpFetcher` when present; `r2Fetcher` fallback otherwise
- `internal/tools/docker/service/mobsf/mobsf_test.go`: SignedFetchURL preference test + UploadRef fallback test
- `DRIFT-LOG.md`: R2 entry inline per Y-DRIFT-LOG-PLACEMENT (a) precedent from V10 + ADR-015

**API side ready (V-DK + V-DH pre-built):**

- `src/app/services/r2.py` — `get_r2_client()` returning boto3 S3 client (`https://<account_id>.r2.cloudflarestorage.com`; `signature_version=s3v4`); `put_object` exposed
- `src/app/services/audit.py` — `ScanAction` enum + `audit()` helper; `SCAN_CREDENTIAL_DECRYPTED` enum precedent from ADR-015 `742faed`
- `src/app/services/orchestrator.py` — `dispatch()` with ADR-015 credential-decrypt block (line 220-280 post-`742faed`) + mobile_config block (line 290-303); `UploadRef` emission as `"r2://{upload.r2_key}"`
- `boto3 ^1.35.0` in `pyproject.toml`; `generate_presigned_url` built-in method available

**API side pending (Stage 3 Commit 2 scope):**

- `src/app/services/r2.py`: `generate_presigned_get_url(r2_key, expiry=600)` async helper extension
- `src/app/services/audit.py`: `SCAN_PRESIGNED_URL_GENERATED = "scan.presigned_url_generated"` ScanAction enum value
- `src/app/services/orchestrator.py`: `dispatch()` mobile_config block extension — call `generate_presigned_get_url` + emit SignedFetchURL alongside UploadRef + `audit()` emission
- `tests/services/test_orchestrator.py`: Positive-path test (SignedFetchURL emission) + audit-row test (SCAN_PRESIGNED_URL_GENERATED)

**Docs side pending (Stage 3 Commit 1 scope):**

- `SPECIFICATION.md` §13 ADR-013 Addendum: Payload Contract Pre-Signed URL Extension
- `SPECIFICATION.md` §13 ADR-014 Addendum: Pre-Signed URL Transit Posture

### 2.2 Architectural Analog to ADR-015 (V-DH grounded)

R2 task is structurally identical to ADR-015 enablement at orchestrator-as-sole-writer-of-time-bounded-access-tokens pattern. Symmetric implementation surface:

- Credential transit (ADR-015): orchestrator decrypts ProjectCredential → emits `{type, data}` in `JobDispatch.Auth` → engine consumes
- URL delegation (this task): orchestrator generates pre-signed URL → emits SignedFetchURL in JobMobileConfig → engine consumes plain-HTTP
- Audit pattern: `SCAN_CREDENTIAL_DECRYPTED` (ADR-015) ↔ `SCAN_PRESIGNED_URL_GENERATED` (this task)
- ADR addendum pattern: ADR-013 payload-contract + ADR-014 transit-posture addendums for both
- Cross-repo trio pattern: docs (ADR addendums) → api (orchestrator + audit + tests) → engine (consumer refactor + tests)

Plan structure inherits ADR-015 implementation plan `00dd2d1` shape directly.

## 3. Architectural Decisions (Plan-Level Locks)

Brainstorming chain Y1+Y2+Q1-Q8 + Q7-refined architectural decisions locked at design doc `b25e9ba`. Plan-level refinements captured below.

### 3.1 Q1 (β) JobMobileConfig SignedFetchURL Field

**Implementation surface:** `internal/events/events.go`:

```go
type JobMobileConfig struct {
    UploadRef       string `json:"upload_ref"`
    SignedFetchURL  string `json:"signed_fetch_url,omitempty"` // NEW; Q1 β
    Platform        string `json:"platform"`
    AnalysisType    string `json:"analysis_type"`
}
```

Tag `omitempty` preserves backward-compat (old orchestrator emitting payload without field; field absent in JSON; engine consumes UploadRef per Q3 (a) fallback). ~3-5 LoC.

### 3.2 Q2 (a) Fixed 600s URL Expiry — Implementation Site

**Orchestrator emission shape:** `dispatch()` mobile_config block extension:

```python
from app.services.r2 import generate_presigned_get_url
signed_url = await generate_presigned_get_url(r2_key=upload.r2_key, expiry=600)
mobile_config = {
    "upload_ref": f"r2://{upload.r2_key}",
    "signed_fetch_url": signed_url,
    "platform": ...,
    "analysis_type": ...,
}
await audit(
    session=self.db,
    organization_id=...,
    actor_id=actor_id,
    action=ScanAction.SCAN_PRESIGNED_URL_GENERATED,
    resource_type="scan",
    resource_id=str(scan.id),
    ip_address=...,
    user_agent=...,
    details={"r2_key": upload.r2_key, "expiry_seconds": 600},
)
```

~15-25 LoC orchestrator delta.

### 3.3 Q3 (a) Engine Prefer-SignedFetchURL + UploadRef Fallback

**Implementation surface:** `internal/tools/docker/service/mobsf/mobsf.go` `NewBuildScan` closure refactor:

```go
var fetcher r2Fetcher
if target.SignedFetchURL != "" {
    fetcher = newHTTPFetcher(target.SignedFetchURL) // NEW; Q3 a primary path
} else {
    fetcher = config.FetcherOverride // r2Fetcher fallback per Q3 a
    if fetcher == nil {
        fetcher = newS3R2Fetcher(config.R2) // existing path
    }
}
localPath, cleanup, err := fetcher.Fetch(ctx, target.MobileUploadRef)
```

~30-50 LoC delta.

**httpFetcher implementation at `http_fetcher.go` (NEW):**

```go
type httpFetcher struct { url string }
func newHTTPFetcher(url string) *httpFetcher { return &httpFetcher{url: url} }
func (f *httpFetcher) Fetch(ctx context.Context, _ string) (string, func(), error) {
    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, f.url, nil)
    resp, err := http.DefaultClient.Do(req)
    // ... temp-file staging + cleanup func; mirrors s3R2Fetcher staging pattern
}
```

~30-50 LoC.

**Plan-level Y-decision Y-HTTP-FETCHER-LOCATION:** (a) new file `http_fetcher.go` OR (b) method on existing struct. Default (a) per separation-of-concerns + `r2.go` parallel; resolve at Stage 3 Commit 3 execution.

### 3.4 Q4 (b) ADR-013 + ADR-014 Addendums (Stage 3 Commit 1 Scope)

**ADR-013 Addendum: Payload Contract Pre-Signed URL Extension**
Orchestrator emits `{upload_ref, signed_fetch_url}` both populated during migration window. Sole-writer responsibility per ADR-013 canonical preserved. Engine consumes SignedFetchURL when present; UploadRef fallback per Q3 (a). Migration close (forward-pinned to *"Begin MobSF R2 migration-close task"*) removes UploadRef emission. ~5-10 LoC at `SPECIFICATION.md` §13 after existing ADR-013 + ADR-015 addendums.

**ADR-014 Addendum: Pre-Signed URL Transit Posture**
Signed URLs at-rest in Redis JobDispatch payloads for queue-residence duration (typically seconds-to-minutes per BullMQ semantics). Mitigations: 600s URL expiry per Q2 (a) + Redis authenticated access per ADR-014 mandatory v1 mitigations + TLS-in-transit. Analog to ADR-015 credential transit posture. ~5-10 LoC at `SPECIFICATION.md` §13 after existing ADR-014 addendums.

**Stage 3 Commit 1 total docs delta:** ~15-25 LoC.

### 3.5 Q5 (a) SCAN_PRESIGNED_URL_GENERATED Audit

**Implementation:** `ScanAction` enum addition at `src/app/services/audit.py`:

```python
SCAN_PRESIGNED_URL_GENERATED = "scan.presigned_url_generated"
```

Audit emission inline at orchestrator `dispatch()` per §3.2 above. ~5-10 LoC across enum + emission.

**Plan-level Y-decision Y-PRESIGN-METHOD:** boto3 `generate_presigned_url` is synchronous; orchestrator `dispatch()` is async. Options: (a) `asyncio.to_thread()` wrap + boto3 call; (b) aiobotocore-based async-native (requires new dep). Default (a) per zero-new-dependency. ~3-5 LoC overhead at `r2.py` helper.

### 3.6 Q6 (a) 3-Commit Cross-Repo Sequencing

**Commit 1 (docs FIRST; ~15-25 LoC):** SPEC §13 ADR-013 + ADR-014 addendums (mirrors ADR-015 `9a57865` multi-addendum shape).

**Commit 2 (api SECOND; ~70-100 LoC):** `r2.py` `generate_presigned_get_url` helper + `audit.py` `ScanAction` enum + `orchestrator.py` `dispatch()` integration + audit emission + `test_orchestrator.py` positive-path + audit-row tests.

**Commit 3 (engine THIRD; ~115-188 LoC):** `events.go` SignedFetchURL field + `runner.go` `Target.SignedFetchURL` (verify at execution) + `http_fetcher.go` (NEW) + `mobsf.go` `NewBuildScan` refactor + `mobsf_test.go` SignedFetchURL preference + UploadRef fallback tests + `DRIFT-LOG.md` inline R2 entry.

**Cross-reference shape:** Mirrors ADR-015 Stage 3 trio (`9a57865` + `742faed` + `b48fef8`) + V10 Stage 3 trio (`d4f6ca7` + `ad7cc94` + `06c444c`) precedent.

### 3.7 Q7-Refined r2.go Retention + Deletion Forward-Pinned

**Decision:** `r2.go` retained in this task to support Q3 (a) UploadRef fallback during migration window. Deletion forward-pinned to ***"Begin MobSF R2 migration-close task"*** when SignedFetchURL adoption empirically stable.

**Migration-stability signal:** Per design doc §3.7 — production metric *"≥30 days of zero UploadRef fallback consumption"* OR explicit migration-close decision.

### 3.8 Q8 (a) r2.py Helper Extension

**Implementation:** `src/app/services/r2.py` extension:

```python
async def generate_presigned_get_url(r2_key: str, expiry: int = 600) -> str:
    client = get_r2_client()
    loop = asyncio.get_event_loop()
    url = await loop.run_in_executor(
        None,
        lambda: client.generate_presigned_url(
            "get_object",
            Params={"Bucket": settings.R2_BUCKET, "Key": r2_key},
            ExpiresIn=expiry,
        ),
    )
    return url
```

~10-15 LoC. Y-PRESIGN-METHOD (a) lock. Zero new dependencies (boto3 built-in).

## 4. Stage 3 Sub-Step Breakdown

### Stage 3 Commit 1 — Docs (SPEC ADR-013/ADR-014 Addendums) (~20-30min)

- **C1.1** Locate ADR-013 section in `SPECIFICATION.md`; identify ADR-013 + ADR-015-enablement addendums area for new R2 addendum insertion. `grep` + `sed` location.
- **C1.2** Append ADR-013 Addendum: Payload Contract Pre-Signed URL Extension after existing addendums. ~5-10 LoC.
- **C1.3** Locate ADR-014 section; identify addendum insertion area.
- **C1.4** Append ADR-014 Addendum: Pre-Signed URL Transit Posture after existing addendums. ~5-10 LoC.
- **C1.5** Pre-commit verification: grep ADR-013 + ADR-014 addendum titles (verify landed); `wc -l SPECIFICATION.md` (delta within bound).
- **C1.6** Single atomic commit. Cross-references future api+engine hashes via placeholders.

**Total Stage 3 Commit 1 LoC delta:** ~15-25 LoC.

### Stage 3 Commit 2 — API (r2.py + audit + orchestrator + tests) (~45-90min)

- **C2.1** `r2.py` extension: `generate_presigned_get_url` async helper per §3.8 Y-PRESIGN-METHOD (a). ~10-15 LoC.
- **C2.2** `audit.py`: `SCAN_PRESIGNED_URL_GENERATED` ScanAction enum value addition. ~1-2 LoC.
- **C2.3** `orchestrator.py` `dispatch()` mobile_config block extension: call `generate_presigned_get_url` + emit SignedFetchURL + audit emission. ~15-25 LoC.
- **C2.4** `test_orchestrator.py` positive-path test: verify SignedFetchURL emitted alongside UploadRef for mobile scan with R2 upload reference. ~25-40 LoC.
- **C2.5** `test_orchestrator.py` audit-row test: verify `SCAN_PRESIGNED_URL_GENERATED` audit row written. ~20-30 LoC.
- **C2.6** Pre-commit verification: `pytest tests/services/test_orchestrator.py` + full suite green.
- **C2.7** Single atomic commit. Cross-references docs Commit 1 hash + engine placeholder.

**Total Stage 3 Commit 2 LoC delta:** ~70-100 LoC.

### Stage 3 Commit 3 — Engine (events.go + http_fetcher + mobsf + tests + DRIFT-LOG) (~60-90min)

- **C3.1** `events.go` `JobMobileConfig` SignedFetchURL field extension per §3.1. ~3-5 LoC.
- **C3.2** `runner.go` `Target.SignedFetchURL` field extension (verify at execution; may already exist OR need addition for plumbing). ~2-3 LoC if needed.
- **C3.3** `http_fetcher.go` NEW per Y-HTTP-FETCHER-LOCATION (a): `newHTTPFetcher` + `Fetch` implementation; mirrors r2Fetcher temp-file staging pattern. ~30-50 LoC.
- **C3.4** `mobsf.go` `NewBuildScan` refactor per §3.3 Q3 (a): branch on `target.SignedFetchURL` presence; httpFetcher primary path; r2Fetcher fallback. ~30-50 LoC.
- **C3.5** `mobsf_test.go`: SignedFetchURL preference test + UploadRef fallback test + edge cases. ~40-60 LoC.
- **C3.6** Y-INTEGRATION-TEST-COVERAGE resolution: default (a) mock HTTP server in `mobsf_test.go` OR `integration_test.go` addition (per V10 `integration_test.go` precedent; verify at execution if existing `integration_test.go` from `ad7cc94` V10 extension OR new addition).
- **C3.7** `DRIFT-LOG.md` R2 entry inline per Y-DRIFT-LOG-PLACEMENT (a). ~10-20 LoC.
- **C3.8** Pre-commit verification: `go vet ./...` + `go test -race ./internal/tools/docker/service/mobsf/` + `go test -race ./...` (25 packages baseline preserved) + integration test if applicable.
- **C3.9** Single atomic commit. Cross-references docs Commit 1 + api Commit 2 hashes concretely.

**Total Stage 3 Commit 3 LoC delta:** ~115-188 LoC.

### Stage 3 Aggregate LoC Forecast

Total across 3 commits: **~200-313 LoC** (docs ~15-25 + api ~70-100 + engine ~115-188). Comparable to ADR-015 Stage 3 (~205-340); smaller than V10 Stage 3 (~500-770).

## 5. D-Deviation Tracking Framework

Per Task 7.6 + ADR-015 + V10 D-PLAN tracking precedent.

**Pre-execution drifts catalogued:** None (pre-verification surface report this session caught zero drifts; calibration accurate; cumulative count stays at 49).

**Expected Stage 3 D-deviation count:** LOW. Pre-verification grounded all V-items + canonical authority text; architectural-decision-only task with strong analog (ADR-015) as template; bounded LoC scope. Expected ~2-4 drifts at execution per typical pattern (compare ADR-015 Stage 3 surfaced 5 drifts; V10 Stage 3 surfaced 3 pre-execution static + 0 post-execution; R2 should land closer to V10 quality given strong analog template).

**Plan-level Y-decisions to resolve at execution:**

- **Y-PRESIGN-METHOD**: boto3 sync-wrapped `asyncio.to_thread` (a) vs aiobotocore async-native (b); default (a) per zero-new-dep; resolve at Stage 3 Commit 2 C2.1
- **Y-HTTP-FETCHER-LOCATION**: new file `http_fetcher.go` (a) vs method on existing (b); default (a) per separation; resolve at Stage 3 Commit 3 C3.3
- **Y-INTEGRATION-TEST-COVERAGE**: mock HTTP server (a) vs real R2 (b) vs unit-only (c); default (a) per V10 precedent; resolve at Stage 3 Commit 3 C3.6

## 6. Out of Scope (per design doc §6 + plan-level refinements)

1. `r2.go` deletion + UploadRef emission removal (Q7-refined; *"Begin MobSF R2 migration-close task"*)
2. Framework-level Fetch-via-URL pattern (Y1 forward-pin; 2nd R2-consumer trigger)
3. Per-scan-deadline-derived URL expiry (Q2 (c) forward-pin to v1.1+)
4. URL refresh-on-expiry pattern (Q2 (d) forward-pin at scale)
5. New ADR creation (Q4 (a) rejected)
6. boto3-alternative SDK on api (Q8 (a) confirmed; zero new dep)
7. Hard cutover deployment (Q3 (b) rejected)
8. Non-MobSF consumer integration (V-DL grounds)
9. Migration timeline / production-stability signal definition
10. Audit log retention + analytics
11. shieldscan-api credential infrastructure modifications outside `r2.py` + `audit.py` + `orchestrator.py` + `test_orchestrator.py`
12. shieldscan-engine modifications outside `events.go` + `runner.go` + `service/mobsf/` files

## 7. Forward-Pins

**Pre-execution forward-pins (Stage 3 entry):**

1. **Stage 3 trigger phrase:** ***"Resume MobSF R2 — Stage 3 cross-repo implementation"***
2. **Y-PRESIGN-METHOD decision context preserved:** boto3 sync-wrapped vs aiobotocore async-native (resolve at C2.1 execution; default a)
3. **Y-HTTP-FETCHER-LOCATION decision context:** new file vs method on existing (resolve at C3.3 execution; default a)
4. **Y-INTEGRATION-TEST-COVERAGE decision context:** mock HTTP server vs real R2 vs unit-only (resolve at C3.6 execution; default a)
5. **Design doc canonical authority:** `b25e9ba` §3 + §4 verbatim drafts

**Post-Stage-3 forward-pins:**

6. **MobSF R2 migration-close task** — ***"Begin MobSF R2 migration-close task"*** (Q7-refined; `r2.go` deletion + UploadRef emission removal when SignedFetchURL adoption empirically stable)
7. **Framework-level Fetch-via-URL pattern** — ***"Begin framework-level Fetch-via-URL pattern task"*** (Y1 forward-pin; 2nd R2-consumer trigger)
8. **v1.1+ URL expiry enhancements** — per-scan-deadline-derived (Q2 (c)) + refresh-on-expiry (Q2 (d)) at scale motivation

## 8. Cross-References

**Engine commits:**

- `ad7cc94` (V10 Stage 3 C2; latest engine state; `mobsf.go` infrastructure consumed)
- `b48fef8` (ADR-015 Stage 3 C3; cookie-injection precedent for fetcher-pattern reference)

**Docs commits:**

- `b25e9ba` (R2 Stage 1 design doc; this plan's canonical authority)
- `06c444c` (V10 P5.A; latest docs state)
- `9a57865` (ADR-015 Stage 3 C1; ADR-013/ADR-014 addendum precedent shape)
- `0347a79` (V10 design doc; structural precedent)
- `7c4fe75` (V10 plan; structural precedent for this plan)
- `00dd2d1` (ADR-015 plan; structural precedent for this plan)
- `40606c5` (Task 7.6 plan; structural precedent)

**API commits:**

- `742faed` (ADR-015 Stage 3 C2; orchestrator `dispatch()` + audit + ScanAction enum precedent for Q5 (a) analog)

**SPEC sections:**

- §13 ADR-013 (sole-writer canonical; Q4 (b) addendum target)
- §13 ADR-014 (Redis Streams transit; Q4 (b) addendum target)
- §13 ADR-015 (credential-delegation; architectural analog)

**Source authorities (Stage 3 sub-step targets):**

- shieldscan-api `src/app/services/r2.py` (C2.1 modification target)
- shieldscan-api `src/app/services/audit.py` (C2.2 modification target)
- shieldscan-api `src/app/services/orchestrator.py` (C2.3 modification target)
- shieldscan-api `tests/services/test_orchestrator.py` (C2.4 + C2.5 modification target)
- shieldscan-engine `internal/events/events.go` (C3.1 modification target)
- shieldscan-engine `internal/tools/runner.go` (C3.2 modification target; verify at execution)
- shieldscan-engine `internal/tools/docker/service/mobsf/http_fetcher.go` (C3.3 NEW file target)
- shieldscan-engine `internal/tools/docker/service/mobsf/mobsf.go` (C3.4 modification target)
- shieldscan-engine `internal/tools/docker/service/mobsf/mobsf_test.go` (C3.5 modification target)
- shieldscan-engine `DRIFT-LOG.md` (C3.7 modification target)

**DRIFT-LOG:**

- shieldscan-engine `DRIFT-LOG.md` (C3.7 inline entry per Y-DRIFT-LOG-PLACEMENT (a) precedent from V10 `ad7cc94` + ADR-015 `b48fef8`)
