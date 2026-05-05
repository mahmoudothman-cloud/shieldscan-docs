# Warm Pool Primitive + DockerRunner Framework — Design Doc

**Date:** 2026-05-03
**Author:** Mahmoud Hassan (Odyssey Technology) with Claude
**Status:** Approved design; pending implementation
**Task:** M7 Task 7.5a (warm pool primitive + DockerRunner framework only)
**Scope:** CLI-shaped Docker tools (Trivy, Nmap, SQLMap); ZAP + MobSF
deferred to Task 7.5b (DockerServiceRunner; separate brainstorming)
**Cross-references:** ADR-006 (Hybrid Native + Persistent Docker),
ADR-023 (NativeRunner OutputFile mode), ADR-026 (this commit; warm
pool + DockerRunner), IMPLEMENTATION-PLAN Task 7.5, SPECIFICATION §3
architecture diagram (lines 229 + 248)

---

## 0. Executive Summary

Implement the **warm pool primitive** + **DockerRunner framework type**
that establishes the architectural pattern for M7 CLI-shaped Docker
tools (Trivy, Nmap, SQLMap).

**This task ships the framework only.** The 3 CLI tools (Trivy, Nmap,
SQLMap) are subsequent tasks (per M6 working pattern: per-task brainstorm
+ scope proposal + implementation cycle). ZAP + MobSF are HTTP-shaped
services per ADR-006 + ADR-008 and use a separate DockerServiceRunner
abstraction (Task 7.5b; not in scope here).

**Estimated effort: ~6-9 hours** (framework type + warm pool primitive
+ Docker SDK integration + tests + DRIFT entries + ADR-026).

**Single architectural artifact: ADR-026.**

**Single-repo task:** shieldscan-engine. No cross-repo coordination.

**Coordination shape: Approach C (single atomic commit).**

**Closes:** IMPLEMENTATION-PLAN Task 7.5 framework primitive obligation
+ project corpus's 5th asymmetric-cost ADR opportunity.

**Establishes:** the architectural pattern for all M7 CLI Docker
tools; future M8+ Docker tools (if any new CLI-shaped) inherit the
warm pool primitive directly.

---

## 1. Brainstorming Output (Locked Decisions)

The brainstorming session locked **six architectural decisions through
six clarifying questions** plus three additional design-confirmation
locks plus one ADR-006 conflict resolution:

| # | Decision | Lock |
|---|----------|------|
| 1 | "Finish the system" goal | **Feature-complete platform** (M7 + M8 + M9 + M10 + M11) |
| 2 | Build order | **Bottom-up** (M7 → M8 → M9 → M10 → M11) |
| 3 | M7 scope | **Full M7 (5 tools)**: Trivy + Nmap + MobSF + ZAP + SQLMap |
| 4 | Container lifecycle | **Warm pool** (per IMPLEMENTATION-PLAN Task 7.5) |
| 5 | Brainstorming depth | **Per-task cycles** (warm pool now; per-tool brainstorm at each tool task; matches M6 pattern) |
| 6 | Pool semantics | **Lazy warm** (pool starts empty; first checkout triggers spin-up; max-bound prevents runaway) |
| 7 | Reuse semantics | **Reuse with cleanup hook** (per-tool cleanup function runs between checkouts; empirical fallback to hybrid tiers if MobSF surfaces unreliability) |

Plus three additional design-confirmation locks:

| # | Decision | Lock |
|---|----------|------|
| 8 | Implementation approach | **Approach B** — DockerRunner framework type wraps WarmPool; per-tool runners are minimal config + parser |
| 9 | ADR-026 title | **(a)** "DockerRunner framework + lazy warm pool — M7 container lifecycle architecture" |
| 10 | This task scope | **Task 7.5a framework only** — 3 CLI tools (Trivy, Nmap, SQLMap) future tasks; ZAP + MobSF use Task 7.5b DockerServiceRunner |

Plus the ADR-006 conflict resolution:

| # | Decision | Lock |
|---|----------|------|
| 11 | ADR-006 conflict resolution | **Option β** — warm pool for CLI tools (Trivy, Nmap, SQLMap); ZAP + MobSF use DockerServiceRunner (HTTP) per ADR-006 + ADR-008; this design doc covers warm pool only |

**Asymmetric-cost meta-principle:** ADR-026 candidacy is the **5th
project-corpus invocation** of asymmetric-cost reasoning (after ADR-022
recon-as-helpers, ADR-023 NativeRunner OutputFile mode, ADR-024 schema
extension, ADR-025 findings-ingest). Pattern is well-established;
DEVELOPMENT-PATTERNS entry codifying this meta-principle is appropriate
at the next architectural decision-point.

---

## 2. Problem Statement

### 2.1 Current State

After M6 close (commit a61601a) and M6-close-followup + findings-ingest
task close, ShieldScan engine has:

- **9 native binary tool runners** (Nuclei, Semgrep, Gitleaks, SSLyze,
  Dep-Check, Checkov, Nikto, Wapiti, CORStest) consumed via
  `tools.NativeRunner` framework
- **1 recon helper package** (`internal/tools/recon/`; Subfinder + httpx;
  not Registry-registered per ADR-022)
- **Zero Docker tool runners.** M7 introduces this entirely new
  architectural surface.

Per ADR-006 (refined 2026-04-18), heavy tools deploy as **persistent
Docker services** to eliminate per-scan container startup latency
(2-3s overhead). The architectural commitment is: warm pool + reuse
semantics, NOT per-scan ephemeral containers.

Per IMPLEMENTATION-PLAN Task 7.5, a `warm_pool.go` module is
specified at `shieldscan-engine/internal/tools/warm_pool.go`
(SPEC §3 architecture diagram, line 248). The primitive is
anticipated; tonight's task delivers it.

### 2.2 The Architectural Decision

Three interrelated commitments lock together:

**(A) Lazy warm semantics.** Pool starts empty; first checkout triggers
spin-up; subsequent checkouts reuse warmed containers; max-bound
prevents runaway resource usage; resource floor is **zero for unused
tools.**

**(B) Cleanup hook between checkouts.** Per-tool cleanup function
runs between scan completion and container return-to-pool. Tenant
isolation guaranteed via cleanup; no state leak between scans.

**(C) DockerRunner framework type wrapping WarmPool.** Per-tool runners
provide minimal config (image, max pool size, exec command builder,
output parser, cleanup hook) + ToolRunner-shaped interface; framework
handles container lifecycle centrally.

### 2.3 Scope Discipline (Per Option β Resolution)

This task ships the warm pool primitive + DockerRunner framework only.
**The 3 CLI consumer tools (Trivy, Nmap, SQLMap) are subsequent tasks**;
each gets per-task brainstorming + scope proposal + implementation
per the M6 working pattern.

**ZAP + MobSF are NOT consumers of this primitive.** Per ADR-006 +
ADR-008, they are HTTP-shaped persistent services accessed via
DockerServiceRunner (Task 7.5b; separate brainstorming + design doc).

The architecture diagram (SPEC §3) lists BOTH primitives explicitly:
- `internal/tools/docker/warm_pool.go` (this task)
- `internal/tools/docker/docker_service.go` (Task 7.5b)

Two-runner-type architecture is explicit project commitment, not
emergent design.

### 2.4 Why The Framework Primitive Matters Now

**Asymmetric-cost analysis:**

- Cost of Approach A (per-tool pool orchestration; ~30-50 LoC per tool):
  ~150-250 LoC duplicated across 3 CLI tools + scattered observability
- Cost of Approach C (two-layer; warm pool primitive + escape hatch):
  premature; YAGNI on escape hatch until concrete second use case
- Cost of Approach B (DockerRunner wraps WarmPool): centralizes
  lifecycle in ~200 LoC framework; per-tool ~10-15 LoC

Total saved: ~100-200 LoC + concentrated test coverage in framework.

**Plus:** future Docker tools inherit the primitive cleanly. M8+
hypothetical Docker tools (if any new CLI-shaped) consume DockerRunner
identically to Trivy/Nmap/SQLMap. The framework asymmetric-cost
compounds across the platform's lifetime.

### 2.5 Why Lazy Warm Specifically

Pre-warming all containers at startup wastes resources on tools never
used in a given deployment. Customer running only SAST scans pays zero
cost for Trivy/Nmap/SQLMap pools. Lazy warm achieves this naturally
via "spin up on first checkout."

Per-task spin-up (no pool) defeats the architectural commitment from
ADR-006 (eliminate 2-3s startup latency).

Always-warm fixed pool wastes resources for unused tools.

Lazy warm is the right size: zero resource floor for unused tools;
warmed reuse for used tools; max-bound prevents runaway.

---

## 3. Solution Architecture

### 3.1 Component Inventory

The solution consists of 4 components, all in `shieldscan-engine`:

| # | Component | Status | Purpose |
|---|-----------|--------|---------|
| 1 | `internal/tools/docker/warmpool.go` | **NEW** | Warm pool primitive (lazy spin-up, max-bound, cleanup hook, health check) |
| 2 | `internal/tools/docker/dockerrunner.go` | **NEW** | Framework type wrapping WarmPool; provides ToolRunner-shaped interface |
| 3 | `internal/tools/docker/container.go` | **NEW** | Container lifecycle wrapper (Docker SDK abstraction) |
| 4 | `internal/tools/docker/*_test.go` | **NEW** | Comprehensive test coverage including concurrent-safety + lifecycle |

### 3.2 WarmPool Type (Component 1)

**File:** `internal/tools/docker/warmpool.go` (shipped in shieldscan-engine commit f5d77c8)

> **Post-implementation correction (2026-05-04).** The original verbatim Go in this section had three concurrency-incorrect patterns that were caught at design-spec-reading time during Phase 2 of Task 7.5a implementation. The corrected exported API + load-bearing concurrency invariants are documented below; the canonical source-of-truth is the shieldscan-engine source files at commit f5d77c8. See SPEC §13 ADR-026 for the architectural reasoning.

The WarmPool primitive ships at `internal/tools/docker/warmpool.go` (per shieldscan-engine commit f5d77c8). The exported API:

```go
type WarmPool struct { /* unexported fields */ }
type Config struct {
    Image       string
    MaxSize     int
    Cleanup     CleanupFunc
    HealthCheck HealthCheckFunc  // optional
}
type CleanupFunc func(ctx context.Context, c *Container) error
type HealthCheckFunc func(ctx context.Context, c *Container) bool

func New(cfg Config, cli dockerClient, log zerolog.Logger) (*WarmPool, error)
func (p *WarmPool) Checkout(ctx context.Context) (*Container, error)
func (p *WarmPool) Return(ctx context.Context, c *Container) error
func (p *WarmPool) Shutdown(ctx context.Context) error
func (p *WarmPool) Size() int
func (p *WarmPool) Available() int

// NoCleanup is the explicit no-op CleanupFunc for stateless tools.
// Statelessness is architecturally visible — no nil functions accepted.
func NoCleanup(ctx context.Context, c *Container) error { return nil }
```

**Concurrency invariants** (load-bearing — see ADR-026 in SPECIFICATION.md §13 + DRIFT-LOG entry 5 in shieldscan-engine for the full deviation history; brief summary):

1. **Done-channel signal pattern (not channel-close-as-signal).** A separate `done chan struct{}` carries the shutdown signal; the `available` channel is never closed. Closing `available` would race with concurrent Return goroutines on the send side (panic: send to closed channel). The closed-flag check + send happen atomically under the pool mutex.

2. **Blocked Checkout observes shutdown signal.** The blocking-receive case includes `<-p.done` in its select, alongside `<-p.available` and `<-ctx.Done()`. Without this, a Checkout blocked at max-size receives the zero-value `*Container` (nil) when Shutdown closes available, returning `(nil, nil)` to caller — silent nil-Container hazard. Pinned by `TestWarmPool_Shutdown_UnblocksWaitingCheckout`.

3. **Health-check replacement is size-neutral on success.** When health check fails, the original container is stopped (-1) and a replacement is spun up (+1) — net zero size change. Size decrement happens only on spin-up failure (original stopped, replacement failed → net -1). The naive "decrement-then-spinUp" pattern leaks size downward by one per replacement.

4. **Race-detector-clean across 500-cycle concurrent stress.** `TestWarmPool_Concurrent_NoRaceCondition` (10 goroutines × 50 cycles) is the load-bearing race detector pin.

**DockerRunner integration cleanup:**

```go
// Inside DockerRunner.Run (excerpt from shieldscan-engine f5d77c8):
defer func() {
    // Detached cleanup ctx survives primary-context cancellation.
    // Cleanup-uses-parent-context anti-pattern explicitly avoided
    // (matches M6.7 Wapiti file-output cleanup pattern; 2nd instance).
    cleanupCtx, cancel := context.WithTimeout(context.WithoutCancel(ctx), 30*time.Second)
    defer cancel()
    if err := r.Pool.Return(cleanupCtx, c); err != nil {
        r.Log.Warn().Err(err).Msg("pool return failed")
    }
}()
```

The detached cleanup context ensures Pool.Return's cleanup hook has a fresh deadline-bounded window even when Run's primary context hit its timeout. Without `context.WithoutCancel`, cleanup would fail-fast against an already-cancelled context, leaving stale containers in the pool.

For the full implementation (including stub-backed test infrastructure with atomic.Int64 unique-ID counter), see `shieldscan-engine/internal/tools/docker/warmpool.go` and `warmpool_test.go` (commit f5d77c8).

<details>
<summary>Original pre-implementation Go verbatim (preserved for traceability — DO NOT use as reference; concurrency-incorrect per the patterns documented above)</summary>

```go
package docker

import (
    "context"
    "errors"
    "fmt"
    "sync"
    "time"

    "github.com/rs/zerolog"
)

// WarmPool manages a lazy-warm pool of Docker containers for a single
// tool image. Pool starts empty; first checkout spins up a container;
// subsequent checkouts reuse warmed containers. Max-bound prevents
// runaway resource usage.
//
// Per ADR-026: matches the "lazy warm" semantics locked in
// brainstorming. Resource floor is zero for unused tools.
//
// Concurrency: WarmPool is safe for concurrent use. Channel-based
// available pool with sync.Mutex for size accounting.
type WarmPool struct {
    image    string
    maxSize  int
    cleanup  CleanupFunc  // runs between checkouts
    health   HealthCheckFunc  // optional pre-checkout validation
    log      zerolog.Logger

    available chan *Container
    mu        sync.Mutex
    size      int  // current pool size (created containers)
}

// CleanupFunc runs between checkouts to ensure tenant isolation.
// Returning an error causes the container to be replaced rather than
// returned to pool.
type CleanupFunc func(ctx context.Context, c *Container) error

// HealthCheckFunc runs before checkout to validate container
// reusability. Returning false causes the container to be replaced.
// Optional; nil HealthCheckFunc means "always healthy."
type HealthCheckFunc func(ctx context.Context, c *Container) bool

// Config configures a WarmPool. Image is required; MaxSize defaults to 4
// if zero; Cleanup is required (no-op cleanup must be explicit per
// brainstorming Decision 7); HealthCheck is optional.
type Config struct {
    Image       string
    MaxSize     int
    Cleanup     CleanupFunc
    HealthCheck HealthCheckFunc  // optional
}

// New creates a WarmPool. Pool is empty; lazy spin-up on first
// Checkout.
func New(cfg Config, log zerolog.Logger) (*WarmPool, error) {
    if cfg.Image == "" {
        return nil, errors.New("warmpool: Image required")
    }
    if cfg.Cleanup == nil {
        return nil, errors.New("warmpool: Cleanup required (use NoCleanup if stateless)")
    }
    maxSize := cfg.MaxSize
    if maxSize == 0 {
        maxSize = 4  // default
    }
    return &WarmPool{
        image:     cfg.Image,
        maxSize:   maxSize,
        cleanup:   cfg.Cleanup,
        health:    cfg.HealthCheck,
        log:       log,
        available: make(chan *Container, maxSize),
    }, nil
}

// Checkout retrieves a container from the pool. If available pool
// has a warmed container, returns it. Otherwise spins up a new
// container if size < maxSize. Otherwise blocks until a container
// becomes available (with ctx-respect timeout).
func (p *WarmPool) Checkout(ctx context.Context) (*Container, error) {
    select {
    case c := <-p.available:
        // Got a warmed container; verify health if configured
        if p.health != nil && !p.health(ctx, c) {
            p.log.Warn().Str("container_id", c.ID).Msg("health check failed; replacing")
            _ = c.Stop(ctx)
            return p.spinUp(ctx)
        }
        return c, nil
    default:
        // No available container; check if we can spin up
        p.mu.Lock()
        if p.size < p.maxSize {
            p.size++
            p.mu.Unlock()
            return p.spinUp(ctx)
        }
        p.mu.Unlock()
        // At max; block until available
        select {
        case c := <-p.available:
            if p.health != nil && !p.health(ctx, c) {
                _ = c.Stop(ctx)
                return p.spinUp(ctx)
            }
            return c, nil
        case <-ctx.Done():
            return nil, ctx.Err()
        }
    }
}

// Return runs cleanup hook + returns container to pool. If cleanup
// fails, container is stopped and pool size is decremented (size
// allows future spin-up).
func (p *WarmPool) Return(ctx context.Context, c *Container) error {
    if err := p.cleanup(ctx, c); err != nil {
        p.log.Warn().Err(err).Str("container_id", c.ID).Msg("cleanup failed; stopping container")
        _ = c.Stop(ctx)
        p.mu.Lock()
        p.size--
        p.mu.Unlock()
        return err
    }
    select {
    case p.available <- c:
        return nil
    default:
        // Channel full (shouldn't happen per max-size logic, but defensive)
        _ = c.Stop(ctx)
        p.mu.Lock()
        p.size--
        p.mu.Unlock()
        return errors.New("warmpool: pool full on return; container stopped")
    }
}

// Shutdown stops all warmed containers + drains pool. Called on
// worker shutdown.
func (p *WarmPool) Shutdown(ctx context.Context) error {
    close(p.available)
    var errs []error
    for c := range p.available {
        if err := c.Stop(ctx); err != nil {
            errs = append(errs, err)
        }
    }
    if len(errs) > 0 {
        return fmt.Errorf("warmpool: shutdown errors: %v", errs)
    }
    return nil
}

// spinUp creates a new container of pool's image. Internal; called
// by Checkout when pool needs to grow.
func (p *WarmPool) spinUp(ctx context.Context) (*Container, error) {
    c, err := newContainer(ctx, p.image, p.log)
    if err != nil {
        p.mu.Lock()
        p.size--  // decrement; spin-up failed
        p.mu.Unlock()
        return nil, fmt.Errorf("warmpool: spin up: %w", err)
    }
    return c, nil
}

// NoCleanup is a CleanupFunc that does nothing. Use for stateless
// tools (e.g., Nmap) where no per-scan state cleanup is needed.
// Explicit no-op rather than nil function makes statelessness
// architecturally explicit.
func NoCleanup(ctx context.Context, c *Container) error { return nil }
```

</details>

**Critical design decisions captured (validated post-implementation):**
- **Lazy spin-up via `Checkout`** (Decision 6: lazy warm) — preserved as designed
- **Cleanup hook between checkouts** (Decision 7: cleanup-with-hook); explicit `NoCleanup` for stateless tools rather than nil function — preserved as designed
- **Concurrency-safe** via channel for available pool + mutex for size accounting — preserved as designed; concurrency invariants tightened during implementation (see numbered list above)
- **Graceful blocking** when at max size; ctx-respect timeout — extended during implementation to also observe `done` shutdown signal
- **Cleanup-failure handling**: container stopped, size decremented, future spin-up unblocked — preserved as designed
- **Health check optional**: stateless tools skip; complex tools opt in — preserved; replacement made size-neutral on success during implementation

### 3.3 Container Type (Component 3)

**File:** `internal/tools/docker/container.go` (new)

```go
package docker

import (
    "context"

    "github.com/docker/docker/api/types"
    "github.com/docker/docker/client"
    "github.com/rs/zerolog"
)

// Container wraps a running Docker container with lifecycle
// management. Created by WarmPool's spinUp; returned via Checkout;
// stopped via Stop or returned to pool via WarmPool.Return.
//
// Per ADR-026: framework abstraction over Docker SDK. Tool runners
// don't directly use Docker SDK; they use Container's Exec method.
type Container struct {
    ID    string
    Image string
    log   zerolog.Logger
    cli   *client.Client
}

// Exec runs a command inside the container; returns stdout + exit
// code + error. Equivalent to `docker exec <id> <cmd...>`.
//
// Tool runners use Container.Exec to invoke the tool's CLI inside
// the warmed container. Stdin/stdout/stderr handling matches existing
// NativeRunner semantics where applicable.
func (c *Container) Exec(ctx context.Context, cmd []string) (stdout []byte, exitCode int, err error) {
    // Implementation: docker.client.ContainerExecCreate +
    // ContainerExecAttach + drain stdout
    // Details deferred to implementation phase; surface deviations
}

// Stop kills the container + removes it. Used on cleanup failure
// or pool shutdown.
func (c *Container) Stop(ctx context.Context) error {
    // Implementation: docker.client.ContainerKill + ContainerRemove
}

// newContainer creates a new long-running container of the given
// image. Used by WarmPool's spinUp.
func newContainer(ctx context.Context, image string, log zerolog.Logger) (*Container, error) {
    // Implementation: docker.client.ImagePull (if needed) +
    // ContainerCreate (long-running with `tail -f /dev/null`-style
    // entrypoint to keep alive) + ContainerStart
}
```

**Critical design decisions captured:**
- **Container is long-running**: created with sleep-equivalent entrypoint to stay alive; tool execution via `Exec`
- **Docker SDK abstraction**: tool runners don't import Docker SDK directly
- **Lifecycle methods**: Exec (during scan), Stop (on cleanup failure or shutdown)

### 3.4 DockerRunner Framework Type (Component 2)

**File:** `internal/tools/docker/dockerrunner.go` (new)

```go
package docker

import (
    "context"
    "fmt"
    "time"

    "github.com/odyssey-tech/shieldscan-engine/internal/events"
    "github.com/odyssey-tech/shieldscan-engine/internal/tools"
    "github.com/rs/zerolog"
)

// DockerRunner is the framework type for CLI-shaped Docker tools
// (Trivy, Nmap, SQLMap; future M7+ CLI Docker tools). Wraps
// WarmPool + provides ToolRunner-shaped interface.
//
// Per ADR-026 + ADR-006: this is the CLI-tool framework. ZAP + MobSF
// use DockerServiceRunner (separate type at internal/tools/docker/
// docker_service.go; Task 7.5b) for HTTP-shaped persistent services.
//
// Symmetric with tools.NativeRunner (M5.2 + ADR-023): per-tool config
// + parser; framework lifecycle + observability.
type DockerRunner struct {
    ToolName        string
    ToolCategory    string
    Pool            *WarmPool

    // BuildArgs constructs the CLI command for the tool, given
    // target + scan config. Returns argv slice to pass to
    // Container.Exec.
    BuildArgs func(target tools.Target, cfg tools.ScanConfig) []string

    // ParseOutput parses the tool's stdout into RawFinding structs.
    // Identical signature to tools.NativeRunner.ParseOutput.
    ParseOutput func(stdout []byte) ([]events.RawFinding, error)

    // ExitCodeLenient: same semantics as NativeRunner.
    // If true, non-zero exit codes are tolerated (tool may exit non-
    // zero on findings present).
    ExitCodeLenient bool

    // Timeout for tool execution within container.
    Timeout time.Duration

    Log zerolog.Logger
}

// Run implements the ToolRunner contract. Checks out a warmed
// container; executes tool; parses output; returns findings + cleans
// up container.
//
// Per ADR-013 sole-writer atomicity: findings are returned to caller
// (not persisted by runner); caller (worker.Processor) handles
// persistence.
func (r *DockerRunner) Run(ctx context.Context, target tools.Target, cfg tools.ScanConfig) ([]events.RawFinding, error) {
    // 1. Checkout container from pool
    runCtx, cancel := context.WithTimeout(ctx, r.Timeout)
    defer cancel()
    c, err := r.Pool.Checkout(runCtx)
    if err != nil {
        return nil, fmt.Errorf("%s: pool checkout: %w", r.ToolName, err)
    }

    // 2. Exec tool inside container
    args := r.BuildArgs(target, cfg)
    stdout, exitCode, execErr := c.Exec(runCtx, args)

    // 3. Always return container to pool (cleanup runs between
    // checkouts; failures handled by pool internally)
    if returnErr := r.Pool.Return(runCtx, c); returnErr != nil {
        r.Log.Warn().Err(returnErr).Str("tool", r.ToolName).Msg("pool return failed")
    }

    // 4. Handle exec failure
    if execErr != nil && !r.ExitCodeLenient {
        return nil, fmt.Errorf("%s: exec: %w", r.ToolName, execErr)
    }
    if !r.ExitCodeLenient && exitCode != 0 {
        return nil, fmt.Errorf("%s: non-zero exit code %d", r.ToolName, exitCode)
    }

    // 5. Parse output
    findings, parseErr := r.ParseOutput(stdout)
    if parseErr != nil {
        return nil, fmt.Errorf("%s: parse: %w", r.ToolName, parseErr)
    }

    return findings, nil
}

// Name + Category implement ToolRunner identity (matches NativeRunner
// pattern).
func (r *DockerRunner) Name() string     { return r.ToolName }
func (r *DockerRunner) Category() string { return r.ToolCategory }
```

**Critical design decisions captured:**
- **ToolRunner contract honored**: Run() returns []RawFinding + error;
  Name() + Category() match identity pattern
- **Sole-writer atomicity**: findings returned to caller; runner doesn't
  persist (per ADR-013)
- **Container always returned to pool**: defer-shaped logic ensures
  cleanup even on exec failure
- **ExitCodeLenient inheritance**: matches NativeRunner semantics
- **Per-tool config**: BuildArgs + ParseOutput closures; tool packages
  provide minimal config

### 3.5 Per-Tool Consumer Shape (Future Tasks)

Each CLI Docker tool task (Trivy at Task 7.1; Nmap at Task 7.2; SQLMap
at Task 7.6 — exact ordering set by per-task brainstorm) constructs
a DockerRunner with tool-specific config:

```go
// Example shape for Trivy (Task 7.1; not in scope here)
package trivy

import (
    "github.com/odyssey-tech/shieldscan-engine/internal/tools/docker"
    "github.com/rs/zerolog"
)

type Config struct {
    Image string  // SHIELDSCAN_TRIVY_IMAGE env or default
}

func NewTrivyRunner(cfg Config, log zerolog.Logger) (*docker.DockerRunner, error) {
    pool, err := docker.New(docker.Config{
        Image:   cfg.Image,
        MaxSize: 4,
        Cleanup: trivyCleanup,  // tool-defined
    }, log)
    if err != nil {
        return nil, err
    }
    return &docker.DockerRunner{
        ToolName:     "trivy",
        ToolCategory: "sca",
        Pool:         pool,
        BuildArgs:    trivyBuildArgs,
        ParseOutput:  trivyParseOutput,
        Timeout:      10 * time.Minute,
        Log:          log,
    }, nil
}
```

Per-tool boilerplate is ~10-15 LoC (config struct + constructor +
cleanup func + buildArgs func + parseOutput func + tests). Compared
to Approach A's ~30-50 LoC per tool, the framework saves ~75-105 LoC
across 3 CLI tools.

---

## 4. Test Strategy

### 4.1 Test Categories

**WarmPool unit tests** (`warmpool_test.go`):
- `TestWarmPool_New_RejectsEmptyImage`
- `TestWarmPool_New_RejectsNilCleanup` (NoCleanup must be explicit)
- `TestWarmPool_New_DefaultMaxSize` (zero MaxSize → default 4)
- `TestWarmPool_Checkout_LazySpinUpOnFirstCall`
- `TestWarmPool_Checkout_ReusesWarmedContainer`
- `TestWarmPool_Checkout_BlocksAtMaxSize` (max=2; 3 concurrent checkouts; 3rd blocks)
- `TestWarmPool_Checkout_RespectsContextCancellation` (blocked checkout returns ctx error)
- `TestWarmPool_Return_RunsCleanupHook`
- `TestWarmPool_Return_StopsContainerOnCleanupFailure`
- `TestWarmPool_Return_DecrementsSizeOnCleanupFailure` (allows future spin-up)
- `TestWarmPool_HealthCheck_ReplacesUnhealthyContainer`
- `TestWarmPool_HealthCheck_NilSkipsCheck`
- `TestWarmPool_Shutdown_StopsAllContainers`
- `TestWarmPool_Concurrent_NoRaceCondition` (10 concurrent checkout/return cycles; no data race)

**Container unit tests** (`container_test.go`):
- `TestContainer_Exec_CapturesStdout`
- `TestContainer_Exec_CapturesExitCode`
- `TestContainer_Exec_RespectsContextCancellation`
- `TestContainer_Stop_RemovesContainer`
- `TestContainer_NewContainer_PullsImageIfMissing`

**DockerRunner unit tests** (`dockerrunner_test.go`):
- `TestDockerRunner_Run_ChecksOutContainer`
- `TestDockerRunner_Run_ReturnsContainerOnSuccess`
- `TestDockerRunner_Run_ReturnsContainerOnExecFailure` (defer semantics)
- `TestDockerRunner_Run_PassesArgsToExec`
- `TestDockerRunner_Run_ParsesOutputCorrectly`
- `TestDockerRunner_Run_RespectsExitCodeLenient` (lenient: tolerates non-zero; strict: errors)
- `TestDockerRunner_Run_ContextTimeout`
- `TestDockerRunner_Name_ReturnsToolName`
- `TestDockerRunner_Category_ReturnsToolCategory`

**Total target: ~28 tests** across 3 files. Mock-based for unit tests
(stub Docker SDK responses; no actual Docker daemon); per-tool integration
tests live in tool tasks (not this framework task).

### 4.2 Mock Strategy

Docker SDK calls are mocked via interface abstraction. The Container
type's Docker SDK client (`*client.Client`) is replaced in tests with
a mock implementing the relevant subset of Docker API methods
(ContainerCreate, ContainerStart, ContainerExecCreate, etc.).

**Why mock-only:** unit tests run without Docker daemon; CI portability;
fast test execution.

**Integration tests:** deferred to per-tool tasks (7.1, 7.2, 7.6).
Each tool task includes one integration test exercising real Docker
daemon + actual tool image.

### 4.3 Concurrency Test

`TestWarmPool_Concurrent_NoRaceCondition` runs `go test -race`. 10
goroutines × 100 checkout/return cycles × 100ms simulated work; no
race conditions detected; all containers eventually returned cleanly.

---

## 5. Implementation Plan

### 5.1 Single Atomic Commit (Approach C)

Per established M6 single-task pattern + SPEC §7.3 Phase 3 precedent:
single atomic commit covering framework type + warm pool primitive +
Container abstraction + tests + DRIFT entries.

**Files:**
```
NEW:
  internal/tools/docker/warmpool.go         (~250 LoC; lazy warm pool)
  internal/tools/docker/container.go        (~150 LoC; Docker SDK wrapper)
  internal/tools/docker/dockerrunner.go     (~120 LoC; framework type)
  internal/tools/docker/warmpool_test.go    (~400 LoC; ~14 tests)
  internal/tools/docker/container_test.go   (~150 LoC; ~5 tests)
  internal/tools/docker/dockerrunner_test.go (~250 LoC; ~9 tests)
  internal/tools/docker/mocks_test.go       (~100 LoC; Docker SDK mocks)

EDIT:
  go.mod                                    (add Docker SDK dependency)
  go.sum                                    (regenerated)
  DRIFT-LOG.md                              (~6 entries)
```

**Estimated total LoC:** ~1500 lines (src + tests).

### 5.2 Phase Decomposition

Per established M6 phase pattern:

**Phase 0: Pre-implementation verification (~30 min)**
- Verify Docker SDK Go module identifier + version pin
  (`github.com/docker/docker` vs alternatives)
- Verify go.mod compatibility (Go version requirements for Docker SDK)
- Verify existing `internal/tools/` package layout for placement
  (`internal/tools/docker/` vs alternative)
- Verify Docker daemon access from worker process (socket path,
  permissions)
- Surface findings before Phase 1

**Phase 1: Container abstraction (~1.5h)**
- Create `internal/tools/docker/container.go`
- Implement Container type + newContainer + Exec + Stop
- Mock Docker SDK responses
- Unit tests for Container
- Run tests in isolation; green

**Phase 2: WarmPool primitive (~2-3h)**
- Create `internal/tools/docker/warmpool.go`
- Implement WarmPool + Config + Checkout + Return + Shutdown +
  spinUp + NoCleanup
- Channel-based available pool + mutex-protected size accounting
- Unit tests for WarmPool (including concurrent test)
- Run tests in isolation with -race; green

**Phase 3: DockerRunner framework type (~1.5h)**
- Create `internal/tools/docker/dockerrunner.go`
- Implement DockerRunner + Run + Name + Category
- Unit tests for DockerRunner
- Run tests in isolation; green

**Phase 4: Quality gate + DRIFT-LOG + commit (~30 min)**
- DRIFT-LOG entries (~6 entries)
- gofmt -w + go vet ./... + golangci-lint run + go test -race
  -count=1 ./... (full suite green)
- Single atomic commit

### 5.3 Effort Summary

| Phase | Effort |
|-------|--------|
| Phase 0 verification | ~30 min |
| Phase 1 Container abstraction + tests | ~1.5h |
| Phase 2 WarmPool primitive + tests | ~2-3h |
| Phase 3 DockerRunner + tests | ~1.5h |
| Phase 4 DRIFT-LOG + commit | ~30 min |
| **Total** | **~6-8h** |

Within brainstorming bandwidth confirmation of 6-9h.

---

## 6. ADR-026 — Canonical Reference

The ADR-026 final text lives in `SPECIFICATION.md` §13 (landed 2026-05-04, commit 7ddace4 in shieldscan-docs). This design doc draft preceded the final ADR; the SPEC version is canonical and supersedes any draft in this section.

**Brief summary of the ADR-026 decision:**

Two M7 framework types — DockerRunner (warm-pool wrapper for CLI-shaped Docker tools; consumers Trivy, Nmap, SQLMap) and DockerServiceRunner (HTTP-API wrapper for persistent Docker services; consumers ZAP, MobSF). Resolves the Option β decision locked in brainstorming: ADR-006's "persistent Docker services for heavy tools (ZAP, MobSF, Trivy, SQLMap)" enumeration is refined — Trivy + SQLMap reclassify to warm-pool CLI framework; persistent service framework scoped to ZAP + MobSF only.

Key architectural commitments documented in SPEC §13 ADR-026:
- Lazy warm pool semantics (zero resource floor for unused tools)
- Cleanup-between-checkouts contract for tenant isolation
- Done-channel signal pattern (not channel-close-as-signal)
- Size-neutral health-check replacement
- Cleanup uses `context.WithoutCancel` (cleanup-uses-parent-context anti-pattern explicitly avoided)
- ToolRunner interface symmetry (NativeRunner / DockerRunner / DockerServiceRunner all satisfy ToolRunner)
- 3-tier timeout precedence symmetric with NativeRunner
- Compile-time interface assertion as regression guard

For full ADR text including Context / Decision / Rationale / Consequences / Anti-patterns / Triggers / Forcing functions / Open follow-ups / Cross-references, see SPECIFICATION.md §13.

**Design doc → SPEC translation notes:**

This design doc was drafted pre-implementation (2026-05-03) and contained verbatim Go code that subsequent implementation (Phases 2-3 of Task 7.5a) caught as concurrency-incorrect. Specifically, the original §3.2 in this document had:
- Shutdown closing the available channel (race with Return)
- Blocking Checkout missing shutdown-signal observation (nil hazard)
- Health-check replacement size accounting leak

§3.2 has been corrected post-implementation to reflect the f5d77c8 actual code shape and concurrency invariants. SPEC §13 ADR-026 documents these invariants as anti-patterns explicitly prevented. See `shieldscan-engine/DRIFT-LOG.md` entry 5 (Task 7.5a entries; 15 deviations enumerated) for the full pre-implementation-vs-implementation deviation history.

<details>
<summary>Original pre-implementation ADR-026 draft (preserved for traceability — DO NOT use as reference; SPEC §13 is canonical)</summary>

```markdown
### ADR-026: DockerRunner framework + lazy warm pool — M7 container lifecycle architecture
**Status:** Accepted (2026-05-XX, Task 7.5a)

**Context.**
M7 introduces Docker tools to ShieldScan's tool surface. Per ADR-006
(Hybrid Native + Persistent Docker, refined 2026-04-18), heavy tools
deploy as persistent Docker services to eliminate 2-3s per-scan
container startup latency.

ADR-006 + ADR-008 specifies persistent services for ZAP + MobSF
(HTTP/REST API consumers). The architecture diagram (SPEC §3) lists
both `warm_pool.go` (line 248) and `docker_service.go` (line 229) as
distinct modules.

**Two-runner-type architecture is explicit project commitment:**
- DockerRunner + warm pool: CLI-shaped tools (Trivy, Nmap, SQLMap)
- DockerServiceRunner: HTTP-shaped persistent services (ZAP, MobSF)

This ADR addresses the warm pool primitive + DockerRunner framework
(CLI tools). DockerServiceRunner is Task 7.5b; separate ADR.

IMPLEMENTATION-PLAN Task 7.5 specifies the warm pool primitive at
`shieldscan-engine/internal/tools/docker/warm_pool.go`. ADR-026
delivers it.

**Decision.**
Implement warm pool primitive + DockerRunner framework with five
architectural commitments:

1. **Lazy warm pool semantics** (`internal/tools/docker/warmpool.go`):
   pool starts empty; first checkout triggers spin-up; subsequent
   checkouts reuse warmed containers; max-bound prevents runaway.
   Resource floor = zero for unused tools.

2. **Cleanup hook between checkouts**: per-tool CleanupFunc runs on
   container return-to-pool. Tenant isolation guaranteed via cleanup;
   no state leak between scans. Stateless tools must use explicit
   `NoCleanup` (no nil functions) to make statelessness architecturally
   visible.

3. **Container abstraction** (`internal/tools/docker/container.go`):
   wraps Docker SDK; provides `Exec`, `Stop`, `newContainer`. Tool
   runners don't import Docker SDK directly; they consume Container.

4. **DockerRunner framework type** (`internal/tools/docker/dockerrunner.go`):
   wraps WarmPool + provides ToolRunner-shaped interface (Run, Name,
   Category). Per-tool config: image, BuildArgs, ParseOutput,
   ExitCodeLenient, Timeout. Symmetric with tools.NativeRunner pattern
   (M5.2 + ADR-023).

5. **Sole-writer atomicity preserved** (per ADR-013): findings
   returned to caller; runner doesn't persist. Worker.Processor
   handles persistence per ADR-017 sequencing contract.

**Rationale.**

Three implementation alternatives considered and rejected:

| Alternative | Why rejected |
|---|---|
| **Approach A: per-tool pool orchestration in each tool package** | ~30-50 LoC duplicated across N tool packages; scattered observability; cross-tool resource limits not natively supported. Framework type centralizes ~150-250 LoC. |
| **Approach C: two-layer (WarmPool primitive + DockerRunner wrapper + escape hatch)** | Premature; YAGNI on escape hatch until concrete second use case surfaces. M9 connection pooling (theoretical) is the only candidate for direct WarmPool consumption; not warranted now. |
| **Per-task containers (no warm pool)** | Defeats ADR-006 architectural commitment. 2-3s cold-start per scan × concurrent scans = customer-facing latency degradation. Lazy warm has zero resource floor for unused tools while delivering warm-reuse benefit. |
| **Always-warm fixed pool** | Wastes resources on tools never used in a given deployment. Customer running only SAST scans pays zero cost for Trivy/Nmap/SQLMap pools under lazy semantics. |

**Cross-reference: asymmetric-cost meta-principle (5th invocation in
project corpus).** Following ADR-022 (recon-as-pre-scan-helpers),
ADR-023 (NativeRunner OutputFile mode), ADR-024 (RawFinding schema
extension), ADR-025 (findings-ingest path), this is the 5th ADR
invoking asymmetric-cost reasoning to justify architectural commitment.

Pattern shape: framework cost (~6-8h single-commit work for 3 CLI
tool consumers) is asymmetrically smaller than alternative costs:
- Approach A's ~150-250 LoC duplication across 3 tools + future
  M7+ Docker CLI tools
- Per-task containers' 2-3s × scans × N concurrent operations
- Always-warm fixed pool's resource waste

**Promotion candidate:** asymmetric-cost meta-principle is now at 5
ADR instances; well above 3-instance threshold. DEVELOPMENT-PATTERNS
entry codifying this meta-principle is appropriate at next
architectural decision-point.

**ADR-006 alignment.**
ADR-006 (Hybrid Native + Persistent Docker) specifies persistent
Docker services for heavy tools. ADR-026 honors ADR-006 by adopting
warm pool semantics for CLI tools (avoiding 2-3s cold-start latency)
while leaving DockerServiceRunner for HTTP-shaped persistent services
(ZAP, MobSF) per ADR-006's "persistent service" framing.

The two abstractions coexist:
- DockerRunner: CLI tools that exec into warmed containers
- DockerServiceRunner: HTTP-shaped tools that talk to running services

Both honor ADR-006's "eliminate per-scan startup latency" intent via
distinct architectural shapes.

**Forcing functions.**

- Empirical fallback to hybrid tier acknowledged: if MobSF (DockerServiceRunner
  consumer) or any future cleanup-hook consumer surfaces unreliability,
  fall back to hybrid tiers (some tools reuse-blindly, some get cleanup,
  some get tear-down-replace). Pattern: factual deviation auto-correct
  + document.
- Concurrency test (`TestWarmPool_Concurrent_NoRaceCondition`) is
  load-bearing: future engineers tempted to "simplify" pool logic
  trip the race detector immediately.
- `NoCleanup` explicit no-op rather than nil function: future engineers
  reading tool packages see explicit cleanup contract for every tool;
  statelessness is architecturally visible.

**Consequences.**

Positive:
- Future M7+ Docker CLI tools inherit DockerRunner cleanly: ~10-15 LoC
  per tool vs ~30-50 LoC under Approach A.
- Resource floor zero for unused tools (lazy semantics).
- Tenant isolation guaranteed via cleanup hook contract.
- Centralized observability: pool metrics in framework, not scattered.
- Symmetric with tools.NativeRunner pattern; familiar shape for
  engineers maintaining the codebase.
- ADR-006 architectural commitment honored cleanly.

Negative:
- **Two-runner-type cognitive load.** DockerRunner (CLI) vs
  DockerServiceRunner (HTTP) requires engineers to classify each new
  Docker tool. Mitigation: ADR-026 + ADR-006 provide explicit
  classification criteria; per-tool brainstorming clarifies at task
  time.
- **Docker SDK dependency added.** New external dependency; version
  pin requires VERSIONS.md update + ongoing security maintenance.
- **First-scan latency.** Lazy warm means first scan of each tool pays
  cold-start cost (~2-3s for Trivy; ~5-15s for SQLMap). Acceptable
  trade-off vs always-warm resource cost; documented in operator
  guide.
- **Cleanup hook required per tool.** Stateless tools (Nmap) must
  explicitly use NoCleanup. Slight ergonomic cost; benefit is
  architectural visibility.

**Alternatives considered (and rejected).** See Rationale table above.

**Anti-patterns this prevents.**
- Per-tool pool orchestration scattered across N packages (Approach A):
  Approach B centralizes lifecycle in framework.
- Per-scan containers defeating ADR-006 cold-start intent: warm pool
  is the canonical M7 lifecycle.
- Always-warm fixed pool wasting resources for unused tools: lazy
  semantics opt in per use.
- Nil cleanup functions hiding statelessness: NoCleanup is explicit.
- Tool packages importing Docker SDK directly: Container abstraction
  keeps SDK-specific knowledge in framework.

**Triggers to revisit.**

1. **MobSF (or future cleanup-hook consumer) surfaces unreliability.**
   Empirical fallback to hybrid tiers (per Decision 7 brainstorming
   acknowledgment).
2. **Approach C escape hatch becomes warranted.** If concrete second
   use case for WarmPool (M9 connection pooling, M11 resource pool)
   surfaces, expose WarmPool as primitive directly + keep DockerRunner
   as wrapper.
3. **Always-warm operational signal.** If first-scan latency becomes
   customer-facing concern (e.g., demo scenarios where cold-start is
   visible), introduce always-warm fixed pool as opt-in `WarmPoolConfig.PreWarm`
   field.
4. **Cross-tool resource limits.** If global Docker container count
   becomes operational constraint (e.g., shared deployment with strict
   container quotas), centralize across-pool limit in framework via
   shared semaphore or per-deployment resource manager.
5. **Docker SDK breaking change.** Pin version in VERSIONS.md; track
   upstream releases; revisit on major version change.

**Open follow-ups.**
- DockerServiceRunner (Task 7.5b; separate ADR; HTTP-shaped tool
  framework).
- Per-tool brainstorming + scope proposal cycles for Trivy (Task 7.1),
  Nmap (Task 7.2), SQLMap (Task 7.6).
- Asymmetric-cost meta-principle promotion to DEVELOPMENT-PATTERNS
  entry (5+ instances; well above threshold).
- Empirical re-evaluation discipline pattern promotion (3+ instances:
  M6.6 Nikto stdout, M6.3 httpx stdin, ADR-026 hybrid-tier fallback
  acknowledgment).

**Cross-references.**
- ADR-006: Hybrid Native + Persistent Docker (load-bearing
  architectural commitment honored).
- ADR-008: MobSF as persistent service (DockerServiceRunner
  consumer; not in this ADR's scope).
- ADR-013: Python sole writer (atomicity preserved; runner returns
  findings to caller).
- ADR-017: Findings inline in job_completed events with sequencing
  (per-runner findings flow into the same pipeline).
- ADR-023: NativeRunner OutputFile mode (framework precedent;
  Approach B pattern).
- ADR-024 + ADR-025: schema extension + findings-ingest (the
  consumer-side pipeline that DockerRunner findings flow into).
- IMPLEMENTATION-PLAN Task 7.5: warm pool primitive obligation
  (this ADR delivers).
- SPECIFICATION §3 architecture diagram (lines 229 + 248): both
  runner types explicitly listed.
```

</details>

---

## 7. DRIFT-LOG Entries Plan (Engine-side; ~6 entries)

1. **Task 7.5a closes** — warm pool primitive + DockerRunner framework
   landed; Task 7.5b (DockerServiceRunner) is separate task; M7 task
   structure now 7 tasks total per Option β resolution.

2. **ADR-026 lands** — 5th asymmetric-cost ADR in project corpus;
   meta-principle well above promotion threshold; DEVELOPMENT-PATTERNS
   entry candidate at next architectural decision-point.

3. **Two-runner-type architecture confirmed** — DockerRunner (CLI;
   warm pool) vs DockerServiceRunner (HTTP; ADR-006 + ADR-008
   persistent services). Architecture diagram (SPEC §3) literal
   honored.

4. **Lazy warm semantics** — pool starts empty; first checkout
   triggers spin-up; max-bound prevents runaway. Resource floor zero
   for unused tools.

5. **Cleanup hook contract** — per-tool CleanupFunc; explicit NoCleanup
   required for stateless tools (no nil functions); statelessness is
   architecturally visible.

6. **Empirical fallback to hybrid tiers acknowledged** — if MobSF or
   future consumer surfaces cleanup unreliability, fall back to
   tiered cleanup (some reuse blindly, some hook, some tear-down-
   replace). Discipline pattern: factual deviation auto-correct +
   document.

---

## 8. Open Questions / Triggers

### 8.1 Pre-Implementation Verification (Phase 0)

1. **Docker SDK Go module identifier + version pin.** The Docker
   SDK has multiple Go module paths; verify the canonical one
   (`github.com/docker/docker` is most likely; confirm).

2. **Go module compatibility.** Current go.mod targets Go 1.22+ (per
   M5/M6 work); verify Docker SDK supports this.

3. **Docker daemon socket access from worker process.** Worker runs
   as non-root user; needs access to `/var/run/docker.sock` or
   equivalent. Surface OPS milestone concern if worker-process Docker
   access requires socket mount or privileged execution.

4. **Existing internal/tools/ package layout.** Verify placement —
   `internal/tools/docker/` vs `internal/tools/dockerrunner/` vs
   alternative. Match project conventions.

### 8.2 Future Triggers (Carried Forward)

1. **MobSF cleanup-hook unreliability** → empirical fallback to
   hybrid tiers (per Decision 7).

2. **Concrete second WarmPool use case** (M9 connection pool, M11
   resource pool, etc.) → expose WarmPool primitive directly +
   keep DockerRunner as wrapper (Approach C migration).

3. **First-scan latency customer-facing concern** → opt-in
   `PreWarm` field on WarmPoolConfig.

4. **Cross-tool resource limits operational signal** → centralized
   container count limit via framework-level semaphore.

5. **Docker SDK breaking change** → version pin upgrade + test
   regression.

6. **Asymmetric-cost meta-principle promotion** to
   DEVELOPMENT-PATTERNS entry: opportunistic at next architectural
   decision-point that warrants framing the meta-principle explicitly.

---

## 9. Brainstorming Process Acknowledgment

This design emerged from a structured brainstorming session that:
1. Locked 6 architectural decisions through 6 clarifying questions
2. Surfaced ADR-006 conflict during pre-design SPEC scan
3. Resolved conflict via Option β (warm pool for CLI tools only;
   DockerServiceRunner for HTTP-shaped services)
4. Locked 3 additional design-confirmation decisions (Approach B;
   ADR-026 title (a); Task 7.5a framework-only scope; MobSF-deferral)

**The ADR-006 conflict is significant.** Pre-implementation guidance
(brainstorming) initially anticipated all 5 M7 tools using warm pool;
SPEC reading caught existing architectural commitment that contradicts.
The discipline pattern (verify before producing artifact) caught the
gap before design doc landed; Option β resolution preserves
brainstorming output for warm pool primitive while honoring ADR-006.

**This is the 4th task in the project corpus that brainstorming
produced a comprehensive design doc for** (after M5 close retrospective,
SPEC §7.3 followup, findings-ingest task, [warm pool primitive as 4th]).
The discipline pattern is mature; design doc quality benefits from
SPEC scan before drafting.

ADR-026 is the 5th project-corpus ADR invoking asymmetric-cost
meta-principle. The pattern is now well-established norm; future ADRs
inherit the cross-reference shape; DEVELOPMENT-PATTERNS entry is
warranted.

---

**Status:** Ready for implementation per writing-plans skill workflow.

**Approval required:** None outstanding; brainstorming locked all
decisions; ADR-006 conflict resolved via Option β.

**Next action:** Mahmoud copies design doc to
`shieldscan-docs/docs/plans/2026-05-03-warm-pool-primitive-design.md`;
then proceeds to Phase 0 verification (Task 7.5a implementation) when
bandwidth permits.

---

**Post-implementation acknowledgment (2026-05-04).** The brainstorming → design → implementation chain for Task 7.5a surfaced 15 factual deviations across Phases 0-3 (3+4+3+5; documented in `shieldscan-engine/DRIFT-LOG.md` entry 5 of the f5d77c8 commit batch). Three of these — concurrency bugs in the original §3.2 WarmPool verbatim — were architecturally consequential: channel-close-as-signal pattern, blocking Checkout silent-nil hazard, and decrement-without-increment size accounting leak. All three were caught at design-spec-reading time during Phase 2 implementation, before any code committed.

This is the discipline pattern working at the layer it matters most. Pre-implementation guidance is unreliable specifically because edge-case interleavings and architectural-symmetry concerns surface during implementation; the verify-then-adapt-then-document workflow catches them transparently. Future engineers reading this design doc see corrections-applied-post-implementation rather than original-buggy-text-preserved.
