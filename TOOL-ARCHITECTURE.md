# ShieldScan — Tool Architecture

**Version:** 1.0
**Date:** 2026-04-18
**Companion to:** `SPECIFICATION.md`
**Audience:** Implementation engineers working on the scan engine (Go workers + orchestration)

> **For Claude Code:** This document defines how every scan tool is integrated, invoked, and how findings are normalized. Read before touching anything in `shieldscan-engine/internal/tools/` or `shieldscan-engine/internal/orchestrator/`.

---

## Table of Contents

1. [Architectural Overview](#1-architectural-overview)
2. [Deployment Model](#2-deployment-model)
3. [The ToolRunner Interface](#3-the-toolrunner-interface)
4. [RawFinding Normalized Struct](#4-rawfinding-normalized-struct)
5. [Complete Tool Inventory](#5-complete-tool-inventory)
6. [Native Tool Specifications](#6-native-tool-specifications)
7. [Persistent Docker Service Specifications](#7-persistent-docker-service-specifications)
8. [Recon-First Pipeline](#8-recon-first-pipeline)
9. [Mobile Security Flow (MobSF)](#9-mobile-security-flow-mobsf)
10. [Scan Type → Tool Matrix](#10-scan-type--tool-matrix)
11. [Worker Startup & Health Checks](#11-worker-startup--health-checks)
12. [Tool Execution Orchestration](#12-tool-execution-orchestration)
13. [Error Handling & Retry Strategy](#13-error-handling--retry-strategy)
14. [Adding a New Tool — Developer Guide](#14-adding-a-new-tool--developer-guide)

---

## 1. Architectural Overview

### 1.1 The Core Problem

ShieldScan integrates **19 security tools** across **9 scan categories**. These tools have vastly different:
- **Invocation models** — some are CLI binaries, some are long-running HTTP services
- **Output formats** — JSON, XML, SARIF, free-form text, proprietary formats
- **Resource profiles** — MobSF needs 2GB RAM; SSLyze runs in 100MB
- **Startup characteristics** — Nuclei starts in 50ms; ZAP takes 8 seconds
- **Lifetime models** — per-scan ephemeral vs. always-on service

The architecture must absorb all this complexity behind a single interface so that the scan orchestrator treats every tool identically.

### 1.2 Design Principles

1. **Uniform interface:** Every tool implements the same `ToolRunner` interface regardless of how it runs.
2. **Normalized output:** Every tool produces the same `RawFinding` struct. No tool-specific downstream logic.
3. **Hybrid deployment:** Native binaries for lightweight tools, persistent Docker services for heavy tools.
4. **Fingerprint-based dedup:** Every finding gets a deterministic fingerprint before AI pipeline.
5. **Recon-first:** Subdomain discovery runs before every web scan, feeding discovered hosts into all engines.
6. **Fail-open:** Tool failures don't block the scan — other tools continue, degraded findings reported.
7. **Health-aware:** Workers self-report tool health; unhealthy workers don't receive jobs.

### 1.3 High-Level Flow

```
┌─────────────────────────────────────────────────────────────┐
│  Scan Job Arrives (Redis Queue → Go Worker)                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  Orchestrator determines tools for scan_type                │
│  (tool_router.go — see Section 10)                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  IF web scan: Recon runs first (Subfinder → httpx)          │
│  Discovered subdomains added to target list                 │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  For each (tool × target) pair:                             │
│    tool.Run(ctx, target, config) → []RawFinding             │
│    Findings published to Redis as discovered                │
│    Progress events streamed via pub/sub                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ↓
┌─────────────────────────────────────────────────────────────┐
│  Raw findings stored in PostgreSQL                          │
│  AllScanJobsCompleted event → AI pipeline triggered         │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Deployment Model

### 2.1 Hybrid Deployment — Why Two Models

| Tool profile | Deployment | Example | Rationale |
|---|---|---|---|
| CLI binary, fast startup (< 100ms), low memory | **Native** | Nuclei, Semgrep, SSLyze, Subfinder | Zero overhead, direct subprocess call |
| HTTP service, slow startup (> 3s), high memory | **Persistent Docker** | MobSF, ZAP, Trivy, SQLMap | Start once on worker boot, reuse indefinitely |

### 2.2 Layer 1 — Native Binaries (11 tools)

Installed directly on worker OS via `provision-worker.sh`. Pinned versions. Located at `/usr/local/bin/`.

```
/usr/local/bin/
├── nuclei            v3.2.0
├── semgrep           v1.60.0
├── subfinder         v2.6.3
├── httpx             v1.6.0
├── gitleaks          v8.18.0
├── sslyze            v6.0.0
├── nikto             v2.5.0
├── wapiti            v3.2.0
├── corstest          (git commit pinned)
├── dependency-check  v9.0.0
└── checkov           v3.2.0
```

**Invocation:** Go worker calls `exec.CommandContext(ctx, binary, args...)`. Output parsed via tool-specific parser.

### 2.3 Layer 2 — Persistent Docker Services (4 tools)

Started once when worker boots via `docker-compose up -d`. Always-on HTTP endpoints. Reused across all scans.

| Tool | Port | Memory Limit | Startup Time | Purpose |
|---|---|---|---|---|
| **MobSF** | 8000 | 2GB | ~15s | Mobile APK/IPA/source analysis |
| **ZAP** | 8080 | 1GB | ~8s | Deep DAST active scanning |
| **Trivy** | 4954 | 512MB | ~3s | SCA + container scanning |
| **SQLMap** | 8775 | 512MB | ~2s | Deep SQL injection |

**Invocation:** Go worker makes HTTP REST API calls to `localhost:{port}`. Total persistent memory: ~4GB.

### 2.4 Layer 3 — Docker Warm Pool (1 tool)

For tools that need fresh isolation per scan but have expensive startup:

| Tool | Pool Size | Reason |
|---|---|---|
| **Nmap** | 2 pre-warmed containers | Needs isolated network namespace per scan |

Warm pool maintains 2 paused containers. On job arrival: unpause (< 200ms) instead of cold-start (~3s). Background replenishment after use.

### 2.5 Worker Server Specifications

| Resource | Minimum | Recommended |
|---|---|---|
| RAM | 8GB | 16GB |
| CPU | 4 cores | 8 cores |
| Storage | 50GB SSD | 100GB SSD |
| OS | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |
| Docker | 24.0+ | 24.0+ |

**Resource budget on 8GB worker:**
- MobSF: 2GB
- ZAP: 1GB
- Trivy + SQLMap: 1GB combined
- Nmap warm pool: 512MB
- Go worker process: 512MB
- Native tool execution (concurrent): 2GB
- OS + buffer: 1GB

Supports **5 concurrent scans** comfortably on 8GB. Mobile-only workers reserve more for MobSF.

---

## 3. The ToolRunner Interface

### 3.1 The Interface

```go
// internal/tools/runner.go
package tools

import "context"

type ToolRunner interface {
    Name() string
    Category() string  // "dast" | "sast" | "mobile" | "ssl" | "recon" | etc.
    Run(ctx context.Context, target Target, config ScanConfig) ([]RawFinding, error)
}
```

Every tool in ShieldScan implements this interface — period. The orchestrator never cares whether a tool is native or Docker, fast or slow, web or mobile. It calls `Run()`.

### 3.2 Supporting Types

```go
type Target struct {
    URL             string
    Domain          string
    TargetType      string  // "web" | "mobile" | "container" | "source"
    DomainVerified  bool
    AuthConfig      *AuthConfig  // nil if no authentication
    MobileUploadRef string       // for mobile targets: r2://... reference
    SourcePath      string       // for SAST: local git clone path
    ContainerImage  string       // for container scans: docker image tag
}

type AuthConfig struct {
    Type   string  // "cookie" | "bearer" | "basic" | "custom_header" | "form"
    Data   string  // encoded credential data (already decrypted)
    Fields map[string]string  // for form auth: field names and values
}

type ScanConfig struct {
    Depth           string            // "quick" | "standard" | "deep"
    Timeout         int               // seconds
    MaxRPS          int               // max requests per second
    TemplateCategories []string       // for Nuclei: ["owasp-top-10", "cves"]
    ExtraArgs       map[string]string // tool-specific extras
}
```

#### §3.2 Addendum: Source-Acquisition Implementation Lock (2026-05-29)

Implements the `SourcePath` design intent canonicalized above (`SourcePath string // for SAST: local git clone path`). Mechanism **(α.1) host-side `os/exec` git clone `--depth=1`** with per-scan tempdir under `$TRIVY_SCAN_BASE_PATH`; engine clones at job-pickup time (lazy per-job at worker); the existing trivy-fs ReadOnly bind mount surfaces the staging tree as `/scan/<scan-id>` inside the container.

**Validator:** `Project.source_repo_url` is HTTPS-scheme-only at v1 (mirrors `target_url` `_validate_https_url`); SSH-key + private-token auth forward-pinned per Q-AUTH.

**Failure semantics:** clone failure is hard-fail with `SCAN_FAILED` structured error (`error.code = "SOURCE_ACQUISITION_FAILED"`; `error.details = {url, git_exit_code, stderr_tail}`) per Q-FAILURE-MODE + Q-EVENTS standard-lifecycle; repairs Drift #54 silent-no-op behavior.

**Cleanup:** per-scan `defer os.RemoveAll(stagingDir)` on the engine host. The container's ReadOnly mount inside trivy-fs does not block host-side cleanup; per Q-CLEANUP.

**Drift #54 root-cause repair:** `FULL_WEB_SOURCE` / `FULL_SPECTRUM` `trivy-fs` dispatch was aspirational-broken end-to-end before this addendum + Stage 3 trio. `Project.source_repo_url` column existed since Milestone 1; orchestrator never threaded it; `JobTarget` wire had no source field; `trivy-fs` silently no-op'd on empty `SourcePath`. Stage 3 trio repairs the wire + orchestrator + engine layers; this addendum lands the canonical authority side. Drift #54 catch-class = stored-design-intent-with-unimplemented-mechanism (distinct from framing-vs-empirical drifts #44/#45/#50/#51-53).

**Scope (v1):** `trivy-fs` SCA scanner only. Future SAST tools (`gitleaks` / `semgrep` / `dependency_check` per IMPLEMENTATION-PLAN.md M6) reuse the same `internal/source/` engine primitive when wired.

**Cross-references:** source-ingestion fix design doc `plans/2026-05-29-source-ingestion-fix-design.md` + implementation plan `plans/2026-05-29-source-ingestion-fix-implementation.md`; Phase 0 v2 empirical anchors (P0v2.A-D ZERO pivot triggers; NodeGoat depth=1 clone 0.76s/3.3M/1.2M `.git`; trivy fs 75 findings against `package-lock.json`; Docker ReadOnly mount enforced — `touch /scan/...` denied); §3.2 design intent (preserved verbatim above).

### 3.3 Two Concrete Implementations

**NativeRunner** — wraps subprocess execution:
```go
type NativeRunner struct {
    ToolName     string
    ToolCategory string
    BinaryPath   string
    BuildArgs    func(target Target, cfg ScanConfig) []string
    ParseOutput  func(out []byte) ([]RawFinding, error)
    Timeout      time.Duration
}

func (n *NativeRunner) Name() string     { return n.ToolName }
func (n *NativeRunner) Category() string { return n.ToolCategory }

func (n *NativeRunner) Run(ctx context.Context, target Target, cfg ScanConfig) ([]RawFinding, error) {
    timeoutCtx, cancel := context.WithTimeout(ctx, n.Timeout)
    defer cancel()

    args := n.BuildArgs(target, cfg)
    cmd := exec.CommandContext(timeoutCtx, n.BinaryPath, args...)

    var stdout, stderr bytes.Buffer
    cmd.Stdout = &stdout
    cmd.Stderr = &stderr

    if err := cmd.Run(); err != nil {
        if errors.Is(timeoutCtx.Err(), context.DeadlineExceeded) {
            return nil, fmt.Errorf("%s timed out after %s", n.ToolName, n.Timeout)
        }
        // Some tools exit non-zero when findings exist (e.g., gitleaks)
        // Parser decides whether to treat exit code as error
    }

    findings, err := n.ParseOutput(stdout.Bytes())
    if err != nil {
        return nil, fmt.Errorf("%s parse failed: %w (stderr: %s)", n.ToolName, err, stderr.String())
    }

    // Enrich findings with tool metadata
    for i := range findings {
        findings[i].ToolName = n.ToolName
        findings[i].EngineCategory = n.ToolCategory
        findings[i].DiscoveredAt = time.Now()
        findings[i].Fingerprint = computeFingerprint(findings[i])
    }
    return findings, nil
}
```

**DockerServiceRunner** — wraps HTTP API calls:
```go
type DockerServiceRunner struct {
    ToolName     string
    ToolCategory string
    BaseURL      string
    APIKey       string
    HTTPClient   *http.Client
    Execute      func(ctx context.Context, client *http.Client, baseURL, apiKey string,
                      target Target, cfg ScanConfig) ([]RawFinding, error)
}

func (d *DockerServiceRunner) Name() string     { return d.ToolName }
func (d *DockerServiceRunner) Category() string { return d.ToolCategory }

func (d *DockerServiceRunner) Run(ctx context.Context, target Target, cfg ScanConfig) ([]RawFinding, error) {
    findings, err := d.Execute(ctx, d.HTTPClient, d.BaseURL, d.APIKey, target, cfg)
    if err != nil {
        return nil, fmt.Errorf("%s service call failed: %w", d.ToolName, err)
    }
    for i := range findings {
        findings[i].ToolName = d.ToolName
        findings[i].EngineCategory = d.ToolCategory
        findings[i].DiscoveredAt = time.Now()
        findings[i].Fingerprint = computeFingerprint(findings[i])
    }
    return findings, nil
}
```

### 3.4 Fingerprint Computation

Every finding gets a deterministic fingerprint for downstream deduplication:

```go
func computeFingerprint(f RawFinding) string {
    components := []string{
        f.ToolName,
        f.FindingType,
        f.TargetURL,
        f.Parameter,
        f.CodeFile,
        strconv.Itoa(f.CodeLine),
    }
    h := sha256.Sum256([]byte(strings.Join(components, "|")))
    return hex.EncodeToString(h[:])
}
```

This fingerprint is used by Qdrant for primary-pass deduplication. If two tools produce the same fingerprint, they're definitely the same finding. If fingerprints differ but embeddings are similar (cosine > 0.92), they're probably the same finding. See AI pipeline in Specification §8.

---

## 4. RawFinding Normalized Struct

All 19 tools produce this struct. Every field has a defined meaning. Mobile-specific fields are nullable for non-mobile tools.

```go
type RawFinding struct {
    // === Identity ===
    ToolName       string    // "nuclei" | "mobsf" | "sslyze" | "semgrep" | ...
    EngineCategory string    // "dast" | "sast" | "mobile" | "ssl" | "recon" |
                             // "infrastructure" | "secrets" | "sca" | "api" |
                             // "container" | "iac"
    ScanID         string    // populated by orchestrator
    OrgID          string    // populated by orchestrator

    // === Classification ===
    Title          string    // Short human-readable title
    Description    string    // Detailed explanation
    Severity       string    // "critical" | "high" | "medium" | "low" | "info"
    FindingType    string    // Machine-readable type: "sqli", "xss", "weak_cipher",
                             // "hardcoded_secret", "exported_component", etc.
    CWEID          string    // "CWE-89" format — blank if unknown
    OWASP          string    // "A03:2021" format — blank if unknown
    CVSSScore      float64   // 0.0-10.0

    // === Evidence (web / API) ===
    TargetURL      string    // Full URL where finding was discovered
    Parameter      string    // Affected parameter name (for injection findings)
    Payload        string    // Attack payload that triggered the finding
    Request        string    // Raw HTTP request (truncated to 4KB)
    Response       string    // Raw HTTP response (truncated to 4KB)

    // === Evidence (source code / SAST) ===
    CodeFile       string    // Relative path to file
    CodeLine       int       // Line number
    CodeSnippet    string    // Surrounding code context

    // === Evidence (mobile — MobSF only) ===
    MobileOS       string    // "android" | "ios" | "" (empty for non-mobile)
    Permission     string    // Dangerous permission name
    ComponentName  string    // Exported Activity/Service/Receiver name

    // === Evidence (SSL/TLS — SSLyze only) ===
    CipherSuite    string    // Weak cipher name if applicable
    CertSubject    string    // Cert subject (for cert issues)

    // === Metadata ===
    RawOutputRef   string    // R2 reference to full tool output
    DiscoveredAt   time.Time
    Fingerprint    string    // SHA-256 hash for deduplication
}
```

### 4.1 Field Population Matrix

Not every tool populates every field. Required fields vs. tool category:

| Field | DAST | SAST | Mobile | SSL | Recon | SCA | API |
|---|---|---|---|---|---|---|---|
| Title | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Severity | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| CWEID | ✓ | ✓ | ✓ | ✓ | — | ✓ | ✓ |
| TargetURL | ✓ | — | — | ✓ | ✓ | — | ✓ |
| Parameter | ✓ | — | — | — | — | — | ✓ |
| Request/Response | ✓ | — | — | — | — | — | ✓ |
| CodeFile | — | ✓ | ✓ | — | — | ✓ | — |
| CodeLine | — | ✓ | ✓ | — | — | ✓ | — |
| MobileOS | — | — | ✓ | — | — | — | — |
| Permission | — | — | ✓ | — | — | — | — |
| ComponentName | — | — | ✓ | — | — | — | — |
| CipherSuite | — | — | — | ✓ | — | — | — |

---

## 5. Complete Tool Inventory

### 5.1 All 19 Tools at a Glance

| # | Tool | Category | Deployment | Runs On | Version |
|---|---|---|---|---|---|
| 1 | Nuclei | DAST | Native | Every web scan | v3.2.0 |
| 2 | ZAP | DAST | Persistent Docker | Full/deep web scans | latest |
| 3 | Wapiti | DAST | Native | Full web scans | v3.2.0 |
| 4 | SQLMap | DAST (SQLi) | Persistent Docker | When Nuclei finds SQLi hint | latest |
| 5 | Semgrep | SAST | Native | When source provided | v1.60.0 |
| 6 | Gitleaks | Secrets | Native | When repo connected | v8.18.0 |
| 7 | **MobSF** | **Mobile** | **Persistent Docker** | **APK/IPA/source upload** | **latest** |
| 8 | Trivy | SCA + Container | Persistent Docker | Deps or container provided | latest |
| 9 | OWASP Dep-Check | SCA | Native | Full SCA scans | v9.0.0 |
| 10 | Nmap | Infrastructure | Warm Pool Docker | Every web scan | latest |
| 11 | Nikto | Infrastructure | Native | Every web scan | v2.5.0 |
| 12 | **Subfinder** | **Recon** | **Native** | **Every web scan (first)** | **v2.6.3** |
| 13 | **httpx** | **Recon** | **Native** | **After Subfinder** | **v1.6.0** |
| 14 | **SSLyze** | **SSL/TLS** | **Native** | **Every web scan** | **v6.0.0** |
| 15 | Nuclei API templates | API | Native | API targets | (via Nuclei) |
| 16 | ZAP API scan | API | Persistent Docker | OpenAPI/GraphQL upload | (via ZAP) |
| 17 | CORStest | API | Native | Every API scan | (git pin) |
| 18 | Checkov | IaC | Native | IaC files provided | v3.2.0 |
| 19 | Trivy (container) | Container | Persistent Docker | Docker image provided | (via Trivy) |

**Bold = added in v4.0 (this spec). Others from v2.**

### 5.2 Scan Category Coverage

| Category | Primary Tool | Backup Tool | What's Covered |
|---|---|---|---|
| DAST | Nuclei | ZAP, Wapiti | OWASP Top 10, CVEs, misconfigs, XSS, CSRF, injection |
| SAST | Semgrep | — | Code patterns, injection flaws, insecure functions |
| Mobile | MobSF | — | APK/IPA/source, permissions, crypto, components, secrets |
| SCA | Trivy | Dep-Check | CVEs in dependencies (npm/pip/gem/go + Java/.NET) |
| Infrastructure | Nmap | Nikto | Open ports, services, server misconfigs |
| Recon | Subfinder + httpx | — | Subdomain discovery + live host probing |
| SSL/TLS | SSLyze | — | Weak ciphers, expired certs, HSTS, TLS versions |
| API | Nuclei API | ZAP API | BOLA, broken auth, mass assignment, CORS |
| IaC + Container | Checkov + Trivy | — | Terraform/K8s + Docker image vulns |
| Secrets | Gitleaks | Semgrep | Hardcoded keys in code and git history |

### 5.3 Deep SQLi Tool (SQLMap) — Trigger Logic

SQLMap is **not** run on every scan — it's too aggressive. It's triggered conditionally:

```go
// Triggered when Nuclei finds a SQLi hint
for _, finding := range nucleiFindings {
    if finding.FindingType == "sqli_hint" && finding.Severity >= "medium" {
        sqlmapJob := ScanJob{
            Engine:    "sqlmap",
            Target:    Target{URL: finding.TargetURL, Parameter: finding.Parameter},
            Config:    ScanConfig{Depth: "confirm_only", Timeout: 300},
        }
        dispatchJob(sqlmapJob)
    }
}
```

SQLMap confirms the vulnerability and produces a proof-of-concept exploit string. This gets attached as evidence to the final vulnerability record.

---

## 6. Native Tool Specifications

Each tool in this section has: command invocation, output parser logic, example output, Go struct.

### 6.1 Nuclei

**Binary:** `/usr/local/bin/nuclei` (v3.2.0)
**Category:** DAST
**Invocation:**
```bash
nuclei -u https://target.com \
       -t owasp-top-10,cves,misconfigurations \
       -rl 50 \
       -j \
       -silent \
       -o /dev/stdout
```

**Output format:** JSON lines (one finding per line).

**Sample output line:**
```json
{
  "template-id": "xss-detected",
  "info": {
    "name": "Cross-Site Scripting",
    "severity": "high",
    "classification": {"cwe-id": ["CWE-79"], "cvss-score": 7.4}
  },
  "host": "https://target.com",
  "matched-at": "https://target.com/search?q=<script>alert(1)</script>",
  "request": "GET /search?q=<script>alert(1)</script> HTTP/1.1\n...",
  "response": "HTTP/1.1 200 OK\n..."
}
```

**Parser logic:**
```go
func parseNucleiOutput(out []byte) ([]RawFinding, error) {
    var findings []RawFinding
    for _, line := range bytes.Split(out, []byte("\n")) {
        if len(line) == 0 { continue }
        var nf NucleiFinding
        if err := json.Unmarshal(line, &nf); err != nil {
            continue // malformed line — log and skip
        }
        cwe := ""
        if len(nf.Info.Classification.CWEIDs) > 0 {
            cwe = nf.Info.Classification.CWEIDs[0]
        }
        findings = append(findings, RawFinding{
            EngineCategory: "dast",
            Title:          nf.Info.Name,
            Description:    nf.Info.Description,
            Severity:       nf.Info.Severity,
            FindingType:    nf.TemplateID,
            CWEID:          cwe,
            CVSSScore:      nf.Info.Classification.CVSSScore,
            TargetURL:      nf.MatchedAt,
            Request:        truncate(nf.Request, 4096),
            Response:       truncate(nf.Response, 4096),
        })
    }
    return findings, nil
}
```

### 6.2 Semgrep

**Binary:** `/usr/local/bin/semgrep` (v1.60.0)
**Category:** SAST
**Invocation:**
```bash
semgrep scan --config=p/default \
             --json \
             --quiet \
             --metrics=off \
             --timeout=120 \
             /path/to/source
```

**Parser logic:** Maps Semgrep rule IDs to OWASP categories. Extracts `check_id`, `path`, `start.line`, `extra.message`, `extra.severity`, `extra.metadata.cwe`, `extra.metadata.owasp[0]`.

### 6.3 Subfinder

**Binary:** `/usr/local/bin/subfinder` (v2.6.3)
**Category:** Recon
**Invocation:**
```bash
subfinder -d example.com -oJ -silent -max-time 60
```

**Output format:** Newline-separated subdomain list.

**Parser logic:**
```go
func parseSubfinderOutput(out []byte) []string {
    var subs []string
    for _, line := range bytes.Split(out, []byte("\n")) {
        s := string(bytes.TrimSpace(line))
        if s != "" && strings.Contains(s, ".") {
            subs = append(subs, s)
        }
    }
    return subs
}
```

**Note:** Subfinder doesn't produce RawFindings directly — it produces subdomain strings consumed by the orchestrator. See §8 Recon-First Pipeline.

### 6.4 httpx

**Binary:** `/usr/local/bin/httpx` (v1.6.0)
**Category:** Recon
**Invocation:**
```bash
echo "api.example.com
admin.example.com
staging.example.com" | httpx -silent -json -status-code -title -tech-detect -web-server
```

**Output format:** JSON lines (one per probed host).

**Sample output:**
```json
{"url":"https://api.example.com","status_code":200,"tech":["nginx","Node.js"]}
{"url":"https://admin.example.com","status_code":403,"tech":["Apache"]}
```

### 6.5 SSLyze

**Binary:** `/usr/local/bin/sslyze` (v6.0.0)
**Category:** SSL/TLS
**Invocation:**
```bash
sslyze --json_out=- \
       --certinfo \
       --heartbleed --robot --openssl_ccs --reneg \
       --sslv2 --sslv3 --tlsv1 --tlsv1_1 --tlsv1_2 --tlsv1_3 \
       --compression --fallback --ems \
       target.com:443
```

**What it finds:**
- SSL 2.0 / SSL 3.0 / TLS 1.0 / TLS 1.1 supported (severity: critical/high)
- Weak cipher suites (RC4, DES, export-grade)
- Certificate expired or invalid chain
- Missing HSTS header
- Heartbleed, ROBOT, CCS Injection vulnerabilities
- Certificate subject mismatch

**Parser produces findings like:**
```go
RawFinding{
    EngineCategory: "ssl",
    Title:          "SSL 3.0 Supported — POODLE Vulnerable",
    Severity:       "high",
    FindingType:    "weak_ssl_protocol",
    CWEID:          "CWE-326",
    CipherSuite:    "SSL_CK_DES_192_EDE3_CBC_WITH_MD5",
}
```

### 6.6 Gitleaks

**Binary:** `/usr/local/bin/gitleaks` (v8.18.0)
**Category:** Secrets
**Invocation:**
```bash
gitleaks detect --source /path/to/repo \
                --report-format json \
                --report-path /dev/stdout \
                --exit-code 0 \
                --no-banner
```

**Scans:** Full git history (every commit), not just current code. Critical for catching secrets that were committed and later "removed" (but still in history).

**Output:** JSON array of leaks.

**Parser populates:**
- `Severity: "critical"` (always — hardcoded secrets are always critical)
- `CWEID: "CWE-798"`
- `CodeFile`, `CodeLine`, `CodeSnippet` with commit hash

### 6.7 Nikto

**Binary:** `/usr/local/bin/nikto` (v2.5.0)
**Category:** Infrastructure
**Invocation:**
```bash
nikto -h target.com:443 -Format xml -ask no -nointeractive
```

**Output format:** XML (`-Format xml` to stdout). Stable across Nikto 2.x; preferred over `-Format txt` for parser reliability.

**Parser extracts:** Web server misconfigs, outdated software versions, directory indexing, dangerous files.

### 6.8 Wapiti

**Binary:** `/usr/local/bin/wapiti` (v3.2.0)
**Category:** DAST (complement to Nuclei/ZAP)
**Invocation:**
```bash
wapiti -u https://target.com \
       -f json \
       -o /tmp/wapiti-output.json \
       --flush-session
```

**Note:** `-o /dev/stdout` corrupts JSON (Wapiti bug; verified at M6.6 pre-prep). Engine uses NativeRunner OutputFile mode (ADR-023 2nd consumer) to mint a per-Run tempfile via `os.CreateTemp`.

**Covers:** XSS, CSRF, XXE, file traversal, command injection, SSRF.

### 6.9 CORStest

**Binary:** `/usr/local/bin/corstest` (git-pinned)
**Category:** API
**Invocation:**
```bash
python3 corstest.py <urlfile> -v
```

CORStest takes a positional URL-list **file**, not a `-u <url>` flag. Engine BuildArgs mints a per-Run tempfile containing the target URL.

**Detects:** Dangerous CORS configurations (wildcard with credentials, reflection, null origin acceptance).

### 6.10 OWASP Dependency-Check

**Binary:** `/usr/local/bin/dependency-check` (v9.0.0)
**Category:** SCA
**Invocation:**
```bash
dependency-check --scan /path/to/project \
                 --format JSON \
                 --out /tmp/depcheck.json
```

**Parser reads:** Generated JSON, extracts CVE IDs for each vulnerable dependency.

### 6.11 Checkov

**Binary:** `/usr/local/bin/checkov` (v3.2.0)
**Category:** IaC
**Invocation:**
```bash
checkov -d /path/to/iac \
        --output json \
        --quiet \
        --framework terraform,kubernetes,cloudformation
```

**Scans:** Terraform, Kubernetes manifests, CloudFormation templates for security misconfigurations.

---

## 7. Persistent Docker Service Specifications

### 7.1 MobSF (Mobile Security Framework)

**See Section 9 for full mobile scan flow.**

**Docker image:** `opensecurity/mobile-security-framework-mobsf:latest`
**Port:** 8000
**API key:** Generated on first boot, stored in `~/.MobSF/secret`
**Health check:** `GET /api/v1/ping` → returns `{"status": "ok"}`

**Integration pattern:** Upload → Scan → Poll → Fetch report.

### 7.2 OWASP ZAP

**Docker image:** `zaproxy/zap-stable:latest`
**Port:** 8080
**Command:** `zap.sh -daemon -port 8080 -host 0.0.0.0 -config api.disablekey=true`
**Health check:** `GET /JSON/core/view/version`

**Integration — Full Active Scan:**

```go
func runZAPScan(ctx context.Context, baseURL string, target Target, cfg ScanConfig) ([]RawFinding, error) {
    // 1. Start spider to discover URLs
    spiderResp, _ := httpGET(ctx, baseURL+"/JSON/spider/action/scan/", map[string]string{
        "url": target.URL,
        "maxChildren": "10",
        "recurse": "true",
    })
    spiderID := extractID(spiderResp)

    // 2. Wait for spider completion
    waitForZAPCompletion(ctx, baseURL, "/JSON/spider/view/status/", spiderID)

    // 3. Start active scan
    activeResp, _ := httpGET(ctx, baseURL+"/JSON/ascan/action/scan/", map[string]string{
        "url": target.URL,
        "recurse": "true",
        "inScopeOnly": "true",
    })
    activeID := extractID(activeResp)

    // 4. Wait for active scan
    waitForZAPCompletion(ctx, baseURL, "/JSON/ascan/view/status/", activeID)

    // 5. Fetch alerts
    alertsResp, _ := httpGET(ctx, baseURL+"/JSON/alert/view/alerts/", map[string]string{
        "baseurl": target.URL,
    })
    return parseZAPAlerts(alertsResp)
}
```

**What ZAP catches that Nuclei doesn't:**
- Business logic flaws requiring session state
- Authenticated-only vulnerabilities
- Complex multi-step attack chains
- CSRF and session management issues

### 7.3 Trivy

**Docker image:** `aquasec/trivy:latest`
**Port:** 4954
**Command:** `server --listen 0.0.0.0:4954`
**Health check:** `GET /healthz`

**Two modes:**
1. **SCA scan** — scans project dependencies
2. **Container scan** — scans Docker images

**Invocation:**
```go
// SCA mode
req := TrivyScanRequest{
    Target: "/path/to/project",
    Type:   "fs",
    Severity: "UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL",
}

// Container mode
req := TrivyScanRequest{
    Target: "nginx:1.21",
    Type:   "image",
}
```

### 7.4 SQLMap

**Docker image:** `paoloo/sqlmap:latest`
**Port:** 8775 (sqlmapapi daemon)
**Health check:** `GET /version`

**Trigger:** Only runs when Nuclei finds a SQLi hint (see §5.3).

**Confirm-only mode** — doesn't attempt data extraction, just confirms exploitability:
```
POST /task/new
POST /scan/{taskid}/start {"url": "...", "data": "...", "level": 3, "risk": 1, "batch": true}
GET  /scan/{taskid}/status
GET  /scan/{taskid}/data
```

---

## 8. Recon-First Pipeline

**This is a key differentiator.** Every web scan begins with subdomain discovery. Discovered live subdomains automatically become scan targets.

### 8.1 The Flow

```
User submits: example.com
        │
        ↓
┌──────────────────────┐
│ 1. Subfinder          │ → api.example.com, admin.example.com,
│    (passive OSINT)    │   staging.example.com, old.example.com,
│                       │   dev.example.com, test.example.com
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ 2. Apply limit        │ → Max 100 subdomains per scan
│    (deduplication)    │   (configurable per tier)
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ 3. httpx probing      │ → Live: api (200), admin (403)
│    (HTTP HEAD probe)  │   Dead: staging, old, dev, test
│                       │   Tech: api→{nginx, Node.js}
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ 4. Build target list  │ → [example.com, api.example.com, admin.example.com]
└──────────┬───────────┘
           │
           ↓
┌──────────────────────┐
│ 5. All tools run on  │ → Nuclei, ZAP, SSLyze, Nikto, etc.
│    all live targets  │   run in parallel against 3 targets
│    in parallel       │
└──────────────────────┘
```

### 8.2 Go Implementation

```go
// internal/orchestrator/scan_executor.go

func (e *ScanExecutor) BuildTargetList(ctx context.Context, rootDomain string, limit int) ([]Target, error) {
    rootTarget := Target{
        URL:            "https://" + rootDomain,
        Domain:         rootDomain,
        TargetType:     "web",
        DomainVerified: true,
    }
    targets := []Target{rootTarget}

    // Publish recon_started event
    e.publisher.Publish("recon_started", map[string]string{"domain": rootDomain})

    // Step 1: Subfinder discovers subdomains
    subdomains, err := runSubfinder(ctx, rootDomain, 60*time.Second)
    if err != nil {
        log.Warn("subfinder failed, continuing with root only", "err", err)
        return targets, nil // fail-open
    }

    // Step 2: Apply tier-based limit
    if len(subdomains) > limit {
        subdomains = subdomains[:limit]
    }

    // Publish subdomains_discovered event
    e.publisher.Publish("subdomains_discovered", map[string]interface{}{
        "count": len(subdomains),
        "subdomains": subdomains,
    })

    if len(subdomains) == 0 {
        return targets, nil
    }

    // Step 3: httpx probes which subdomains are live
    liveHosts, err := runHttpx(ctx, subdomains, 120*time.Second)
    if err != nil {
        log.Warn("httpx failed, using all subdomains unprobed", "err", err)
        // Fallback: try all subdomains anyway
        for _, sub := range subdomains {
            targets = append(targets, Target{URL: "https://" + sub, Domain: sub, TargetType: "web"})
        }
        return targets, nil
    }

    // Step 4: Add live hosts as scan targets
    for _, host := range liveHosts {
        if host.URL == rootTarget.URL { continue }
        targets = append(targets, Target{
            URL:        host.URL,
            Domain:     extractDomain(host.URL),
            TargetType: "web",
        })
    }

    // Store attack surface in PostgreSQL for later retrieval via API
    e.storeAttackSurface(ctx, rootDomain, liveHosts, subdomains)

    return targets, nil
}
```

### 8.3 Safety Guardrails

| Guardrail | Rule |
|---|---|
| **Subdomain limit** | Max 100 per scan (Starter: 10, Growth: 50, Business: 500) |
| **Ownership scope** | Only scan subdomains of the verified root domain |
| **Wildcard detection** | If > 50% of subdomains return same response, treat as wildcard DNS and abort recon |
| **Rate limit** | Recon phase capped at 60s — if incomplete, proceed with what was found |
| **Cost control** | Recon counts as one "scan unit" against tier limits |

### 8.4 User Experience

The Attack Surface Map tab in the dashboard shows:
- Total discovered subdomains
- Live vs. dead count
- Per-subdomain status + last probe time
- Per-subdomain vulnerability count (after scan completes)
- Tech stack detected
- "Quick-scan this subdomain" button

This often reveals **forgotten staging environments, abandoned admin panels, and legacy APIs** that clients didn't know existed — a huge sales differentiator.

---

## 9. Mobile Security Flow (MobSF)

### 9.1 The Full Mobile Scan Pipeline

```
┌─────────────────────────────────────────────────────────┐
│ 1. User uploads APK/IPA/ZIP via dashboard               │
│    POST /orgs/:org_id/projects/:pid/mobile/upload       │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 2. FastAPI validates file                               │
│    • Extension: .apk, .ipa, .zip                        │
│    • Magic bytes match extension                        │
│    • Size ≤ 500 MB                                      │
│    • ClamAV malware scan passes                         │
│    • Platform detected (android/ios/unknown)            │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 3. File stored in Cloudflare R2                         │
│    r2://mobile/{org_id}/{project_id}/{uuid}.apk         │
│    mobile_uploads row created in PostgreSQL             │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 4. User triggers mobile scan                            │
│    POST /orgs/:org_id/projects/:pid/scans               │
│    { "scan_type": "mobile", "mobile_config": {...} }    │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 5. Orchestrator dispatches to mobile queue              │
│    Redis: shieldscan:queue:mobile                       │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 6. Go Worker picks up job                               │
│    • Downloads file from R2 to /tmp/                    │
│    • Calls MobSFRunner.Run()                            │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 7. MobSF Integration (REST API)                         │
│    a. POST /api/v1/upload        → file_hash            │
│    b. POST /api/v1/scan          → triggers analysis    │
│    c. GET  /api/v1/scan_status   → poll every 10s       │
│    d. GET  /api/v1/report_json   → full findings        │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 8. Parser converts MobSF report → []RawFinding          │
│    • Dangerous permissions                              │
│    • Hardcoded secrets                                  │
│    • Exported components without permission             │
│    • Weak cryptography usage                            │
│    • Insecure data storage                              │
│    • SSL pinning bypass detection                       │
│    • Debug flags enabled                                │
│    • ATS misconfigurations (iOS)                        │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 9. Raw findings → AI pipeline                           │
│    (same dedup + score + fix pipeline as web)           │
│    Mobile-specific fix prompts include platform context │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 10. Report + PDF + dashboard update                     │
│     R2 file auto-deleted 30 days post-scan              │
└─────────────────────────────────────────────────────────┘
```

### 9.2 MobSF REST API Integration (Go)

```go
// internal/tools/mobsf.go

type MobSFRunner struct {
    BaseURL string
    APIKey  string
    Client  *http.Client
}

func NewMobSFRunner(baseURL, apiKey string) ToolRunner {
    return &DockerServiceRunner{
        ToolName:     "mobsf",
        ToolCategory: "mobile",
        BaseURL:      baseURL,
        APIKey:       apiKey,
        HTTPClient:   &http.Client{Timeout: 15 * time.Minute},
        Execute:      mobsfExecute,
    }
}

func mobsfExecute(ctx context.Context, client *http.Client, baseURL, apiKey string,
                   target Target, cfg ScanConfig) ([]RawFinding, error) {

    // Step 1: Upload file to MobSF
    uploadResp, err := mobsfUpload(ctx, client, baseURL, apiKey, target.MobileUploadRef)
    if err != nil {
        return nil, fmt.Errorf("mobsf upload: %w", err)
    }
    fileHash := uploadResp.Hash

    // Step 2: Trigger scan
    if err := mobsfTriggerScan(ctx, client, baseURL, apiKey, fileHash); err != nil {
        return nil, fmt.Errorf("mobsf trigger: %w", err)
    }

    // Step 3: Poll for completion (max 10 min)
    deadline := time.Now().Add(10 * time.Minute)
    for time.Now().Before(deadline) {
        status, err := mobsfGetStatus(ctx, client, baseURL, apiKey, fileHash)
        if err != nil {
            return nil, fmt.Errorf("mobsf status: %w", err)
        }
        if status == "completed" { break }
        if status == "error" { return nil, fmt.Errorf("mobsf scan error") }

        select {
        case <-ctx.Done(): return nil, ctx.Err()
        case <-time.After(10 * time.Second):
        }
    }

    // Step 4: Fetch JSON report
    report, err := mobsfGetReport(ctx, client, baseURL, apiKey, fileHash)
    if err != nil {
        return nil, fmt.Errorf("mobsf report: %w", err)
    }

    return parseMobSFReport(report, target), nil
}
```

### 9.3 MobSF Report Parsing

MobSF produces a comprehensive JSON report. Key sections parsed:

```go
type MobSFReport struct {
    AppName       string                 `json:"app_name"`
    PackageName   string                 `json:"package_name"`
    AndroidAPI    int                    `json:"android_api,omitempty"`
    IOSMinVersion string                 `json:"ios_min_version,omitempty"`

    Permissions       map[string]MobSFPermission    `json:"permissions"`
    Secrets           []MobSFSecret                 `json:"secrets"`
    ExportedComponents []MobSFComponent             `json:"exported_components"`
    ManifestAnalysis  []MobSFManifestIssue          `json:"manifest_analysis"`
    CodeAnalysis      map[string]MobSFCodeIssue     `json:"code_analysis"`
    URLs              []MobSFURL                    `json:"urls"`
    Domains           []string                      `json:"domains"`
    Trackers          []MobSFTracker                `json:"trackers"`
    Certificate       MobSFCertificate              `json:"certificate_analysis"`
    BinaryAnalysis    []MobSFBinaryIssue            `json:"binary_analysis"`
    Niap              []MobSFNiapIssue              `json:"niap"`
}

func parseMobSFReport(r *MobSFReport, target Target) []RawFinding {
    var findings []RawFinding
    platform := detectPlatform(target.MobileUploadRef)

    // Dangerous permissions
    for perm, details := range r.Permissions {
        if details.Status == "dangerous" {
            findings = append(findings, RawFinding{
                EngineCategory: "mobile",
                Title:          fmt.Sprintf("Dangerous Permission: %s", perm),
                Severity:       "medium",
                FindingType:    "dangerous_permission",
                CWEID:          "CWE-250",
                MobileOS:       platform,
                Permission:     perm,
                Description:    details.Description,
            })
        }
    }

    // Hardcoded secrets in mobile app
    for _, s := range r.Secrets {
        findings = append(findings, RawFinding{
            EngineCategory: "mobile",
            Title:          "Hardcoded Secret in Mobile App",
            Severity:       "critical",
            FindingType:    "hardcoded_secret",
            CWEID:          "CWE-798",
            MobileOS:       platform,
            CodeFile:       s.File,
            CodeLine:       s.Line,
            CodeSnippet:    s.Type + ": " + s.Preview,
        })
    }

    // Exported components (Android-specific)
    for _, c := range r.ExportedComponents {
        findings = append(findings, RawFinding{
            EngineCategory: "mobile",
            Title:          fmt.Sprintf("Exported %s Without Permission: %s", c.Type, c.Name),
            Severity:       "high",
            FindingType:    "exported_component",
            CWEID:          "CWE-926",
            MobileOS:       "android",
            ComponentName:  c.Name,
            Description:    fmt.Sprintf("%s is exported and accessible without permission check.", c.Type),
        })
    }

    // Weak cryptography
    for _, bi := range r.BinaryAnalysis {
        if strings.Contains(bi.Issue, "weak_crypto") || strings.Contains(bi.Issue, "md5") ||
           strings.Contains(bi.Issue, "sha1") || strings.Contains(bi.Issue, "des") {
            findings = append(findings, RawFinding{
                EngineCategory: "mobile",
                Title:          "Weak Cryptography Detected",
                Severity:       "high",
                FindingType:    "weak_crypto",
                CWEID:          "CWE-327",
                MobileOS:       platform,
                Description:    bi.Description,
            })
        }
    }

    // iOS-specific: ATS misconfigurations
    if platform == "ios" {
        for _, m := range r.ManifestAnalysis {
            if strings.Contains(m.Title, "NSAllowsArbitraryLoads") {
                findings = append(findings, RawFinding{
                    EngineCategory: "mobile",
                    Title:          "App Transport Security (ATS) Disabled",
                    Severity:       "high",
                    FindingType:    "ats_disabled",
                    CWEID:          "CWE-311",
                    MobileOS:       "ios",
                    Description:    m.Description,
                })
            }
        }
    }

    return findings
}
```

### 9.4 Mobile-Specific AI Fix Prompt

When Claude generates fixes for mobile findings, the prompt includes platform context:

```
TARGET PLATFORM: Android (API level 33)
LANGUAGE: Kotlin
COMPONENT TYPE: Exported Activity
CODE LOCATION: src/main/java/com/example/app/LoginActivity.kt:42

FINDING: Exported Activity Without Permission
DESCRIPTION: LoginActivity is declared exported in AndroidManifest.xml but has no
             permission attribute. Any installed app can launch this activity.

CURRENT CODE:
<activity android:name=".LoginActivity" android:exported="true" />

Generate a secure fix following Android Security Guidelines. Show the corrected
AndroidManifest.xml entry and explain the security improvement.
```

Claude produces a fix like:
```xml
<activity
    android:name=".LoginActivity"
    android:exported="false" />

<!-- If this activity must be launched by other apps, require a custom permission: -->
<permission
    android:name="com.example.app.permission.LAUNCH_LOGIN"
    android:protectionLevel="signature" />

<activity
    android:name=".LoginActivity"
    android:exported="true"
    android:permission="com.example.app.permission.LAUNCH_LOGIN" />
```

---

## 10. Scan Type → Tool Matrix

### 10.1 The Matrix

```go
// internal/orchestrator/tool_router.go

var scanTypeMatrix = map[string]ScanTypeDefinition{
    "quick": {
        Tools:         []string{"subfinder", "httpx", "nuclei_fast", "sslyze"},
        MaxDuration:   5 * time.Minute,
        MinTier:       "free",
    },
    "full_web": {
        Tools:         []string{"subfinder", "httpx", "nuclei", "zap", "wapiti", "nikto", "nmap", "sslyze"},
        MaxDuration:   30 * time.Minute,
        MinTier:       "starter",
    },
    "full_web_source": {
        Tools:         []string{"subfinder", "httpx", "nuclei", "zap", "wapiti", "nikto", "nmap", "sslyze",
                                 "semgrep", "gitleaks", "dependency_check"},
        MaxDuration:   60 * time.Minute,
        MinTier:       "growth",
    },
    "api": {
        Tools:         []string{"nuclei_api", "zap_api", "corstest"},
        MaxDuration:   20 * time.Minute,
        MinTier:       "starter",
    },
    "mobile": {
        Tools:         []string{"mobsf"},
        MaxDuration:   15 * time.Minute,
        MinTier:       "growth",
    },
    "container": {
        Tools:         []string{"trivy_image", "checkov"},
        MaxDuration:   10 * time.Minute,
        MinTier:       "growth",
    },
    "full_spectrum": {
        Tools:         []string{"subfinder", "httpx", "nuclei", "zap", "wapiti", "nikto", "nmap", "sslyze",
                                 "semgrep", "gitleaks", "dependency_check", "trivy",
                                 "corstest", "checkov"},
        MaxDuration:   120 * time.Minute,
        MinTier:       "business",
    },
}

type ScanTypeDefinition struct {
    Tools       []string
    MaxDuration time.Duration
    MinTier     string
}
```

### 10.2 Conditional Tools

Some tools only run under specific conditions:

| Tool | Condition |
|---|---|
| `sqlmap` | Triggered when Nuclei finds SQLi hint (severity ≥ medium) |
| `zap` (full active) | Only on `full_web` and `full_spectrum`, not quick |
| `gitleaks` | Only when project has `source_repo_url` configured |
| `semgrep` | Only when project has `source_repo_url` configured |
| `dependency_check` | Only when project has `source_repo_url` configured |
| `checkov` | Only when IaC files detected in source repo |
| `trivy_image` | Only when project has `container_image` configured |

### 10.3 Tool Execution Concurrency

Tools for a scan run **in parallel** up to a per-worker concurrency limit:

```go
// Worker allows max 5 concurrent tool executions
const MaxConcurrentToolsPerWorker = 5

func (e *ScanExecutor) ExecuteTools(ctx context.Context, tools []ToolRunner, targets []Target) {
    sem := make(chan struct{}, MaxConcurrentToolsPerWorker)
    var wg sync.WaitGroup

    for _, tool := range tools {
        for _, target := range targets {
            wg.Add(1)
            sem <- struct{}{}  // acquire
            go func(t ToolRunner, tg Target) {
                defer wg.Done()
                defer func() { <-sem }()  // release

                findings, err := t.Run(ctx, tg, e.config)
                if err != nil {
                    log.Error("tool failed", "tool", t.Name(), "err", err)
                    e.publishJobFailed(t.Name(), tg, err)
                    return
                }
                e.storeFindingsAndPublish(findings)
            }(tool, target)
        }
    }
    wg.Wait()
}
```

---

## 11. Worker Startup & Health Checks

### 11.1 Startup Sequence

When a Go worker boots, it must verify all tools are ready before accepting jobs:

```go
// internal/worker/startup.go

func (w *Worker) Startup(ctx context.Context) error {
    log.Info("worker startup beginning")

    // Phase 1: Verify native binaries exist and are executable
    nativeBinaries := map[string]string{
        "nuclei":           "/usr/local/bin/nuclei",
        "semgrep":          "/usr/local/bin/semgrep",
        "subfinder":        "/usr/local/bin/subfinder",
        "httpx":            "/usr/local/bin/httpx",
        "gitleaks":         "/usr/local/bin/gitleaks",
        "sslyze":           "/usr/local/bin/sslyze",
        "nikto":            "/usr/local/bin/nikto",
        "wapiti":           "/usr/local/bin/wapiti",
        "corstest":         "/usr/local/bin/corstest",
        "dependency-check": "/usr/local/bin/dependency-check",
        "checkov":          "/usr/local/bin/checkov",
    }

    for name, path := range nativeBinaries {
        if _, err := os.Stat(path); err != nil {
            return fmt.Errorf("required native binary missing: %s at %s", name, path)
        }
        // Verify executable
        if info, _ := os.Stat(path); info.Mode().Perm()&0111 == 0 {
            return fmt.Errorf("binary not executable: %s", path)
        }
    }
    log.Info("all native binaries verified", "count", len(nativeBinaries))

    // Phase 2: Wait for Docker services to become healthy (30s timeout each)
    services := []DockerService{
        {Name: "mobsf", HealthURL: "http://localhost:8000/api/v1/ping"},
        {Name: "zap", HealthURL: "http://localhost:8080/JSON/core/view/version"},
        {Name: "trivy", HealthURL: "http://localhost:4954/healthz"},
        {Name: "sqlmap", HealthURL: "http://localhost:8775/version"},
    }

    for _, svc := range services {
        if err := w.waitForService(ctx, svc, 30*time.Second); err != nil {
            return fmt.Errorf("docker service not ready: %s — %w", svc.Name, err)
        }
        log.Info("docker service healthy", "service", svc.Name)
    }

    // Phase 3: Initialize Nmap warm pool
    if err := w.warmPool.Init(ctx); err != nil {
        log.Warn("warm pool init failed, will cold-start", "err", err)
    }

    // Phase 4: Register worker as available in Redis
    if err := w.redis.RegisterWorker(ctx, w.workerID, w.capabilities()); err != nil {
        return fmt.Errorf("failed to register worker: %w", err)
    }

    log.Info("worker ready to accept jobs", "worker_id", w.workerID)
    return nil
}

func (w *Worker) waitForService(ctx context.Context, svc DockerService, timeout time.Duration) error {
    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        resp, err := http.Get(svc.HealthURL)
        if err == nil && resp.StatusCode < 500 {
            resp.Body.Close()
            return nil
        }
        select {
        case <-ctx.Done(): return ctx.Err()
        case <-time.After(2 * time.Second):
        }
    }
    return fmt.Errorf("health check timed out after %s", timeout)
}
```

### 11.2 Runtime Health Check

Workers continuously monitor tool health during operation. Exposed via `/health` endpoint and published to Redis:

```go
type ToolStatus struct {
    Name      string    `json:"name"`
    Category  string    `json:"category"`
    Status    string    `json:"status"`    // "healthy" | "degraded" | "unhealthy"
    LastCheck time.Time `json:"last_check"`
    Error     string    `json:"error,omitempty"`
}

func (w *Worker) HealthCheckLoop(ctx context.Context) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done(): return
        case <-ticker.C:
            statuses := w.checkAllTools(ctx)
            w.redis.PublishWorkerHealth(ctx, w.workerID, statuses)

            // Mark worker as degraded if any critical tool is down
            criticalDown := countUnhealthy(statuses, criticalTools)
            if criticalDown > 0 {
                w.redis.MarkWorkerDegraded(ctx, w.workerID)
            } else {
                w.redis.MarkWorkerHealthy(ctx, w.workerID)
            }
        }
    }
}
```

### 11.3 Orchestrator Awareness

The orchestrator only dispatches jobs to healthy workers. If a worker is degraded:

- Jobs already in progress continue
- New jobs route to other healthy workers
- If ALL workers are degraded for a tool, jobs queue with warning to user
- Alerts sent to ops team via PagerDuty/Slack

---

## 12. Tool Execution Orchestration

### 12.1 Job Lifecycle

```
Queued → Claimed → Running → Completed | Failed | Canceled
```

**Detailed states:**

```go
type JobStatus string

const (
    JobQueued     JobStatus = "queued"
    JobClaimed    JobStatus = "claimed"   // Worker picked up
    JobReconning  JobStatus = "reconning" // Subfinder/httpx running
    JobRunning    JobStatus = "running"
    JobCompleted  JobStatus = "completed"
    JobFailed     JobStatus = "failed"
    JobCanceled   JobStatus = "canceled"
    JobRetrying   JobStatus = "retrying"  // Failed, will retry
)
```

### 12.2 Progress Publishing

Each tool publishes progress events to Redis as it works:

```go
func (t *NucleiTool) Run(ctx context.Context, target Target, cfg ScanConfig) ([]RawFinding, error) {
    publisher := NewProgressPublisher(ctx, t.scanID)

    publisher.Publish("job_started", map[string]interface{}{
        "engine": "nuclei", "target": target.URL,
    })

    // Run nuclei with progress streaming
    cmd := exec.CommandContext(ctx, t.BinaryPath, t.args(target, cfg)...)
    stdout, _ := cmd.StdoutPipe()

    if err := cmd.Start(); err != nil { return nil, err }

    var findings []RawFinding
    scanner := bufio.NewScanner(stdout)
    totalTemplates := 120  // estimated
    completed := 0

    for scanner.Scan() {
        completed++
        line := scanner.Bytes()

        // Parse finding if JSON
        if f, err := parseNucleiLine(line); err == nil {
            findings = append(findings, f)
            publisher.Publish("finding_discovered", map[string]interface{}{
                "severity": f.Severity,
                "title":    f.Title,
                "url":      f.TargetURL,
            })
        }

        // Publish progress every 10 templates
        if completed%10 == 0 {
            publisher.Publish("job_progress", map[string]interface{}{
                "engine":    "nuclei",
                "progress":  (completed * 100) / totalTemplates,
                "message":   fmt.Sprintf("Running template %d/%d", completed, totalTemplates),
            })
        }
    }

    publisher.Publish("job_completed", map[string]interface{}{
        "engine":         "nuclei",
        "finding_count":  len(findings),
    })

    return findings, nil
}
```

### 12.3 Scan Cancellation

See Specification §7.4 for the cancellation flow. Key implementation in the tool executor:

```go
func (e *ScanExecutor) ExecuteScan(ctx context.Context, scan Scan) error {
    scanCtx, cancel := context.WithCancel(ctx)
    defer cancel()

    // Listen for cancel signal in background
    go func() {
        ch := e.redis.Subscribe(scanCtx, "shieldscan:cancel:"+scan.ID)
        defer ch.Close()
        select {
        case <-ch.Channel():
            log.Info("scan cancel received", "scan_id", scan.ID)
            cancel()
        case <-scanCtx.Done():
            return
        }
    }()

    // Execute all tools with cancellable context
    err := e.runTools(scanCtx, scan)
    if errors.Is(err, context.Canceled) {
        e.publishJobStatus(scan.ID, JobCanceled, "User requested cancellation")
        return nil // don't retry canceled scans
    }
    return err
}
```

---

## 13. Error Handling & Retry Strategy

### 13.1 Error Categories

| Error Type | Example | Strategy |
|---|---|---|
| **Transient** | Network timeout, DNS failure, target rate-limited | Retry up to 3x with exponential backoff |
| **Tool crash** | Nuclei panic, ZAP OOM | Retry once with increased timeout |
| **Invalid target** | Domain not verified, URL malformed | Fail immediately, report to user |
| **Quota exceeded** | Tier limit hit mid-scan | Stop scan, notify user, save partial results |
| **Tool missing/broken** | Worker misconfigured | Mark worker degraded, reroute to healthy worker |
| **User canceled** | Explicit cancel action | Stop all tools, cleanup, don't retry |

### 13.2 Partial Success Handling

**Key principle:** If 5 of 8 tools succeed and 3 fail, still report results from the 5 successful tools.

```go
type ScanResult struct {
    ScanID          string
    Status          string  // "completed" | "partial" | "failed"
    ToolResults     map[string]ToolResult
    TotalFindings   int
    FailedTools     []string
}

type ToolResult struct {
    Name         string
    Status       string // "success" | "failed" | "timeout"
    FindingCount int
    Error        string
    Duration     time.Duration
}
```

- If all tools fail → scan status = `failed`
- If some tools fail → scan status = `partial`, user notified which engines didn't run
- If all succeed → scan status = `completed`

### 13.3 Retry Configuration

```go
var retryPolicy = map[string]RetryConfig{
    "nuclei":   {MaxAttempts: 3, BackoffBase: 5 * time.Second,  Timeout: 30 * time.Minute},
    "zap":      {MaxAttempts: 2, BackoffBase: 10 * time.Second, Timeout: 45 * time.Minute},
    "mobsf":    {MaxAttempts: 2, BackoffBase: 15 * time.Second, Timeout: 20 * time.Minute},
    "sqlmap":   {MaxAttempts: 1, BackoffBase: 0,                Timeout: 10 * time.Minute},
    "recon":    {MaxAttempts: 2, BackoffBase: 3 * time.Second,  Timeout: 2 * time.Minute},
    // ... other tools
}
```

---

## 14. Adding a New Tool — Developer Guide

When a new security tool becomes available (open-source or custom), integration is straightforward thanks to the ToolRunner interface.

### 14.1 Checklist

```
[ ] 1. Determine category (dast | sast | mobile | ssl | recon | sca | api | container | iac | secrets)
[ ] 2. Determine deployment model (native binary | persistent Docker service)
[ ] 3. Pin version in provision-worker.sh (native) or docker-compose.services.yml (Docker)
[ ] 4. Create runner file: internal/tools/<toolname>.go
[ ] 5. Implement ToolRunner interface (use NativeRunner or DockerServiceRunner)
[ ] 6. Implement output parser → []RawFinding
[ ] 7. Map tool's finding types to CWE IDs in finding_type_to_cwe.go
[ ] 8. Add tool to scanTypeMatrix in tool_router.go
[ ] 9. Register tool in worker startup health checks
[ ] 10. Write unit tests for parser (sample outputs in testdata/)
[ ] 11. Write integration test with real tool binary
[ ] 12. Document tool in this file (TOOL-ARCHITECTURE.md §6 or §7)
[ ] 13. Update deployment provisioning script
[ ] 14. Test on staging before promoting to production
```

### 14.2 Example: Adding Retire.js

Let's walk through adding retire.js (a JavaScript dependency vulnerability scanner).

**1. Category:** `sca` (Software Composition Analysis)
**2. Deployment:** Native (it's a Node.js CLI, lightweight)
**3. Version pin:** `retire@5.2.0`

**4. Create `internal/tools/retirejs.go`:**
```go
package tools

func NewRetireJSRunner() ToolRunner {
    return &NativeRunner{
        ToolName:     "retirejs",
        ToolCategory: "sca",
        BinaryPath:   "/usr/local/bin/retire",
        Timeout:      5 * time.Minute,
        BuildArgs: func(target Target, cfg ScanConfig) []string {
            return []string{"--path", target.SourcePath, "--outputformat", "json"}
        },
        ParseOutput: parseRetireJSOutput,
    }
}

func parseRetireJSOutput(out []byte) ([]RawFinding, error) {
    var report struct {
        Data []struct {
            File    string `json:"file"`
            Results []struct {
                Component       string `json:"component"`
                Version         string `json:"version"`
                Vulnerabilities []struct {
                    Severity    string   `json:"severity"`
                    Identifiers struct{ CVE []string } `json:"identifiers"`
                    Info        []string `json:"info"`
                } `json:"vulnerabilities"`
            } `json:"results"`
        } `json:"data"`
    }
    if err := json.Unmarshal(out, &report); err != nil {
        return nil, err
    }

    var findings []RawFinding
    for _, d := range report.Data {
        for _, r := range d.Results {
            for _, v := range r.Vulnerabilities {
                cve := ""
                if len(v.Identifiers.CVE) > 0 { cve = v.Identifiers.CVE[0] }
                findings = append(findings, RawFinding{
                    EngineCategory: "sca",
                    Title:          fmt.Sprintf("Vulnerable JS library: %s@%s", r.Component, r.Version),
                    Severity:       v.Severity,
                    FindingType:    "vulnerable_dependency",
                    CWEID:          "CWE-1104",  // Use of Unmaintained Third Party Components
                    CodeFile:       d.File,
                    Description:    strings.Join(v.Info, " "),
                })
            }
        }
    }
    return findings, nil
}
```

**5. Register in tool matrix:**
```go
// tool_router.go
scanTypeMatrix["full_web_source"].Tools = append(
    scanTypeMatrix["full_web_source"].Tools, "retirejs")
```

**6. Add to provisioning:**
```bash
# provision-worker.sh
npm install -g retire@5.2.0
```

**7. Add to startup health check:**
```go
nativeBinaries["retire"] = "/usr/local/bin/retire"
```

**8. Write tests, update docs, ship.**

That's the entire integration. The AI pipeline, dedup, correlation, and reporting all work automatically because the tool produces normalized RawFindings.

---

*End of tool architecture. See `IMPLEMENTATION-PLAN.md` for task-by-task build plan and `OPERATIONS-RUNBOOK.md` for deployment.*
