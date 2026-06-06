# Source-Ingestion Fix: Implementation Plan

**Status:** Ready for Stage 3 cross-repo implementation. Pre-implementation grounded against source-ingestion fix design doc `90fc933` Y/Q-chain + pre-verification V-IA-V-II + Phase 0 v2 P0v2.A-D empirical confirmation. Plan structure grounded against verified TOOL-ARCH §3.2 design-intent + trivy-fs runner shape + `Project.source_repo_url` field state + container-security envelope.

**Date:** 2026-05-29.

**Authority:** source-ingestion fix design doc canonical authority (shieldscan-docs commit `90fc933`; Y1+Y2+Y3+12 Q-chain locks architectural-decision authority); pre-verification surface report (this session; V-IA-V-II grounding); Phase 0 v2 empirical confirmation (this session; P0v2.A-D ZERO pivot triggers); revocation implementation plan structural precedent (commit `fdad021`; 338 LoC); R2 implementation plan structural precedent (commit `721f788`; 357 LoC); V10 implementation plan structural precedent (commit `7c4fe75`; 304 LoC); ADR-015 implementation plan structural precedent (commit `00dd2d1`; 278 LoC); Task 7.6 implementation plan structural precedent (commit `40606c5`; 269 LoC); 54+ cumulative session-tail framing-drift discipline (Drift #54 catalogued; clean Stage 1 design doc entry); milestone-completion-constraint locked this session.

**Related:** Stage 3 cross-repo implementation trigger phrase: ***"Resume source-ingestion fix — Stage 3 cross-repo implementation"*** (after this plan lands).

---

## 1. Authority + Scope Lock

**Y1+Y2+Y3 (locked at design doc `90fc933` §1):** Y1 (α.1) host-side `os/exec` git clone per TOOL-ARCH §3.2 design intent + P0v2 empirical confirmation; Y2 Phase 0 v2 (this session; bounded empirical confirmation; ZERO pivot triggers); Y3 direct Q-chain (post-Phase-0-v2-grounded).

**In scope (Stage 3 sub-steps; Q-COMMITS 3-commit shape):**

1. shieldscan-docs TOOL-ARCHITECTURE.md §3.2 addendum: Source-Acquisition Implementation Lock (Stage 3 Commit 1; ~25-50 LoC)
2. shieldscan-api `src/app/schemas/projects.py` HTTPS-scheme validator on `source_repo_url` (Stage 3 Commit 2; ~5-10 LoC)
3. shieldscan-api `src/app/services/orchestrator.py` `_build_job_payload` `source_repo_url` threading (Stage 3 Commit 2; ~10-20 LoC)
4. shieldscan-api `src/app/schemas/scans.py` ScanCreateRequest validator (Stage 3 Commit 2; ~5-15 LoC)
5. shieldscan-api tests: validator + orchestrator threading + failure-mode tests (Stage 3 Commit 2; ~40-60 LoC)
6. shieldscan-engine `internal/events/events.go` `JobTarget.SourceRepoURL` field (Stage 3 Commit 3; ~3-5 LoC)
7. shieldscan-engine `internal/source/` NEW package (Y-PACKAGE-LOCATION a default): `cloneRepo` + per-scan tempdir manager (Stage 3 Commit 3; ~50-100 LoC)
8. shieldscan-engine `internal/tools/runner.go` `Target.SourceRepoURL` field (Stage 3 Commit 3; ~2-3 LoC)
9. shieldscan-engine `internal/worker/processor.go` `jobDispatchToTarget` threading (Stage 3 Commit 3; ~3-5 LoC)
10. shieldscan-engine `internal/tools/docker/trivy/trivy.go`/`scan.go` refactor with cloneRepo + defer cleanup (Stage 3 Commit 3; ~20-40 LoC)
11. shieldscan-engine tests: cloneRepo unit test + trivy-fs integration test extension + DRIFT-LOG inline (Stage 3 Commit 3; ~30-50 LoC)

**Out of scope (forward-pinned per design doc §6):** SSH-key git clone auth; private-token git clone auth; `go-git` library; SOURCE_ACQUISITION_* event types; recon-time source-clone (Task 8.1 territory); R2-upload-source-tarball variant; LRU staging-cache; `Project.source_repo_url` backfill; engine source-tool parser changes; UI surfaces; SCA `ScanType.SCA` enablement (Task 2 of 2 per SCA decomposition); composite-ScanType partial-success-after-source-fail.

**Brainstorming chain Q-locks recap (12):** Q-CLONE-LIB (a) `os/exec`; Q-STAGING-PATH (a) `$TRIVY_SCAN_BASE_PATH/<scan-id>/`; Q-DEPTH `--depth=1`; Q-CLEANUP per-scan `defer os.RemoveAll`; Q-AUTH public-HTTPS v1; Q-VALIDATOR HTTPS-scheme-only; Q-WIRE engine clones (wire carries URL); Q-PARSER NO changes; Q-FAILURE-MODE hard-fail with `SCAN_FAILED`; Q-EVENTS standard lifecycle; Q-RECON-TIMING lazy per-job at worker; Q-ADR TOOL-ARCH addendum; Q-COMMITS 3-commit cross-repo per design doc `90fc933` §1 + §3.

## 2. Pre-Implementation State

### 2.1 Engine Infrastructure Ready Matrix (V-IB + V-IC + V-IH + Phase 0 v2)

**Engine side ready:**

- `internal/tools/docker/trivy/trivy.go` + `scan.go`: `NewFsRunner` + `buildArgsFs` short-circuits on empty `SourcePath` (preserves existing test invariant)
- `internal/tools/runner.go`: `Target` struct with `TargetType` + existing `SourcePath` field for "source" TargetType
- `internal/events/events.go`: `JobTarget` struct with URL + TargetType + DomainVerified (no source-related field yet)
- `internal/worker/processor.go`: `jobDispatchToTarget` maps `job.Target` → `tools.Target`; pattern established
- `internal/tools/recon/`: subfinder + httpx + recon.go (DNS+HTTP only; no source-clone surface)
- `go.mod`: no `go-git` import; `os/exec` viable per P0v2.A git binary v2.43.0 at `/usr/bin/git`
- Container-security envelope (V-IH): default bridge network outbound; ReadOnly mount enforced (P0v2.D)
- Phase 0 v2 artifacts: `/tmp/sif-p0v2-1780787784/NodeGoat/` (3.3M) + `trivy-fs-p0v2.json` (75-finding fixture)

**Engine side pending (Stage 3 Commit 3 scope):**

- `internal/events/events.go`: `JobTarget.SourceRepoURL` field per Q-WIRE wire-carries-URL lock (~3-5 LoC)
- `internal/source/` NEW package per Y-PACKAGE-LOCATION (a default): `cloneRepo` + staging-dir manager (~50-100 LoC)
- `internal/tools/runner.go`: `Target.SourceRepoURL` field (verify if needed OR plumb via existing `SourcePath`; ~0-3 LoC)
- `internal/worker/processor.go`: `jobDispatchToTarget` threading `SourceRepoURL` (~3-5 LoC)
- `internal/tools/docker/trivy/`: cloneRepo invocation + `SourcePath` population + defer cleanup; preserve `buildArgsFs` guard (~20-40 LoC)
- `DRIFT-LOG.md`: source-ingestion fix LANDED entry inline

**API side ready (V-IE + V-IG + Phase 0 v2):**

- `src/app/models/projects.py`: `Project.source_repo_url = String(500), nullable=True` (orphaned column; never read by orchestrator per Drift #54)
- `src/app/schemas/projects.py`: ProjectCreateRequest + ProjectUpdateRequest with `source_repo_url: str | None, max_length=500`; **no validator** (V-IE freeform gap)
- `src/app/services/orchestrator.py`: `_build_job_payload` target block emits `{url, target_type, domain_verified}`; no `source_repo_url` threading (Drift #54 root)
- `src/app/schemas/scans.py`: ScanCreateRequest with no source-related field; FULL_WEB_SOURCE accepts same request as FULL_WEB (V-IG UX gap)
- `src/app/services/orchestrator.py` `SCAN_TYPE_TOOLS`: FULL_WEB_SOURCE + FULL_SPECTRUM dispatch `trivy-fs` (silently no-ops today per Drift #54)
- `src/app/services/orchestrator.py` `_target_type_for`: maps MOBILE → "mobile"; CONTAINER → "container"; API → "api"; default → "web"; no "source" mapping

**API side pending (Stage 3 Commit 2 scope):**

- `src/app/schemas/projects.py`: HTTPS-scheme validator on `source_repo_url` mirrors `_validate_https_url` for `target_url` (~5-10 LoC)
- `src/app/services/orchestrator.py` `_build_job_payload`: thread `source_repo_url` from `scan.project` → wire `target.source_repo_url` (conditional on `scan_type` ∈ source-requiring set) (~10-20 LoC)
- `src/app/schemas/scans.py`: ScanCreateRequest server-side validator — when `scan_type` ∈ {FULL_WEB_SOURCE, FULL_SPECTRUM} and `project.source_repo_url is None`, raise 422 (route-handler validation; ~5-15 LoC)
- `tests/schemas/test_projects.py`: HTTPS validator tests + freeform rejection tests
- `tests/services/test_orchestrator.py`: `source_repo_url` threading test + failure-mode test
- `tests/routes/test_scans.py`: FULL_WEB_SOURCE without `source_repo_url` 422 test (~40-60 LoC total tests)

**Docs side pending (Stage 3 Commit 1 scope):**

- TOOL-ARCHITECTURE.md §3.2 addendum: Source-Acquisition Implementation Lock — mechanism (α.1) + design-intent canonicalization + validator + hard-fail + Drift #54 root-cause repair (~25-50 LoC)

### 2.2 Architectural Analog: Source-Acquisition Pattern Origin

**No prior task analog for source-acquisition.** Unlike R2 (orchestrator-as-sole-writer + signed-URL analog of ADR-015) and revocation (audit-emission + Task 4.5 cancel-fanout dual analog), source-acquisition is a net-new architectural pattern at engine layer. Analog availability is LIMITED:

- Host-clone-then-ReadOnly-mount (architectural shape): no direct prior; closest is Trivy production warm-pool `$TRIVY_SCAN_BASE_PATH` → `/scan` mount (V-IC) — same mount semantics but never previously source-clone-fed
- JobTarget wire-field addition: R2 `SignedFetchURL` extension precedent (events.go addition pattern at `3ccf5b8`) — directly applicable
- `SCAN_FAILED` with structured error: existing error-emission pattern per Task 4.5 + Task 5.x — directly applicable

**Implication for D-deviation forecast:** likely 2-4 drifts at execution per dual-novel-pattern territory; lower than V10 first-iOS-section (~5 drifts) but higher than R2 ZERO-drift (which had unusually-strong analog template).

### 2.3 Phase 0 v2 Empirical Anchors

- git binary present (P0v2.A) + clone semantics empirically bounded (P0v2.B; 0.76s/3.3M/1.2M .git for NodeGoat depth=1)
- trivy fs scan against cloned tree produces 75 findings (P0v2.C; `parseTrivyJSON` contract unchanged per Q-PARSER NO-changes lock)
- Docker ReadOnly mount enforced (P0v2.D; touch denied empirically)
- Phase 0 v2 testbed preserved at `/tmp/sif-p0v2-1780787784/NodeGoat/` + `trivy-fs-p0v2.json`; Stage 3 integration test reuses fixture

## 3. Architectural Decisions (Plan-Level Locks)

Brainstorming chain Y1+Y2+Y3+12 Q-locks architectural decisions locked at design doc `90fc933`. Plan-level refinements captured below.

### 3.1 Y1 (α.1) Host-Side `os/exec` git clone — Implementation Site

**Implementation surface:** `internal/source/` NEW package (Y-PACKAGE-LOCATION a default; resolve at execution):

```go
package source

import ("context"; "fmt"; "net/url"; "os"; "os/exec"; "path/filepath")

func CloneRepo(ctx context.Context, gitURL string, stagingDir string) error {
    // Defense-in-depth: HTTPS-scheme guard (api validator is primary)
    u, err := url.Parse(gitURL)
    if err != nil || u.Scheme != "https" {
        return fmt.Errorf("source.CloneRepo: HTTPS required; got %q", gitURL)
    }
    if err := os.MkdirAll(filepath.Dir(stagingDir), 0o755); err != nil {
        return fmt.Errorf("source.CloneRepo: mkdir base: %w", err)
    }
    cmd := exec.CommandContext(ctx, "git", "clone", "--depth=1", gitURL, stagingDir)
    output, err := cmd.CombinedOutput()
    if err != nil {
        return fmt.Errorf("source.CloneRepo: %w (output: %s)", err, output)
    }
    return nil
}
```

~30-50 LoC at `internal/source/clone.go` + ~20-50 LoC for staging-dir manager (creation + per-scan-id path + cleanup). Total ~50-100 LoC.

**Plan-level Y-decision Y-PACKAGE-LOCATION:** (a) `internal/source/` standalone (default; separation-of-concerns + future-multi-tool-consumer); (b) `internal/tools/source/` subpackage; (c) `internal/tools/docker/trivy/source.go` inline. Default (a); resolve at Stage 3 Commit 3 C3.2 execution per existing package-organization convention.

### 3.2 Q-CLONE-LIB (a) `os/exec`

Implementation per §3.1. `exec.CommandContext` integrates with ADR-021 Rule 1 (ctx cancel → SIGKILLs subprocess naturally). No new Go dep.

### 3.3 Q-STAGING-PATH (a) `$TRIVY_SCAN_BASE_PATH/<scan-id>/`

**Staging dir manager:** `internal/source/staging.go`:

```go
type StagingManager struct { basePath string }

func NewStagingManager(basePath string) *StagingManager {
    return &StagingManager{basePath: basePath}
}

func (s *StagingManager) StagingDirForScan(scanID string) string {
    return filepath.Join(s.basePath, scanID)
}

func (s *StagingManager) Cleanup(scanID string) error {
    return os.RemoveAll(s.StagingDirForScan(scanID))
}
```

Container-relative path: trivy-fs sees `/scan/<scan-id>` per existing mount semantics. trivy_fs setup populates `target.SourcePath = "/scan/" + scanID`. ~20-40 LoC.

### 3.4 Q-DEPTH `--depth=1`

Locked at CloneRepo via `--depth=1` flag. Empirically validated (P0v2.B).

### 3.5 Q-CLEANUP per-scan `defer os.RemoveAll`

**trivy-fs integration:**

```go
// In trivy-fs Run-equivalent setup site
scanID := target.ScanID
stagingDir := stagingMgr.StagingDirForScan(scanID)
if err := source.CloneRepo(ctx, target.SourceRepoURL, stagingDir); err != nil {
    return nil, fmt.Errorf("trivy-fs: source acquisition failed: %w", err)
}
defer stagingMgr.Cleanup(scanID) // tolerates ReadOnly mount inside container
target.SourcePath = "/scan/" + scanID
// existing buildArgsFs invocation proceeds with populated SourcePath
```

~20-40 LoC at trivy-fs refactor site.

### 3.6 Q-AUTH public-HTTPS-v1 + Q-VALIDATOR HTTPS-scheme-only

**API validator (`src/app/schemas/projects.py`):** mirrors `_validate_https_url` for `target_url`. ~5-10 LoC.

```python
@field_validator("source_repo_url")
@classmethod
def _validate_source_repo_url(cls, v: str | None) -> str | None:
    if v is None:
        return None
    return _validate_https_url(v)
```

### 3.7 Q-WIRE engine-clones (wire carries URL) — Implementation Site

**Plan-level Y-decision Y-WIRE-FIELD-NAME:** (a) `JobTarget.SourceRepoURL` matches DB column; (b) `JobTarget.SourcePath` symmetric with engine `Target.SourcePath`; (c) both fields. Default (a) per DB-column-symmetry + explicit URL semantics; resolve at Stage 3 Commit 3 C3.1 execution.

Wire JSON: `{"url":..., "target_type":..., "domain_verified":..., "source_repo_url":"https://github.com/...", ...}`.

**Engine plumbing:** `JobTarget.SourceRepoURL string \`json:"source_repo_url,omitempty"\``; `Target.SourceRepoURL` field; `jobDispatchToTarget` mapping. ~8-13 LoC across 3 files.

**Orchestrator threading:** `_build_job_payload` conditional on `scan_type ∈ {FULL_WEB_SOURCE, FULL_SPECTRUM}`: emit `source_repo_url` alongside existing target fields. ~10-20 LoC.

### 3.8 Q-PARSER NO changes

Existing `parseTrivyJSON` consumer contract preserved unchanged per P0v2.C empirical confirmation. ZERO parser scope.

### 3.9 Q-FAILURE-MODE hard-fail + Q-EVENTS standard lifecycle — Implementation Site

**Plan-level Y-decision Y-CLONE-FAILURE-EVENT-SHAPE:** (a) `SCAN_FAILED` with `error.code = "SOURCE_ACQUISITION_FAILED"` + `error.details = {url, git_exit_code, stderr_tail}`; (b) generic `SCAN_FAILED` + clone-specific details only; (c) new event type. Default (a) per Q-EVENTS standard-lifecycle lock + structured-error precedent + forensic-detail preservation. ~10-15 LoC at trivy-fs error-emission site.

### 3.10 Q-RECON-TIMING lazy per-job at worker

Lazy clone happens at trivy-fs invocation (worker-side; per-job; just-before-scan). Orchestrator dispatches scan as today; engine clones at job-pickup. ~0 LoC for this lock (architectural property of §3.5 placement).

### 3.11 Q-ADR (c) TOOL-ARCHITECTURE.md addendum — Implementation Site (Stage 3 Commit 1)

**TOOL-ARCHITECTURE.md addendum content scope (~25-50 LoC):**

- Source-Acquisition Implementation Lock (2026-05-29): mechanism (α.1) host-side `os/exec` git clone `--depth=1` + per-scan tempdir under `$TRIVY_SCAN_BASE_PATH`
- HTTPS-scheme validator + hard-fail-on-clone-failure + `SCAN_FAILED` structured error
- Drift #54 root-cause repair: §3.2 design intent (`SourcePath string // for SAST: local git clone path`) canonicalized to implementation-lock
- Cross-reference to source-ingestion fix design doc `90fc933` + Phase 0 v2 empirical anchors

**Mirrors revocation ADR-015 Credential Lifecycle Extension addendum precedent shape (`291ded3`) but at TOOL-ARCH §3.2 location instead of SPEC §13 ADR-015.**

### 3.12 Q-COMMITS (a) 3-Commit Cross-Repo Sequencing

**Commit 1 (docs FIRST; ~25-50 LoC):** TOOL-ARCHITECTURE.md §3.2 addendum.

**Commit 2 (api SECOND; ~60-105 LoC):** validator + orchestrator threading + ScanCreateRequest validation + tests.

**Commit 3 (engine THIRD; ~108-203 LoC):** events.go field + runner.go field + `source/` package + processor.go threading + trivy-fs refactor + tests + DRIFT-LOG inline.

**Cross-reference shape:** Commit 1 references future api+engine hashes via placeholders; Commit 2 references docs Commit 1 hash + engine placeholder; Commit 3 references both docs + api hashes concretely.

### 3.13 Engine Source-Acquisition Test Strategy

**Unit test:** `internal/source/clone_test.go`: `CloneRepo` against httptest-served git OR mock subprocess (avoid network in CI). Pure unit coverage.

**Integration test:** `internal/tools/docker/trivy/integration_test.go` extension: end-to-end test with mock cloneRepo OR pre-staged tempdir + real trivy docker invocation against Phase 0 v2 fixture analog. Reuses Task 7.1 testdata pattern.

## 4. Stage 3 Sub-Step Breakdown

### Stage 3 Commit 1 — Docs (TOOL-ARCH §3.2 Addendum) (~15-25min)

**C1.1** Locate TOOL-ARCHITECTURE.md §3.2 section; identify existing content + insertion point. grep §3.2 + adjacent sections.

**C1.2** Append Source-Acquisition Implementation Lock addendum per §3.11. ~25-50 LoC.

**C1.3** Pre-commit verification: grep `"Source-Acquisition\|SourcePath\|local git clone path"` TOOL-ARCHITECTURE.md; wc -l TOOL-ARCHITECTURE.md (delta within bound).

**C1.4** Single atomic commit. Cross-references future api+engine hashes via placeholders.

**Total Stage 3 Commit 1 LoC delta:** ~25-50 LoC.

### Stage 3 Commit 2 — API (validator + orchestrator + ScanCreate validator + tests) (~45-90min)

**C2.1** `schemas/projects.py`: HTTPS-scheme validator on `source_repo_url` per §3.6. ~5-10 LoC.

**C2.2** `services/orchestrator.py` `_build_job_payload`: thread `source_repo_url` from `scan.project` conditional on `scan_type` per §3.7. ~10-20 LoC.

**C2.3** `schemas/scans.py` OR `routes/scans.py`: ScanCreateRequest validator — raise 422 when `scan_type` requires source + `project.source_repo_url is None` per §2.1 V-IG repair. ~5-15 LoC. (Note: server-side resolution at route handler likely cleanest since validator needs DB lookup of project.)

**C2.4** `tests/schemas/test_projects.py` OR `tests/routes/test_projects.py`: HTTPS validator tests (HTTPS accepts; http/git/ssh rejects). ~15-25 LoC.

**C2.5** `tests/services/test_orchestrator.py`: `source_repo_url` threading test (FULL_WEB_SOURCE with `source_repo_url` → payload contains it). ~10-20 LoC.

**C2.6** `tests/routes/test_scans.py`: FULL_WEB_SOURCE without `source_repo_url` → 422 test. ~10-20 LoC.

**C2.7** Pre-commit verification: pytest validator + orchestrator + routes tests green; pytest full suite green (no regressions per baseline).

**C2.8** Single atomic commit. Cross-references docs Commit 1 hash + engine placeholder.

**Total Stage 3 Commit 2 LoC delta:** ~60-105 LoC.

### Stage 3 Commit 3 — Engine (events.go + source/ package + runner.go + processor.go + trivy-fs + tests + DRIFT-LOG) (~60-120min)

**C3.1** `events.go` `JobTarget.SourceRepoURL` field per §3.7 Y-WIRE-FIELD-NAME (a) default. ~3-5 LoC.

**C3.2** `internal/source/` NEW package per §3.1 + §3.3 Y-PACKAGE-LOCATION (a) default: `clone.go` + `staging.go`. ~50-100 LoC.

**C3.3** `internal/tools/runner.go` `Target.SourceRepoURL` field. ~2-3 LoC.

**C3.4** `internal/worker/processor.go` `jobDispatchToTarget` mapping `job.Target.SourceRepoURL → tools.Target.SourceRepoURL`. ~3-5 LoC.

**C3.5** `internal/tools/docker/trivy/` refactor: cloneRepo invocation + SourcePath population + defer cleanup per §3.5; preserve `buildArgsFs` guard. Hard-fail with structured error per §3.9 Y-CLONE-FAILURE-EVENT-SHAPE (a) default. ~20-40 LoC.

**C3.6** `internal/source/clone_test.go`: unit test with mocked subprocess. ~15-25 LoC.

**C3.7** `internal/tools/docker/trivy/integration_test.go`: extension with mock cloneRepo OR pre-staged tempdir + Phase 0 v2 fixture analog. ~15-30 LoC.

**C3.8** Pre-commit verification: `go vet ./...` clean; `go test -race ./internal/source/` green; `go test -race ./internal/tools/docker/trivy/` green; `go test -race ./...` green (full suite baseline preserved).

**C3.9** `DRIFT-LOG.md`: source-ingestion fix LANDED entry inline per engine DRIFT-LOG convention. ~25-40 LoC.

**C3.10** Single atomic commit. Cross-references docs Commit 1 + api Commit 2 hashes concretely.

**Total Stage 3 Commit 3 LoC delta:** ~108-203 LoC.

### Stage 3 Aggregate LoC Forecast

Total across 3 commits: ~193-358 LoC (docs ~25-50 + api ~60-105 + engine ~108-203). Between revocation Stage 3 (~88-145) and R2 Stage 3 (~200-313).

## 5. D-Deviation Tracking Framework

Per Task 7.6 + ADR-015 + V10 + R2 + revocation D-PLAN tracking precedent.

**Pre-execution drifts catalogued:** None (pre-verification + Phase 0 v2 this session caught zero new drifts; Drift #54 catalogued at Y0 surface report; cumulative count stays at 54).

**Expected Stage 3 D-deviation count:** MODERATE (~2-4 drifts) per §2.2 analog limitation observation. Lower than V10 (~5) given Phase 0 v2 empirical grounding; higher than R2 ZERO-drift trio given dual-novel-pattern territory. Comparable to revocation Stage 3 (~3 drifts at execution; #51-#53 framing-vs-empirical-reality).

**Plan-level Y-decisions to resolve at execution:**

- **Y-PACKAGE-LOCATION:** `internal/source/` (a default) vs `internal/tools/source/` (b) vs inline (c); resolve at C3.2
- **Y-WIRE-FIELD-NAME:** `SourceRepoURL` (a default) vs `SourcePath` (b) vs both (c); resolve at C3.1
- **Y-CLONE-FAILURE-EVENT-SHAPE:** `SCAN_FAILED` + structured error (a default) vs generic (b) vs new event (c); resolve at C3.5

## 6. Out of Scope (per design doc §6 + plan-level refinements)

1. SSH-key git clone auth (Q-AUTH forward-pin)
2. Private-token git clone auth (Q-AUTH forward-pin)
3. `go-git` library adoption (Q-CLONE-LIB forward-pin)
4. SOURCE_ACQUISITION_* event types (Q-EVENTS forward-pin)
5. Recon-time source-clone (Q-RECON-TIMING forward-pin; Task 8.1 territory)
6. R2-upload-source-tarball variant (Y1 β forward-pin)
7. LRU staging-cache (Q-CLEANUP forward-pin)
8. `Project.source_repo_url` backfill / data migration
9. Engine source-tool parser changes (Q-PARSER NO-changes lock)
10. UI surfaces for `source_repo_url` display
11. SCA `ScanType.SCA` enum addition (Task 2 of 2 per SCA decomposition)
12. Composite-ScanType partial-success-after-source-fail (Q-FAILURE-MODE hard-fail lock)
13. shieldscan-api modifications outside `schemas/projects.py` + `services/orchestrator.py` + `schemas/scans.py` + relevant tests
14. shieldscan-engine modifications outside `events.go` + `tools/runner.go` + `worker/processor.go` + new `source/` + `tools/docker/trivy/` + relevant tests + DRIFT-LOG
15. shieldscan-docs modifications outside TOOL-ARCHITECTURE.md (no SPEC §13 ADR changes; addendum at §3.2 sub-location)

## 7. Forward-Pins

**Pre-execution forward-pins (Stage 3 entry):**

1. **Stage 3 trigger phrase:** ***"Resume source-ingestion fix — Stage 3 cross-repo implementation"***
2. **Y-PACKAGE-LOCATION decision context preserved:** `internal/source/` (a default) vs alternatives; resolve at C3.2
3. **Y-WIRE-FIELD-NAME decision context preserved:** `SourceRepoURL` (a default) vs alternatives; resolve at C3.1
4. **Y-CLONE-FAILURE-EVENT-SHAPE decision context preserved:** structured error (a default) vs alternatives; resolve at C3.5
5. **Design doc canonical authority:** `90fc933` §3 + §4 verbatim drafts
6. **Phase 0 v2 testbed:** `/tmp/sif-p0v2-1780787784/NodeGoat/` + `trivy-fs-p0v2.json` for Stage 3 integration test fixture

**Post-Stage-3 forward-pins:**

7. **SCA `ScanType.SCA` enablement task** — Task 2 of 2 per SCA decomposition; mechanical compressed-lifecycle on now-working source-ingestion path
8. **SSH-key git clone auth** — Q-AUTH forward-pin
9. **Private-token git clone auth** — Q-AUTH forward-pin
10. **`go-git` library** — Q-CLONE-LIB forward-pin if richer API surfaces
11. **SOURCE_ACQUISITION_* events** — Q-EVENTS forward-pin
12. **Recon-time source-clone (Task 8.1 integration)** — Q-RECON-TIMING forward-pin
13. **R2-upload-source-tarball variant** — Y1 β forward-pin
14. **LRU staging-cache** — Q-CLEANUP forward-pin
15. **`Project.source_repo_url` backfill** — out-of-v1
16. **M5/M6/M7/M8 completeness audit** — post-task-close milestone-completion-constraint audit (locked this session; runs after Stage 4 Phase 5 close)

## 8. Cross-References

**Engine commits:**

- `99e2c31` (F2 close; latest engine state; ZERO scope until Stage 3 Commit 3)
- `3ccf5b8` (R2 Stage 3 C3; events.go field addition + DRIFT-LOG inline precedent)
- `ad7cc94` (V10 Stage 3 C2; runner.go Target plumbing pattern)
- `b48fef8` (ADR-015 Stage 3 C3; engine credential consumer pattern)
- Task 4.5 `processor.go` cancel infrastructure (V-FG canonical)
- Task 7.1 `internal/tools/docker/trivy/` (canonical trivy-fs runner)
- Task 7.1 `internal/tools/recon/` (canonical subfinder+httpx; reference for new `internal/source/` standalone shape per Y-PACKAGE-LOCATION a)

**Docs commits:**

- `90fc933` (source-ingestion fix Stage 1 design doc; this plan's canonical authority)
- `56b8fce` (revocation P5.A; latest docs state pre-source-ingestion-fix)
- `0e55a4f` (revocation design doc; structural precedent)
- `fdad021` (revocation plan; structural precedent)
- `b25e9ba` (R2 design doc; structural precedent)
- `721f788` (R2 plan; structural precedent)
- `b344d0c` (ADR-015 design doc)
- `00dd2d1` (ADR-015 plan)
- `9a57865` (ADR-015 Stage 3 C1; multi-addendum precedent)
- `8f71b01` (R2 Stage 3 C1; multi-addendum continuation)
- `291ded3` (revocation Stage 3 C1; ADR addendum precedent)
- `40606c5` (Task 7.6 plan; structural precedent)

**API commits:**

- `40ce2f1` (cancel-helper extraction; latest api state pre-source-ingestion-fix)
- `453a80b` (revocation Stage 3 C2; orchestrator+validator pattern)
- `824853c` (R2 Stage 3 C2; orchestrator+enum pattern)
- `742faed` (ADR-015 Stage 3 C2; orchestrator decrypt+emit pattern)

**SPEC sections:**

- TOOL-ARCHITECTURE.md §3.2 (canonical design-intent authority; Q-ADR addendum target)
- §13 ADR-008 (Service-shape; container-mount semantics analog)
- §13 ADR-013 (sole-writer canonical; orchestrator-as-sole-writer pattern)
- §13 ADR-015 (credential-delegation pattern analog)

**Source authorities (Stage 3 sub-step targets):**

- shieldscan-docs TOOL-ARCHITECTURE.md §3.2 (C1.2 modification target)
- shieldscan-api `src/app/schemas/projects.py` (C2.1 modification target)
- shieldscan-api `src/app/services/orchestrator.py` (C2.2 modification target)
- shieldscan-api `src/app/schemas/scans.py` OR `routes/scans.py` (C2.3 modification target)
- shieldscan-api `tests/schemas/test_projects.py` + `tests/services/test_orchestrator.py` + `tests/routes/test_scans.py` (C2.4-C2.6 modification targets)
- shieldscan-engine `internal/events/events.go` (C3.1 modification target)
- shieldscan-engine `internal/source/` NEW package (C3.2 NEW target)
- shieldscan-engine `internal/tools/runner.go` (C3.3 modification target)
- shieldscan-engine `internal/worker/processor.go` (C3.4 modification target)
- shieldscan-engine `internal/tools/docker/trivy/` (C3.5 modification target)
- shieldscan-engine `internal/source/clone_test.go` + `internal/tools/docker/trivy/integration_test.go` (C3.6-C3.7 modification targets)
- shieldscan-engine `DRIFT-LOG.md` (C3.9 modification target)

**Phase 0 v2 artifacts:**

- `/tmp/sif-p0v2-1780787784/NodeGoat/` (depth=1 clone testbed; 3.3M)
- `/tmp/sif-p0v2-1780787784/trivy-fs-p0v2.json` (75-finding fixture; Stage 3 integration test reuse)

**Pre-verification artifacts:** V-IA-V-II surface report + Phase 0 v2 surface report (this session).

**Cumulative drift count:** 54 catches at execution time (Drift #54 source-ingestion-broken-end-to-end newly catalogued at Y0; ZERO new drifts through SIF_PV + Phase 0 v2 + brainstorming + Stage 1 design doc; clean Stage 2 entry).

**Milestone-completion-constraint context:** locked this session; post-Stage-4 Phase 5 P5.A close triggers M5/M6/M7/M8 completeness audit per the constraint's intent before any new task/milestone entry.
