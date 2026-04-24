# ShieldScan — Software Versions & Verification

**Version:** 1.0
**Date:** 2026-04-20
**Status:** Authoritative — overrides any version specified in SPECIFICATION.md, TOOL-ARCHITECTURE.md, IMPLEMENTATION-PLAN.md, or OPERATIONS-RUNBOOK.md.

> **For Claude Code — READ THIS FIRST.** This document pins every tool, language, library, and Docker image used in ShieldScan to a current-as-of-2026-04-20 stable version. Before installing anything, Claude Code MUST run the verification step in §3 to confirm each version is still current and stable. If a pinned version has a newer stable release, follow the upgrade decision matrix in §4.

---

## Table of Contents

1. [Why Version Pinning Matters](#1-why-version-pinning-matters)
2. [Complete Pinned Version Matrix](#2-complete-pinned-version-matrix)
3. [Version Verification Procedure (MUST RUN FIRST)](#3-version-verification-procedure-must-run-first)
4. [Upgrade Decision Matrix](#4-upgrade-decision-matrix)
5. [Version Pin Locations in Code](#5-version-pin-locations-in-code)
6. [Update Schedule](#6-update-schedule)

---

## 1. Why Version Pinning Matters

Security scanning tools update frequently. A new Nuclei template for a just-disclosed CVE might ship tomorrow. At the same time, **unpinned versions cause three failures**:

1. **Production drift** — "it worked on dev" because dev had v3.7.1 and prod pulled v3.8.0 with a breaking CLI flag change
2. **Unreproducible scans** — a customer's "baseline" scan from last month can't be compared to today's scan if the scanner version changed
3. **Supply chain attacks** — `:latest` tags on Docker Hub have been hijacked

**The rule:** Pin everything. Pin Docker image digests, not just tags, where possible. Update deliberately on a schedule, not accidentally on a rebuild.

---

## 2. Complete Pinned Version Matrix

### 2.1 Core Languages & Runtimes

| Component | Pinned Version | Released | Notes |
|---|---|---|---|
| **Python** | **3.13.13** | 2026-04-07 | Python 3.13.13, released April 7, 2026, latest maintenance release of 3.13. 3.14 is newer but 3.13 has longer LTS. |
| **Go** | **1.26.2** | 2026-04-07 | go1.26.2 (released 2026-04-07) includes security fixes. Go 1.26 released Feb 2026. |
| **Node.js** | **22 LTS** | 2024-10-29 | Latest LTS ("Jod"). Use for frontend tooling. |
| **Ubuntu (server OS)** | **24.04 LTS** | 2024-04-25 | "Noble Numbat" — LTS support until April 2029. Ships Python 3.12 + Go 1.22 natively (we install newer versions manually). MobSF officially supports 24.04 as of 2024. **See Appendix C for 24.04-specific provisioning quirks.** |

### 2.2 Databases & Data Stores

| Component | Pinned Version | Released | Notes |
|---|---|---|---|
| **PostgreSQL** | **18.3** | 2026-02-26 | PostgreSQL 18.3, released Feb 26, 2026 — out-of-cycle release that fixes several regressions. Use 18.3 not 18.2 (regression fixes critical). |
| **Redis** | **8.6.2** | 2026-02 | 8.6.2 is the current latest stable tag on Docker Hub. Pin image digest for production. |
| **Qdrant** | **1.11.x** | 2026-02 | Check `github.com/qdrant/qdrant/releases` for latest at install time. |
| **Docker Engine** | **26.0+** | 2024+ | Any recent version works; used for building, not for production runtime. |

### 2.3 Python Libraries (shieldscan-api)

```toml
[tool.poetry.dependencies]
python = "^3.13"
fastapi = "^0.135.0"          # FastAPI 0.135+ requires Python 3.10+
uvicorn = {extras = ["standard"], version = "^0.32.0"}
sqlalchemy = "^2.0.36"
alembic = "^1.14.0"
asyncpg = "^0.30.0"
pydantic = "^2.9.0"
pydantic-settings = "^2.6.0"
redis = "^5.2.0"
qdrant-client = "^1.12.0"
anthropic = "^0.40.0"
openai = "^1.54.0"
stripe = "^11.0.0"
boto3 = "^1.35.0"
python-jose = {extras = ["cryptography"], version = "^3.3.0"}
bcrypt = "^5.0"                # direct bcrypt (not passlib) — see ADR-010
python-slugify = "^8.0"        # organization slug generation in identity orchestrator
tldextract = "^5.1"            # PSL-aware root-domain extraction for projects (M3 Task 3.1)
dnspython = "^2.7"             # async DNS resolver for domain verification (M3 Task 3.2)
cryptography = "^43.0.0"
weasyprint = "^62.3"
jinja2 = "^3.1.4"
httpx = "^0.28.0"
email-validator = "^2.2.0"
sse-starlette = "^2.1.0"
prometheus-fastapi-instrumentator = "^7.0.0"
sentry-sdk = {extras = ["fastapi"], version = "^2.17.0"}

[tool.poetry.group.dev.dependencies]
pytest = "^8.3.0"
pytest-asyncio = "^0.24.0"
pytest-cov = "^5.0.0"
ruff = "^0.8.0"
mypy = "^1.13.0"
fakeredis = "^2.35"            # hermetic Redis for rate-limiter + revocation tests
```

**Note:** Minor version floors (`^`) allow compatible upgrades. Exact pinning happens in `poetry.lock` — commit the lockfile.

### 2.4 Go Libraries (shieldscan-engine)

```go
// go.mod
module github.com/odyssey/shieldscan-engine

go 1.26

require (
    github.com/hibiken/asynq v0.25.1
    github.com/redis/go-redis/v9 v9.7.0
    github.com/docker/docker v27.3.1+incompatible
    github.com/lib/pq v1.10.9
    github.com/aws/aws-sdk-go-v2 v1.32.0
    github.com/aws/aws-sdk-go-v2/service/s3 v1.66.0
    github.com/stretchr/testify v1.9.0
    github.com/rs/zerolog v1.33.0
    github.com/spf13/cobra v1.8.1
    github.com/prometheus/client_golang v1.20.0
    github.com/getsentry/sentry-go v0.29.0
)
```

### 2.5 Security Scanning Tools (the 19)

**Layer 1 — Native binaries:**

| Tool | Pinned Version | Installation Command | Notes |
|---|---|---|---|
| **Nuclei** | **v3.7.1** | `go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@v3.7.1` | v3.7.1 released March 5, 2026, latest stable |
| **Semgrep** | **1.95.0** | `pip install semgrep==1.95.0` | Install via pip for Python runtime integration |
| **Subfinder** | **v2.6.7** | `go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@v2.6.7` | Check releases page for latest at install |
| **httpx** | **v1.6.10** | `go install github.com/projectdiscovery/httpx/cmd/httpx@v1.6.10` | |
| **Gitleaks** | **v8.21.2** | `go install github.com/gitleaks/gitleaks/v8@v8.21.2` | |
| **SSLyze** | **6.1.0** | `pip install sslyze==6.1.0` | |
| **Nikto** | **2.5.0** | `apt install nikto` | OS package — version varies by Ubuntu |
| **Wapiti** | **3.2.4** | `pip install wapiti3==3.2.4` | |
| **CORStest** | git HEAD | `git clone https://github.com/RUB-NDS/CORStest.git` | Pin to specific commit SHA |
| **OWASP Dep-Check** | **9.2.0** | Download from `github.com/jeremylong/DependencyCheck/releases` | |
| **Checkov** | **3.2.340** | `pip install checkov==3.2.340` | |

**Layer 2 — Persistent Docker services:**

| Service | Pinned Image | Notes |
|---|---|---|
| **MobSF** | `opensecurity/mobile-security-framework-mobsf:v4.4.6` | v4.4.6 (March 2026) patched an SQL injection vulnerability in the SQLite DB viewer component — use v4.4.6 or newer, **never use :latest in production** |
| **ZAP** | `zaproxy/zap-stable:2.16.0` | Stable-track preferred over weekly |
| **Trivy** | `aquasec/trivy:0.58.0` | Pin minor version |
| **SQLMap** | `paoloo/sqlmap:1.9` | Community image; pin and verify before use |
| **Nmap** | `instrumentisto/nmap:7.95` | |

**CRITICAL:** In production, replace all `:tag` with `:tag@sha256:<digest>` to prevent image substitution attacks. Example:
```yaml
image: opensecurity/mobile-security-framework-mobsf:v4.4.6@sha256:abc123...
```

Get digests via: `docker inspect --format='{{index .RepoDigests 0}}' <image>`

### 2.6 Frontend (shieldscan-web)

```json
{
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.28.0",
    "@tanstack/react-query": "^5.60.0",
    "axios": "^1.7.0",
    "tailwindcss": "^3.4.15",
    "vite": "^5.4.11",
    "typescript": "^5.6.0",
    "react-dropzone": "^14.3.0",
    "react-syntax-highlighter": "^15.6.0",
    "recharts": "^2.13.0",
    "@microsoft/fetch-event-source": "^2.0.1",
    "lucide-react": "^0.468.0"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.0",
    "vitest": "^2.1.0",
    "@testing-library/react": "^16.1.0",
    "@testing-library/user-event": "^14.5.0",
    "eslint": "^9.15.0"
  }
}
```

**Note:** We're intentionally staying on React 18 (not React 19) for ecosystem stability. Upgrade to React 19 after major libraries (TanStack Query, etc.) have mature React 19 support.

### 2.7 External SaaS / Cloud

| Service | Pinned API Version | Notes |
|---|---|---|
| **Anthropic (Claude API)** | `2023-06-01` (API version header) | Models: `claude-opus-4-7`, `claude-sonnet-4-6`, `claude-haiku-4-5-20251001` |
| **OpenAI Embeddings** | `text-embedding-3-small` | Cheapest per-token embedding model |
| **Stripe** | `2024-11-20.acacia` | Pin API version in code, don't rely on account default |
| **Cloudflare R2** | S3-compatible | Use AWS SDK v2 |
| **SendGrid** | v3 API | Or Resend (better DX) |

---

## 3. Version Verification Procedure (MUST RUN FIRST)

**Before ANY installation, Claude Code runs this verification script.** It confirms each pinned version either:
- Is still the current stable version, OR
- Has been superseded by a newer stable that should be evaluated

### 3.1 Verification Script — `scripts/verify-versions.sh`

```bash
#!/bin/bash
# ShieldScan Version Verification
# Run BEFORE any installation to confirm pinned versions are still current
# Usage: ./scripts/verify-versions.sh

set -e
BOLD='\033[1m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m' # No Color

echo -e "${BOLD}ShieldScan Version Verification${NC}"
echo "==============================="
echo ""

# Helper: fetch latest GitHub release tag
gh_latest() {
    curl -s "https://api.github.com/repos/$1/releases/latest" | grep -Po '"tag_name": "\K[^"]+' || echo "unknown"
}

# Helper: compare versions and print result
compare() {
    local name=$1
    local pinned=$2
    local latest=$3

    if [ "$pinned" = "$latest" ]; then
        echo -e "  ${GREEN}✓${NC} $name: $pinned (current)"
    else
        echo -e "  ${YELLOW}!${NC} $name: pinned=$pinned, latest=$latest ${YELLOW}(review)${NC}"
    fi
}

echo "Checking core languages..."
PY_LATEST=$(curl -s https://www.python.org/api/v2/downloads/release/?is_published=true | python3 -c "import json,sys; data=json.load(sys.stdin); latest=[r for r in data if '3.13' in r['name']]; print(latest[0]['name'].replace('Python ',''))" 2>/dev/null || echo "3.13.13")
compare "Python" "3.13.13" "$PY_LATEST"

GO_LATEST=$(curl -s "https://go.dev/VERSION?m=text" | head -1 | sed 's/go//')
compare "Go" "1.26.2" "$GO_LATEST"

PG_LATEST=$(curl -s https://www.postgresql.org/versions.json | python3 -c "import json,sys; print(json.load(sys.stdin)[0]['latestMinor'])" 2>/dev/null || echo "unknown")
compare "PostgreSQL" "18.3" "$PG_LATEST"

echo ""
echo "Checking security scanning tools..."

NUCLEI_LATEST=$(gh_latest "projectdiscovery/nuclei")
compare "Nuclei" "v3.7.1" "$NUCLEI_LATEST"

SUBFINDER_LATEST=$(gh_latest "projectdiscovery/subfinder")
compare "Subfinder" "v2.6.7" "$SUBFINDER_LATEST"

HTTPX_LATEST=$(gh_latest "projectdiscovery/httpx")
compare "httpx" "v1.6.10" "$HTTPX_LATEST"

GITLEAKS_LATEST=$(gh_latest "gitleaks/gitleaks")
compare "Gitleaks" "v8.21.2" "$GITLEAKS_LATEST"

MOBSF_LATEST=$(gh_latest "MobSF/Mobile-Security-Framework-MobSF")
compare "MobSF" "v4.4.6" "$MOBSF_LATEST"

TRIVY_LATEST=$(gh_latest "aquasecurity/trivy")
compare "Trivy" "v0.58.0" "$TRIVY_LATEST"

ZAP_LATEST=$(gh_latest "zaproxy/zaproxy")
compare "ZAP" "2.16.0" "$ZAP_LATEST"

echo ""
echo "Checking Python packages (PyPI)..."

check_pypi() {
    local name=$1
    local pinned=$2
    local latest=$(curl -s "https://pypi.org/pypi/$name/json" | python3 -c "import json,sys; print(json.load(sys.stdin)['info']['version'])" 2>/dev/null || echo "unknown")
    compare "$name" "$pinned" "$latest"
}

check_pypi "fastapi" "0.135.0"
check_pypi "sqlalchemy" "2.0.36"
check_pypi "pydantic" "2.9.0"
check_pypi "anthropic" "0.40.0"
check_pypi "semgrep" "1.95.0"
check_pypi "sslyze" "6.1.0"
check_pypi "checkov" "3.2.340"
check_pypi "wapiti3" "3.2.4"

echo ""
echo -e "${BOLD}Legend:${NC}"
echo -e "  ${GREEN}✓${NC} Version current — proceed with installation"
echo -e "  ${YELLOW}!${NC} Newer version available — see §4 Upgrade Decision Matrix in VERSIONS.md"
echo ""
echo "If any tool shows YELLOW, check the changelog before upgrading."
echo "Security patches in pinned versions should ALWAYS be applied."
```

### 3.2 Running Verification

Claude Code runs this **before** the provisioning script and **before** `poetry install` / `go mod download`:

```bash
cd shieldscan-api  # or shieldscan-engine
chmod +x scripts/verify-versions.sh
./scripts/verify-versions.sh
```

### 3.3 What to Do With Results

| Result | Action |
|---|---|
| All GREEN | Proceed with installation using pinned versions |
| Some YELLOW (patch bumps, e.g., 3.13.13 → 3.13.14) | Apply — patch releases contain security fixes |
| Some YELLOW (minor bumps, e.g., 0.135.0 → 0.136.0) | Review changelog, apply if non-breaking |
| Some YELLOW (major bumps, e.g., Python 3.13 → 3.14) | Do NOT auto-upgrade. Add as tech debt ticket. |
| Any RED / error | Stop. Report to user before proceeding. |

---

## 4. Upgrade Decision Matrix

When verification shows newer versions available:

### 4.1 For Security Scanning Tools (Nuclei, MobSF, etc.)

**Always upgrade to latest stable** — these tools need current vulnerability signatures. Outdated scanners = missed CVEs = angry customers.

**Exception:** If a tool release is < 7 days old, wait. Fresh releases sometimes have regressions (as PostgreSQL 18.2 demonstrated in Feb 2026, requiring an out-of-cycle 18.3 fix).

### 4.2 For Core Languages (Python, Go)

**Patch releases (3.13.12 → 3.13.13):** Upgrade immediately. Contains security fixes.

**Minor releases (3.13 → 3.14):** Wait 3 months after release. Verify all dependencies support new version.

**Major releases (3.x → 4.x):** Plan as a dedicated project. Budget 1-2 weeks minimum.

### 4.3 For Databases (PostgreSQL, Redis)

**Patch releases (18.3 → 18.4):** Upgrade during next maintenance window.

**Major releases (PostgreSQL 18 → 19):** Plan carefully. Test in staging for 2 weeks minimum. Run pg_dump backup immediately before upgrade.

### 4.4 For Python/Go Libraries

**Follow SemVer:**
- Patch (x.y.z → x.y.z+1): Auto-upgrade via Dependabot
- Minor (x.y.z → x.y+1.0): Review changelog weekly, apply non-breaking
- Major (x.y.z → x+1.0.0): Breaking change — plan migration

### 4.5 For Frontend Libraries

React 18 → 19, TanStack Query v5 → v6, etc.: **Stay 1 major version behind cutting edge** to ensure ecosystem compatibility.

---

## 5. Version Pin Locations in Code

When updating versions, you must update **every** location. Forgetting one causes the drift problem.

### 5.1 Python (shieldscan-api)

| File | Pins |
|---|---|
| `pyproject.toml` | All Python library versions |
| `poetry.lock` | **Committed to git** — exact resolved versions |
| `Dockerfile` | Python base image tag |
| `.python-version` | Local Python version for pyenv |
| `scripts/verify-versions.sh` | Pinned versions for verification |
| `.github/workflows/*.yml` | Python version in CI setup |
| `docs/SPECIFICATION.md` § Tech Stack | Human-readable version list |

### 5.2 Go (shieldscan-engine)

| File | Pins |
|---|---|
| `go.mod` | Go version + direct dependencies |
| `go.sum` | **Committed** — exact module checksums |
| `Dockerfile` | Go base image tag |
| `deploy/provision-worker.sh` | All security tool versions |
| `scripts/verify-versions.sh` | Pinned versions for verification |
| `deploy/docker-compose.services.yml` | Docker image tags (MobSF, ZAP, Trivy, SQLMap) |

### 5.3 Frontend (shieldscan-web)

| File | Pins |
|---|---|
| `package.json` | All JS library versions |
| `package-lock.json` or `pnpm-lock.yaml` | **Committed** — exact resolved versions |
| `.nvmrc` | Node.js version |
| `Dockerfile` | Node base image |

### 5.4 Infrastructure

| File | Pins |
|---|---|
| `deploy/docker-compose.services.yml` | All Docker image tags |
| `deploy/provision-worker.sh` | OS package versions |
| `.github/workflows/*.yml` | Action versions (uses: `actions/checkout@v4`) |
| `Dockerfile` (each service) | Base image tag |

---

## 6. Update Schedule

### 6.1 Regular Cadence

| Type | Cadence | Who |
|---|---|---|
| Security tool versions | **Weekly** | Automated via Dependabot |
| Python/Go library patches | **Weekly** | Dependabot auto-PR, human merges |
| Python/Go minor versions | **Monthly** | Review + merge |
| Docker image digests | **Monthly** | Pull latest, re-pin digest |
| Core language versions | **Quarterly** | Planned project |
| Database major versions | **Annually** | Major project with staging validation |

### 6.2 Emergency Updates

**Immediate (< 24 hours):** CVE with CVSS ≥ 7.0 in any dependency, or public exploit available.

**Fast track (< 7 days):** CVE with CVSS 4.0-6.9, or maintainer security advisory.

**Normal:** All other updates follow regular cadence.

### 6.3 Update Procedure

```
1. Dependabot opens PR
2. CI runs full test suite
3. Human reviews changelog (look for breaking changes, deprecations)
4. Merge to staging branch
5. Monitor staging for 24 hours
6. Merge to main → auto-deploy to production
7. Monitor production Sentry for 48 hours
8. Update VERSIONS.md with new pinned version
```

---

## Appendix A: Version Sources

Authoritative sources for version checks:

| Component | Source |
|---|---|
| Python | https://www.python.org/downloads/ |
| Go | https://go.dev/dl/ |
| PostgreSQL | https://www.postgresql.org/download/ |
| Redis | https://redis.io/download |
| Qdrant | https://github.com/qdrant/qdrant/releases |
| Nuclei | https://github.com/projectdiscovery/nuclei/releases |
| MobSF | https://github.com/MobSF/Mobile-Security-Framework-MobSF/releases |
| ZAP | https://github.com/zaproxy/zaproxy/releases |
| Trivy | https://github.com/aquasecurity/trivy/releases |
| FastAPI | https://pypi.org/project/fastapi/ |
| SQLAlchemy | https://pypi.org/project/sqlalchemy/ |
| Anthropic SDK | https://pypi.org/project/anthropic/ |

---

## Appendix B: Override Notice

**Claude Code:** If anything in `SPECIFICATION.md`, `TOOL-ARCHITECTURE.md`, `IMPLEMENTATION-PLAN.md`, or `OPERATIONS-RUNBOOK.md` specifies a different version than this document, **this document wins**. Those documents were written first and contain older version references for illustration. Always use versions from VERSIONS.md.

Specifically, the following version references in the other 4 docs are superseded:

- "Python ^3.12" → Use Python 3.13.13
- "FastAPI ^0.110.0" → Use FastAPI ^0.135.0
- "Pydantic ^2.6.0" → Use Pydantic ^2.9.0
- "SQLAlchemy ^2.0.25" → Use SQLAlchemy ^2.0.36
- "Nuclei v3.2.0" → Use Nuclei v3.7.1
- "Go 1.22" → Use Go 1.26
- "PostgreSQL 16" → Use PostgreSQL 18.3
- "Redis 7" → Use Redis 8.6.2
- "anthropic ^0.18.0" → Use anthropic ^0.40.0
- "openai ^1.12.0" → Use openai ^1.54.0
- "stripe ^8.0.0" → Use stripe ^11.0.0
- "Ubuntu 22.04 LTS" → Use Ubuntu 24.04 LTS (see Appendix C)
- All other unpinned or older version specs

---

## Appendix C: Ubuntu 24.04 LTS Provisioning Notes

**Target OS:** Ubuntu Server 24.04 LTS ("Noble Numbat"), 64-bit, Long-Term Support until April 2029.

Ubuntu 24.04 introduces several changes that affect our provisioning scripts. This section supersedes OPERATIONS-RUNBOOK.md §4.1 where specified.

### C.1 What 24.04 Gives You For Free

| Component | 24.04 Default | Our Pin | Action |
|---|---|---|---|
| Python 3 | 3.12.3 | 3.13.13 | Install 3.13 via deadsnakes PPA |
| Go | 1.22 | 1.26.2 | Manual download (unchanged from 22.04) |
| OpenJDK | 17 (via apt) | 17 | Use apt default |
| Git | 2.43 | Any recent | Use apt default |
| Docker | N/A (not installed) | 26+ | Install via official repo |
| Node.js | 18.19 | 22 LTS | Install via NodeSource or nvm |

### C.2 The PEP 668 Gotcha — CRITICAL

**Problem:** Ubuntu 24.04 enforces PEP 668. Running `pip install <package>` globally fails with:

```
error: externally-managed-environment
× This environment is externally managed
╰─> To install Python packages system-wide, try apt install
    python3-xyz, where xyz is the package you are trying to
    install.
```

**Solution: Use `pipx` for CLI tools, venv for app dependencies.**

`pipx` installs each CLI tool in its own isolated venv and exposes the binary on PATH. This is exactly what we want for our Python-based security tools (Semgrep, SSLyze, Wapiti, Checkov, mobsf CLI, etc.).

**Install pipx first:**
```bash
apt-get install -y pipx
pipx ensurepath
```

### C.3 Updated Install Commands (Replaces OPERATIONS-RUNBOOK.md §4.1)

**Python-based CLI tools — use pipx instead of pip:**

```bash
# OLD (22.04 way — DON'T use on 24.04):
pip3 install --no-cache-dir sslyze==6.1.0
pip3 install --no-cache-dir semgrep==1.95.0
pip3 install --no-cache-dir wapiti3==3.2.4
pip3 install --no-cache-dir checkov==3.2.340

# NEW (24.04 way — use this):
pipx install sslyze==6.1.0
pipx install semgrep==1.95.0
pipx install wapiti3==3.2.4
pipx install checkov==3.2.340

# Verify installations
pipx list
```

Each tool gets its own venv under `/root/.local/pipx/venvs/<tool>/`. Binaries are symlinked to `/root/.local/bin/` which `pipx ensurepath` added to PATH.

**For worker systemd service:** Add `/root/.local/bin` to the service's PATH environment variable, OR symlink the binaries to `/usr/local/bin/`:

```bash
ln -sf /root/.local/bin/sslyze /usr/local/bin/sslyze
ln -sf /root/.local/bin/semgrep /usr/local/bin/semgrep
ln -sf /root/.local/bin/wapiti /usr/local/bin/wapiti
ln -sf /root/.local/bin/checkov /usr/local/bin/checkov
```

### C.4 Installing Python 3.13 on 24.04

Ubuntu 24.04 ships Python 3.12.3. ShieldScan pins Python 3.13.13 for the API. Three options:

**Option 1 — deadsnakes PPA (simplest, recommended for production):**
```bash
add-apt-repository ppa:deadsnakes/ppa -y
apt-get update
apt-get install -y python3.13 python3.13-venv python3.13-dev
# Use 'python3.13' explicitly; don't replace system 'python3'
python3.13 --version  # Should show 3.13.x
```

**Option 2 — pyenv (most flexible, good for dev):**
```bash
curl https://pyenv.run | bash
# Add to .bashrc
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
pyenv install 3.13.13
pyenv global 3.13.13
```

**Option 3 — Build from source (not recommended):**
Avoid unless you have a specific reason. Takes 15+ minutes, harder to update.

**For production workers:** Use Option 1 (deadsnakes). It's simpler to automate, updates via apt, and doesn't require per-user shell configuration.

**For the shieldscan-api Poetry project:**
```bash
cd shieldscan-api
poetry env use python3.13
poetry install
```

### C.5 Docker Installation on 24.04

Same commands as 22.04 (Docker supports 24.04 officially):

```bash
# Add Docker's official GPG key
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify
docker --version  # Should show 26.x or 27.x
```

### C.6 Node.js 22 LTS on 24.04

24.04 ships Node.js 18.19. For Node 22:

```bash
# NodeSource LTS repo
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs

node --version  # Should show v22.x
npm --version
```

### C.7 Firewall (ufw) — Same Commands

UFW works identically on 24.04. Use the OPERATIONS-RUNBOOK.md §4.1 firewall config unchanged.

### C.8 systemd and Service Management — Same

24.04 uses systemd 255 (22.04 used 249). Service files are fully compatible. Our `shieldscan-worker.service` from OPERATIONS-RUNBOOK.md §4.1 works unchanged — **but add `/root/.local/bin` to the PATH** if using pipx-installed tools:

```ini
[Service]
Type=simple
User=shieldscan
WorkingDirectory=/opt/shieldscan-engine
EnvironmentFile=/opt/shieldscan-engine/.env
Environment="PATH=/root/.local/bin:/usr/local/bin:/usr/bin:/bin"
ExecStart=/opt/shieldscan-engine/worker
Restart=always
RestartSec=5s
```

### C.9 Summary: Complete Updated provision-worker.sh Structure

The structure stays the same as OPERATIONS-RUNBOOK.md §4.1 with these specific substitutions:

```bash
#!/bin/bash
# provision-worker.sh — UPDATED FOR UBUNTU 24.04 LTS

set -euo pipefail

# Check we're on 24.04
if ! grep -q "Ubuntu 24.04" /etc/os-release; then
    echo "ERROR: This script requires Ubuntu 24.04 LTS"
    exit 1
fi

# ─── 1. System updates + base tools ─────────────────────────────
apt-get update
apt-get install -y \
    curl wget git build-essential \
    python3-pip python3-venv \
    pipx \
    software-properties-common \
    openjdk-17-jre-headless \
    nmap-common \
    ca-certificates gnupg lsb-release \
    ufw fail2ban \
    htop jq unzip

pipx ensurepath

# ─── 2. Install Python 3.13 via deadsnakes ──────────────────────
add-apt-repository ppa:deadsnakes/ppa -y
apt-get update
apt-get install -y python3.13 python3.13-venv python3.13-dev

# ─── 3. Install Go 1.26.2 ───────────────────────────────────────
GO_VERSION=1.26.2
wget https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz
rm -rf /usr/local/go
tar -C /usr/local -xzf go${GO_VERSION}.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin:/root/go/bin' >> /root/.bashrc
export PATH=$PATH:/usr/local/go/bin:/root/go/bin

# ─── 4. Install Docker (official repo) ──────────────────────────
# (same commands as §C.5 above)

# ─── 5. Install Python CLI security tools via pipx ──────────────
pipx install sslyze==6.1.0
pipx install semgrep==1.95.0
pipx install wapiti3==3.2.4
pipx install checkov==3.2.340

# Symlink pipx binaries to /usr/local/bin for systemd services
for tool in sslyze semgrep wapiti checkov; do
    ln -sf /root/.local/bin/$tool /usr/local/bin/$tool
done

# ─── 6. Install Go-based security tools ─────────────────────────
go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@v3.7.1
go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@v2.6.7
go install github.com/projectdiscovery/httpx/cmd/httpx@v1.6.10
go install github.com/gitleaks/gitleaks/v8@v8.21.2
go install github.com/trufflesecurity/trufflehog/v3@v3.88.0
go install github.com/OJ/gobuster/v3@v3.6.2
go install github.com/owasp-amass/amass/v4/...@v4.2.0
go install github.com/BishopFox/jsluice/cmd/jsluice@v0.0.6

# Move Go binaries to /usr/local/bin
for bin in nuclei subfinder httpx gitleaks trufflehog gobuster amass jsluice; do
    mv /root/go/bin/$bin /usr/local/bin/$bin
done

# ─── 7. Kiterunner (manual download) ────────────────────────────
KR_VERSION=1.0.2
wget -q https://github.com/assetnote/kiterunner/releases/download/v${KR_VERSION}/kiterunner_${KR_VERSION}_linux_amd64.tar.gz -O /tmp/kr.tar.gz
tar -xzf /tmp/kr.tar.gz -C /tmp
mv /tmp/kr /usr/local/bin/kr
rm /tmp/kr.tar.gz

# ─── 8. apt-installed tools ─────────────────────────────────────
apt-get install -y nikto
ln -sf /usr/bin/nikto /usr/local/bin/nikto

# ─── 9. OWASP Dependency-Check (jar wrapper) ────────────────────
DC_VERSION=9.2.0
wget -q https://github.com/jeremylong/DependencyCheck/releases/download/v${DC_VERSION}/dependency-check-${DC_VERSION}-release.zip
unzip -q dependency-check-${DC_VERSION}-release.zip -d /opt/
ln -sf /opt/dependency-check/bin/dependency-check.sh /usr/local/bin/dependency-check
rm dependency-check-${DC_VERSION}-release.zip

# ─── 10. CORStest (git clone) ───────────────────────────────────
git clone https://github.com/RUB-NDS/CORStest.git /opt/corstest
chmod +x /opt/corstest/corstest.py
ln -sf /opt/corstest/corstest.py /usr/local/bin/corstest

# ─── 11. Wordlists (SecLists + Kitebuilder) ─────────────────────
git clone --depth=1 --branch=2025.09.release \
    https://github.com/danielmiessler/SecLists.git /opt/seclists

mkdir -p /opt/kiterunner
wget -q https://wordlists-cdn.assetnote.io/data/kiterunner/routes-small.kite \
    -O /opt/kiterunner/routes-small.kite
wget -q https://wordlists-cdn.assetnote.io/data/kiterunner/routes-large.kite \
    -O /opt/kiterunner/routes-large.kite

# ─── 12. Docker images for persistent services ──────────────────
docker pull opensecurity/mobile-security-framework-mobsf:v4.4.6
docker pull zaproxy/zap-stable:2.16.0
docker pull aquasec/trivy:0.58.0
docker pull paoloo/sqlmap:1.9
docker pull instrumentisto/nmap:7.95

# ─── 13. Verify all binaries ────────────────────────────────────
REQUIRED_BINARIES=(
    # Go-based
    nuclei subfinder httpx gitleaks trufflehog gobuster amass jsluice kr
    # Python-based (via pipx)
    sslyze semgrep wapiti checkov
    # Other
    nikto corstest dependency-check
    # System
    docker
)
echo "Verifying installed binaries..."
for bin in "${REQUIRED_BINARIES[@]}"; do
    if ! command -v "$bin" &> /dev/null; then
        echo "ERROR: $bin not found in PATH"
        exit 1
    fi
    echo "  ✓ $bin"
done

# ─── 14. Firewall, systemd service, worker user ─────────────────
# (unchanged from OPERATIONS-RUNBOOK.md §4.1)

echo "=== Ubuntu 24.04 provisioning complete ==="
```

### C.10 Differences Summary Table

| Aspect | 22.04 | 24.04 | Change Required |
|---|---|---|---|
| pip install | Works | **Fails (PEP 668)** | Use pipx |
| Default Python | 3.10 | 3.12 | Install 3.13 via deadsnakes |
| Default Go | 1.18 | 1.22 | Manual install 1.26 (same as 22.04) |
| Default Docker | Not installed | Not installed | Same install (Docker repo) |
| Default Node | 12.x | 18.19 | Install 22 via NodeSource |
| OpenSSL | 3.0.2 | 3.0.13 | None (compatible) |
| systemd | 249 | 255 | None |
| iptables | legacy default | nftables default | None (Docker handles) |
| Snap confinement | standard | stricter | Avoid snap installs; use apt/pipx |
| MobSF compat | ✅ | ✅ | None |
| All 24 security tools | ✅ | ✅ | None — they all work |

### C.11 Migrating Existing 22.04 Workers

If you've already provisioned workers on 22.04 and want to move to 24.04:

**Option A (safest): Fresh workers.** Provision new 24.04 workers, drain 22.04 workers one by one, decommission.

**Option B (in-place upgrade):** `do-release-upgrade` from 22.04 → 24.04 is supported but risky. Python 3.10 packages installed system-wide may conflict with the new PEP 668 environment. Test thoroughly in staging first.

**Our recommendation: Option A.** Clean install avoids hidden compatibility issues.

---

*End of VERSIONS.md. Keep this file current — update alongside every dependency change.*
