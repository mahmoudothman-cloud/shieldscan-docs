# ShieldScan — Tools 5 Feature Addendum

**Version:** 1.0
**Date:** 2026-04-20
**Status:** Authoritative — extends SPECIFICATION.md, TOOL-ARCHITECTURE.md, IMPLEMENTATION-PLAN.md, and VERSIONS.md
**Timeline Impact:** +1 week (13 → 14 weeks to launch)

> **For Claude Code:** This document adds 5 native security tools to the ShieldScan scanner, bringing the total from 19 to 24. Read AFTER the base 9 docs and ADDENDUM-CRITICAL-5.md. When conflicts arise, this document wins for anything it specifies; the original docs win for everything else. All 5 tools follow the existing `NativeRunner` pattern from TOOL-ARCHITECTURE.md §3 — no new architecture required.

---

## Table of Contents

1. [Summary of Changes](#1-summary-of-changes)
2. [Updated Tool Inventory (24 Tools)](#2-updated-tool-inventory-24-tools)
3. [Tool 1 — Trufflehog (Verified Secret Scanning)](#3-tool-1--trufflehog-verified-secret-scanning)
4. [Tool 2 — Gobuster (Directory/File Discovery)](#4-tool-2--gobuster-directoryfile-discovery)
5. [Tool 3 — Amass (Deep Subdomain Enumeration)](#5-tool-3--amass-deep-subdomain-enumeration)
6. [Tool 4 — Kiterunner (API Route Discovery)](#6-tool-4--kiterunner-api-route-discovery)
7. [Tool 5 — jsluice (JavaScript Bundle Analysis)](#7-tool-5--jsluice-javascript-bundle-analysis)
8. [RawFinding Schema Extensions](#8-rawfinding-schema-extensions)
9. [Scan Type Matrix Updates](#9-scan-type-matrix-updates)
10. [Wordlist Management](#10-wordlist-management)
11. [Provisioning Script Updates](#11-provisioning-script-updates)
12. [Version Pinning Additions](#12-version-pinning-additions)
13. [Implementation Milestone M6.5](#13-implementation-milestone-m65)
14. [Cross-References to Updated Docs](#14-cross-references-to-updated-docs)

---

## 1. Summary of Changes

### 1.1 What's Added

Five native Go/Python binaries, each filling a distinct coverage gap:

| Tool | Category | Unique Capability | Integration |
|---|---|---|---|
| **trufflehog** | Secrets | **Verifies** secrets are live (makes validation API calls) | Native binary |
| **Gobuster** | Discovery | Brute-forces hidden directories and files | Native binary |
| **Amass** | Recon | Active DNS enumeration, cert transparency, ASN analysis | Native binary |
| **Kiterunner** | API | Discovers undocumented API endpoints | Native binary |
| **jsluice** | SPA | Extracts secrets/URLs/endpoints from JavaScript bundles | Native binary |

### 1.2 Architecture Impact

**Zero architectural changes.** All 5 tools:
- Implement the existing `ToolRunner` interface via `NativeRunner`
- Produce standard `RawFinding` structs (with minor extensions documented in §8)
- Deploy as native binaries on existing worker pools (no new Docker services)
- Add ~60 seconds to full-spectrum scan time
- Require ~200MB of RAM collectively during execution

### 1.3 Data Model Impact

One new boolean field on `raw_findings`:

```sql
ALTER TABLE raw_findings ADD COLUMN secret_verified BOOLEAN DEFAULT NULL;
-- NULL for non-secret findings
-- TRUE for trufflehog-verified live secrets
-- FALSE for pattern-match-only secrets (Gitleaks or trufflehog unverified)
```

No other schema changes required.

### 1.4 Timeline Impact

- Implementation: 7 engineer-days
- Testing + integration: 2 days
- **Total: ~1 week added → 13 → 14 weeks to launch**

---

## 2. Updated Tool Inventory (24 Tools)

Replaces the table in TOOL-ARCHITECTURE.md §5.1.

| # | Tool | Category | Deployment | Purpose |
|---|---|---|---|---|
| 1 | Nuclei | DAST | Native | Template-based vulnerability scanning |
| 2 | ZAP | DAST | Persistent Docker | Deep active DAST |
| 3 | Wapiti | DAST | Native | Complement to Nuclei/ZAP |
| 4 | SQLMap | DAST (SQLi) | Persistent Docker | SQLi confirmation |
| 5 | Semgrep | SAST | Native | Code pattern analysis |
| 6 | Gitleaks | Secrets | Native | Pattern-based secret detection |
| **7** | **trufflehog** ⭐ | **Secrets** | **Native** | **Verified live secret detection** |
| 8 | MobSF | Mobile | Persistent Docker | APK/IPA analysis |
| 9 | Trivy | SCA + Container | Persistent Docker | Dependency CVEs |
| 10 | OWASP Dep-Check | SCA | Native | Deep dependency analysis |
| 11 | Nmap | Infrastructure | Warm Pool | Port scanning |
| 12 | Nikto | Infrastructure | Native | Web server misconfigurations |
| 13 | Subfinder | Recon | Native | Passive subdomain discovery |
| **14** | **Amass** ⭐ | **Recon** | **Native** | **Active DNS + cert transparency + ASN** |
| 15 | httpx | Recon | Native | HTTP probing after subdomain discovery |
| **16** | **Gobuster** ⭐ | **Discovery** | **Native** | **Directory/file brute-forcing** |
| 17 | SSLyze | SSL/TLS | Native | TLS/cipher/cert analysis |
| 18 | Nuclei API | API | Native | API template scanning (via Nuclei) |
| 19 | ZAP API | API | Persistent Docker | OpenAPI/GraphQL deep scan |
| 20 | CORStest | API | Native | CORS misconfiguration |
| **21** | **Kiterunner** ⭐ | **API** | **Native** | **Undocumented API endpoint discovery** |
| **22** | **jsluice** ⭐ | **SPA** | **Native** | **JS bundle secret/URL extraction** |
| 23 | Checkov | IaC | Native | Terraform/K8s misconfigurations |
| 24 | Trivy (container mode) | Container | Persistent Docker | Docker image scanning |

**New tools marked with ⭐.**

### 2.1 New Scan Category

Added: `spa` (Single-Page Application) category for jsluice findings. Existing categories unchanged.

---

## 3. Tool 1 — Trufflehog (Verified Secret Scanning)

### 3.1 What It Adds That Gitleaks Can't

**Gitleaks** produces findings like:
```
HIGH: Pattern matching "AWS Access Key" found at src/config.js:42
```

**Trufflehog** produces findings like:
```
CRITICAL: VERIFIED AWS access key AKIAIOSFODNN7EXAMPLE found at src/config.js:42.
Credentials are currently active. Key belongs to account 123456789012.
Associated permissions include: s3:*, iam:GetUser.
```

The difference: trufflehog **calls the actual API** to verify the credential works. A verified secret = definitive proof of compromise.

### 3.2 Configuration

**Pinned version:** `v3.88.0` (verify current via `verify-versions.sh`)

**Install command:**
```bash
go install github.com/trufflesecurity/trufflehog/v3@v3.88.0
```

**Binary location:** `/usr/local/bin/trufflehog`

### 3.3 CLI Invocation

Git history scan (for repos):
```bash
trufflehog git file:///path/to/repo \
    --json \
    --only-verified=false \
    --no-update \
    --concurrency=4
```

Filesystem scan (for source uploads):
```bash
trufflehog filesystem /path/to/source \
    --json \
    --only-verified=false \
    --no-update
```

**Key flags:**
- `--only-verified=false` — report both verified and unverified; we distinguish in output
- `--no-update` — prevent runtime self-update (security)
- `--concurrency=4` — parallel detector execution

### 3.4 Sample Output

```json
{
  "SourceMetadata": {
    "Data": {
      "Git": {
        "commit": "a3f2b1c4d5e6f7",
        "file": "src/config.js",
        "email": "dev@example.com",
        "timestamp": "2026-03-15T14:23:10Z",
        "line": 42
      }
    }
  },
  "SourceType": 7,
  "SourceName": "trufflehog - git",
  "DetectorType": 2,
  "DetectorName": "AWS",
  "Verified": true,
  "Raw": "AKIAIOSFODNN7EXAMPLE",
  "RawV2": "",
  "Redacted": "AKIA************AMPLE",
  "ExtraData": {
    "account": "123456789012",
    "permissions": "s3:*, iam:GetUser"
  }
}
```

### 3.5 Parser Implementation

```go
// internal/tools/trufflehog.go
package tools

import (
    "bytes"
    "encoding/json"
    "fmt"
    "time"
)

type TrufflehogFinding struct {
    SourceMetadata struct {
        Data struct {
            Git struct {
                Commit    string `json:"commit"`
                File      string `json:"file"`
                Email     string `json:"email"`
                Timestamp string `json:"timestamp"`
                Line      int    `json:"line"`
            } `json:"Git"`
            Filesystem struct {
                File string `json:"file"`
                Line int    `json:"line"`
            } `json:"Filesystem"`
        } `json:"Data"`
    } `json:"SourceMetadata"`
    DetectorName string            `json:"DetectorName"`
    Verified     bool              `json:"Verified"`
    Redacted     string            `json:"Redacted"`
    ExtraData    map[string]string `json:"ExtraData"`
}

func NewTrufflehogRunner() ToolRunner {
    return &NativeRunner{
        ToolName:     "trufflehog",
        ToolCategory: "secrets",
        BinaryPath:   "/usr/local/bin/trufflehog",
        Timeout:      15 * time.Minute,
        BuildArgs: func(target Target, cfg ScanConfig) []string {
            if target.SourcePath != "" {
                return []string{
                    "filesystem", target.SourcePath,
                    "--json", "--only-verified=false", "--no-update",
                    "--concurrency=4",
                }
            }
            // Git scan mode
            return []string{
                "git", "file://" + target.SourcePath,
                "--json", "--only-verified=false", "--no-update",
                "--concurrency=4",
            }
        },
        ParseOutput: parseTrufflehogOutput,
    }
}

func parseTrufflehogOutput(out []byte) ([]RawFinding, error) {
    var findings []RawFinding

    for _, line := range bytes.Split(out, []byte("\n")) {
        if len(line) == 0 {
            continue
        }
        var tf TrufflehogFinding
        if err := json.Unmarshal(line, &tf); err != nil {
            continue // malformed line - skip
        }

        // Extract file and line from either Git or Filesystem source
        file := tf.SourceMetadata.Data.Git.File
        lineNum := tf.SourceMetadata.Data.Git.Line
        if file == "" {
            file = tf.SourceMetadata.Data.Filesystem.File
            lineNum = tf.SourceMetadata.Data.Filesystem.Line
        }

        severity := "high"
        title := fmt.Sprintf("Pattern-matched %s secret", tf.DetectorName)
        description := fmt.Sprintf("A string matching the %s secret pattern was found in source code.", tf.DetectorName)
        verifiedFlag := false

        if tf.Verified {
            severity = "critical"
            title = fmt.Sprintf("VERIFIED LIVE %s credential", tf.DetectorName)
            description = fmt.Sprintf(
                "A currently-active %s credential was found and successfully validated against the API. "+
                    "This credential must be rotated IMMEDIATELY as it may already be compromised. "+
                    "Details: %v",
                tf.DetectorName, tf.ExtraData,
            )
            verifiedFlag = true
        }

        findings = append(findings, RawFinding{
            EngineCategory: "secrets",
            Title:          title,
            Description:    description,
            Severity:       severity,
            FindingType:    fmt.Sprintf("secret_%s", tf.DetectorName),
            CWEID:          "CWE-798",  // Use of Hard-coded Credentials
            OWASP:          "A07:2021",
            CodeFile:       file,
            CodeLine:       lineNum,
            CodeSnippet:    fmt.Sprintf("%s (redacted: %s)", tf.DetectorName, tf.Redacted),
            SecretVerified: &verifiedFlag, // New field — see §8
        })
    }
    return findings, nil
}
```

### 3.6 Relationship With Gitleaks

**Both run together.** They are complementary:

- **Gitleaks** finds broader pattern matches, catches more potential secrets
- **Trufflehog** verifies which of those are actually live

Dedup via fingerprinting: if both tools find the same secret at the same file:line, the AI pipeline merges them, preferring trufflehog's verified flag.

### 3.7 Triggered When

| Scan Type | Runs? |
|---|---|
| `quick` | No |
| `full_web` | No (no source available) |
| `full_web_source` | **Yes** |
| `api` | No |
| `mobile` | No |
| `container` | No |
| `full_spectrum` | **Yes** |

### 3.8 Implementation Task (M6.5.1)

**Files:**
- Create: `shieldscan-engine/internal/tools/trufflehog.go`
- Create: `shieldscan-engine/internal/tools/testdata/trufflehog_output.jsonl`
- Test: `shieldscan-engine/internal/tools/trufflehog_test.go`

**Step 1: Failing tests**

```go
func TestTrufflehogParser_VerifiedSecret(t *testing.T) {
    raw, _ := os.ReadFile("testdata/trufflehog_output.jsonl")
    findings, err := parseTrufflehogOutput(raw)
    require.NoError(t, err)

    verified := findings[0]
    assert.Equal(t, "secrets", verified.EngineCategory)
    assert.Equal(t, "critical", verified.Severity)
    assert.Contains(t, verified.Title, "VERIFIED LIVE")
    assert.NotNil(t, verified.SecretVerified)
    assert.True(t, *verified.SecretVerified)
    assert.Equal(t, "CWE-798", verified.CWEID)
}

func TestTrufflehogParser_UnverifiedSecret(t *testing.T) {
    raw := []byte(`{"DetectorName":"Slack","Verified":false,"Redacted":"xoxb-***","SourceMetadata":{"Data":{"Git":{"file":"api.js","line":15}}}}`)
    findings, err := parseTrufflehogOutput(raw)
    require.NoError(t, err)

    assert.Equal(t, "high", findings[0].Severity)
    assert.NotContains(t, findings[0].Title, "VERIFIED LIVE")
    assert.False(t, *findings[0].SecretVerified)
}

func TestTrufflehogParser_HandlesMalformedLines(t *testing.T) {
    raw := []byte(`{"valid":"json"}\nnot json\n{"Verified":true,"DetectorName":"AWS","SourceMetadata":{"Data":{"Git":{"file":"x","line":1}}}}`)
    findings, err := parseTrufflehogOutput(raw)
    require.NoError(t, err)
    assert.Len(t, findings, 1) // Only the last valid line with required fields
}
```

**Step 2:** Implement per §3.5 above.

**Step 3: Commit**
```bash
git commit -m "feat(engine): add trufflehog runner for verified secret scanning"
```

---

## 4. Tool 2 — Gobuster (Directory/File Discovery)

### 4.1 What It Fills

Nuclei has templates for *known* exposed paths (e.g., `/.git/HEAD`). Gobuster **brute-forces** paths against a wordlist, finding things no template knows about.

**Common findings Gobuster catches that nothing else does:**
- `/admin`, `/administrator`, `/wp-admin` on custom stacks
- `/backup.sql`, `/db.sql`, `/dump.sql`
- `/.env`, `/.env.production`, `/.env.local`
- `/.git/config`, `/.svn/entries`
- `/staging`, `/beta`, `/old`
- `/phpmyadmin`, `/adminer.php`
- `/config.php.bak`, `/.DS_Store`
- Custom API paths unique to target's framework

### 4.2 Configuration

**Pinned version:** `v3.6.2`

**Install command:**
```bash
go install github.com/OJ/gobuster/v3@v3.6.2
```

**Binary location:** `/usr/local/bin/gobuster`

### 4.3 CLI Invocation

```bash
gobuster dir \
    -u https://target.com \
    -w /usr/share/seclists/Discovery/Web-Content/common.txt \
    -t 30 \
    -o /tmp/gobuster-out.txt \
    --no-error \
    --no-color \
    -q \
    --random-agent \
    --timeout 10s
```

**Key flags:**
- `-t 30` — 30 concurrent threads (balance speed vs target load)
- `--timeout 10s` — per-request timeout
- `--random-agent` — rotate user agents to avoid trivial blocking
- `-q` — quiet mode, cleaner output parsing

### 4.4 Wordlist Strategy

Gobuster needs wordlists. Use **SecLists** (industry standard):

**Three-tier wordlist strategy** by scan depth:

| Scan Depth | Wordlist | Size | Scan Time |
|---|---|---|---|
| Quick | `common.txt` | 4,700 entries | ~2 min |
| Standard | `directory-list-2.3-medium.txt` | 220,000 entries | ~20 min |
| Deep | `directory-list-2.3-big.txt` | 1,200,000 entries | ~3 hours |

Installed via provisioning script (see §11).

### 4.5 Sample Output

Gobuster outputs plain text:

```
/admin                (Status: 301) [Size: 178] [--> https://target.com/admin/]
/api                  (Status: 200) [Size: 2341]
/backup.sql           (Status: 200) [Size: 5432100]
/.env                 (Status: 200) [Size: 847]
/.git/HEAD            (Status: 200) [Size: 21]
/phpmyadmin           (Status: 200) [Size: 5421]
```

### 4.6 Parser Implementation

```go
// internal/tools/gobuster.go
package tools

import (
    "bytes"
    "regexp"
    "strconv"
    "strings"
    "time"
)

var gobusterLineRegex = regexp.MustCompile(
    `^(\S+)\s+\(Status:\s+(\d+)\)\s+\[Size:\s+(\d+)\](.*)$`,
)

// High-severity paths that warrant elevated findings
var criticalPaths = map[string]bool{
    "/.env":              true,
    "/.env.production":   true,
    "/.env.local":        true,
    "/.git/HEAD":         true,
    "/.git/config":       true,
    "/.svn/entries":      true,
    "/.DS_Store":         true,
    "/backup.sql":        true,
    "/db.sql":            true,
    "/dump.sql":          true,
    "/database.sql":      true,
    "/config.php.bak":    true,
    "/wp-config.php.bak": true,
    "/id_rsa":            true,
    "/.ssh/id_rsa":       true,
}

var highRiskPaths = map[string]bool{
    "/admin":        true,
    "/administrator": true,
    "/phpmyadmin":   true,
    "/adminer.php":  true,
    "/wp-admin":     true,
    "/manager":      true,
    "/console":      true,
}

func NewGobusterRunner(wordlistPath string) ToolRunner {
    return &NativeRunner{
        ToolName:     "gobuster",
        ToolCategory: "discovery",
        BinaryPath:   "/usr/local/bin/gobuster",
        Timeout:      25 * time.Minute,
        BuildArgs: func(target Target, cfg ScanConfig) []string {
            wordlist := wordlistPath
            if cfg.Depth == "quick" {
                wordlist = "/opt/seclists/Discovery/Web-Content/common.txt"
            } else if cfg.Depth == "deep" {
                wordlist = "/opt/seclists/Discovery/Web-Content/directory-list-2.3-big.txt"
            } else {
                wordlist = "/opt/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt"
            }
            return []string{
                "dir",
                "-u", target.URL,
                "-w", wordlist,
                "-t", "30",
                "--timeout", "10s",
                "--random-agent",
                "-q",
                "--no-error",
                "--no-color",
            }
        },
        ParseOutput: parseGobusterOutput,
    }
}

func parseGobusterOutput(out []byte) ([]RawFinding, error) {
    var findings []RawFinding

    for _, line := range bytes.Split(out, []byte("\n")) {
        lineStr := strings.TrimSpace(string(line))
        matches := gobusterLineRegex.FindStringSubmatch(lineStr)
        if len(matches) < 4 {
            continue
        }

        path := matches[1]
        statusCode, _ := strconv.Atoi(matches[2])
        size, _ := strconv.Atoi(matches[3])

        // Classify severity based on what was found
        severity, title, description := classifyGobusterFinding(path, statusCode)

        // Skip redirects to known login pages unless they're interesting
        if statusCode == 401 || statusCode == 403 {
            // Auth-required paths are useful but lower severity
            severity = downgradeSeverity(severity)
        }

        findings = append(findings, RawFinding{
            EngineCategory: "discovery",
            Title:          title,
            Description:    description,
            Severity:       severity,
            FindingType:    "exposed_path",
            CWEID:          "CWE-538", // Insertion of Sensitive Information into Externally-Accessible File
            OWASP:          "A05:2021", // Security Misconfiguration
            TargetURL:      path,
            Response:       fmt.Sprintf("HTTP %d, %d bytes", statusCode, size),
        })
    }
    return findings, nil
}

func classifyGobusterFinding(path string, statusCode int) (severity, title, description string) {
    if criticalPaths[path] {
        return "critical",
            fmt.Sprintf("Sensitive file exposed: %s", path),
            fmt.Sprintf("The path %s was found publicly accessible (HTTP %d). This file type typically contains credentials, database dumps, or version control metadata that must not be public.", path, statusCode)
    }
    if highRiskPaths[path] {
        return "high",
            fmt.Sprintf("Administrative interface exposed: %s", path),
            fmt.Sprintf("An administrative interface was found at %s (HTTP %d). Even with authentication, exposing administrative paths increases attack surface.", path, statusCode)
    }
    if strings.HasSuffix(path, ".bak") || strings.HasSuffix(path, ".old") ||
       strings.HasSuffix(path, "~") || strings.HasSuffix(path, ".swp") {
        return "high",
            fmt.Sprintf("Backup file exposed: %s", path),
            fmt.Sprintf("A backup file was found publicly accessible at %s. Backup files often contain source code or configuration data.", path)
    }
    if statusCode == 200 && strings.Contains(path, "api") {
        return "medium",
            fmt.Sprintf("API endpoint discovered: %s", path),
            fmt.Sprintf("An API endpoint was found at %s. Review to ensure it requires proper authentication and authorization.", path)
    }
    return "low",
        fmt.Sprintf("Path discovered: %s", path),
        fmt.Sprintf("The path %s was discovered via directory enumeration (HTTP %d, %d bytes). Review whether public access is intended.", path, statusCode, 0)
}

func downgradeSeverity(current string) string {
    switch current {
    case "critical":
        return "high"
    case "high":
        return "medium"
    case "medium":
        return "low"
    default:
        return "info"
    }
}
```

### 4.7 Triggered When

| Scan Type | Runs? | Depth |
|---|---|---|
| `quick` | No | — |
| `full_web` | **Yes** | Standard wordlist |
| `full_web_source` | **Yes** | Standard wordlist |
| `api` | No (Kiterunner handles APIs) | — |
| `mobile` | No | — |
| `full_spectrum` | **Yes** | Deep wordlist |

### 4.8 Implementation Task (M6.5.2)

**Files:**
- Create: `shieldscan-engine/internal/tools/gobuster.go`
- Create: `shieldscan-engine/internal/tools/testdata/gobuster_output.txt`
- Test: `shieldscan-engine/internal/tools/gobuster_test.go`

**Step 1: Failing tests**

```go
func TestGobusterParser_CriticalExposure(t *testing.T) {
    raw := []byte(`/.env (Status: 200) [Size: 847]
/.git/HEAD (Status: 200) [Size: 21]
/backup.sql (Status: 200) [Size: 5432100]`)
    findings, err := parseGobusterOutput(raw)
    require.NoError(t, err)
    require.Len(t, findings, 3)
    for _, f := range findings {
        assert.Equal(t, "critical", f.Severity)
        assert.Equal(t, "CWE-538", f.CWEID)
    }
}

func TestGobusterParser_HighRiskAdminPath(t *testing.T) {
    raw := []byte(`/admin (Status: 200) [Size: 2341]`)
    findings, _ := parseGobusterOutput(raw)
    require.Len(t, findings, 1)
    assert.Equal(t, "high", findings[0].Severity)
    assert.Contains(t, findings[0].Title, "Administrative interface")
}

func TestGobusterParser_DowngradesFor403(t *testing.T) {
    raw := []byte(`/.env (Status: 403) [Size: 0]`)
    findings, _ := parseGobusterOutput(raw)
    // Critical → High because it's 403 (auth required, less exposed)
    assert.Equal(t, "high", findings[0].Severity)
}

func TestGobusterParser_APIEndpoint(t *testing.T) {
    raw := []byte(`/api/users (Status: 200) [Size: 150]`)
    findings, _ := parseGobusterOutput(raw)
    assert.Equal(t, "medium", findings[0].Severity)
    assert.Contains(t, findings[0].Title, "API endpoint")
}
```

**Step 2:** Implement per §4.6 above.

**Step 3: Commit**
```bash
git commit -m "feat(engine): add Gobuster runner for directory/file discovery with severity classification"
```

---

## 5. Tool 3 — Amass (Deep Subdomain Enumeration)

### 5.1 What It Adds Over Subfinder

Subfinder is **passive-only** — queries OSINT sources (Shodan, VirusTotal, crt.sh, etc.). Amass does everything Subfinder does **plus**:

- **Active DNS brute-forcing** against custom wordlists
- **Certificate transparency log analysis** (deeper than Subfinder)
- **ASN-based discovery** — finds subdomains by IP ownership patterns
- **Reverse DNS sweeps** on target IP ranges
- **DNS zone transfer attempts** (where allowed)
- **Permutation-based generation** (dev → dev-1 → dev-staging → etc.)

**Typical result:** Subfinder finds ~40-60% of subdomains. Amass finds ~80-90%.

### 5.2 Configuration

**Pinned version:** `v4.2.0`

**Install command:**
```bash
go install github.com/owasp-amass/amass/v4/...@v4.2.0
```

**Binary location:** `/usr/local/bin/amass`

### 5.3 CLI Invocation

**Passive mode (fast, ~30s):**
```bash
amass enum -passive -d target.com -silent -json /tmp/amass-out.jsonl
```

**Active mode (thorough, ~5-10 min):**
```bash
amass enum -active -d target.com \
    -brute \
    -w /opt/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
    -silent \
    -json /tmp/amass-out.jsonl \
    -timeout 10
```

**Key flags:**
- `-passive` — no active DNS queries (uses OSINT only)
- `-active` — enables DNS brute-forcing and cert transparency
- `-brute` — brute-force against wordlist
- `-timeout 10` — 10 minute hard cap
- `-silent` — suppress banner and progress output
- `-json` — structured output for parsing

### 5.4 Sample Output

JSON lines format:

```json
{"name":"api.target.com","domain":"target.com","addresses":[{"ip":"203.0.113.5","cidr":"203.0.113.0/24","asn":12345,"desc":"CLOUDFLARE"}],"tag":"cert","sources":["Crtsh"]}
{"name":"staging-internal.target.com","domain":"target.com","addresses":[{"ip":"203.0.113.8","cidr":"203.0.113.0/24","asn":12345,"desc":"CLOUDFLARE"}],"tag":"brute","sources":["DNS"]}
```

### 5.5 Parser Implementation

Amass output feeds into the **existing recon pipeline**. It doesn't produce `RawFinding` records directly — it produces subdomain strings that become scan targets.

```go
// internal/tools/amass.go
package tools

import (
    "bytes"
    "context"
    "encoding/json"
    "os/exec"
    "time"
)

type AmassOutput struct {
    Name   string `json:"name"`
    Domain string `json:"domain"`
    Tag    string `json:"tag"`    // "cert" | "brute" | "dns" | etc.
    Sources []string `json:"sources"`
}

// runAmass executes Amass in the specified mode and returns discovered subdomains.
// This is called from the recon pipeline (not as a standard ToolRunner) because
// Amass produces subdomain strings, not RawFindings.
func runAmass(ctx context.Context, domain string, mode string, timeout time.Duration) ([]string, error) {
    var args []string
    if mode == "passive" {
        args = []string{
            "enum", "-passive",
            "-d", domain,
            "-silent",
            "-timeout", "2",
        }
    } else {
        args = []string{
            "enum", "-active",
            "-d", domain,
            "-brute",
            "-w", "/opt/seclists/Discovery/DNS/subdomains-top1million-20000.txt",
            "-silent",
            "-timeout", "10",
        }
    }

    cmdCtx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()

    cmd := exec.CommandContext(cmdCtx, "/usr/local/bin/amass", args...)
    var stdout bytes.Buffer
    cmd.Stdout = &stdout

    if err := cmd.Run(); err != nil {
        // Amass exits non-zero when no results found — that's not an error for us
        if cmdCtx.Err() != context.DeadlineExceeded {
            // Only return error on timeout, not on no-results
        }
    }

    var subdomains []string
    seen := make(map[string]bool)

    for _, line := range bytes.Split(stdout.Bytes(), []byte("\n")) {
        if len(line) == 0 {
            continue
        }
        var ao AmassOutput
        if err := json.Unmarshal(line, &ao); err != nil {
            // Some Amass output is plain text; try that
            name := string(bytes.TrimSpace(line))
            if isValidSubdomain(name, domain) && !seen[name] {
                subdomains = append(subdomains, name)
                seen[name] = true
            }
            continue
        }
        if ao.Name != "" && !seen[ao.Name] {
            subdomains = append(subdomains, ao.Name)
            seen[ao.Name] = true
        }
    }
    return subdomains, nil
}

func isValidSubdomain(name, parentDomain string) bool {
    // Must end with parent domain
    if !strings.HasSuffix(name, "."+parentDomain) && name != parentDomain {
        return false
    }
    // Must be a valid hostname
    if _, err := net.LookupHost(name); err != nil {
        return true // Still valid format even if doesn't resolve right now
    }
    return true
}
```

### 5.6 Integration With Recon Pipeline

Update TOOL-ARCHITECTURE.md §8.2 — `BuildTargetList` now runs **Subfinder AND Amass in parallel**, merges results, then deduplicates:

```go
// internal/orchestrator/scan_executor.go (updated)

func (e *ScanExecutor) BuildTargetList(ctx context.Context, rootDomain string, cfg ReconConfig) ([]Target, error) {
    rootTarget := Target{
        URL:            "https://" + rootDomain,
        Domain:         rootDomain,
        TargetType:     "web",
        DomainVerified: true,
    }
    targets := []Target{rootTarget}

    e.publisher.Publish("recon_started", map[string]string{"domain": rootDomain})

    // Run Subfinder and Amass in parallel
    var subdomains []string
    var mu sync.Mutex
    var wg sync.WaitGroup

    wg.Add(2)
    go func() {
        defer wg.Done()
        subs, err := runSubfinder(ctx, rootDomain, 60*time.Second)
        if err == nil {
            mu.Lock()
            subdomains = append(subdomains, subs...)
            mu.Unlock()
        }
    }()

    go func() {
        defer wg.Done()
        // Use passive mode for quick/standard, active for deep
        amassMode := "passive"
        amassTimeout := 90 * time.Second
        if cfg.Depth == "deep" {
            amassMode = "active"
            amassTimeout = 10 * time.Minute
        }
        subs, err := runAmass(ctx, rootDomain, amassMode, amassTimeout)
        if err == nil {
            mu.Lock()
            subdomains = append(subdomains, subs...)
            mu.Unlock()
        }
    }()
    wg.Wait()

    // Deduplicate
    subdomains = dedupe(subdomains)

    // Apply tier limit
    if len(subdomains) > cfg.SubdomainLimit {
        subdomains = subdomains[:cfg.SubdomainLimit]
    }

    e.publisher.Publish("subdomains_discovered", map[string]interface{}{
        "count": len(subdomains),
        "sources": []string{"subfinder", "amass"},
    })

    // Continue with httpx probing as before...
}
```

### 5.7 Resource Considerations

**Amass is more resource-intensive than Subfinder.** To prevent abuse:

- **Active mode** only available to Business+ tier (DNS brute-forcing is noisy)
- **Passive mode** available to all tiers
- **Concurrent Amass executions per worker:** max 2 (compared to unlimited Subfinder)
- **Rate limiting:** 10 Amass runs per hour per organization

### 5.8 Implementation Task (M6.5.3)

**Files:**
- Create: `shieldscan-engine/internal/tools/amass.go`
- Modify: `shieldscan-engine/internal/orchestrator/scan_executor.go`
- Test: `shieldscan-engine/internal/tools/amass_test.go`

**Step 1: Failing tests**

```go
func TestAmassParser_ExtractsSubdomains(t *testing.T) {
    raw := `{"name":"api.target.com","domain":"target.com"}
{"name":"staging.target.com","domain":"target.com"}
not-valid-json
{"name":"admin.target.com","domain":"target.com"}`

    subs := parseAmassOutput([]byte(raw), "target.com")
    assert.ElementsMatch(t, []string{
        "api.target.com", "staging.target.com", "admin.target.com",
    }, subs)
}

func TestReconPipeline_MergesSubfinderAndAmass(t *testing.T) {
    // Mock both tools to return different subdomains
    mockSubfinder := func(ctx context.Context, domain string, timeout time.Duration) ([]string, error) {
        return []string{"api.target.com", "www.target.com"}, nil
    }
    mockAmass := func(ctx context.Context, domain string, mode string, timeout time.Duration) ([]string, error) {
        return []string{"api.target.com", "staging.target.com"}, nil // 1 overlap, 1 unique
    }

    subs := mergeReconResults([]string{"api.target.com", "www.target.com"},
                              []string{"api.target.com", "staging.target.com"})
    assert.ElementsMatch(t, []string{
        "api.target.com", "www.target.com", "staging.target.com",
    }, subs)
}
```

**Step 2:** Implement per §5.5-5.6 above.

**Step 3: Commit**
```bash
git commit -m "feat(engine): add Amass runner and integrate with Subfinder in recon pipeline"
```

---

## 6. Tool 4 — Kiterunner (API Route Discovery)

### 6.1 What It Fills

Your existing API scanning requires customers to provide an OpenAPI spec. Many don't have one, or their spec is incomplete. Kiterunner discovers API endpoints by:

- Brute-forcing paths with API-specific wordlists (different from web paths)
- Understanding REST conventions (`/api/v{n}/resource/{id}`)
- Testing HTTP methods (GET, POST, PUT, DELETE) per path
- Using **kitebuilder schemas** — pre-built wordlists of actual API route patterns from real applications

**Typical finding:** An `/api/internal/v2/admin/users` endpoint that isn't linked from anywhere, isn't in the OpenAPI spec, and has no authentication.

### 6.2 Configuration

**Pinned version:** `v1.0.2` (note: infrequently updated, last release 2022, but stable)

**Install command:**
```bash
# Binary download (preferred — kiterunner doesn't go install cleanly)
wget https://github.com/assetnote/kiterunner/releases/download/v1.0.2/kiterunner_1.0.2_linux_amd64.tar.gz
tar -xzf kiterunner_1.0.2_linux_amd64.tar.gz -C /tmp
mv /tmp/kr /usr/local/bin/kr
```

**Binary location:** `/usr/local/bin/kr`

### 6.3 Wordlist (Kitebuilder Schemas)

Kiterunner uses special `.kite` files. Pre-built schemas:
- `routes-large.kite` — 2.3GB, 2M routes (deep scans)
- `routes-small.kite` — 150MB, 160K routes (standard)

Download via provisioning script (see §11).

### 6.4 CLI Invocation

```bash
kr scan https://api.target.com \
    -A=/opt/kiterunner/routes-small.kite \
    -o json \
    --output-filename /tmp/kiterunner-out.json \
    --fail-status-codes 400,401,403,404,503 \
    --max-connection-per-host 5 \
    --timeout 10s \
    -q
```

**Key flags:**
- `-A` — assetlist (the .kite file)
- `--fail-status-codes 400,401,403,404,503` — ignore these (not real endpoints)
- `--max-connection-per-host 5` — don't hammer target
- `--timeout 10s` — per-request timeout

### 6.5 Sample Output

```json
{
  "url": "https://api.target.com/v2/admin/users",
  "method": "GET",
  "status_code": 200,
  "body_length": 2341,
  "reason": "found via kitebuilder schema: github.com/auth0/routes.json",
  "headers": {
    "content-type": "application/json",
    "server": "nginx/1.24.0"
  }
}
```

### 6.6 Parser Implementation

```go
// internal/tools/kiterunner.go
package tools

import (
    "bytes"
    "encoding/json"
    "fmt"
    "time"
)

type KiterunnerResult struct {
    URL        string            `json:"url"`
    Method     string            `json:"method"`
    StatusCode int               `json:"status_code"`
    BodyLength int               `json:"body_length"`
    Reason     string            `json:"reason"`
    Headers    map[string]string `json:"headers"`
}

func NewKiterunnerRunner() ToolRunner {
    return &NativeRunner{
        ToolName:     "kiterunner",
        ToolCategory: "api",
        BinaryPath:   "/usr/local/bin/kr",
        Timeout:      20 * time.Minute,
        BuildArgs: func(target Target, cfg ScanConfig) []string {
            wordlist := "/opt/kiterunner/routes-small.kite"
            if cfg.Depth == "deep" {
                wordlist = "/opt/kiterunner/routes-large.kite"
            }
            return []string{
                "scan", target.URL,
                "-A=" + wordlist,
                "-o", "json",
                "--fail-status-codes", "400,401,403,404,503",
                "--max-connection-per-host", "5",
                "--timeout", "10s",
                "-q",
            }
        },
        ParseOutput: parseKiterunnerOutput,
    }
}

func parseKiterunnerOutput(out []byte) ([]RawFinding, error) {
    var findings []RawFinding

    for _, line := range bytes.Split(out, []byte("\n")) {
        if len(line) == 0 {
            continue
        }
        var kr KiterunnerResult
        if err := json.Unmarshal(line, &kr); err != nil {
            continue
        }

        // Only report useful discoveries
        if kr.StatusCode < 200 || kr.StatusCode >= 400 {
            continue
        }

        severity := classifyKiterunnerFinding(kr)
        title := fmt.Sprintf("Undocumented API endpoint discovered: %s %s", kr.Method, kr.URL)

        description := fmt.Sprintf(
            "The API endpoint %s responds with HTTP %d (%d bytes). This endpoint was not in the provided OpenAPI spec and was discovered via route fingerprinting. "+
                "Review to ensure it requires proper authentication and authorization.",
            kr.URL, kr.StatusCode, kr.BodyLength,
        )

        findings = append(findings, RawFinding{
            EngineCategory: "api",
            Title:          title,
            Description:    description,
            Severity:       severity,
            FindingType:    "undocumented_api_endpoint",
            CWEID:          "CWE-1059", // Insufficient Technical Documentation
            OWASP:          "A05:2021", // Security Misconfiguration
            TargetURL:      kr.URL,
            Parameter:      kr.Method,
            Response:       fmt.Sprintf("HTTP %d, %d bytes", kr.StatusCode, kr.BodyLength),
        })
    }
    return findings, nil
}

func classifyKiterunnerFinding(kr KiterunnerResult) string {
    url := kr.URL
    // High severity: admin, internal, debug endpoints
    if strings.Contains(url, "/admin") || strings.Contains(url, "/internal") ||
       strings.Contains(url, "/debug") || strings.Contains(url, "/_") {
        return "high"
    }
    // Medium severity: 200 responses to POST/PUT/DELETE (write endpoints)
    if kr.Method == "POST" || kr.Method == "PUT" || kr.Method == "DELETE" {
        if kr.StatusCode == 200 || kr.StatusCode == 201 {
            return "medium"
        }
    }
    // Info: read-only endpoints
    return "info"
}
```

### 6.7 Integration With Existing API Scanning

Kiterunner runs **before** Nuclei API templates and ZAP API scan. The discovered endpoints feed into their target lists, improving downstream coverage.

### 6.8 Triggered When

| Scan Type | Runs? | Wordlist |
|---|---|---|
| `api` | **Yes** | Small (160K routes) |
| `full_spectrum` | **Yes** | Small for standard, Large for deep |

### 6.9 Implementation Task (M6.5.4)

**Files:**
- Create: `shieldscan-engine/internal/tools/kiterunner.go`
- Create: `shieldscan-engine/internal/tools/testdata/kiterunner_output.json`
- Test: `shieldscan-engine/internal/tools/kiterunner_test.go`

**Step 1: Failing tests**

```go
func TestKiterunnerParser_AdminEndpointHighSeverity(t *testing.T) {
    raw := []byte(`{"url":"https://api.target.com/admin/users","method":"GET","status_code":200,"body_length":2341}`)
    findings, err := parseKiterunnerOutput(raw)
    require.NoError(t, err)
    require.Len(t, findings, 1)
    assert.Equal(t, "high", findings[0].Severity)
    assert.Contains(t, findings[0].Title, "admin")
}

func TestKiterunnerParser_WriteEndpointMediumSeverity(t *testing.T) {
    raw := []byte(`{"url":"https://api.target.com/v1/users","method":"POST","status_code":201,"body_length":0}`)
    findings, _ := parseKiterunnerOutput(raw)
    assert.Equal(t, "medium", findings[0].Severity)
}

func TestKiterunnerParser_IgnoresFailedResponses(t *testing.T) {
    raw := []byte(`{"url":"https://api.target.com/v1/users","method":"GET","status_code":404,"body_length":0}
{"url":"https://api.target.com/v1/products","method":"GET","status_code":200,"body_length":150}`)
    findings, _ := parseKiterunnerOutput(raw)
    assert.Len(t, findings, 1) // Only the 200 response
}
```

**Step 2:** Implement per §6.6 above.

**Step 3: Commit**
```bash
git commit -m "feat(engine): add Kiterunner for undocumented API endpoint discovery"
```

---

## 7. Tool 5 — jsluice (JavaScript Bundle Analysis)

### 7.1 What It Fills

Modern SPAs (React, Vue, Angular, Next.js) bundle everything into JavaScript files served publicly. Those bundles often contain:

- **Hardcoded secrets** — Stripe keys, Firebase credentials, API keys
- **Internal API URLs** — undocumented endpoints
- **Admin URLs** — referenced in code but not linked from UI
- **GraphQL introspection** — leaked schema
- **Feature flags** — revealing unreleased features
- **Cloud config** — S3 bucket names, CDN URLs

Your existing stack doesn't analyze JS files. jsluice extracts all of the above.

### 7.2 Configuration

**Pinned version:** `v0.0.6`

**Install command:**
```bash
go install github.com/BishopFox/jsluice/cmd/jsluice@v0.0.6
```

**Binary location:** `/usr/local/bin/jsluice`

### 7.3 CLI Invocation

jsluice takes JavaScript files as input. Workflow:

1. **Discover JS files** from the target (via httpx during recon)
2. **Download each JS file** to a temp directory
3. **Run jsluice on each**

```bash
# Secret extraction
jsluice secrets /tmp/main.a3f2b.js

# URL extraction
jsluice urls /tmp/main.a3f2b.js
```

### 7.4 Sample Output

```json
# jsluice secrets output:
{"kind":"stripe","data":{"match":"pk_live_51Hb...","key":"stripe","stripe_key":"pk_live_51Hb..."},"filename":"main.a3f2b.js","severity":"high"}
{"kind":"firebase","data":{"match":"AIzaSyB...","key":"firebase"},"filename":"main.a3f2b.js","severity":"medium"}
{"kind":"aws_secret","data":{"match":"wJalrXUt..."},"filename":"main.a3f2b.js","severity":"critical"}

# jsluice urls output:
{"url":"/api/internal/v2/admin","filename":"main.a3f2b.js","type":"absolute-path"}
{"url":"https://s3.amazonaws.com/internal-bucket/","filename":"main.a3f2b.js","type":"absolute-url"}
{"url":"/graphql","filename":"main.a3f2b.js","type":"absolute-path"}
```

### 7.5 Parser Implementation

```go
// internal/tools/jsluice.go
package tools

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "os"
    "path/filepath"
    "strings"
    "time"
)

type JsluiceSecret struct {
    Kind     string            `json:"kind"`
    Data     map[string]string `json:"data"`
    Filename string            `json:"filename"`
    Severity string            `json:"severity"`
}

type JsluiceURL struct {
    URL      string `json:"url"`
    Filename string `json:"filename"`
    Type     string `json:"type"`
}

func NewJsluiceRunner() ToolRunner {
    return &NativeRunner{
        ToolName:     "jsluice",
        ToolCategory: "spa",
        BinaryPath:   "/usr/local/bin/jsluice",
        Timeout:      10 * time.Minute,
        // jsluice requires custom execution logic — override Execute
        Execute: executeJsluice,
    }
}

func executeJsluice(ctx context.Context, target Target, cfg ScanConfig) ([]RawFinding, error) {
    // Step 1: Discover JS files from target
    jsURLs, err := discoverJSFiles(ctx, target.URL)
    if err != nil {
        return nil, fmt.Errorf("JS discovery failed: %w", err)
    }

    if len(jsURLs) == 0 {
        return nil, nil // No JS files found; not an error
    }

    // Step 2: Download JS files to temp directory
    tempDir, err := os.MkdirTemp("", "jsluice-*")
    if err != nil {
        return nil, err
    }
    defer os.RemoveAll(tempDir)

    jsFiles := make(map[string]string) // filename -> URL
    for i, jsURL := range jsURLs {
        if i >= 20 {
            break // Cap at 20 JS files per scan
        }
        localPath, err := downloadJSFile(ctx, jsURL, tempDir, i)
        if err != nil {
            continue // Skip failed downloads
        }
        jsFiles[localPath] = jsURL
    }

    var allFindings []RawFinding

    // Step 3: Run jsluice on each file
    for localPath, originalURL := range jsFiles {
        // Secrets extraction
        secretFindings, _ := runJsluiceSecrets(ctx, localPath, originalURL)
        allFindings = append(allFindings, secretFindings...)

        // URL extraction
        urlFindings, _ := runJsluiceURLs(ctx, localPath, originalURL)
        allFindings = append(allFindings, urlFindings...)
    }

    return allFindings, nil
}

func discoverJSFiles(ctx context.Context, targetURL string) ([]string, error) {
    // Use httpx or internal crawler to find JS files
    // Implementation uses existing httpx integration from recon pipeline
    // Returns list of JS file URLs
    // ...
    return []string{}, nil // Placeholder — see full implementation in internal/tools/js_discovery.go
}

func downloadJSFile(ctx context.Context, url, destDir string, idx int) (string, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    client := &http.Client{Timeout: 30 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    if resp.StatusCode != 200 {
        return "", fmt.Errorf("HTTP %d", resp.StatusCode)
    }

    // Limit file size (some JS bundles are huge)
    maxSize := int64(10 * 1024 * 1024) // 10MB
    body, err := io.ReadAll(io.LimitReader(resp.Body, maxSize))
    if err != nil {
        return "", err
    }

    localPath := filepath.Join(destDir, fmt.Sprintf("bundle_%d.js", idx))
    if err := os.WriteFile(localPath, body, 0644); err != nil {
        return "", err
    }
    return localPath, nil
}

func runJsluiceSecrets(ctx context.Context, jsFile, originalURL string) ([]RawFinding, error) {
    cmd := exec.CommandContext(ctx, "/usr/local/bin/jsluice", "secrets", jsFile)
    var stdout bytes.Buffer
    cmd.Stdout = &stdout
    if err := cmd.Run(); err != nil {
        return nil, err
    }

    var findings []RawFinding
    for _, line := range bytes.Split(stdout.Bytes(), []byte("\n")) {
        if len(line) == 0 {
            continue
        }
        var secret JsluiceSecret
        if err := json.Unmarshal(line, &secret); err != nil {
            continue
        }

        severity := mapJsluiceSecretSeverity(secret.Kind)
        findings = append(findings, RawFinding{
            EngineCategory: "spa",
            Title:          fmt.Sprintf("Secret exposed in JavaScript bundle: %s", secret.Kind),
            Description: fmt.Sprintf(
                "A %s credential was found embedded in the public JavaScript file %s. "+
                    "Secrets in client-side code are visible to anyone who views the page source. "+
                    "This credential must be revoked and moved to a secure backend.",
                secret.Kind, originalURL,
            ),
            Severity:    severity,
            FindingType: fmt.Sprintf("js_secret_%s", secret.Kind),
            CWEID:       "CWE-798",
            OWASP:       "A07:2021",
            TargetURL:   originalURL,
            CodeSnippet: fmt.Sprintf("Match: %s (redacted in report)", redactSecret(secret.Data["match"])),
        })
    }
    return findings, nil
}

func runJsluiceURLs(ctx context.Context, jsFile, originalURL string) ([]RawFinding, error) {
    cmd := exec.CommandContext(ctx, "/usr/local/bin/jsluice", "urls", jsFile)
    var stdout bytes.Buffer
    cmd.Stdout = &stdout
    if err := cmd.Run(); err != nil {
        return nil, err
    }

    var findings []RawFinding
    for _, line := range bytes.Split(stdout.Bytes(), []byte("\n")) {
        if len(line) == 0 {
            continue
        }
        var jsurl JsluiceURL
        if err := json.Unmarshal(line, &jsurl); err != nil {
            continue
        }

        // Filter interesting URLs
        if !isInterestingURL(jsurl.URL) {
            continue
        }

        findings = append(findings, RawFinding{
            EngineCategory: "spa",
            Title:          fmt.Sprintf("Internal URL exposed in JavaScript: %s", jsurl.URL),
            Description: fmt.Sprintf(
                "The URL %s was found referenced in the public JavaScript file %s. "+
                    "This may reveal internal endpoints, admin interfaces, or backend architecture.",
                jsurl.URL, originalURL,
            ),
            Severity:    classifyURLSeverity(jsurl.URL),
            FindingType: "js_url_leak",
            CWEID:       "CWE-200", // Information Disclosure
            OWASP:       "A01:2021",
            TargetURL:   jsurl.URL,
        })
    }
    return findings, nil
}

func mapJsluiceSecretSeverity(kind string) string {
    switch kind {
    case "aws_secret", "stripe_live", "gcp_service_account":
        return "critical"
    case "stripe", "firebase", "slack", "github_token":
        return "high"
    case "google_api", "sendgrid":
        return "medium"
    default:
        return "low"
    }
}

func isInterestingURL(url string) bool {
    // Ignore CDN, fonts, images, etc.
    if strings.Contains(url, "fonts.googleapis.com") ||
        strings.Contains(url, "cdn.jsdelivr.net") ||
        strings.HasSuffix(url, ".png") ||
        strings.HasSuffix(url, ".jpg") ||
        strings.HasSuffix(url, ".svg") ||
        strings.HasSuffix(url, ".woff") ||
        strings.HasSuffix(url, ".woff2") {
        return false
    }
    // Interesting: admin, internal, api, debug, staging
    interestingKeywords := []string{"/admin", "/internal", "/debug", "/api", "/staging", "/dev"}
    for _, kw := range interestingKeywords {
        if strings.Contains(url, kw) {
            return true
        }
    }
    // Interesting: s3, blob storage
    if strings.Contains(url, "s3.amazonaws.com") ||
        strings.Contains(url, "blob.core.windows.net") ||
        strings.Contains(url, "storage.googleapis.com") {
        return true
    }
    return false
}

func classifyURLSeverity(url string) string {
    if strings.Contains(url, "/admin") || strings.Contains(url, "/internal") {
        return "high"
    }
    if strings.Contains(url, "s3.amazonaws.com") && strings.Contains(url, "internal") {
        return "high"
    }
    return "medium"
}

func redactSecret(secret string) string {
    if len(secret) < 10 {
        return "***"
    }
    return secret[:4] + "***" + secret[len(secret)-4:]
}
```

### 7.6 Triggered When

| Scan Type | Runs? |
|---|---|
| `quick` | No |
| `full_web` | **Yes** |
| `full_web_source` | **Yes** |
| `api` | No |
| `full_spectrum` | **Yes** |

### 7.7 Implementation Task (M6.5.5)

**Files:**
- Create: `shieldscan-engine/internal/tools/jsluice.go`
- Create: `shieldscan-engine/internal/tools/js_discovery.go`
- Test: `shieldscan-engine/internal/tools/jsluice_test.go`

**Step 1: Failing tests**

```go
func TestJsluiceSecretParser_CriticalAWSKey(t *testing.T) {
    raw := []byte(`{"kind":"aws_secret","data":{"match":"wJalrXUt..."},"filename":"main.js"}`)
    findings := parseJsluiceSecrets(raw, "https://target.com/main.js")
    require.Len(t, findings, 1)
    assert.Equal(t, "critical", findings[0].Severity)
    assert.Equal(t, "CWE-798", findings[0].CWEID)
}

func TestJsluiceURLFilter_IgnoresFonts(t *testing.T) {
    urls := []string{
        "https://fonts.googleapis.com/css",
        "https://target.com/admin/users",
        "https://cdn.jsdelivr.net/npm/react",
    }
    interesting := filterInterestingURLs(urls)
    assert.Equal(t, []string{"https://target.com/admin/users"}, interesting)
}

func TestJsluiceRedactSecret(t *testing.T) {
    assert.Equal(t, "AKIA***MPLE", redactSecret("AKIAIOSFODNN7EXAMPLE"))
    assert.Equal(t, "***", redactSecret("short"))
}
```

**Step 2:** Implement per §7.5 above.

**Step 3: Commit**
```bash
git commit -m "feat(engine): add jsluice for JavaScript bundle secret and URL extraction"
```

---

## 8. RawFinding Schema Extensions

### 8.1 New Field

Add to `RawFinding` struct in `internal/tools/runner.go`:

```go
type RawFinding struct {
    // ... existing fields ...

    // New field for trufflehog verified secrets
    SecretVerified *bool `json:"secret_verified,omitempty"`
    // nil for non-secret findings
    // &true for trufflehog-verified live secrets
    // &false for pattern-matched-only secrets
}
```

### 8.2 Database Migration

```sql
-- Add to existing migrations

ALTER TABLE raw_findings ADD COLUMN secret_verified BOOLEAN DEFAULT NULL;

CREATE INDEX idx_raw_findings_verified_secret
    ON raw_findings(scan_id)
    WHERE secret_verified = TRUE;
```

The partial index makes "show me all verified live secrets" queries fast.

### 8.3 AI Pipeline Awareness

Update the AI fix-generation prompt to emphasize when a secret is verified (SPECIFICATION.md §8.4):

```python
if vuln.secret_verified:
    prompt += """
    CRITICAL CONTEXT: This credential has been VERIFIED as currently active.
    Prioritize immediate rotation over mere removal from source code.
    The credential may already be compromised if this code has been
    public for any duration.
    """
```

### 8.4 UI Display

In the vulnerability detail component, verified secrets get a distinct badge:

```tsx
{vuln.secret_verified === true && (
    <Badge variant="critical">
        VERIFIED LIVE — Rotate Immediately
    </Badge>
)}
```

---

## 9. Scan Type Matrix Updates

Replaces TOOL-ARCHITECTURE.md §10.1.

```go
// internal/orchestrator/tool_router.go (updated)

var scanTypeMatrix = map[string]ScanTypeDefinition{
    "quick": {
        Tools: []string{
            "subfinder", "httpx", "nuclei_fast", "sslyze",
        },
        MaxDuration: 5 * time.Minute,
        MinTier:     "free",
    },

    "full_web": {
        Tools: []string{
            "subfinder", "amass_passive",     // recon — passive only
            "httpx",                           // probe
            "gobuster",                        // NEW: directory discovery
            "nuclei", "zap", "wapiti",         // DAST
            "nikto", "nmap",                   // infrastructure
            "sslyze",                          // TLS
            "jsluice",                         // NEW: SPA analysis
        },
        MaxDuration: 45 * time.Minute,
        MinTier:     "starter",
    },

    "full_web_source": {
        Tools: []string{
            "subfinder", "amass_passive",
            "httpx",
            "gobuster",
            "nuclei", "zap", "wapiti",
            "nikto", "nmap",
            "sslyze",
            "jsluice",
            "semgrep",                         // SAST
            "gitleaks", "trufflehog",          // NEW: secrets (both)
            "dependency_check",                // SCA
        },
        MaxDuration: 75 * time.Minute,
        MinTier:     "growth",
    },

    "api": {
        Tools: []string{
            "nuclei_api",
            "kiterunner",                      // NEW: endpoint discovery
            "zap_api",
            "corstest",
        },
        MaxDuration: 30 * time.Minute,
        MinTier:     "starter",
    },

    "mobile": {
        Tools:       []string{"mobsf"},
        MaxDuration: 15 * time.Minute,
        MinTier:     "growth",
    },

    "container": {
        Tools: []string{
            "trivy_image", "checkov",
        },
        MaxDuration: 10 * time.Minute,
        MinTier:     "growth",
    },

    "full_spectrum": {
        Tools: []string{
            "subfinder", "amass_active",      // NEW: active Amass for deep scans
            "httpx",
            "gobuster",                        // NEW
            "nuclei", "zap", "wapiti",
            "nikto", "nmap",
            "sslyze",
            "jsluice",                         // NEW
            "semgrep",
            "gitleaks", "trufflehog",          // NEW
            "dependency_check", "trivy",
            "kiterunner",                      // NEW
            "corstest", "checkov",
        },
        MaxDuration: 180 * time.Minute,
        MinTier:     "business",
    },
}
```

### 9.1 Scan Duration Impact

| Scan Type | Before | After | Change |
|---|---|---|---|
| quick | 5 min | 5 min | No change |
| full_web | 30 min | 45 min | +15 min (Gobuster + jsluice + Amass) |
| full_web_source | 60 min | 75 min | +15 min (+ trufflehog) |
| api | 20 min | 30 min | +10 min (Kiterunner) |
| mobile | 15 min | 15 min | No change |
| full_spectrum | 120 min | 180 min | +60 min (all new tools + active Amass) |

Customers can still choose `quick` for fast scans. The full-spectrum time increase is acceptable for business-tier customers doing deep audits.

---

## 10. Wordlist Management

Gobuster, Amass (active mode), and Kiterunner require wordlists. Centralize management.

### 10.1 SecLists Installation

```bash
# In provision-worker.sh
git clone --depth=1 https://github.com/danielmiessler/SecLists.git /opt/seclists
# Pin to specific commit for reproducibility:
cd /opt/seclists && git checkout 2025.09.release  # or latest stable tag
```

**Wordlists used by ShieldScan:**
- `/opt/seclists/Discovery/Web-Content/common.txt` (Gobuster quick)
- `/opt/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt` (Gobuster standard)
- `/opt/seclists/Discovery/Web-Content/directory-list-2.3-big.txt` (Gobuster deep)
- `/opt/seclists/Discovery/DNS/subdomains-top1million-20000.txt` (Amass active)

### 10.2 Kitebuilder Schemas

```bash
mkdir -p /opt/kiterunner
cd /opt/kiterunner

# Small schema (standard scans)
wget https://wordlists-cdn.assetnote.io/data/kiterunner/routes-small.kite
# Large schema (deep scans only)
wget https://wordlists-cdn.assetnote.io/data/kiterunner/routes-large.kite

chmod 644 routes-*.kite
```

### 10.3 Update Schedule

Wordlists don't need frequent updates. Schedule:
- **SecLists:** Monthly pull (security researchers add new patterns)
- **Kitebuilder schemas:** Quarterly (update with new API patterns)

Automated via cron on one worker; results replicated to all workers.

---

## 11. Provisioning Script Updates

Addition to `deploy/provision-worker.sh` (OPERATIONS-RUNBOOK.md §4.1):

```bash
# ─── Add after section 4 (Install native security tools) ────────

echo "Installing additional security tools (Tools 5 Addendum)..."

# trufflehog
go install github.com/trufflesecurity/trufflehog/v3@v3.88.0
mv /root/go/bin/trufflehog /usr/local/bin/trufflehog

# Gobuster
go install github.com/OJ/gobuster/v3@v3.6.2
mv /root/go/bin/gobuster /usr/local/bin/gobuster

# Amass
go install github.com/owasp-amass/amass/v4/...@v4.2.0
mv /root/go/bin/amass /usr/local/bin/amass

# Kiterunner (binary download — doesn't go install cleanly)
KR_VERSION=1.0.2
wget -q https://github.com/assetnote/kiterunner/releases/download/v${KR_VERSION}/kiterunner_${KR_VERSION}_linux_amd64.tar.gz -O /tmp/kr.tar.gz
tar -xzf /tmp/kr.tar.gz -C /tmp
mv /tmp/kr /usr/local/bin/kr
rm /tmp/kr.tar.gz

# jsluice
go install github.com/BishopFox/jsluice/cmd/jsluice@v0.0.6
mv /root/go/bin/jsluice /usr/local/bin/jsluice

# ─── Install wordlists ────────────────────────────────────────────

echo "Installing SecLists wordlists..."
if [ ! -d /opt/seclists ]; then
    git clone --depth=1 --branch=2025.09.release \
        https://github.com/danielmiessler/SecLists.git /opt/seclists
fi

echo "Installing Kiterunner schemas..."
mkdir -p /opt/kiterunner
cd /opt/kiterunner
if [ ! -f routes-small.kite ]; then
    wget -q https://wordlists-cdn.assetnote.io/data/kiterunner/routes-small.kite
fi
if [ ! -f routes-large.kite ]; then
    wget -q https://wordlists-cdn.assetnote.io/data/kiterunner/routes-large.kite
fi
chmod 644 /opt/kiterunner/routes-*.kite
cd -

# ─── Update verification ─────────────────────────────────────────

REQUIRED_BINARIES+=(
    "trufflehog" "gobuster" "amass" "kr" "jsluice"
)
```

### 11.1 Disk Space Impact

| Item | Size |
|---|---|
| New binaries (5 tools) | ~150MB |
| SecLists | ~1GB |
| Kitebuilder schemas | ~2.5GB |
| **Total added per worker** | **~3.7GB** |

Update worker specs in OPERATIONS-RUNBOOK.md §4.5:
- Minimum storage: 50GB → **55GB**
- Recommended: 100GB → **120GB**

---

## 12. Version Pinning Additions

Additions to VERSIONS.md §2.5 (replaces Native binaries table):

| Tool | Pinned Version | Installation Command |
|---|---|---|
| **trufflehog** ⭐ | **v3.88.0** | `go install github.com/trufflesecurity/trufflehog/v3@v3.88.0` |
| **Gobuster** ⭐ | **v3.6.2** | `go install github.com/OJ/gobuster/v3@v3.6.2` |
| **Amass** ⭐ | **v4.2.0** | `go install github.com/owasp-amass/amass/v4/...@v4.2.0` |
| **Kiterunner** ⭐ | **v1.0.2** | Binary download (see §11) |
| **jsluice** ⭐ | **v0.0.6** | `go install github.com/BishopFox/jsluice/cmd/jsluice@v0.0.6` |

### 12.1 Addition to verify-versions.sh

```bash
# Add to scripts/verify-versions.sh

echo ""
echo "Checking Tools 5 addendum..."

TRUFFLEHOG_LATEST=$(gh_latest "trufflesecurity/trufflehog")
compare "trufflehog" "v3.88.0" "$TRUFFLEHOG_LATEST"

GOBUSTER_LATEST=$(gh_latest "OJ/gobuster")
compare "Gobuster" "v3.6.2" "$GOBUSTER_LATEST"

AMASS_LATEST=$(gh_latest "owasp-amass/amass")
compare "Amass" "v4.2.0" "$AMASS_LATEST"

# Kiterunner — rarely updated, don't alarm if newer
KR_LATEST=$(gh_latest "assetnote/kiterunner")
if [ "$KR_LATEST" != "v1.0.2" ]; then
    echo -e "  ${YELLOW}!${NC} Kiterunner: pinned=v1.0.2, latest=$KR_LATEST (kiterunner is rarely updated; verify before upgrading)"
else
    echo -e "  ${GREEN}✓${NC} Kiterunner: v1.0.2 (current)"
fi

JSLUICE_LATEST=$(gh_latest "BishopFox/jsluice")
compare "jsluice" "v0.0.6" "$JSLUICE_LATEST"
```

### 12.2 Special Note on Kiterunner

Kiterunner has infrequent releases (last major: 2022). Do NOT auto-upgrade. Verify that any newer version:
- Still works with existing `.kite` schemas
- Hasn't changed CLI flags we depend on
- Is cryptographically signed by BishopFox

Track via manual review quarterly.

---

## 13. Implementation Milestone M6.5

Inserts between existing M6 (Native Tool Runners) and M7 (Persistent Docker Service Runners) in IMPLEMENTATION-PLAN.md.

### 13.1 Milestone Summary

**Goal:** Integrate 5 additional native tools (trufflehog, Gobuster, Amass, Kiterunner, jsluice) with full TDD coverage.

**Duration:** 7 engineer-days (1 week).

**Dependencies:** M6 complete (existing native runners working).

### 13.2 Task Breakdown

| Task | Description | Duration | Dependencies |
|---|---|---|---|
| M6.5.0 | Database migration: add `secret_verified` column | 0.5 day | M1.5 |
| M6.5.1 | Trufflehog runner + parser | 1 day | M6.5.0 |
| M6.5.2 | Gobuster runner + parser + classification | 1 day | M6 |
| M6.5.3 | Amass runner + recon pipeline integration | 2 days | M8 |
| M6.5.4 | Kiterunner runner + parser | 1 day | M6 |
| M6.5.5 | jsluice runner + JS discovery + parser | 1.5 days | M6 |
| M6.5.6 | Update scan type matrix + tool registry | 0.5 day | All above |
| M6.5.7 | Update provisioning script + wordlist setup | 0.5 day | All above |

**Total:** 8 task-days over 7 calendar days (some parallelization possible).

### 13.3 Task M6.5.0 — Schema Migration

**Files:**
- Create: `shieldscan-api/alembic/versions/xxx_add_secret_verified.py`
- Modify: `shieldscan-api/src/app/models/raw_findings.py`
- Modify: `shieldscan-engine/internal/tools/runner.go`

**Migration:**
```python
def upgrade():
    op.add_column('raw_findings',
        sa.Column('secret_verified', sa.Boolean(), nullable=True)
    )
    op.create_index(
        'idx_raw_findings_verified_secret',
        'raw_findings',
        ['scan_id'],
        postgresql_where=sa.text('secret_verified = TRUE'),
    )

def downgrade():
    op.drop_index('idx_raw_findings_verified_secret')
    op.drop_column('raw_findings', 'secret_verified')
```

**Commit:**
```bash
git commit -m "feat(db): add secret_verified column to raw_findings for trufflehog integration"
```

### 13.4 Task Integration Order

Execute in this order (respects dependencies):

```
Day 1: M6.5.0 (schema) + M6.5.1 (trufflehog)
Day 2-3: M6.5.2 (Gobuster) + M6.5.4 (Kiterunner)
Day 4-5: M6.5.3 (Amass + recon integration)
Day 5-6: M6.5.5 (jsluice)
Day 7: M6.5.6 (matrix update) + M6.5.7 (provisioning)
```

### 13.5 Post-Milestone Validation

After M6.5 complete, run integration test against a known vulnerable target:

```bash
# scripts/validate-m6-5.sh
#!/bin/bash

# Run full_web_source scan against OWASP Juice Shop
./shieldscan scan --target https://vulnerable-site.com \
    --type full_web_source \
    --source-repo /path/to/repo-with-secrets \
    --verbose

# Expected results:
# - At least 1 Gobuster finding (/admin or /.env)
# - At least 1 trufflehog verified finding
# - At least 5 subdomains from Amass+Subfinder
# - At least 1 jsluice URL leak
```

---

## 14. Cross-References to Updated Docs

### 14.1 Changes to SPECIFICATION.md

**Section §4 (Bounded Contexts):** Update Scan Execution context to include 5 new tools.

**Section §5 (Data Model):** Add `secret_verified` column to `raw_findings` table definition.

**Section §8 (AI Pipeline):** Update §8.4 (Fix Generation) to include verified secret context in Claude prompts.

**Section §13 (ADRs):** Add:
- **ADR-013:** Why we run Gitleaks AND trufflehog together (complementary, not redundant)
- **ADR-014:** Why Amass runs in passive mode by default (noise + abuse concerns)
- **ADR-015:** Why Kiterunner pinned to 2022 version (stability over freshness)

### 14.2 Changes to TOOL-ARCHITECTURE.md

**Section §5 (Complete Tool Inventory):** Replace with updated 24-tool table from §2 of this addendum.

**Section §6 (Native Tool Specifications):** Add subsections:
- §6.12: trufflehog
- §6.13: Gobuster
- §6.14: Amass
- §6.15: Kiterunner
- §6.16: jsluice

**Section §8 (Recon-First Pipeline):** Update to show Subfinder + Amass parallel execution.

**Section §10 (Scan Type Matrix):** Replace with updated matrix from §9 of this addendum.

### 14.3 Changes to IMPLEMENTATION-PLAN.md

Insert Milestone M6.5 between M6 and M7. Update timeline summary table:

| Week | Milestones | Duration |
|---|---|---|
| Week 3-4 | M6 (native runners) | Unchanged |
| **Week 4** | **M6.5 (Tools 5 Addendum)** | **+1 week** |
| Week 4-5 | M7 (Docker runners) | Shifted by 1 week |

### 14.4 Changes to OPERATIONS-RUNBOOK.md

**Section §4.1 (Fresh Worker Setup):** Add 5 new tool install commands + wordlist setup.

**Section §4.5 (Worker Specifications):** Update storage requirements (50→55GB min, 100→120GB recommended).

**Section §7.2 (Metrics):** Add new Prometheus metrics:
- `worker_tool_invocations_total{tool_name="trufflehog"}`
- `worker_tool_invocations_total{tool_name="gobuster"}`
- `worker_tool_invocations_total{tool_name="amass"}`
- `worker_tool_invocations_total{tool_name="kiterunner"}`
- `worker_tool_invocations_total{tool_name="jsluice"}`

**Section §12.2 (Cost):** Negligible (<$5/month). No change to budget.

### 14.5 Changes to VERSIONS.md

Add 5 new pinned versions to §2.5 per §12 above. Update `scripts/verify-versions.sh` per §12.1.

### 14.6 Changes to CONSTITUTION.md

**Section §6.2 (Scan Authorization):** Add:

> Kiterunner and Amass (active mode) perform aggressive enumeration that can trip intrusion detection systems. For Business+ tier customers, these tools are enabled by default. For Starter/Growth tier, they are opt-in with explicit acknowledgment that the target's hosting provider may flag the activity.

### 14.7 Changes to CLAUDE.md

**New Gotcha:** Add Gotcha 10:

> **Gotcha 10: Gitleaks and Trufflehog Run Together — Don't Skip the Dedup**
>
> Both Gitleaks and trufflehog are in `full_web_source` and `full_spectrum` scans. They will find overlapping findings for the same secret (e.g., same AWS key at same file:line). The AI pipeline's fingerprint-based dedup handles this — **do not remove Gitleaks thinking trufflehog replaces it**. Gitleaks catches pattern matches trufflehog doesn't try; trufflehog verifies which ones are live. Keep both. The dedup works because both tools populate the same CodeFile + CodeLine + finding_type, producing identical fingerprints.

**Update Gotcha 2 (Recon Runs Before Web Scans):** Note that Amass runs in parallel with Subfinder, both feeding into httpx.

### 14.8 Changes to ADDENDUM-CRITICAL-5.md

No direct changes. The public scan (Feature 1) uses only lightweight tools and does NOT run any of the 5 new tools (they're too slow for 60-second public scans).

---

## Appendix A: Expected Finding Volume Impact

With 5 new tools, expect **30-50% more findings per full-spectrum scan** on a typical SME website.

### A.1 Breakdown by Tool (typical SME scan)

| Tool | Typical findings per scan | Signal quality |
|---|---|---|
| trufflehog | 0-3 | Very high (verified live secrets) |
| Gobuster | 5-15 | High (exposed files are real issues) |
| Amass | 0 (produces targets, not findings) | — |
| Kiterunner | 0-5 | High (undocumented APIs = attack surface) |
| jsluice | 2-8 | Medium-high (secrets in JS are common) |

**New findings per scan:** 7-31 typically, skewing toward 10-15 for the average customer.

### A.2 Impact on AI Pipeline Cost

More findings = more AI embedding + correlation calls. Estimated cost increase:

| Tier | Per-scan cost before | Per-scan cost after |
|---|---|---|
| Starter | $0.25 | $0.32 |
| Growth | $0.60 | $0.78 |
| Business | $1.50 | $2.00 |

Adjust per-scan AI budgets accordingly (SPECIFICATION.md §9).

---

## Appendix B: Integration Testing Targets

**OWASP Juice Shop** remains the canonical test target. Additional vulnerable targets useful for M6.5 validation:

| Target | Purpose | URL |
|---|---|---|
| OWASP Juice Shop | General web vulnerabilities | https://juice-shop.herokuapp.com |
| VAmPI | Vulnerable API | https://github.com/erev0s/VAmPI |
| DVGA | Vulnerable GraphQL API | https://github.com/dolevf/Damn-Vulnerable-GraphQL-Application |
| tiredful-api | Vulnerable REST API | https://github.com/payatu/Tiredful-API |
| vulnerable-apk | Android app for MobSF testing | https://github.com/payatu/diva-android |

Set up self-hosted instances of these for integration testing in staging environment.

---

*End of Addendum. This document authorizes the 5 additional security scanning tools and their full integration into ShieldScan. Post-launch, additional tools should be evaluated through a standard ADR process rather than further addenda.*
