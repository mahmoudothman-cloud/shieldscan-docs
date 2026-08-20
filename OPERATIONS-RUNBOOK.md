# ShieldScan — Operations Runbook

**Version:** 1.0
**Date:** 2026-04-18
**Companion to:** `SPECIFICATION.md`, `TOOL-ARCHITECTURE.md`, `IMPLEMENTATION-PLAN.md`
**Audience:** DevOps / SRE engineers deploying and operating ShieldScan in production

> **For Claude Code:** This document is the source of truth for deployment and operations. Read §1–§4 before setting up infrastructure. Reference §8–§12 during incidents.

---

## Table of Contents

1. [Infrastructure Overview](#1-infrastructure-overview)
2. [Production Architecture Topology](#2-production-architecture-topology)
3. [Environment Setup (Development → Production)](#3-environment-setup-development--production)
4. [Worker Provisioning](#4-worker-provisioning)
5. [Deployment Procedures](#5-deployment-procedures)
6. [Database Operations](#6-database-operations)
7. [Monitoring & Observability](#7-monitoring--observability)
8. [Incident Response Playbooks](#8-incident-response-playbooks)
9. [Backup & Disaster Recovery](#9-backup--disaster-recovery)
10. [Scaling Strategy](#10-scaling-strategy)
11. [Security Operations](#11-security-operations)
12. [Cost Management](#12-cost-management)
13. [Maintenance Windows & Schedules](#13-maintenance-windows--schedules)
14. [Runbook Index](#14-runbook-index)

---

## 1. Infrastructure Overview

### 1.1 Component Summary

| Component | Purpose | Deployment | HA |
|---|---|---|---|
| FastAPI (API + AI) | Main application, AI pipeline | Fly.io / Hetzner containers | Multi-region, 3+ instances |
| PostgreSQL 16 | Primary data store with RLS | Neon / self-hosted | Streaming replication + PITR |
| Redis 7 | Job queue + pub/sub + cache | Redis Cloud / self-hosted | Sentinel HA (3 nodes) |
| Qdrant | Vector similarity search | Qdrant Cloud / self-hosted | Clustered (3 nodes) |
| Go Workers | Scan execution | Hetzner dedicated servers | Horizontal pool (3+) |
| Cloudflare R2 | Reports + mobile uploads | Cloudflare | Built-in durability |
| Nginx / Cloudflare | Edge + SSL termination | Cloudflare | Anycast |
| Sentry | Error tracking | SaaS | N/A |
| Prometheus + Grafana | Metrics + dashboards | Self-hosted | Single region OK |
| Better Uptime | External uptime monitoring | SaaS | N/A |
| SendGrid / Resend | Transactional email | SaaS | N/A |

### 1.2 Infrastructure Budget (Monthly — 1K active orgs)

| Item | Provider | Cost |
|---|---|---|
| API servers (3× 4GB) | Fly.io / Hetzner | $60 |
| PostgreSQL (primary + replica) | Neon / Hetzner | $120 |
| Redis Sentinel (3 nodes) | Hetzner | $45 |
| Qdrant (3 nodes) | Hetzner | $60 |
| Go worker pool (5× 16GB, 8 cores) | Hetzner | $250 |
| Cloudflare R2 (1TB) | Cloudflare | $15 |
| Domain + SSL | Cloudflare | Free |
| Sentry | SaaS | $26 |
| SendGrid (100K emails/mo) | SaaS | $20 |
| Better Uptime | SaaS | $24 |
| Backup storage (B2) | Backblaze | $12 |
| **Total** | | **~$632/mo** |

At 1K orgs paying an average $79/mo blended, infrastructure is roughly 0.8% of revenue — healthy unit economics.

### 1.3 Why Hetzner (Not AWS)

Worker servers run security scanning tools at high network throughput. Hetzner dedicated servers offer:
- 16GB RAM / 8 cores for $50/mo (vs ~$200/mo AWS equivalent)
- Unmetered 1Gbps bandwidth (vs $0.09/GB egress on AWS)
- Direct support for Docker + nested virtualization
- Frankfurt DC provides low-latency to MENA region

**Use managed services where the operational overhead isn't worth it:** Cloudflare R2 for storage, Neon for PostgreSQL if team is small.

---

## 2. Production Architecture Topology

### 2.1 Network Diagram

```
                                    ┌────────────────────┐
                                    │  Cloudflare Edge   │
                                    │  (SSL + WAF + DDoS)│
                                    └─────────┬──────────┘
                                              │
                            ┌─────────────────┴─────────────────┐
                            │                                   │
                 ┌──────────▼───────────┐         ┌─────────────▼────────────┐
                 │  api.shieldscan.io    │         │   app.shieldscan.io      │
                 │  (FastAPI, 3 nodes)   │         │   (React static CDN)     │
                 └──────────┬───────────┘         └─────────────────────────┘
                            │
           ┌────────────────┼───────────────┬────────────────────┐
           │                │               │                    │
  ┌────────▼─────────┐ ┌────▼────┐  ┌───────▼──────┐  ┌──────────▼──────┐
  │ PostgreSQL 16    │ │ Redis 7 │  │   Qdrant     │  │  Cloudflare R2  │
  │ Primary + Replica│ │Sentinel │  │  (3 nodes)   │  │                 │
  └──────────────────┘ └────┬────┘  └──────────────┘  └─────────────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
              │  (Worker job queue)       │
              │                           │
              ▼                           ▼
  ┌──────────────────────┐    ┌──────────────────────┐
  │  Go Worker #1         │    │  Go Worker #2 ... #N │
  │  ┌─────────────────┐  │    │  ┌─────────────────┐ │
  │  │ Native binaries │  │    │  │ Native binaries │ │
  │  │ (11 tools)      │  │    │  │ (11 tools)      │ │
  │  └─────────────────┘  │    │  └─────────────────┘ │
  │  ┌─────────────────┐  │    │  ┌─────────────────┐ │
  │  │ Docker Services │  │    │  │ Docker Services │ │
  │  │ MobSF, ZAP,     │  │    │  │ MobSF, ZAP,     │ │
  │  │ Trivy, SQLMap   │  │    │  │ Trivy, SQLMap   │ │
  │  └─────────────────┘  │    │  └─────────────────┘ │
  └──────────────────────┘    └──────────────────────┘
```

### 2.2 Worker Pool Composition

Workers are organized into **pools** by capability:

| Pool | Workers | Capability | Job Routing |
|---|---|---|---|
| `web` | 3× (8GB) | Web scanning tools only (no MobSF) | `scan_type in (quick, full_web, full_web_source, api)` |
| `mobile` | 2× (16GB) | MobSF + web tools | `scan_type = mobile` |
| `full` | 1× (16GB) | All tools (full spectrum) | `scan_type = full_spectrum` |

Workers register their capability on startup. Orchestrator routes jobs to matching pools.

**Why this split:** MobSF needs 2GB RAM at minimum. Running MobSF on every worker wastes money for web-only traffic. Dedicated mobile workers keep web worker density high.

### 2.3 Domain Configuration

| Subdomain | Purpose | Backend |
|---|---|---|
| `shieldscan.io` | Marketing site | Vercel / Framer |
| `app.shieldscan.io` | React dashboard | Cloudflare Pages |
| `api.shieldscan.io` | REST API | FastAPI containers |
| `staging-api.shieldscan.io` | Staging API | Staging FastAPI containers |
| `docs.shieldscan.io` | Documentation | Mintlify / custom |
| `status.shieldscan.io` | Status page | Better Uptime |

---

## 3. Environment Setup (Development → Production)

### 3.1 Three Environments

| Environment | Purpose | Data | Deployment |
|---|---|---|---|
| **Local** | Developer machine | Docker Compose, seed data | Manual |
| **Staging** | Integration testing | Synthetic data, weekly reset | Auto-deploy on `main` merge |
| **Production** | Live customers | Real customer data | Manual promotion from staging |

### 3.2 Local Development Setup

**Prerequisites:**
- Docker + Docker Compose
- Python 3.12 + Poetry
- Go 1.22
- Node.js 20
- PostgreSQL client (`psql`)
- Git

**Setup steps:**

```bash
# Clone repos
git clone git@github.com:odyssey/shieldscan-api.git
git clone git@github.com:odyssey/shieldscan-engine.git
cd shieldscan-api

# Copy environment template
cp .env.example .env
# Edit .env with local development values

# Start dependencies (Postgres, Redis, Qdrant, Docker services for workers)
docker compose -f docker-compose.dev.yml up -d

# Install Python dependencies
poetry install

# Run database migrations
poetry run alembic upgrade head

# Seed development data
poetry run python scripts/seed_dev_data.py

# Start API
poetry run uvicorn app.main:app --reload

# In another terminal — start Go worker
cd ../shieldscan-engine
go run ./cmd/worker

# In another terminal — start frontend
cd ../shieldscan-api/shieldscan-web
npm install
npm run dev

# Visit http://localhost:5173
```

### 3.3 .env Files — Full Reference

**`shieldscan-api/.env`:**
```bash
# Database
DATABASE_URL=postgresql+asyncpg://shieldscan:dev@localhost:5432/shieldscan_dev

# Redis
REDIS_URL=redis://localhost:6379/0

# Qdrant
QDRANT_URL=http://localhost:6333

# AI
ANTHROPIC_API_KEY=sk-ant-xxx
OPENAI_API_KEY=sk-xxx

# Stripe
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# Cloudflare R2
R2_ACCOUNT_ID=xxx
R2_ACCESS_KEY=xxx
R2_SECRET_KEY=xxx
R2_BUCKET=shieldscan-dev

# Auth
JWT_SECRET_KEY=your-256-bit-secret-generated-with-openssl-rand-hex-32
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=15
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7
FERNET_KEY=44-byte-base64-key-generated-with-Fernet.generate_key

# CORS
CORS_ORIGINS=["http://localhost:5173","https://app.shieldscan.io"]

# Environment
ENVIRONMENT=development
SENTRY_DSN=

# Email
SENDGRID_API_KEY=SG.xxx
EMAIL_FROM=noreply@shieldscan.io

# Frontend
FRONTEND_URL=http://localhost:5173
```

**`shieldscan-engine/.env`:**
```bash
# Redis
REDIS_URL=redis://localhost:6379/0

# Database (for storing findings)
DATABASE_URL=postgresql://shieldscan:dev@localhost:5432/shieldscan_dev

# R2
R2_ACCOUNT_ID=xxx
R2_ACCESS_KEY=xxx
R2_SECRET_KEY=xxx
R2_BUCKET=shieldscan-dev

# Tool ports (Docker services)
MOBSF_URL=http://localhost:8000
MOBSF_API_KEY=auto-generated-on-first-boot
ZAP_URL=http://localhost:8080
TRIVY_URL=http://localhost:4954
SQLMAP_URL=http://localhost:8775

# Worker config
WORKER_ID=dev-worker-1
WORKER_POOL=full  # web | mobile | full
MAX_CONCURRENT_SCANS=5

# Limits
MAX_UPLOAD_SIZE_MB=500
RECON_SUBDOMAIN_LIMIT=100

# Logging
LOG_LEVEL=info
LOG_FORMAT=json
SENTRY_DSN=
```

### 3.4 Local Docker Compose

**`docker-compose.dev.yml`** — runs all dependencies for local dev:

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: shieldscan
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: shieldscan_dev
    ports: ["5432:5432"]
    volumes: ["pg_data:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  qdrant:
    image: qdrant/qdrant:v1.7.4
    ports: ["6333:6333", "6334:6334"]
    volumes: ["qdrant_data:/qdrant/storage"]

  mobsf:
    image: opensecurity/mobile-security-framework-mobsf:latest
    ports: ["8000:8000"]
    volumes: ["mobsf_data:/home/mobsf/.MobSF"]
    mem_limit: 2g

  zap:
    image: zaproxy/zap-stable:latest
    command: zap.sh -daemon -port 8080 -host 0.0.0.0 -config api.disablekey=true
    ports: ["8080:8080"]
    mem_limit: 1g

  trivy:
    image: aquasec/trivy:latest
    command: server --listen 0.0.0.0:4954
    ports: ["4954:4954"]
    mem_limit: 512m

  sqlmap:
    image: paoloo/sqlmap:latest
    ports: ["8775:8775"]
    mem_limit: 512m

volumes:
  pg_data:
  qdrant_data:
  mobsf_data:
```

---

## 4. Worker Provisioning

### 4.1 Fresh Worker Server Setup

Target OS: **Ubuntu 22.04 LTS** on Hetzner dedicated server.

**`deploy/provision-worker.sh`** — single script that sets up a worker from scratch:

```bash
#!/bin/bash
set -euo pipefail

echo "=== ShieldScan Worker Provisioning ==="
echo "Target OS: $(lsb_release -ds)"

# ─── 1. System updates + base tools ─────────────────────────────
apt-get update
apt-get install -y \
    curl wget git build-essential \
    python3-pip python3-venv \
    openjdk-17-jre-headless \
    nmap-common \
    ca-certificates gnupg lsb-release \
    ufw fail2ban \
    htop jq unzip

# ─── 2. Install Go 1.22 ─────────────────────────────────────────
GO_VERSION=1.22.0
wget https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz
rm -rf /usr/local/go
tar -C /usr/local -xzf go${GO_VERSION}.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin:/root/go/bin' >> /root/.bashrc
export PATH=$PATH:/usr/local/go/bin:/root/go/bin

# ─── 3. Install Docker (for persistent services) ────────────────
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
    https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
    tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# ─── 4. Install native security tools ───────────────────────────
echo "Installing native binaries..."

# Nuclei
go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@v3.2.0
mv /root/go/bin/nuclei /usr/local/bin/nuclei

# Subfinder
go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@v2.6.3
mv /root/go/bin/subfinder /usr/local/bin/subfinder

# httpx
go install github.com/projectdiscovery/httpx/cmd/httpx@v1.6.0
mv /root/go/bin/httpx /usr/local/bin/httpx

# Gitleaks
go install github.com/gitleaks/gitleaks/v8@v8.18.0
mv /root/go/bin/gitleaks /usr/local/bin/gitleaks

# SSLyze (pip)
pip3 install --no-cache-dir sslyze==6.0.0
ln -sf $(which sslyze) /usr/local/bin/sslyze

# Semgrep (pip)
pip3 install --no-cache-dir semgrep==1.60.0
ln -sf $(which semgrep) /usr/local/bin/semgrep

# Wapiti (pip)
pip3 install --no-cache-dir wapiti3==3.2.0
ln -sf $(which wapiti) /usr/local/bin/wapiti

# Checkov (pip)
pip3 install --no-cache-dir checkov==3.2.0
ln -sf $(which checkov) /usr/local/bin/checkov

# Nikto (apt)
apt-get install -y nikto
ln -sf /usr/bin/nikto /usr/local/bin/nikto

# OWASP Dependency-Check (jar wrapper)
DC_VERSION=9.0.0
wget -q https://github.com/jeremylong/DependencyCheck/releases/download/v${DC_VERSION}/dependency-check-${DC_VERSION}-release.zip
unzip -q dependency-check-${DC_VERSION}-release.zip -d /opt/
ln -sf /opt/dependency-check/bin/dependency-check.sh /usr/local/bin/dependency-check
rm dependency-check-${DC_VERSION}-release.zip

# CORStest (git clone)
git clone https://github.com/RUB-NDS/CORStest.git /opt/corstest
chmod +x /opt/corstest/corstest.py
ln -sf /opt/corstest/corstest.py /usr/local/bin/corstest

# ─── 5. Verify all binaries ─────────────────────────────────────
echo "Verifying installed binaries..."
REQUIRED_BINARIES=(
    nuclei semgrep subfinder httpx gitleaks
    sslyze nikto wapiti corstest dependency-check checkov
)
for bin in "${REQUIRED_BINARIES[@]}"; do
    if ! command -v "$bin" &> /dev/null; then
        echo "ERROR: $bin not found in PATH"
        exit 1
    fi
    echo "  ✓ $bin"
done

# ─── 6. Pull Docker images for persistent services ──────────────
echo "Pulling Docker images..."
docker pull opensecurity/mobile-security-framework-mobsf:latest
docker pull zaproxy/zap-stable:latest
docker pull aquasec/trivy:latest
docker pull paoloo/sqlmap:latest
docker pull instrumentisto/nmap:latest

# ─── 7. Firewall configuration ──────────────────────────────────
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp   # SSH
ufw allow from 10.0.0.0/8 to any port 8000  # MobSF (internal only)
ufw allow from 10.0.0.0/8 to any port 8080  # ZAP (internal only)
ufw allow from 10.0.0.0/8 to any port 4954  # Trivy (internal only)
ufw allow from 10.0.0.0/8 to any port 8775  # SQLMap (internal only)
ufw --force enable

# ─── 8. Create worker user (rootless) ───────────────────────────
useradd -r -m -s /bin/bash shieldscan
usermod -aG docker shieldscan

# ─── 9. Deploy worker binary + systemd ──────────────────────────
# (assumes worker binary already copied to /opt/shieldscan-engine/worker)
cat > /etc/systemd/system/shieldscan-worker.service <<EOF
[Unit]
Description=ShieldScan Scan Worker
After=network.target docker.service
Requires=docker.service

[Service]
Type=simple
User=shieldscan
WorkingDirectory=/opt/shieldscan-engine
EnvironmentFile=/opt/shieldscan-engine/.env
ExecStart=/opt/shieldscan-engine/worker
Restart=always
RestartSec=5s
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable shieldscan-worker

# ─── 10. Deploy docker-compose for persistent services ──────────
cat > /opt/shieldscan-engine/docker-compose.services.yml <<'EOF'
# See IMPLEMENTATION-PLAN.md M7 for full services
EOF

systemctl start shieldscan-worker

echo "=== Provisioning complete ==="
echo "Worker is ready. Check status with: systemctl status shieldscan-worker"
```

### 4.2 Worker Health Dashboard

Every worker exposes health endpoint at `http://localhost:9100/health`:

```json
{
  "worker_id": "worker-hetzner-fra-1",
  "pool": "web",
  "status": "healthy",
  "uptime_seconds": 183421,
  "active_scans": 3,
  "max_concurrent": 5,
  "tools": {
    "nuclei": {"status": "healthy", "version": "3.2.0"},
    "semgrep": {"status": "healthy", "version": "1.60.0"},
    "mobsf": {"status": "healthy", "api_reachable": true, "memory_mb": 1856},
    "zap": {"status": "healthy", "api_reachable": true},
    "trivy": {"status": "degraded", "error": "database not updated in 48h"},
    "...": "..."
  },
  "last_heartbeat": "2026-04-18T14:32:15Z"
}
```

Aggregated across all workers via `/orgs/:org_id/tools/health` API endpoint.

### 4.3 Adding a New Worker to the Pool

```bash
# On new Hetzner server
curl -O https://deploy.shieldscan.io/provision-worker.sh
chmod +x provision-worker.sh
sudo ./provision-worker.sh

# Deploy worker binary
scp worker root@new-worker.shieldscan.io:/opt/shieldscan-engine/
scp .env root@new-worker.shieldscan.io:/opt/shieldscan-engine/

# Start service
ssh root@new-worker.shieldscan.io "systemctl start shieldscan-worker"

# Verify registration in Redis
redis-cli -h production-redis.shieldscan.io KEYS "worker:*"
# Should show new worker ID

# Verify it appears in /tools/health
curl -H "Authorization: Bearer admin-key" https://api.shieldscan.io/v1/admin/workers
```

### 4.4 Worker Removal / Drain

Graceful removal without disrupting running scans:

```bash
# 1. Mark worker as draining (Redis flag)
redis-cli -h production-redis.shieldscan.io \
    HSET worker:worker-id-xxx status "draining"

# Orchestrator stops dispatching new jobs, existing jobs finish

# 2. Wait for active scans to complete (check worker /health)
while [ $(curl -s http://worker:9100/health | jq .active_scans) -gt 0 ]; do
    sleep 30
done

# 3. Stop worker service
ssh root@worker.shieldscan.io "systemctl stop shieldscan-worker"

# 4. Unregister from Redis
redis-cli -h production-redis.shieldscan.io DEL worker:worker-id-xxx

# 5. Decommission server
```

### 4.5 In-Flight Jobs and Stranded-Job Recovery (2026-08-20 — engine Drift #70)

A worker holds each job it is working on in `shieldscan:processing:{worker_id}:{priority}` for the whole run, and removes it only once a terminal completion event has been published. This is the authoritative answer to "what is this worker actually doing right now":

```bash
redis-cli --scan --pattern 'shieldscan:processing:*' | while read k; do echo "$k -> $(redis-cli LLEN "$k")"; done
```

An entry belonging to a worker with no live heartbeat key (`shieldscan:workers:{id}`, 60s TTL) is recovered automatically: every running worker sweeps on startup and then every 60s, requeueing the job or publishing a `failed` completion for it. Nothing to do by hand.

```bash
# Who is alive right now
redis-cli --scan --pattern 'shieldscan:workers:*'

# What the sweep did (worker log)
journalctl -u shieldscan-worker | grep 'job reclaim'
```

**Jobs stranded before this shipped** have no processing-list entry — their payload was destroyed at checkout — so they need the one-shot backfill. It is dry-run by default and safe to run twice:

```bash
python scripts/backfill_stranded_jobs.py            # report only
python scripts/backfill_stranded_jobs.py --apply    # mark them failed
```

It marks each stranded job `failed` with a distinct `error_message` (`stranded: …`), which releases the parent scan to aggregate and, where the scan has raw findings, to enter the AI pipeline. It never re-dispatches: re-running a scan is the customer's call.

A job is only touched if it is non-terminal in PostgreSQL **and** its payload is absent from every queue and every processing list. A job sitting in a queue, or being run right now, is never selected.

---

## 5. Deployment Procedures

### 5.1 Deployment Flow

```
Developer pushes to main
        ↓
GitHub Actions runs tests (unit + integration)
        ↓
Tests pass → Deploy to staging automatically
        ↓
Smoke tests on staging (15 min)
        ↓
Manual approval in GitHub
        ↓
Deploy to production (blue-green)
        ↓
Health check on new deployment
        ↓
Traffic shifted from blue → green
        ↓
Old deployment kept for 1 hour (rollback safety)
```

### 5.2 GitHub Actions CI/CD Pipeline

**`.github/workflows/deploy.yml`:**

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  test-api:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
      redis:
        image: redis:7
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install poetry
      - run: poetry install
      - run: poetry run alembic upgrade head
      - run: poetry run pytest --cov=app --cov-report=xml
      - uses: codecov/codecov-action@v4

  test-engine:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: "1.22" }
      - run: go test ./... -race -coverprofile=coverage.out

  deploy-staging:
    needs: [test-api, test-engine]
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Build API container
        run: docker build -t shieldscan-api:${{ github.sha }} .
      - name: Push to registry
        run: |
          docker tag shieldscan-api:${{ github.sha }} registry.shieldscan.io/api:staging
          docker push registry.shieldscan.io/api:staging
      - name: Deploy to Fly.io staging
        run: flyctl deploy --app shieldscan-api-staging --image registry.shieldscan.io/api:staging
      - name: Run smoke tests
        run: ./scripts/smoke-test.sh https://staging-api.shieldscan.io

  deploy-production:
    needs: [deploy-staging]
    runs-on: ubuntu-latest
    environment: production  # requires manual approval
    steps:
      - uses: actions/checkout@v4
      - name: Promote staging image to production
        run: |
          docker pull registry.shieldscan.io/api:staging
          docker tag registry.shieldscan.io/api:staging registry.shieldscan.io/api:${{ github.sha }}
          docker tag registry.shieldscan.io/api:staging registry.shieldscan.io/api:production
          docker push registry.shieldscan.io/api:${{ github.sha }}
          docker push registry.shieldscan.io/api:production
      - name: Blue-green deploy
        run: flyctl deploy --app shieldscan-api --image registry.shieldscan.io/api:production --strategy bluegreen
      - name: Health check
        run: |
          for i in {1..30}; do
            if curl -f https://api.shieldscan.io/health; then exit 0; fi
            sleep 10
          done
          exit 1
      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          webhook: ${{ secrets.SLACK_DEPLOY_WEBHOOK }}
          payload: '{"text":"ShieldScan deployed ${{ github.sha }} to production"}'
```

### 5.3 Database Migration Deployment

**CRITICAL: Never break running production code.** All migrations must be **additive** in the same release as the code that uses them. See Spec v3 §23.

**Safe migration sequence:**

```
1. Alembic migration: ADD new columns (nullable), ADD new tables, ADD new indexes
2. Deploy new code that writes to new columns (still reads from old if fallback needed)
3. Run backfill job: populate new columns from existing data
4. Next release: Alembic migration to set NOT NULL, drop old columns, remove compatibility code
```

**Running a migration in production:**

```bash
# Step 1: Take DB snapshot before migration
ssh postgres-primary.shieldscan.io \
    "pg_dump shieldscan_prod | gzip > /backups/pre-migration-$(date +%Y%m%d-%H%M%S).sql.gz"

# Step 2: Run migration (from a deployed API instance)
flyctl ssh console -a shieldscan-api -C "alembic upgrade head"

# Step 3: Verify schema
ssh postgres-primary.shieldscan.io \
    "psql shieldscan_prod -c '\d+ scans'"

# Step 4: Monitor for errors in Sentry for next 30 min
```

**Rollback a migration:**

```bash
# Ideally never needed — but if it is:
flyctl ssh console -a shieldscan-api -C "alembic downgrade -1"

# If downgrade fails (data incompatibility), restore from snapshot:
ssh postgres-primary.shieldscan.io \
    "gunzip < /backups/pre-migration-xxx.sql.gz | psql shieldscan_prod"
```

### 5.4 Worker Deployment

Workers are longer-lived than API containers. Deploy via:

```bash
# Build new worker binary
cd shieldscan-engine
GOOS=linux GOARCH=amd64 go build -o bin/worker ./cmd/worker

# Roll workers one at a time
for worker in worker-1 worker-2 worker-3; do
    # Mark as draining
    redis-cli HSET worker:$worker status "draining"

    # Wait for active scans to finish (up to 30 min)
    timeout 1800 sh -c "while [ \$(ssh $worker.shieldscan.io 'curl -s localhost:9100/health | jq .active_scans') -gt 0 ]; do sleep 30; done"

    # Upload new binary
    scp bin/worker root@$worker.shieldscan.io:/opt/shieldscan-engine/worker.new

    # Atomic swap + restart
    ssh root@$worker.shieldscan.io '
        mv /opt/shieldscan-engine/worker /opt/shieldscan-engine/worker.old
        mv /opt/shieldscan-engine/worker.new /opt/shieldscan-engine/worker
        chmod +x /opt/shieldscan-engine/worker
        systemctl restart shieldscan-worker
    '

    # Wait for healthy
    until curl -s http://$worker.shieldscan.io:9100/health | jq -e '.status == "healthy"'; do
        sleep 5
    done

    # Remove draining flag
    redis-cli HSET worker:$worker status "healthy"

    echo "✓ $worker deployed"
done
```

### 5.5 Frontend Deployment

React frontend is static — deploys via Cloudflare Pages:

```bash
cd shieldscan-web
npm run build
# dist/ folder contains static assets

# Cloudflare Pages auto-deploys on push to main via GitHub integration
# No manual steps needed for production
```

---

## 6. Database Operations

### 6.1 Connection Pool Configuration

**FastAPI (SQLAlchemy async):**
```python
# src/app/db.py
engine = create_async_engine(
    settings.DATABASE_URL,
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,
    pool_recycle=3600,
    echo=False,
)
```

**Expected connections at peak:**
- 3 API instances × 30 max connections = 90 connections
- 5 Go workers × 10 max connections = 50 connections
- Reserve 10 for admin tools
- **Total: 150 connections** — PostgreSQL configured for `max_connections = 200`.

### 6.2 Index Strategy

Key indexes (verify via `EXPLAIN ANALYZE`):

```sql
-- Critical hot paths
CREATE INDEX idx_scans_org_created ON scans(organization_id, created_at DESC);
CREATE INDEX idx_vulns_scan_severity ON vulnerabilities(scan_id, severity);
CREATE INDEX idx_vulns_org_status ON vulnerabilities(organization_id, status) WHERE status = 'open';
CREATE INDEX idx_raw_findings_scan ON raw_findings(scan_id);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);  -- auth lookups
CREATE INDEX idx_attack_surface_scan ON attack_surface(scan_id);
CREATE INDEX idx_mobile_uploads_project ON mobile_uploads(project_id, created_at DESC);

-- Audit logs (write-heavy, index only essential fields)
CREATE INDEX idx_audit_org_time ON audit_logs(organization_id, created_at DESC);
CREATE INDEX idx_audit_actor ON audit_logs(actor_id, created_at DESC);
```

**Partitioning** for high-volume tables (post-launch if needed):
- `audit_logs` → partition by month (`created_at`)
- `raw_findings` → partition by month (auto-drop partitions > 90 days)

### 6.3 Query Performance Monitoring

Enable slow query logging:

```sql
ALTER SYSTEM SET log_min_duration_statement = '500ms';
ALTER SYSTEM SET log_statement = 'mod';  -- log all writes
SELECT pg_reload_conf();
```

Use `pg_stat_statements`:

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 slowest queries
SELECT query, calls, mean_exec_time, max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### 6.4 Vacuum + Analyze Schedule

PostgreSQL auto-vacuum defaults are fine for most tables. High-churn tables benefit from tuned settings:

```sql
-- raw_findings gets deleted frequently (retention cleanup)
ALTER TABLE raw_findings SET (
    autovacuum_vacuum_scale_factor = 0.05,  -- vacuum when 5% dead
    autovacuum_analyze_scale_factor = 0.02
);

-- audit_logs grows indefinitely — needs regular ANALYZE
ALTER TABLE audit_logs SET (autovacuum_analyze_scale_factor = 0.01);
```

### 6.5 Data Retention Cleanup Job

Nightly cron runs on one API instance:

```python
# scripts/cleanup_retention.py
async def cleanup_old_data():
    async with async_session() as db:
        # Raw findings > 90 days
        await db.execute(text("DELETE FROM raw_findings WHERE created_at < NOW() - INTERVAL '90 days'"))

        # Mobile uploads > 30 days post-scan
        await db.execute(text("""
            DELETE FROM mobile_uploads
            WHERE created_at < NOW() - INTERVAL '30 days'
            AND id NOT IN (SELECT mobile_upload_id FROM scans WHERE mobile_upload_id IS NOT NULL)
        """))
        # Also delete R2 objects for mobile uploads

        # Soft-deleted projects > 30 days
        await db.execute(text("DELETE FROM projects WHERE archived_at < NOW() - INTERVAL '30 days'"))

        # Keep audit_logs for 7 years (compliance)
        await db.execute(text("DELETE FROM audit_logs WHERE created_at < NOW() - INTERVAL '7 years'"))

        await db.commit()
```

Cron: `0 3 * * * poetry run python scripts/cleanup_retention.py`

---

## 7. Monitoring & Observability

### 7.1 The Four Signals

Every service is monitored on:
1. **Latency** — request response time (p50, p95, p99)
2. **Traffic** — requests per second
3. **Errors** — error rate per endpoint
4. **Saturation** — queue depth, CPU, memory, disk

### 7.2 Metrics Collection (Prometheus)

**FastAPI exposes `/metrics` endpoint** using `prometheus-fastapi-instrumentator`:

```python
from prometheus_fastapi_instrumentator import Instrumentator

Instrumentator().instrument(app).expose(app)
```

**Go workers expose metrics on :9100/metrics** via `prometheus/client_golang`:

```go
import "github.com/prometheus/client_golang/prometheus/promhttp"

http.Handle("/metrics", promhttp.Handler())
http.ListenAndServe(":9100", nil)
```

**Key metrics tracked:**

```
# API
http_requests_total{method, endpoint, status}
http_request_duration_seconds{endpoint}
active_scans_gauge
queued_scans_gauge
ai_api_calls_total{provider, status}
ai_cost_usd_total

# Workers
worker_active_scans{worker_id, pool}
worker_tool_invocations_total{tool_name, status}
worker_tool_duration_seconds{tool_name}
worker_redis_consume_errors_total

# Scanning
scan_duration_seconds{scan_type, status}
findings_discovered_total{engine, severity}
mobile_scans_total{platform, status}
recon_subdomains_discovered_total
```

### 7.3 Grafana Dashboards

Required dashboards (saved as JSON in `deploy/grafana/`):

1. **API Health** — request rate, latency, error rate by endpoint
2. **Scan Throughput** — active scans, queue depth, completion rate, scan duration distribution
3. **Worker Pool Health** — worker uptime, tool health per worker, active scans per worker
4. **AI Costs** — Claude + OpenAI spend per hour, per scan average, running monthly total
5. **Mobile Scans** — uploads per hour, average APK/IPA size, MobSF processing time
6. **Database** — connections, slow queries, replication lag
7. **Revenue** — MRR, new subscriptions, churn, usage per tier

### 7.4 Alerting Rules

**Critical alerts** (wake up on-call):

```yaml
groups:
  - name: critical
    rules:
      - alert: APIDown
        expr: up{job="shieldscan-api"} == 0
        for: 2m
        annotations:
          summary: "API instance {{ $labels.instance }} down"

      - alert: WorkerPoolExhausted
        expr: sum(worker_active_scans) / sum(worker_max_concurrent) > 0.95
        for: 10m
        annotations:
          summary: "All workers at capacity — scans queuing"

      - alert: DatabaseReplicationLag
        expr: pg_replication_lag_seconds > 30
        for: 5m

      - alert: AICostAnomaly
        expr: rate(ai_cost_usd_total[1h]) > 50
        annotations:
          summary: "AI spend > $50/hr — possible runaway loop"

      - alert: MobSFDown
        expr: up{job="mobsf",worker_pool="mobile"} == 0
        for: 5m

      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.01
        for: 10m
```

**Warning alerts** (Slack only):

```yaml
  - name: warnings
    rules:
      - alert: QueueBacklog
        expr: queued_scans_gauge > 100
        for: 15m

      - alert: SlowScanDuration
        expr: histogram_quantile(0.95, scan_duration_seconds) > 3600
        for: 30m

      - alert: WorkerToolDegraded
        expr: worker_tool_healthy == 0
        for: 10m
```

### 7.5 Logging Strategy

**Structured JSON logs** everywhere. Fields:

```json
{
  "timestamp": "2026-04-18T14:32:15.123Z",
  "level": "info",
  "service": "shieldscan-api",
  "environment": "production",
  "request_id": "req_abc123",
  "organization_id": "org_xxx",
  "user_id": "usr_yyy",
  "event": "scan_started",
  "scan_id": "scn_zzz",
  "scan_type": "full_web",
  "message": "Scan dispatched to worker pool",
  "duration_ms": 42
}
```

**Log aggregation:** Grafana Loki (self-hosted) or Axiom (managed).

**Retention:**
- Application logs: 30 days hot, 90 days cold (S3 archive)
- Access logs: 90 days
- Audit logs (in PostgreSQL): 7 years

### 7.6 Distributed Tracing

Instrument with OpenTelemetry:

```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

FastAPIInstrumentor.instrument_app(app)
tracer = trace.get_tracer(__name__)
```

Trace propagation via `traceparent` header across API → Redis → Worker → Claude API.

**Backend:** Jaeger or Grafana Tempo.

---

## 8. Incident Response Playbooks

### 8.1 Incident Severity Levels

| Level | Example | Response SLA |
|---|---|---|
| **SEV-1** | Full API outage, data loss, security breach | Immediate; wake on-call; public status page |
| **SEV-2** | Degraded scan performance, partial outage, high error rate | < 30 min; notify on-call; update status |
| **SEV-3** | Single customer affected, non-critical feature broken | < 4 hours business hours |
| **SEV-4** | Minor bug, cosmetic issue | Next business day |

### 8.2 On-Call Rotation

- Primary on-call: 1 engineer, 7-day rotation
- Secondary (escalation): Founder
- Rotation managed in PagerDuty
- All SEV-1/SEV-2 require post-incident review within 48 hours

### 8.3 Playbook: API Outage

**Symptoms:** API health checks failing, user reports of 500 errors, Sentry spike.

**Response steps:**

```
1. CONFIRM — check status page uptime monitor + Sentry dashboard
2. ASSESS — determine scope:
   - All regions? → infrastructure issue
   - Single region? → regional deploy issue
   - Subset of endpoints? → route-specific bug
3. MITIGATE:
   a. If recent deploy: immediate rollback via Fly.io:
      flyctl releases -a shieldscan-api
      flyctl deploy --image <previous-sha>
   b. If database issue: check pg replication, connection pool exhaustion
   c. If Redis issue: verify Sentinel failover, restart if stuck
4. COMMUNICATE:
   - Update status.shieldscan.io within 5 min
   - Post in #incidents Slack
   - Email affected Business+ tier customers if > 15 min
5. RESOLVE:
   - Verify recovery via health checks + synthetic transactions
   - Update status page to "resolved"
6. POSTMORTEM:
   - Timeline of events
   - Root cause analysis
   - Action items with owners
   - Share publicly if customer-impacting
```

### 8.4 Playbook: Scan Queue Backed Up

**Symptoms:** `queued_scans_gauge > 100`, customer complaints about slow scan start.

**Response:**

```
1. Check worker health: curl https://api.shieldscan.io/v1/admin/workers
2. Identify cause:
   - Workers down? → restart via systemctl on affected hosts
   - All workers busy? → scan_duration anomaly?
     - Check if specific scan is hung (Nmap on huge network, for example)
     - Cancel hung scan: POST /v1/admin/scans/:id/force-cancel
   - AI pipeline stuck? → check Claude API status, check Qdrant
3. Scale horizontally:
   - Provision new worker from snapshot: see §4.3
   - Target: new worker serving traffic within 10 min
4. If persistent > 30 min: engage SEV-2 protocol
```

### 8.5 Playbook: MobSF Container Crashed

**Symptoms:** Mobile scans failing, `worker_tool_invocations_total{tool="mobsf",status="error"}` spiking.

**Response:**

```bash
# On affected worker
ssh root@mobile-worker-1.shieldscan.io

# Check container
docker ps -a | grep mobsf
docker logs mobsf --tail 200

# Common causes:
# A. Out of memory — increase mem_limit in docker-compose
# B. Disk full — /home/mobsf/.MobSF cache grew too large
#    Fix: docker exec mobsf rm -rf /home/mobsf/.MobSF/downloads/*
# C. Corrupt image — pull fresh
#    docker pull opensecurity/mobile-security-framework-mobsf:latest

# Restart
docker compose -f /opt/shieldscan-engine/docker-compose.services.yml restart mobsf

# Wait for health
until curl -f http://localhost:8000/api/v1/ping; do sleep 5; done

# Verify from worker's perspective
curl http://localhost:9100/health | jq .tools.mobsf
```

### 8.6 Playbook: Database Connection Exhaustion

**Symptoms:** `psycopg.OperationalError: too many connections`, API latency spike.

**Response:**

```sql
-- Check current connections
SELECT count(*), state, usename FROM pg_stat_activity GROUP BY state, usename;

-- Identify long-running queries
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC
LIMIT 20;

-- Kill stuck queries (careful!)
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE now() - query_start > INTERVAL '10 minutes'
AND state = 'active';
```

**Root cause fixes:**
- Increase API pool size if legitimate load
- Add connection pool (PgBouncer) in front of Postgres
- Find and fix connection leaks in code

### 8.7 Playbook: AI Cost Spike

**Symptoms:** `AICostAnomaly` alert fires, Anthropic/OpenAI bill spike.

**Response:**

```
1. Check AI call logs for repeated same-prompt loops:
   grep "ai_call" logs | jq -s 'group_by(.prompt_hash) | map({hash: .[0].prompt_hash, count: length}) | sort_by(-.count) | .[:10]'

2. Circuit breaker: set Redis flag to disable non-essential AI:
   redis-cli SET ai:circuit_breaker "enabled"

3. The flag pauses:
   - Fix generation (still show vulnerabilities without fixes)
   - Executive summary generation
   Core scanning continues.

4. Identify and kill runaway scan:
   Find scan with abnormal finding count:
   SELECT scan_id, count(*) FROM vulnerabilities GROUP BY scan_id ORDER BY 2 DESC LIMIT 5;

5. Once resolved: redis-cli DEL ai:circuit_breaker
```

### 8.8 Playbook: Data Breach / Security Incident

**CRITICAL — follow this exactly:**

```
0. DON'T PANIC. Don't delete logs. Preserve evidence.

1. CONTAIN (first 15 min):
   - If active attack: block source IP at Cloudflare WAF
   - If compromised credentials: rotate immediately
   - If malicious internal actor: disable account
   - Do NOT shut down servers — we need the logs

2. ASSESS (first hour):
   - What data was accessed? Query audit_logs
   - Which organizations affected?
   - Scope: read-only access vs data modification vs data exfiltration

3. ESCALATE:
   - Notify founder + legal counsel
   - Engage external incident response firm if scope > 1000 records

4. COMMUNICATE (within 72 hours per GDPR, sooner for affected customers):
   - Direct email to all affected organizations
   - Public blog post if > 100 orgs affected
   - Regulatory notification (Egypt DPL, Saudi PDPL, GDPR)

5. REMEDIATE:
   - Patch the vulnerability
   - Force password reset for affected users
   - Rotate all API keys for affected orgs
   - Issue new encryption keys if encryption compromised

6. POSTMORTEM:
   - Full public disclosure within 14 days
   - Action items with deadlines
   - Third-party security audit
```

### 8.9 Playbook: Worker Scanning a Target It Shouldn't

**Symptoms:** User reports their "forgotten" domain was scanned without authorization.

**Response:**

```
1. IMMEDIATE: cancel all active scans for this organization:
   POST /v1/admin/orgs/:org_id/cancel-all-scans

2. AUDIT: query audit_logs for this organization's scan history:
   SELECT * FROM audit_logs WHERE organization_id = '...' AND event = 'scan_started';

3. VERIFY DOMAIN OWNERSHIP:
   - Check projects.domain_verified
   - If false but scanned: CRITICAL BUG — escalate immediately
   - If true but dispute: user may have lost control of the domain

4. COMPENSATE the user:
   - Refund + credits
   - Written incident report
   - Action items to prevent recurrence
```

---

## 9. Backup & Disaster Recovery

### 9.1 Backup Strategy

| Asset | Frequency | Retention | Destination |
|---|---|---|---|
| PostgreSQL | Continuous WAL + hourly base | 30 days | Backblaze B2 (encrypted) |
| Redis snapshot | Every 6 hours | 7 days | Backblaze B2 |
| Qdrant snapshot | Daily | 7 days | Backblaze B2 |
| Cloudflare R2 objects | Cross-region replication | Indefinite | R2 (another region) |
| Infrastructure as Code | Git (GitHub) | Indefinite | GitHub |
| Secrets (Vault) | Daily encrypted backup | 30 days | Offline encrypted volume |

### 9.2 RTO / RPO Targets

- **RTO (Recovery Time Objective):** < 4 hours for full disaster recovery
- **RPO (Recovery Point Objective):** < 15 minutes of data loss maximum

### 9.3 Backup Verification

**Weekly cron:**

```bash
# scripts/verify-backup.sh
#!/bin/bash
set -e

# Download most recent backup
rclone copy b2:shieldscan-backups/postgres/latest.sql.gz /tmp/

# Restore to isolated test instance
docker run -d --name backup-verify -e POSTGRES_PASSWORD=test postgres:16
gunzip -c /tmp/latest.sql.gz | docker exec -i backup-verify psql -U postgres

# Run integrity checks
docker exec backup-verify psql -U postgres -c "SELECT COUNT(*) FROM organizations;"
docker exec backup-verify psql -U postgres -c "SELECT COUNT(*) FROM scans;"

# Cleanup
docker rm -f backup-verify

# Alert if any step failed
```

### 9.4 Disaster Recovery Drill

Quarterly exercise — restore full production from backups in a staging region:

```
1. Provision new infrastructure in alternate region (Terraform)
2. Restore PostgreSQL from latest base backup + WAL
3. Restore Redis snapshot
4. Restore Qdrant snapshot
5. Point staging DNS at new region
6. Run smoke tests
7. Measure time-to-recovery
8. Tear down drill environment
9. Document lessons learned
```

---

## 10. Scaling Strategy

### 10.1 Growth Milestones

| Milestone | Infrastructure Change |
|---|---|
| 100 orgs | Default deployment — 3 API + 3 workers + 1 Postgres |
| 1,000 orgs | Add read replica, scale to 5 workers, upgrade Redis |
| 10,000 orgs | Shard PostgreSQL by org_id hash, 10+ workers, dedicated AI inference cache |
| 100,000 orgs | Multi-region, dedicated mobile infrastructure, proprietary vuln database |

### 10.2 Horizontal Scaling

**API:** Stateless — scale by adding Fly.io instances. Health check required before traffic routing.

**Workers:** Pool-based. Add workers by running `provision-worker.sh` on new Hetzner server, deploying worker binary, starting systemd service. Worker auto-registers in Redis.

**Database:** Vertical scaling first (Neon scales to 1TB transparently). Beyond that: read replicas for analytical queries, eventual sharding by `organization_id`.

### 10.3 Autoscaling Rules (Fly.io)

```toml
[[services]]
  internal_port = 8000
  protocol = "tcp"

  [[services.ports]]
    port = 443
    handlers = ["tls", "http"]

  [services.concurrency]
    type = "requests"
    hard_limit = 250
    soft_limit = 200

[scaling]
  min_machines = 3
  max_machines = 10
  # Scale up when soft limit hit; down when < 50% capacity for 10 min
```

### 10.4 Worker Queue Scaling Heuristic

Monitor `queued_scans_gauge / worker_max_concurrent * num_workers`:

- **< 0.5** — scale down (remove a worker)
- **0.5 - 0.8** — healthy
- **0.8 - 0.95** — warning, prepare new worker
- **> 0.95** — critical, provision new worker now

---

## 11. Security Operations

### 11.1 Secrets Management

**Production secrets stored in HashiCorp Vault or Fly.io secrets.** Never in:
- Git repositories (even private)
- Environment variables in shell history
- CI/CD logs
- Application logs

**Rotation schedule:**

| Secret | Rotation | Trigger |
|---|---|---|
| JWT_SECRET_KEY | Every 90 days | Calendar + rotation ceremony |
| Fernet encryption key | Every 180 days | Requires data re-encryption |
| Database passwords | Every 180 days | Coordinated with app restart |
| Stripe webhook secret | As needed | Rotation via Stripe dashboard |
| API keys (customer) | 180-day forced rotation | Warning at 90 days |

### 11.2 Eating Our Own Dogfood

ShieldScan scans itself continuously:

```yaml
# .github/workflows/self-scan.yml
on:
  push:
    branches: [main]
  schedule:
    - cron: '0 3 * * *'  # daily at 3am UTC

jobs:
  self-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: odyssey/shieldscan-action@v1
        with:
          api-key: ${{ secrets.SHIELDSCAN_INTERNAL_KEY }}
          target: https://api.shieldscan.io
          scan-type: full_spectrum
          fail-on: high
```

All critical/high findings must be fixed within 7 days, medium within 30 days, low within 90 days.

### 11.3 Access Control

**Principle of least privilege:**

| Role | Access |
|---|---|
| Developer | Staging read-write, production read-only |
| SRE / On-call | Production read-write, database read-only |
| Founder | Full access |
| Customer Support | Customer data via admin API only (audited) |

All production access requires:
- MFA on GitHub + Fly.io + cloud providers
- SSH via bastion host only
- All commands logged via auditd

### 11.4 Compliance Posture

Targets:
- **SOC 2 Type II** — within 12 months of launch
- **ISO 27001** — within 18 months
- **Egypt DPL / Saudi PDPL** compliance documented from day 1

Ongoing:
- Quarterly internal security review
- Annual external penetration test
- Vulnerability disclosure program
- Security.txt at https://shieldscan.io/.well-known/security.txt

### 11.5 PostgreSQL Role Model

Two database roles exist in production:

- **`shieldscan` (admin)** — superuser, used only for migrations and maintenance. Connections from this role **BYPASS RLS**. Never used by the application.

- **`shieldscan_app` (runtime)** — non-superuser, used by the application at runtime. Subject to RLS policies. Has `SELECT`/`INSERT`/`UPDATE`/`DELETE` on tables it needs, no schema DDL.

**Rationale:** PostgreSQL superusers and roles with `rolbypassrls=true` bypass RLS regardless of `FORCE ROW LEVEL SECURITY`. Running the app as a non-superuser role is the only way to guarantee tenant isolation.

**Failure mode:** if the production app ever connects as `shieldscan` (admin) by accident, cross-tenant data leaks become possible. Enforced via:

- Startup check in `app/main.py` that asserts `current_user != 'shieldscan'`
- Connection string validation in config
- Alerting on `pg_stat_activity` showing `shieldscan` with app-like query patterns

See `tests/conftest.py` `use_org()` for the tenant context mechanism.

### 11.6 Redis as a Hard Dependency of the Auth Path

**Introduced with M2.3.** Once the rate limiter and refresh-token revocation list ship, the API cannot authenticate requests if Redis is unreachable. We deliberately **fail closed**: no bypass, no degraded "auth without rate limiting" mode. A bypass would let an attacker induce Redis unavailability (e.g. via connection-pool exhaustion on Redis) to disable brute-force protection.

**What depends on Redis:**

| Auth operation | Redis reason | Failure behavior |
|---|---|---|
| `POST /auth/register` | rate limiter INCR | 503 (retry with backoff) |
| `POST /auth/login` | rate limiter INCR (IP + email) | 503 |
| `POST /auth/refresh` | revocation-list check on incoming jti, INCR of revoked jti | 401 on uncertainty — never "let the token through because Redis is down" |
| `POST /auth/logout` | INCR of both access + refresh jti into revocation list | 503 — client MUST retry until logout is acknowledged |
| any authenticated request | revocation-list check on access jti | 401 on uncertainty |

**Liveness vs readiness:**
- **Liveness probe** (`/health`) — stays green as long as the process is alive. Does NOT depend on Redis (otherwise a Redis blip kills the pod unnecessarily).
- **Readiness probe** (`/ready`, added with M2.3) — fails if Redis PING does not return PONG within 500ms. Load balancer stops routing until Redis recovers.

**Alerting threshold:** PagerDuty P1 if `/ready` returns non-200 for more than 30 seconds across more than 50% of API pods. Auth is a revenue-critical path.

**Recovery order after Redis outage:**
1. Redis itself is restored (see §6.x Redis operations).
2. API pods' readiness probes flip green within ~5s as the singleton client reconnects (redis-py's async client reconnects lazily on next command).
3. Load balancer returns pods to the rotation.
4. **Do NOT flush Redis** during recovery — the revocation list contains jti's for compromised tokens. Flushing re-enables them until their natural expiry.
5. Rate-limit counters ARE lost on Redis restart. This is acceptable (attackers get one free window) and correct (legitimate users are not penalized for our outage).

**What tests cover:**
- Rate limiter interface via `fakeredis.aioredis` (shieldscan-api commit `e425ace`). No live Redis in CI.
- Integration smoke in docker-compose.dev.yml brings up a real Redis 8.6.2.

---

## 12. Cost Management

### 12.1 AI Cost Controls

**Hard budget per scan by tier:**

```python
SCAN_COST_BUDGETS = {
    "free":     0.10,   # $0.10 max AI spend
    "starter":  0.25,
    "growth":   0.60,
    "business": 1.50,
    "enterprise": 5.00,
}

# If mid-scan AI cost exceeds budget:
if current_scan_cost > SCAN_COST_BUDGETS[org.tier]:
    skip_fix_generation()  # still generate vulnerabilities, skip fixes
    log_budget_exceeded_event(scan_id, org_id)
```

**Caching layers** (reduce cost 50-70%):

1. **Embedding cache** — Redis, keyed by finding fingerprint. TTL 30 days.
2. **Fix cache** — PostgreSQL, keyed by `(cwe_id, finding_type, language, framework)`. Cache hit rate target: 40%.
3. **Executive summary cache** — skip if similar scan within 24 hours.

### 12.2 Infrastructure Cost Monitoring

**Monthly review:**

```sql
-- Claude API spend this month
SELECT
    DATE_TRUNC('day', created_at) as day,
    SUM(cost_usd) as daily_spend,
    COUNT(*) as call_count,
    AVG(cost_usd) as avg_cost_per_call
FROM ai_api_calls
WHERE created_at > NOW() - INTERVAL '30 days'
GROUP BY day
ORDER BY day;
```

Alert if monthly spend > $5K (well above projected 1K-org baseline of $250/mo).

### 12.3 Cost per Organization

Track unit economics:

```sql
-- Cost to serve each organization this month
WITH ai_cost AS (
    SELECT organization_id, SUM(cost_usd) as ai_cost
    FROM ai_api_calls
    WHERE created_at > DATE_TRUNC('month', NOW())
    GROUP BY organization_id
),
storage_cost AS (
    SELECT organization_id, SUM(file_size_bytes) / (1024.0^3) * 0.015 as storage_cost  -- R2 pricing
    FROM (SELECT organization_id, file_size_bytes FROM mobile_uploads
          UNION ALL
          SELECT organization_id, size_bytes FROM reports) u
    GROUP BY organization_id
)
SELECT
    o.name,
    o.tier,
    COALESCE(ai.ai_cost, 0) as ai_cost,
    COALESCE(s.storage_cost, 0) as storage_cost,
    COALESCE(ai.ai_cost, 0) + COALESCE(s.storage_cost, 0) as total_cost
FROM organizations o
LEFT JOIN ai_cost ai ON ai.organization_id = o.id
LEFT JOIN storage_cost s ON s.organization_id = o.id
ORDER BY total_cost DESC;
```

Flag orgs where cost > 50% of their subscription revenue for review.

---

## 13. Maintenance Windows & Schedules

### 13.1 Scheduled Maintenance

**Weekly** (no user impact):
- Sunday 03:00 UTC — log rotation, backup verification, dependency vulnerability scan

**Monthly** (brief API pause, < 5 min):
- First Sunday 04:00 UTC — database VACUUM FULL on large tables (with concurrent index rebuilds)

**Quarterly** (1-hour maintenance window):
- Database major version upgrade (when applicable)
- Redis upgrade
- Disaster recovery drill

**Announced 7 days in advance via:**
- Status page banner
- Email to Business+ tier customers
- In-app notification

### 13.2 Dependency Update Cadence

| Dependency | Update Cadence |
|---|---|
| Security patches (all languages) | Within 48 hours |
| Python minor versions | Within 1 month |
| Go minor versions | Within 1 month |
| Major framework versions (FastAPI, SQLAlchemy) | Evaluated quarterly |
| Docker base images | Monthly rebuild |
| Security tool versions (Nuclei, ZAP, etc.) | Monthly — coordinated with release notes |

Automated via Dependabot + Renovate.

---

## 14. Runbook Index

Quick reference for common operational tasks:

| Task | Section |
|---|---|
| Deploy new API version | §5.2 |
| Deploy new worker version | §5.4 |
| Run database migration | §5.3 |
| Add new worker to pool | §4.3 |
| Remove worker from pool | §4.4 |
| Scale API horizontally | §10.3 |
| Rotate secrets | §11.1 |
| Respond to API outage | §8.3 |
| Respond to scan queue backup | §8.4 |
| Fix MobSF crash | §8.5 |
| Handle DB connection exhaustion | §8.6 |
| Handle AI cost spike | §8.7 |
| Respond to security incident | §8.8 |
| Run disaster recovery drill | §9.4 |
| Verify backups | §9.3 |
| Monitor AI costs | §12.1 |
| Review org-level cost | §12.3 |

---

## Appendix A: Emergency Contact Sheet

| Service | Status Page | Support |
|---|---|---|
| Anthropic (Claude API) | status.anthropic.com | support@anthropic.com |
| OpenAI | status.openai.com | help.openai.com |
| Cloudflare | cloudflarestatus.com | support portal |
| Hetzner | status.hetzner.com | support@hetzner.com |
| Fly.io | status.flyio.net | community.fly.io |
| Stripe | status.stripe.com | support.stripe.com |
| Neon (PostgreSQL) | status.neon.tech | support@neon.tech |
| SendGrid | status.sendgrid.com | support portal |

---

## Appendix B: Runbook Changelog

| Date | Change | Author |
|---|---|---|
| 2026-04-18 | Initial runbook v1.0 | Mahmoud Hassan |

---

*End of Operations Runbook. Keep this document current — update within 48 hours of any infrastructure change.*
