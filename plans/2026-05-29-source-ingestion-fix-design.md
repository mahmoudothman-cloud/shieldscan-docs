# Source-Ingestion Fix: Design

**Status:** Brainstorming chain Y1+Y2+Y3+12 Q-locks complete; Phase 0 v2 empirically confirmed (ZERO pivot triggers); ready for implementation plan landing.

**Date:** 2026-05-29.

**Authority:** Y1 mechanism (α.1 host-side `os/exec` git clone per TOOL-ARCH §3.2 design intent + Phase 0 v2 empirical confirmation) + Y2 Phase 0 v2 (this session; bounded confirmation; ZERO pivot triggers) + Y3 direct Q-chain (post-Phase-0-v2-grounded) + Q-CLONE-LIB (a) + Q-STAGING-PATH (a) + Q-DEPTH `--depth=1` + Q-CLEANUP per-scan-defer-rm + Q-AUTH public-HTTPS-v1 + Q-VALIDATOR HTTPS-scheme + Q-WIRE engine-clones + Q-PARSER no-changes + Q-FAILURE-MODE hard-fail + Q-EVENTS standard-lifecycle + Q-RECON-TIMING lazy-per-job-at-worker + Q-ADR TOOL-ARCH addendum + Q-COMMITS 3-commit-trio brainstorming chain locks this session; pre-verification surface report V-IA-V-II + Phase 0 v2 surface report (this session); Drift #54 canonical authority (Y0 surface report + SIF_PV §J; stored-design-intent-unimplemented catch-class); `TOOL-ARCHITECTURE.md` §3.2 design intent (`SourcePath string // for SAST: local git clone path`); R2 design doc structural precedent (`b25e9ba`; 252 LoC); revocation design doc structural precedent (`0e55a4f`; 272 LoC).

**Related:** Implementation plan landing trigger phrase: ***"Begin source-ingestion fix implementation plan landing"*** (after this design doc lands).

---

## 1. Authority + Brainstorming Chain Summary

**Y1 (locked by TOOL-ARCH §3.2 design intent + P0v2 empirical confirmation) Mechanism (α.1) host-side `os/exec` git clone:** TOOL-ARCHITECTURE.md §3.2 canonically documented design intent (`SourcePath` as local git clone path); Phase 0 v2 empirically validated end-to-end (git binary present at `/usr/bin/git` v2.43.0; NodeGoat depth=1 clone 0.76s/3.3M; trivy fs 75 findings; Docker ReadOnly mount enforced; ZERO pivot triggers). Engine-side `os/exec` git clone `--depth=1` with per-scan tempdir under `$TRIVY_SCAN_BASE_PATH`; existing `trivy-fs` runner mount surfaces it without infrastructure change.

**Y2 (locked by SIF_PV V-IH + V-IB) Phase 0 v2 (this session):** Bounded ~30-45min empirical confirmation (P0v2.A-D); confirmed (α.1) viability end-to-end; revised Stage 3 LoC envelope downward (engine ~80-150 vs ~150-250 initial; aggregate ~150-300 vs ~220-400). Phase 0 v2 confirmation-style (not architectural-decision-style) per TOOL-ARCH §3.2 design-intent precedent.

**Y3 (locked by Phase 0 v2 completion + zero pivot triggers) Direct Q-chain:** All architectural decisions empirically grounded; no further empirical territory needed before Stage 1.

**Brainstorming Q-chain locks (12 Q's; high confidence per Phase 0 v2 + canonical authority):**

- **Q-CLONE-LIB (a) `os/exec`** — P0v2.A git binary baseline at `/usr/bin/git` v2.43.0; avoids `go-git` 5MB+ vendor footprint + version-pin discipline; integrates with ADR-021 Rule 1 (ctx-aware `exec.CommandContext`)
- **Q-STAGING-PATH (a) co-located `$TRIVY_SCAN_BASE_PATH/<scan-id>/`** — P0v2.D mount semantics; ZERO mount infrastructure changes (existing trivy-fs bind mount `/tmp` → `/scan` RO surfaces it automatically)
- **Q-DEPTH `--depth=1`** — P0v2.B empirically bounded (3.3M total / 1.2M .git on NodeGoat; ~36% overhead acceptable; `--filter=blob:none` forward-pinned if scale concern)
- **Q-CLEANUP per-scan `defer os.RemoveAll(stagingDir)`** — V-IH host-cleanup ReadOnly-mount-permissible (container's RO mount blocks container-side mutation, not host-side rm)
- **Q-AUTH public-HTTPS v1** — YAGNI; SSH-key + private-token forward-pinned to Q-AUTH expansion task
- **Q-VALIDATOR HTTPS-scheme-only on `Project.source_repo_url`** — V-IE current freeform gap + V-IH HTTPS-only network posture; mirrors `target_url` `_validate_https_url` pattern
- **Q-WIRE engine clones (wire carries URL; not path)** — separation-of-concerns; orchestrator-stateless-wrt-filesystem invariant preserved; `JobTarget.source_repo_url` added (analog to MobSF R2 `signed_fetch_url` field shape)
- **Q-PARSER NO changes** — P0v2.C empirical `parseTrivyJSON` contract preserved unchanged (existing JSON shape consumed; 75 findings parsed cleanly)
- **Q-FAILURE-MODE (a) hard-fail with `SCAN_FAILED` event** — Drift #54 silent-no-op repair; honest fix demands fail-fast (empty `SourceRepoURL` for source-requiring ScanType → hard error; clone failure → hard error; no degraded-success path)
- **Q-EVENTS (a) standard lifecycle** — no new event types; existing `SCAN_DISPATCHED` + `SCAN_FAILED` + `SCAN_COMPLETED` sufficient; clone-failure surfaces as ScanJob-level `FAILED` with diagnostic detail
- **Q-RECON-TIMING (a) lazy per-job at worker** — matches host-clone-then-mount; orchestrator-stateless invariant preserved; recon-pipeline-integration deferred to Task 8.1 territory
- **Q-ADR (c) TOOL-ARCHITECTURE.md addendum** — continues §3.2 design-intent canonical story to implementation-lock; no new ADR-NNN (SPEC §13 unchanged); follows TOOL-ARCH §3.2 + tool-trigger-table consistency
- **Q-COMMITS (a) 3-commit cross-repo (docs → api → engine)** — R2/V10 precedent + scope matches; docs FIRST (TOOL-ARCH addendum) → api SECOND (validator + orchestrator threading + tests) → engine THIRD (source package + wire field + trivy-fs integration + tests)

**Forward-pin chain closure:** Drift #54 (FULL_WEB_SOURCE/FULL_SPECTRUM aspirational-broken end-to-end) operationally settled with Stage 3 implementation; canonical-design-intent canonicalization §3.2 → addendum → implementation chain closes.

## 2. Pre-Verification + Phase 0 v2 Findings

Pre-verification (this session) grounded source-ingestion actual scope + git-clone infrastructure state + trivy-fs input expectations + container-security envelope + design-intent canonical-authority before brainstorming. Critical findings:

**V-IB Git-clone infrastructure (absent):** No `go-git`/`git2go`/`exec-shell git-clone` in engine `internal/` or `cmd/`; no git library in `go.mod`. Net-new (α.1) `os/exec` wrapper required. P0v2.A confirms git binary at `/usr/bin/git` v2.43.0 — provisioning-time baseline; no new Go dep needed.

**V-IC trivy-fs CLI shape:** `trivy fs --format json --scanners vuln --exit-code 0 <path>` per `buildArgsFs`. Bind mount `$TRIVY_SCAN_BASE_PATH` → `/scan` ReadOnly; `SourcePath` must be `/scan/<subpath>` container-relative; Trivy walks recursively for lockfiles/manifests.

**V-IE `Project.source_repo_url` validation (gap):** `String(500), nullable=True` on ORM; `str | None, max_length=500` on schemas; **no `@field_validator`** (unlike `target_url` which uses `_validate_https_url`). Freeform; URL-format-not-enforced. Q-VALIDATOR HTTPS-scheme-only addition closes gap.

**V-IF Task 8.1 recon (absent):** No `plans/2026-*task-8.1*.md`; engine recon at `internal/tools/recon/` is DNS+HTTP subdomain discovery only; no source-clone precedent. Recon-time-clone integration forward-pinned.

**V-IG `ScanCreateRequest` FULL_WEB_SOURCE shape:** No source-related field; no FULL_WEB_SOURCE validator (no requirement that `Project.source_repo_url` be present). Today users can `POST scan_type=FULL_WEB_SOURCE` against a project with `source_repo_url=NULL` → dispatch succeeds → source tools silently no-op (Drift #54 surface).

**V-IH container-security envelope (favorable):** Default bridge network → outbound HTTPS available; no `Privileged`, no `CapAdd`, no `SecurityOpt`; trivy mount ReadOnly. (α.1) host-side clone trust-boundary clean: **host clones (trusted process; ctx-aware via `exec.CommandContext`) → ReadOnly mount → container scans (untrusted-input-zone)**.

**V-II TOOL-ARCH §3.2 design intent (DOMINANT):** `SourcePath string // for SAST: local git clone path` — design intent canonically documented at TOOL-ARCHITECTURE.md §3.2; mechanism deferred → never implemented (Drift #54 root). Tool-trigger table at §10.3-ish documents `gitleaks/semgrep/dependency_check` gated on `Project.source_repo_url` configured (trivy-fs implied parallel).

**Phase 0 v2 empirical confirmation (P0v2.A-D):**

- **P0v2.A:** `git --version` → 2.43.0 at `/usr/bin/git`; host-side `os/exec` path empirically viable
- **P0v2.B:** `git clone --depth=1 https://github.com/OWASP/NodeGoat.git` → 0.76s wall-clock; 3.3M total / 1.2M .git; `package-lock.json` + `package.json` at root
- **P0v2.C:** `docker run aquasec/trivy:0.70.0 fs ...` → exit 0; 488KB JSON / 14,112 lines; **75 findings** (CRITICAL 10 + HIGH 39 + MEDIUM 16 + LOW 10); sample `CVE-2024-45590` `body-parser@1.18.3 → 1.20.3` HIGH
- **P0v2.D:** `-v $STAGE:/scan:ro` mount end-to-end; `touch /scan/ro_test` → `Read-only file system; exit=1` (ReadOnly enforced)

**ZERO pivot triggers across all 4 P0v2 targets.** All Q-chain pre-decisions surfaced with empirical grounding.

**Drift #54 catch-class:** **stored-design-intent-with-unimplemented-mechanism** — distinct from framing-vs-empirical (#44/#45/#50/#51-#53). Discipline implication: catch arrives from **documentation contact** (TOOL-ARCH §3.2) NOT code contact; pre-verification's V-II finding surfaced the canonical-authority ↔ implementation gap. This is a **new entry in the catch-class taxonomy** — a class where the design intent shipped but the mechanism didn't, silently leaving an aspirational dispatch path. Disciplinary value: cross-reference canonical authority against orchestrator/wire/engine code paths during pre-verification; if a documented field has zero readers, flag.

## 3. Architectural Decisions

Cross-references Q-chain locks (§1) + pre-verification findings (§2).

### 3.1 Y1 (α.1) Host-Side `os/exec` git clone

**Decision:** Engine internal package (e.g. `internal/source/` NEW; final location TBD at execution per existing convention review). `exec.CommandContext(ctx, "git", "clone", "--depth=1", url, stagingDir)`. ctx cancel → SIGKILLs subprocess (ADR-021 Rule 1).

**Rationale:** P0v2.A baseline; V-II TOOL-ARCH §3.2 design intent; avoids `go-git` vendor footprint + version-pin discipline; existing `os/exec` patterns in engine (e.g. native tools).

**Rejected alternatives:** (a.2) container-side clone (Trivy image lacks `git`; sidecar inferior); (b) `go-git` library (5MB+ vendor; richer API not required for v1 public-HTTPS scope).

### 3.2 Q-STAGING-PATH (a) Co-Located `$TRIVY_SCAN_BASE_PATH/<scan-id>/`

**Decision:** Source-acquisition writes to `$TRIVY_SCAN_BASE_PATH/<scan-id>/` on host. Existing trivy-fs warm-pool bind mount (`$TRIVY_SCAN_BASE_PATH` → `/scan` RO) surfaces it automatically as `/scan/<scan-id>/`. `Target.SourcePath` populated as `/scan/<scan-id>` (container-relative).

**Rationale:** P0v2.D mount semantics verified; ZERO mount infrastructure changes; preserves Task 7.5e Mounts capability invariants.

### 3.3 Q-DEPTH `--depth=1`

**Decision:** Shallow clone (HEAD only).

**Rationale:** P0v2.B empirically bounded (3.3M total / 1.2M .git on NodeGoat); Trivy fs only walks manifests + lockfiles (no history dependency); minimum bandwidth + disk.

**Forward-pin:** `--filter=blob:none` if scale concern surfaces (e.g. monorepos with large binaries committed).

### 3.4 Q-CLEANUP Per-Scan `defer os.RemoveAll(stagingDir)`

**Decision:** Per-scan tempdir; `defer os.RemoveAll(stagingDir)` after trivy-fs runner returns.

**Rationale:** V-IH host-side `rm` permitted despite container's ReadOnly mount (RO blocks container-side mutation only); simplest invariant; bounds disk usage per scan lifecycle.

**Forward-pin:** LRU staging-cache if monorepo re-scan disk pressure surfaces; per-project staging-dir invalidation policy.

### 3.5 Q-AUTH Public-HTTPS v1

**Decision:** Public HTTPS clone only; no SSH-key support; no PAT/OAuth token support.

**Rationale:** YAGNI for v1; matches `Project.target_url` HTTPS-only validator pattern; defers credential-handling complexity.

**Forward-pin:** SSH-key auth task; PAT/OAuth integration task (with credential-storage analog to `ProjectCredential`).

### 3.6 Q-VALIDATOR HTTPS-scheme-only on `Project.source_repo_url`

**Decision:** Add `@field_validator("source_repo_url")` to `ProjectCreateRequest` + `ProjectUpdateRequest` mirroring `_validate_https_url`.

**Rationale:** V-IE current freeform gap; V-IH HTTPS-only network posture; prevents `git://` + `ssh://` + malformed input from reaching engine clone path.

**Trade-off:** Existing rows with non-HTTPS URLs would surface validation errors on PATCH; v1 default behavior leaves stored rows untouched (validator runs on write path only).

### 3.7 Q-WIRE Engine Clones (Wire Carries URL)

**Decision:** Wire carries the URL via `JobTarget.source_repo_url`; engine clones at job-execution time.

**Rationale:** Separation-of-concerns (orchestrator stateless wrt filesystem); analog to MobSF R2 `signed_fetch_url` sibling-field pattern; engine owns staging-dir lifecycle entirely.

**Rejected alternative:** orchestrator-side pre-clone + signed-tarball pattern (R2-analog Y1 β) — deferred to forward-pin if private-repo / git-protocol-unavailable use cases surface.

### 3.8 Q-PARSER NO Changes

**Decision:** `parseTrivyJSON` consumed unchanged.

**Rationale:** P0v2.C empirical: JSON shape (`SchemaVersion` 2 + `Results[].Target` + `Results[].Vulnerabilities[]`) matches existing parser contract; 75 findings parsed cleanly.

### 3.9 Q-FAILURE-MODE (a) Hard-Fail with `SCAN_FAILED` Event

**Decision:** Source-requiring ScanType + null `source_repo_url` → hard-error at orchestrator validation (`ScanCreateRequest` validator). Clone failure at engine → ScanJob `FAILED` with diagnostic detail; no degraded-success path.

**Rationale:** Drift #54 silent-no-op repair; honest fix demands fail-fast; user surfacing of misconfiguration > silent-empty-finding-set.

**Rejected alternatives:** soft-skip (preserves Drift #54 silent failure); per-tool conditional dispatch (orchestrator complexity creep).

### 3.10 Q-EVENTS (a) Standard Lifecycle

**Decision:** No new event types. Clone-failure surfaces as ScanJob-level `FAILED` event via existing pipeline.

**Rationale:** YAGNI for v1; existing observability sufficient (`SCAN_DISPATCHED` + `SCAN_FAILED` + `SCAN_COMPLETED`).

**Forward-pin:** `SOURCE_ACQUISITION_STARTED` / `_COMPLETED` / `_FAILED` events if richer observability surfaces.

### 3.11 Q-RECON-TIMING (a) Lazy Per-Job at Worker

**Decision:** Clone happens at the worker, lazily per source-requiring job, not at recon stage.

**Rationale:** Matches host-clone-then-mount pattern; orchestrator-stateless invariant preserved; recon-pipeline-integration is Task 8.1 territory (absent today).

**Forward-pin:** Recon-pipeline source-clone integration if multi-tool source-aware scans benefit from single shared clone.

### 3.12 Q-ADR (c) TOOL-ARCHITECTURE.md Addendum

**Decision:** Land Source-Acquisition Implementation Lock as TOOL-ARCHITECTURE.md addendum after §3.2.

**Rationale:** §3.2 already canonically documents the design intent (`SourcePath` field comment); addendum completes the canonical-authority → implementation chain; no new ADR-NNN warranted (SPEC §13 unchanged).

**Rejected alternatives:** new SPEC §13 ADR-NNN (over-coverage); ADR-013 addendum (sole-writer canonical unchanged; orchestrator threads URL not path, no payload-contract architectural shift).

### 3.13 Q-COMMITS (a) 3-Commit Cross-Repo Trio

**Decision:** docs FIRST (TOOL-ARCH addendum) → api SECOND (validator + orchestrator threading + tests) → engine THIRD (source package + wire field + trivy-fs integration + tests).

**Rationale:** R2/V10 trio precedent; scope distribution matches.

**Cross-reference shape:** Commit 1 docs references future api+engine hashes via placeholders; Commit 2 api references docs Commit 1 hash concretely; Commit 3 engine references docs + api hashes concretely.

## 4. Cross-Repo Implementation Surface

### 4.1 Docs Side (Stage 3 Commit 1)

**TOOL-ARCHITECTURE.md addendum after §3.2:** Source-Acquisition Implementation Lock — mechanism (α.1) host-side `os/exec` git clone `--depth=1` with per-scan tempdir under `$TRIVY_SCAN_BASE_PATH`; ReadOnly mount surfaces it; HTTPS-only validator; hard-fail semantics; Drift #54 root-cause repair acknowledgment.

**Total docs delta:** ~25-50 LoC.

### 4.2 API Side (Stage 3 Commit 2)

- `src/app/schemas/projects.py`: HTTPS-scheme validator on `source_repo_url` field on `ProjectCreateRequest` + `ProjectUpdateRequest` (mirrors `target_url` `_validate_https_url`) — ~5-10 LoC
- `src/app/services/orchestrator.py`: `_build_job_payload` threading `scan.project.source_repo_url` → `target.source_repo_url` (conditional on `scan_type` ∈ source-requiring set: `FULL_WEB_SOURCE`, `FULL_SPECTRUM`, future `SCA`) — ~10-20 LoC
- `src/app/schemas/scans.py`: `ScanCreateRequest` validator — when `scan_type` requires source, assert `project.source_repo_url is not None` (server-side resolution at route handler) — ~5-15 LoC
- `tests/schemas/test_projects_credentials.py` OR equivalent: validator tests (HTTPS-scheme accept; non-HTTPS reject; empty reject) — ~15-25 LoC
- `tests/services/test_orchestrator.py`: orchestrator threading test (FULL_WEB_SOURCE scan with source_repo_url → payload contains it) + failure-mode test (null source_repo_url on source-requiring ScanType → validator raises) — ~25-35 LoC

**Total api delta:** ~60-105 LoC.

### 4.3 Engine Side (Stage 3 Commit 3)

- `internal/events/events.go`: `JobTarget.SourceRepoURL` field (analog to `MobileConfig.SignedFetchURL` sibling shape) — ~3-5 LoC
- `internal/source/` NEW package: `cloneRepo(ctx, url, stagingDir)` helper + per-scan tempdir manager (`mkdtemp` + `defer rm`) + HTTPS-scheme guard (defensive belt-and-braces vs api validator) — ~50-100 LoC
- `internal/tools/runner.go`: `Target.SourceRepoURL` field — ~2-3 LoC
- `internal/worker/processor.go`: `jobDispatchToTarget` threading `SourceRepoURL` → `Target` — ~3-5 LoC
- `internal/tools/docker/trivy/trivy.go` OR `scan.go`: pre-clone step before trivy-fs runner invocation — clone via `internal/source.cloneRepo` if `SourceRepoURL` non-empty + populate `SourcePath` as `/scan/<scan-id>` + `defer os.RemoveAll(stagingDir)` cleanup; preserve `buildArgsFs` empty-target guard semantics — ~20-40 LoC
- `internal/source/source_test.go`: `cloneRepo` unit test (against public NodeGoat or sibling testbed) — ~20-30 LoC
- `internal/tools/docker/trivy/integration_test.go`: extension with source-clone-then-scan end-to-end test — ~15-25 LoC
- `DRIFT-LOG.md`: inline Drift #54 root-cause + LANDED entry — ~15-25 LoC

**Total engine delta:** ~128-233 LoC.

### 4.4 Aggregate Stage 3 LoC

Total: ~213-388 LoC. Between revocation Stage 3 (~88-145) and R2 Stage 3 (~200-313); closer to R2 upper-band per Q-COMMITS trio + engine source package addition.

## 5. Phase Structure

Per Q-COMMITS (a) 3-commit cross-repo + Y3 direct Q-chain (post-Phase-0-v2-grounded):

- **Stage 1 — Design Doc Landing (THIS COMMIT; this session)** — ~280-330 LoC
- **Stage 2 — Implementation Plan Landing (~30-45min)** — Y-decisions surface; plan at `plans/2026-05-29-source-ingestion-fix-implementation.md` (~250-320 LoC; mirrors R2 plan)
- **Stage 3 — 3-Commit Cross-Repo (~3-5h)** — per §4
- **Stage 4 — Phase 5 Sub-Phases (~30-60min)** — drift annotation + dispositions; Phase 0 v2 grounding makes P5.A likely lighter

## 6. Out of Scope

1. SSH-key git clone auth (Q-AUTH forward-pin)
2. Private-token / PAT git clone auth (Q-AUTH forward-pin)
3. `go-git` library adoption (Q-CLONE-LIB forward-pin if richer API surfaces)
4. `SOURCE_ACQUISITION_*` event types (Q-EVENTS forward-pin)
5. Recon-time source-clone (Q-RECON-TIMING forward-pin to Task 8.1 territory)
6. R2-upload source-tarball variant (Y1 (β) forward-pin if private-repo / git-protocol-unavailable surfaces)
7. LRU staging-cache (Q-CLEANUP forward-pin if disk pressure surfaces)
8. `Project.source_repo_url` backfill / data migration (per-existing-project; out-of-v1)
9. Engine source-tool parser changes (Q-PARSER NO-changes lock)
10. UI surfaces for `source_repo_url` display (api scope only)
11. **`ScanType.SCA` enum addition** (deferred to Task 2 of SCA decomposition per Option B; this task fixes the pipe, SCA task exposes the public ScanType)
12. Composite-ScanType partial-success-after-source-fail (Q-FAILURE-MODE hard-fail lock)

## 7. Forward-Pins

**Pre-execution forward-pins (Stage 2 entry):**

1. **Stage 2 plan trigger:** ***"Begin source-ingestion fix implementation plan landing"***
2. **Engine package location:** `internal/source/` (default) vs `internal/tools/source/` (alternate; under `tools` umbrella for trivy-fs co-location); resolve at Stage 3 Commit 3 execution per file-organization-convention surface
3. **Y-FAIL-EVENT-DETAIL:** failure detail string format (free-form vs structured); default free-form; resolve at C3 execution
4. **Design doc canonical authority:** this commit's hash for Stage 2 plan reference

**Post-Stage-3 forward-pins:**

5. ***"Begin SCA ScanType.SCA enablement task"*** (Task 2 of SCA decomposition; mechanical compressed-lifecycle on now-working source-ingestion pipe)
6. ***"Begin Q-AUTH SSH-key/PAT credential support task"*** (auth-expansion if private-repo demand surfaces)
7. ***"Begin Q-CLONE-LIB go-git adoption task"*** (richer-API trigger; LRU caching, partial clone, in-memory tree access)
8. ***"Begin Q-EVENTS richer observability task"*** (SOURCE_ACQUISITION_* event types)
9. ***"Begin Q-RECON-TIMING recon-pipeline integration task"*** (Task 8.1 territory; multi-tool shared-clone)
10. ***"Begin R2-upload source-tarball variant task"*** (Y1 β forward-pin)
11. ***"Begin LRU staging-cache task"*** (Q-CLEANUP forward-pin)
12. ***"Begin Project.source_repo_url backfill task"*** (data migration)

## 8. Cross-References

**Engine commits:**
- `99e2c31` (F2 close; latest engine state)
- `3ccf5b8` (R2 Stage 3 C3; httpFetcher + http_fetcher pattern precedent for engine staging-then-consume shape)
- `ad7cc94` (V10 Stage 3 C2; integration_test.go pattern precedent)
- `b48fef8` (ADR-015 Stage 3 C3; integration test V4 baseline pattern)
- Task 7.1 trivy-fs runner (`internal/tools/docker/trivy/`; consumed unchanged in this task)

**Docs commits:**
- `56b8fce` (revocation P5.A; latest docs state pre-this-commit)
- `b25e9ba` (R2 design doc; structural precedent)
- `0e55a4f` (revocation design doc; structural precedent)
- `88b192c` (R2 P5.A; latest pre-revocation docs state)
- `b344d0c` (ADR-015 design doc; ADR-canonical-authority precedent)
- TOOL-ARCHITECTURE.md §3.2 (CANONICAL design-intent authority for `SourcePath` field)
- TOOL-ARCHITECTURE.md §10.3-ish tool-trigger table (gitleaks/semgrep/dependency_check source-gated)

**API commits:**
- `40ce2f1` (cancel-helper extraction; latest api state)
- `453a80b` (revocation Stage 3 C2; `_build_job_payload` extension precedent)
- `824853c` (R2 Stage 3 C2; orchestrator `_build_job_payload` MobileConfig threading precedent)
- `742faed` (ADR-015 Stage 3 C2; orchestrator credential-decrypt + emit precedent)

**SPEC sections:**
- §13 ADR-008 (Service-shape; trivy-fs uses warm-pool DockerRunner not Service)
- §13 ADR-013 (sole-writer canonical; orchestrator threads URL not state)
- §13 ADR-015 (credential-delegation pattern analog for orchestrator-as-sole-writer-of-time-bounded-access-fields)
- §3.2 trivy.go directory placement (SCA + container commitment)

**Source authorities (Stage 3 sub-step targets):**
- shieldscan-engine `internal/tools/docker/trivy/trivy.go` (NewFsRunner; consumed unchanged)
- shieldscan-engine `internal/tools/docker/trivy/scan.go` (`buildArgsFs`; consumed unchanged)
- shieldscan-engine `internal/events/events.go` (`JobTarget` wire shape; SourceRepoURL field add)
- shieldscan-engine `internal/tools/runner.go` (`Target` shape; SourceRepoURL field add)
- shieldscan-engine `internal/worker/processor.go` (`jobDispatchToTarget`; SourceRepoURL mapping add)
- shieldscan-engine `internal/source/` NEW package (cloneRepo helper)
- shieldscan-api `src/app/schemas/projects.py` (HTTPS validator add on `source_repo_url`)
- shieldscan-api `src/app/services/orchestrator.py` (`_build_job_payload` threading)
- shieldscan-api `src/app/schemas/scans.py` (`ScanCreateRequest` validator for source-requiring ScanType)
- shieldscan-api `src/app/models/projects.py` (`Project.source_repo_url` column; consumed unchanged)

**Phase 0 v2 artifacts (preserved for forward-pin reference + integration test fixture):**
- `/tmp/sif-p0v2-1780787784/NodeGoat/` (depth=1 clone testbed; 3.3M; 1.2M .git)
- `/tmp/sif-p0v2-1780787784/trivy-fs-p0v2.json` (75-finding JSON fixture; 488KB; reusable as engine integration test fixture mirroring Task 7.1 testdata pattern)

**Pre-verification artifacts:**
- V-IA-V-II surface report (this session)
- Phase 0 v2 surface report (P0v2.A-D; this session)

**Cumulative drift count:** 54 catches at execution time (Drift #54 source-ingestion-broken-end-to-end newly catalogued at Y0; ZERO new drifts through SIF_PV + Phase 0 v2 + brainstorming).
