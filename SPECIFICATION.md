# ShieldScan — Technical Specification

**Version:** 4.0 (consolidated)
**Date:** 2026-04-18
**Author:** Odyssey Technology
**Status:** Approved for implementation
**Supersedes:** v2.0, v3.0 addendum

> **For Claude Code:** This is the authoritative product specification. Read fully before implementation. See companion documents:
> - `TOOL-ARCHITECTURE.md` — deep dive on the 19-tool scan engine
> - `IMPLEMENTATION-PLAN.md` — task-by-task build plan
> - `OPERATIONS-RUNBOOK.md` — deployment and operational guide

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Product Vision & Market Position](#2-product-vision--market-position)
3. [System Architecture](#3-system-architecture)
4. [Bounded Contexts & Domain Events](#4-bounded-contexts--domain-events)
5. [Database Design](#5-database-design)
6. [API Reference](#6-api-reference)
7. [Inter-Service Contracts](#7-inter-service-contracts)
8. [AI Analysis Pipeline](#8-ai-analysis-pipeline)
9. [Pricing & Business Model](#9-pricing--business-model)
10. [Security Hardening](#10-security-hardening)
11. [Performance Requirements](#11-performance-requirements)
12. [Risk Register](#12-risk-register)
13. [Architectural Decision Records](#13-architectural-decision-records)
14. [Glossary](#14-glossary)

---

## 1. Executive Summary

**Product:** ShieldScan is an AI-powered full-spectrum security testing platform covering 9 scan categories through 19 integrated tools, with AI-driven correlation, deduplication, and automated remediation.

**Goal:** Deliver enterprise-grade security coverage at SME-accessible pricing, purpose-built for the MENA market, with mobile, web, API, container, and infrastructure scanning unified in one platform.

**Architecture:** FastAPI (Python) for API + AI layer, Go for scan workers, PostgreSQL with row-level security for multi-tenancy, Redis for job queues and real-time progress, Qdrant for vector-based deduplication, React + TypeScript for the dashboard. Hybrid tool deployment: native binaries for lightweight tools + persistent Docker services for heavy tools.

**Tech Stack:**
- **API:** FastAPI, SQLAlchemy 2.0, Alembic, Pydantic v2, PostgreSQL 16, Redis 7, Qdrant
- **Scan Engine:** Go 1.22, Nuclei CLI, Docker SDK, MobSF REST API
- **Frontend:** React 18, TypeScript, Tailwind CSS, Vite, TanStack Query
- **AI:** Claude API (Opus/Sonnet/Haiku), OpenAI Embeddings
- **Infrastructure:** Cloudflare R2, Stripe, SendGrid/Resend

**Scan Categories Covered:**
1. DAST (Dynamic Application Security Testing)
2. SAST (Static Application Security Testing)
3. Mobile Security (APK/IPA/source — Android + iOS)
4. SCA (Software Composition Analysis)
5. Infrastructure & Network Scanning
6. Reconnaissance (subdomain auto-expansion)
7. SSL/TLS Analysis
8. API Security
9. IaC & Container Security

**Core Differentiators (vs. market):**
- Only platform offering full 9-category scanning — competitors cover 2–5 categories
- AI cross-layer correlation (DAST↔SAST) — no competitor does this natively
- AI deduplication via vector similarity — 40–60% noise reduction
- Recon-first subdomain auto-expansion — scans targets clients forgot existed
- Mobile security included — MobSF integration covers Android + iOS
- Arabic-language reports and WhatsApp alerts for MENA market (Phase 2)
- 40–50% cheaper than enterprise competitors

---

## 2. Product Vision & Market Position

### 2.1 The Problem

Security testing tools today force a painful trade-off:

| Segment | Examples | Strength | Weakness |
|---|---|---|---|
| Enterprise | Veracode, Checkmarx, Fortify | Deep coverage, compliance | $15K–100K+/yr, slow, poor DX |
| Dev-first SaaS | Snyk, StackHawk, Detectify | Great DX, CI/CD native | Limited scope (Snyk: no DAST; StackHawk: no SAST) |
| Open-source | ZAP, Nuclei, Semgrep, MobSF | Powerful, free | Expert setup required, raw output, no correlation |
| Mobile-specific | NowSecure | Deep mobile coverage | Expensive, mobile-only, no web coverage |

No single platform covers all scan types, correlates findings across layers, or uses AI to transform raw scanner output into actionable intelligence.

### 2.2 ShieldScan's Position

ShieldScan sits at the intersection: enterprise-grade coverage, developer-first experience, AI-powered intelligence, accessible pricing, MENA-native.

**Competitive advantages:**
- 9 scan categories vs. Snyk's 4 (no DAST) or StackHawk's 2 (no SAST) or Detectify's 1 (DAST only)
- Includes mobile security (Android + iOS) that most competitors completely omit
- AI cross-layer correlation: DAST finding + SAST finding on same vulnerability = high-confidence critical
- AI attack chain discovery: redirect → XSS → session hijack (Phase 2)
- AI deduplication reducing noise by 40–60%
- Marketplace for community-contributed scanners
- On-prem agent as a single Go binary for private network scanning
- Recon-first pipeline: auto-discovers subdomains and scans them

### 2.3 Phase Roadmap

**Phase 1 (MVP — 10 weeks):**
- Full 9-category scanning with all 19 tools
- AI pipeline: dedup + cross-layer correlation + fix generation + executive summary
- React dashboard, REST API, CLI, GitHub Action
- Stripe billing, SOC2 + ISO 27001 compliance mapping
- Domain verification, audit logs, tamper-proof evidence
- On-prem agent

**Phase 2 (MENA expansion + advanced features):**
- Arabic UI and Arabic-language reports
- WhatsApp-native alerts via Twilio
- AI attack chain discovery (Claude Opus)
- IDE plugin (VS Code)
- Auto-fix pull request creation
- Team collaboration (assign, comment, track)
- Jira integration
- Security Reputation Badge for public websites
- Business Impact Pricing Engine (EGP/SAR financial risk calculation)
- Investor Security Pack generator
- Regulatory Change Monitoring (Saudi PDPL, Egypt DPL)
- White-label portal for IT resellers
- Dark web monitoring bundle
- Insurance premium integration
- Acquisition due diligence product
- Security escrow for software agencies

---

## 3. System Architecture

### 3.1 High-Level Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│  User (Dashboard / CLI / CI/CD / Webhook / WhatsApp)        │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ↓
┌─────────────────────────────────────────────────────────────┐
│  FastAPI (Python) — API + AI Pipeline + Orchestrator        │
│  • Authentication (JWT + API keys)                          │
│  • Project & scan management                                │
│  • AI analysis (Claude + OpenAI embeddings)                 │
│  • Billing (Stripe)                                          │
└──────────────────┬──────────────────────────────────────────┘
                   │
        ┌──────────┴──────────┐
        ↓                     ↓
┌──────────────┐       ┌──────────────┐
│ PostgreSQL 16│       │   Redis 7    │ ←── Job queues + pub/sub
│ (RLS tenant) │       │              │
└──────────────┘       └──────┬───────┘
        ↑                     │
        │                     ↓
┌──────────────┐       ┌─────────────────────────────────────┐
│   Qdrant     │ ←──── │  Go Workers (scan execution)        │
│ (embeddings) │       │  • 19 tools (native + Docker)       │
└──────────────┘       │  • Recon-first pipeline             │
                       │  • Progress streaming via Redis     │
                       └────────────┬────────────────────────┘
                                    │
                                    ↓
                       ┌─────────────────────────────────────┐
                       │ Cloudflare R2 (reports, APK/IPA)    │
                       └─────────────────────────────────────┘
```

### 3.2 Repository Structure

Two repositories with a shared Redis interface:

**`shieldscan-api`** (Python + React)
```
shieldscan-api/
├── src/app/
│   ├── main.py                    # FastAPI entry point
│   ├── config.py                  # Pydantic Settings
│   ├── models/                    # SQLAlchemy ORM
│   │   ├── base.py                # Base + TenantMixin (RLS)
│   │   ├── identity.py            # organizations, users, api_keys
│   │   ├── projects.py            # projects, credentials
│   │   ├── scans.py               # scans, scan_jobs, raw_findings
│   │   ├── vulnerabilities.py     # vulnerabilities, evidence, history
│   │   ├── mobile.py              # mobile_uploads, mobile_scans
│   │   ├── recon.py               # attack_surface, subdomains
│   │   ├── compliance.py          # frameworks, controls, mappings
│   │   ├── marketplace.py         # templates, ratings
│   │   ├── audit.py               # audit_logs
│   │   └── billing.py             # plans, subscriptions, usage
│   ├── schemas/                   # Pydantic request/response
│   ├── routes/                    # FastAPI routes
│   ├── services/
│   │   ├── auth.py
│   │   ├── orchestrator.py        # Scan orchestration + dispatch
│   │   ├── mobile.py              # APK/IPA upload handling
│   │   └── ai/
│   │       ├── pipeline.py        # Main AI pipeline
│   │       ├── embeddings.py      # OpenAI embeddings
│   │       ├── deduplication.py   # Qdrant similarity search
│   │       ├── correlation.py     # DAST↔SAST cross-layer match
│   │       ├── scoring.py         # Severity scoring formula
│   │       ├── fix_generation.py  # Claude fix generation
│   │       └── summary.py         # Executive summary generation
│   ├── templates/
│   │   ├── report.html            # Jinja2 PDF template
│   │   └── emails/
│   └── alembic/                   # Database migrations
├── tests/
└── shieldscan-web/                # React frontend (subdirectory)
    └── src/
        ├── pages/
        ├── components/
        ├── hooks/
        └── lib/
```

**`shieldscan-engine`** (Go)
```
shieldscan-engine/
├── cmd/
│   ├── worker/main.go             # Scan worker daemon
│   └── agent/main.go              # On-prem agent binary
├── internal/
│   ├── tools/
│   │   ├── runner.go              # ToolRunner interface
│   │   ├── native.go              # NativeRunner (subprocess)
│   │   ├── docker_service.go      # DockerServiceRunner (HTTP)
│   │   ├── nuclei.go
│   │   ├── zap.go
│   │   ├── semgrep.go
│   │   ├── nmap.go
│   │   ├── mobsf.go               # ← Mobile security
│   │   ├── sslyze.go              # ← SSL/TLS
│   │   ├── recon.go               # ← Subfinder + httpx
│   │   ├── gitleaks.go            # ← Secrets in git history
│   │   ├── nikto.go
│   │   ├── wapiti.go
│   │   ├── corstest.go
│   │   ├── sqlmap.go
│   │   ├── trivy.go               # ← SCA + container
│   │   ├── dependency_check.go
│   │   └── checkov.go             # ← IaC
│   ├── orchestrator/
│   │   ├── scan_executor.go
│   │   ├── tool_router.go         # Scan type → tools mapping
│   │   └── warm_pool.go           # Docker warm pool
│   ├── worker/
│   │   ├── startup.go             # Tool health checks on boot
│   │   └── processor.go           # Job processing loop
│   ├── redis/
│   │   ├── queue.go
│   │   └── pubsub.go
│   └── docker/
│       └── runner.go              # Docker SDK wrapper
├── deploy/
│   ├── docker-compose.services.yml
│   ├── provision-worker.sh
│   └── Dockerfile.worker
└── shieldscan-cli/                # CLI tool (subdirectory)
    ├── cmd/shieldscan/main.go
    └── internal/
```

### 3.3 Technology Justifications

See Section 13 (ADRs) for detailed rationale.

---

## 4. Bounded Contexts & Domain Events

### 4.1 Seven Bounded Contexts

| Context | Responsibility | Language | Communication |
|---|---|---|---|
| Identity & Billing | Auth, teams, API keys, subscriptions, Stripe | Python | REST API |
| Project Management | Targets, config, credentials, schedules, mobile uploads | Python | REST API |
| Scan Orchestration | Decompose scans, dispatch jobs, track progress, aggregate | Python | Redis pub/sub |
| Scan Execution | Pull jobs, run 19 tools, normalize findings, report results | Go | Redis queues |
| AI Analysis | Deduplicate, correlate, score, generate fixes, summarize | Python | Internal service calls |
| Marketplace | Community templates, custom scanners, ratings | Python | REST API |
| Agent | On-prem scanning for private networks | Go | HTTPS to cloud API |

### 4.2 Domain Events

```
ScanRequested           → Orchestrator creates scan_jobs and dispatches to Redis
ScanJobDispatched       → Go worker picks up job from queue
ReconStarted            → Subfinder + httpx run first (if web target)
SubdomainsDiscovered    → Live hosts added to target list
ScanJobProgress         → Worker publishes progress → Redis pub/sub → SSE
FindingDiscovered       → Individual vuln appears (streamed live)
ScanJobCompleted        → Raw findings stored in PostgreSQL
ScanJobFailed           → Retry (up to 3x) or mark as failed
AllScanJobsCompleted    → AI pipeline triggered (dedup → correlate → score → fix → report)
AnalysisCompleted       → Dashboard updated, PDF/SARIF generated, webhooks fired
MobileScanCompleted     → Mobile-specific webhooks fired (e.g., WhatsApp alert in Phase 2)
UsageRecorded           → Billing context increments usage counter
AuditLogWritten         → Tamper-proof log entry for every significant action
```

---

## 5. Database Design

### 5.1 Core Tables (21 total — 19 from v2/v3 + 2 new)

| # | Table | Purpose | New? |
|---|---|---|---|
| 1 | organizations | Tenant root | |
| 2 | users | Platform users | |
| 3 | memberships | User ↔ org with role | |
| 4 | api_keys | Programmatic access keys | |
| 5 | projects | Scan targets | |
| 6 | project_credentials | Encrypted auth config | |
| 7 | scans | Scan execution records | |
| 8 | scan_jobs | Per-engine job tracking | |
| 9 | raw_findings | Unprocessed tool output | |
| 10 | vulnerabilities | Deduplicated, scored, AI-enriched | |
| 11 | vulnerability_history | Status change audit | |
| 12 | evidence | Request/response/code evidence | |
| 13 | compliance_frameworks | SOC2, ISO 27001, PCI | |
| 14 | compliance_controls | Individual controls per framework | |
| 15 | cwe_control_mappings | CWE → control mapping | |
| 16 | marketplace_templates | Community scanners | |
| 17 | marketplace_ratings | Reviews | |
| 18 | audit_logs | Tamper-proof action log | |
| 19 | plans | Pricing tiers | |
| 20 | subscriptions | Active Stripe subscriptions | |
| 21 | usage_records | Usage metering | |
| **22** | **mobile_uploads** | **APK/IPA/ZIP file metadata** | **NEW** |
| **23** | **attack_surface** | **Recon results per scan** | **NEW** |

### 5.2 Row-Level Security (RLS)

All tenant-scoped tables (everything except `compliance_frameworks`, `compliance_controls`, `cwe_control_mappings`, `plans`, and `marketplace_templates`) enforce RLS by `organization_id`. PostgreSQL policies ensure zero cross-tenant data leakage at the database level.

```sql
ALTER TABLE scans ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON scans
  USING (organization_id = current_setting('app.current_org_id')::uuid);
```

### 5.3 Key Schema Additions

**`mobile_uploads` (NEW):**
```sql
CREATE TABLE mobile_uploads (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    project_id UUID NOT NULL REFERENCES projects(id),
    uploaded_by UUID NOT NULL REFERENCES users(id),
    filename TEXT NOT NULL,
    file_extension TEXT NOT NULL CHECK (file_extension IN ('.apk', '.ipa', '.zip')),
    file_size_bytes BIGINT NOT NULL,
    platform TEXT NOT NULL CHECK (platform IN ('android', 'ios', 'unknown')),
    r2_key TEXT NOT NULL,
    mobsf_file_hash TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);
CREATE INDEX idx_mobile_uploads_project ON mobile_uploads(project_id, created_at DESC);
```

**`attack_surface` (NEW):**
```sql
CREATE TABLE attack_surface (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    scan_id UUID NOT NULL REFERENCES scans(id),
    root_domain TEXT NOT NULL,
    subdomain TEXT NOT NULL,
    full_url TEXT NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('live', 'dead', 'timeout')),
    status_code INTEGER,
    tech_stack JSONB,
    discovered_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_probed_at TIMESTAMPTZ
);
CREATE INDEX idx_attack_surface_scan ON attack_surface(scan_id);
CREATE UNIQUE INDEX idx_attack_surface_unique ON attack_surface(scan_id, subdomain);
```

**Extensions to `scans`:**
```sql
ALTER TABLE scans ADD COLUMN scan_type TEXT NOT NULL DEFAULT 'full_web'
  CHECK (scan_type IN ('quick', 'full_web', 'full_web_source', 'api', 'mobile', 'container', 'full_spectrum'));
ALTER TABLE scans ADD COLUMN mobile_upload_id UUID REFERENCES mobile_uploads(id);
ALTER TABLE scans ADD COLUMN recon_enabled BOOLEAN NOT NULL DEFAULT TRUE;
ALTER TABLE scans ADD COLUMN subdomain_limit INTEGER NOT NULL DEFAULT 100;
```

**Extensions to `raw_findings` for mobile:**
```sql
ALTER TABLE raw_findings ADD COLUMN mobile_os TEXT CHECK (mobile_os IN ('android', 'ios', NULL));
ALTER TABLE raw_findings ADD COLUMN mobile_permission TEXT;
ALTER TABLE raw_findings ADD COLUMN mobile_component_name TEXT;
ALTER TABLE raw_findings ADD COLUMN engine_category TEXT NOT NULL
  CHECK (engine_category IN ('dast', 'sast', 'sca', 'mobile', 'infrastructure', 'recon', 'ssl', 'api', 'iac', 'secrets', 'container'));
```

### 5.4 Data Retention

| Data | Retention | Reason |
|---|---|---|
| Active scans, vulnerabilities, evidence | Indefinite | Core product data |
| Raw findings | 90 days | AI pipeline re-runs within this window |
| Audit logs | 7 years | Compliance (ISO 27001 A.12.4) |
| Mobile upload files (R2) | 30 days post-scan | Privacy, storage cost |
| Deleted projects | 30-day soft delete, then hard delete | GDPR right to erasure |
| Scan artifacts (PDFs in R2) | 365 days | User access to historical reports |

---

## 6. API Reference

### 6.1 Base URL & Authentication

- **Production:** `https://api.shieldscan.io/v1`
- **Staging:** `https://staging-api.shieldscan.io/v1`

**Authentication methods:**
1. **JWT tokens** — dashboard users. Access token (15 min) + refresh token (7 days). httpOnly cookies.
2. **API keys** — CLI, CI/CD, programmatic. Prefixed `ss_live_` or `ss_test_`. `Authorization: Bearer ss_live_xxx`.

**Rate limits (per organization):**
| Tier | Daily requests | Burst |
|---|---|---|
| Free | 100 | 10/min |
| Starter | 1,000 | 60/min |
| Growth | 10,000 | 300/min |
| Business | 50,000 | 600/min |
| Enterprise | 200,000 | 2,000/min |

### 6.2 Complete Endpoint Inventory (68 endpoints)

> **Maintenance note:** Subsection headers carry endpoint counts (e.g. "Projects (8)"). When adding endpoints, update **both** the line count **and** the subsection header. Total Phase 1 count must equal the sum of subsection counts. Last full recount: 2026-04-30 (Checkpoint 2, M4→M5 transition).

**Auth & Users (10)**
```
POST   /auth/register
POST   /auth/login
POST   /auth/refresh
POST   /auth/logout
GET    /auth/me
POST   /auth/oauth/github
POST   /auth/forgot-password
POST   /auth/reset-password
POST   /auth/verify-email          ← new (v3 gap #9)
POST   /auth/resend-verification   ← new (v3 gap #9)
```

**Organizations & Teams (7)**
```
POST   /orgs
GET    /orgs/:org_id
PATCH  /orgs/:org_id
GET    /orgs/:org_id/members
POST   /orgs/:org_id/members
DELETE /orgs/:org_id/members/:id
PATCH  /orgs/:org_id/members/:id
```

**API Keys (3)**
```
POST   /orgs/:org_id/api-keys
GET    /orgs/:org_id/api-keys
DELETE /orgs/:org_id/api-keys/:id
```

**Projects (8)**
```
POST   /orgs/:org_id/projects
GET    /orgs/:org_id/projects
GET    /orgs/:org_id/projects/:id
PATCH  /orgs/:org_id/projects/:id
DELETE /orgs/:org_id/projects/:id
POST   /orgs/:org_id/projects/:id/verify
PATCH  /orgs/:org_id/projects/:id/credentials
GET    /orgs/:org_id/projects/:id/stats
```

**Mobile Scanning (1 new)**
```
POST   /orgs/:org_id/projects/:id/mobile/upload   ← NEW
```

**Scans (9)**
```
POST   /orgs/:org_id/projects/:pid/scans
GET    /orgs/:org_id/projects/:pid/scans
GET    /orgs/:org_id/scans/:scan_id
GET    /orgs/:org_id/scans/:scan_id/progress        # SSE
DELETE /orgs/:org_id/scans/:scan_id
GET    /orgs/:org_id/scans/:scan_id/jobs
GET    /orgs/:org_id/scans/:scan_id/jobs/:jid
GET    /orgs/:org_id/scans/:scan_id/attack-surface  ← NEW (recon)
POST   /orgs/:org_id/scans/compare                  ← NEW (v3 gap #15)
```

**Vulnerabilities (7)**
```
GET    /orgs/:org_id/scans/:scan_id/vulnerabilities
GET    /orgs/:org_id/vulnerabilities
GET    /orgs/:org_id/vulnerabilities/:vid
PATCH  /orgs/:org_id/vulnerabilities/:vid
GET    /orgs/:org_id/vulnerabilities/:vid/evidence
GET    /orgs/:org_id/vulnerabilities/:vid/fix
GET    /orgs/:org_id/vulnerabilities/:vid/history
```

**Reports (5)**
```
GET    /orgs/:org_id/scans/:scan_id/report
GET    /orgs/:org_id/scans/:scan_id/report/pdf
GET    /orgs/:org_id/scans/:scan_id/report/sarif
GET    /orgs/:org_id/scans/:scan_id/report/json
GET    /orgs/:org_id/scans/:scan_id/report/executive
```

**Compliance (3)**
```
GET    /orgs/:org_id/compliance/frameworks
GET    /orgs/:org_id/scans/:scan_id/compliance/:fw
GET    /orgs/:org_id/compliance/posture
```

**Billing (6)**
```
GET    /orgs/:org_id/billing/subscription
POST   /orgs/:org_id/billing/subscribe
POST   /orgs/:org_id/billing/portal
GET    /orgs/:org_id/billing/usage
GET    /orgs/:org_id/billing/invoices
POST   /orgs/:org_id/billing/webhooks/stripe
```

**Integrations (6)**
```
POST   /orgs/:org_id/integrations/github
POST   /orgs/:org_id/integrations/slack
POST   /orgs/:org_id/integrations/jira       (Phase 2)
GET    /orgs/:org_id/integrations
DELETE /orgs/:org_id/integrations/:id
POST   /orgs/:org_id/integrations/webhooks
```

**Marketplace (Phase 2 — 4)**
```
GET    /marketplace/templates
GET    /marketplace/templates/:id
POST   /marketplace/templates/:id/install
POST   /marketplace/templates/:id/rate
```

**Tool Health (1 new)**
```
GET    /orgs/:org_id/tools/health   ← NEW
```

**Health & Meta (2)**
```
GET    /health
GET    /v1/openapi.json
```

**Total Phase 1: 68 endpoints. Phase 2 adds 4 marketplace endpoints.**

### 6.3 Request/Response Examples

**Start a Full Web Scan:**
```http
POST /orgs/{org_id}/projects/{pid}/scans
Content-Type: application/json
Authorization: Bearer ss_live_xxx

{
  "scan_type": "full_web",
  "recon_enabled": true,
  "subdomain_limit": 100,
  "dast_config": {
    "depth": "standard",
    "template_categories": ["owasp-top-10", "cves", "misconfigurations"],
    "max_requests_per_second": 50,
    "authenticated": true
  },
  "priority": "normal",
  "callback_url": "https://example.com/webhook"
}
```

**Start a Mobile Scan:**
```http
POST /orgs/{org_id}/projects/{pid}/scans
Content-Type: application/json

{
  "scan_type": "mobile",
  "mobile_config": {
    "upload_ref": "r2://uploads/org_xxx/app_v2.apk",
    "platform": "android",
    "analysis_type": "both"
  },
  "priority": "normal"
}
```

**Upload APK/IPA:**
```http
POST /orgs/{org_id}/projects/{pid}/mobile/upload
Content-Type: multipart/form-data

file: <binary APK/IPA/ZIP, max 500MB>

Response 200:
{
  "upload_ref": "r2://uploads/org_xxx/project_yyy/abc123.apk",
  "platform_detected": "android",
  "file_size_mb": 18.42,
  "filename": "myapp.apk"
}
```

**Attack Surface Response:**
```http
GET /orgs/{org_id}/scans/{scan_id}/attack-surface

Response 200:
{
  "root_domain": "example.com",
  "total_discovered": 24,
  "live": 8,
  "dead": 16,
  "subdomains": [
    {
      "url": "https://api.example.com",
      "status": "live",
      "status_code": 200,
      "tech_stack": ["nginx", "Node.js", "React"],
      "vulnerability_count": 3,
      "last_probed_at": "2026-04-18T14:32:00Z"
    }
  ]
}
```

### 6.4 SSE Streaming

```
GET /orgs/:org_id/scans/:scan_id/progress
```

Event types:
```
event: scan_status
data: {"scan_id": "xxx", "status": "running", "progress": 45}

event: recon_complete
data: {"scan_id": "xxx", "subdomains_found": 24, "live_hosts": 8}

event: job_progress
data: {"job_id": "xxx", "engine": "nuclei", "progress": 34, "message": "..."}

event: finding_discovered
data: {"severity": "high", "title": "SQL Injection", "url": "/api/users"}

event: scan_completed
data: {"scan_id": "xxx", "status": "completed", "total_vulns": 23, "critical": 2}
```

**Reconnection:** Client sends `Last-Event-ID` header. Server replays from last 60s of Redis stream.

### 6.5 Pagination

Dual-mode pagination. Offset for user-facing lists, cursor for API consumers:

```
GET /orgs/:org_id/vulnerabilities?page=2&per_page=50        # offset
GET /orgs/:org_id/vulnerabilities?cursor=eyJpZCI6...&limit=100  # cursor
```

Response includes both `meta.next_cursor` and `meta.page_info` where applicable.

### 6.6 Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable explanation",
    "details": {
      "field": "scan_type",
      "reason": "must be one of: quick, full_web, mobile, ..."
    },
    "request_id": "req_abc123"
  }
}
```

**Rate limited (429):**
```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded. 10000 requests per day allowed on Growth plan.",
    "retry_after": 3600,
    "limit": 10000,
    "remaining": 0,
    "reset_at": "2026-04-19T00:00:00Z"
  }
}
```

---

## 7. Inter-Service Contracts

Python API ↔ Go workers communicate exclusively via Redis. All schemas are explicit.

### 7.1 Job Dispatch (Python → Redis → Go)

Queue: `shieldscan:queue:{priority}` where priority is `critical|high|normal|low`.

```json
{
  "id": "job_a1b2c3d4",
  "scan_id": "scn_x1y2z3",
  "organization_id": "org_m1n2o3",
  "engine": "nuclei",
  "idempotency_key": "scn_x1y2z3:nuclei:1711720200",
  "target": {
    "url": "https://app.example.com",
    "target_type": "web",
    "domain_verified": true
  },
  "auth": {
    "type": "cookie",
    "data": "session=abc123; csrf=xyz789"
  },
  "config": {
    "depth": "standard",
    "template_categories": ["owasp-top-10", "cves"],
    "max_requests_per_second": 50,
    "timeout_seconds": 1800
  },
  "mobile_config": null,
  "callback_channel": "shieldscan:progress:scn_x1y2z3",
  "created_at": "2026-04-18T14:30:00Z"
}
```

For mobile jobs:
```json
{
  "engine": "mobsf",
  "target": {
    "target_type": "mobile"
  },
  "mobile_config": {
    "upload_ref": "r2://uploads/org_xxx/app.apk",
    "platform": "android",
    "analysis_type": "both"
  }
}
```

### 7.2 Progress Events (Go → Redis Streams → Python → SSE)

Stream: `shieldscan:progress:{scan_id}` (Redis Streams, not Pub/Sub — see ADR-014).

Producer (Go worker): `XADD shieldscan:progress:{scan_id} MAXLEN ~ 1000 * event <json>`. The single-field `event` value carries the JSON payload below; `MAXLEN ~ 1000` bounds retention to ~60s of events at peak emit rate per §6.4 replay requirement.

Consumer (Python SSE): `XRANGE` for `Last-Event-ID` replay, `XREAD BLOCK` for live tail. The two compose seamlessly — replay yields up to the latest committed entry, live picks up from that id forward without duplicate or skip.

```json
{
  "event_type": "job_progress",
  "job_id": "job_a1b2c3d4",
  "scan_id": "scn_x1y2z3",
  "engine": "nuclei",
  "status": "running",
  "progress": 34,
  "message": "Running XSS checks... 34/120 templates",
  "finding_count": 12,
  "timestamp": "2026-04-18T14:32:15Z"
}
```

Event types: `job_started`, `recon_started`, `subdomains_discovered`, `job_progress`, `finding_discovered`, `job_completed`, `job_failed`, `job_canceled`.

Cancel signals (`shieldscan:cancel:{scan_id}`) and completions (`shieldscan:completions`) remain Pub/Sub — see §7.3, §7.4 and ADR-014. Mixed-primitive use is intentional, not accidental.

### 7.3 Job Completion (Go → Redis → Python)

Channel: `shieldscan:completions`

```json
{
  "event_type": "job_completed",
  "job_id": "job_a1b2c3d4",
  "scan_id": "scn_x1y2z3",
  "engine": "nuclei",
  "status": "completed",
  "finding_count": 47,
  "duration_ms": 145000,
  "idempotency_key": "scn_x1y2z3:nuclei:1711720200",
  "timestamp": "2026-04-18T14:34:27Z",
  "findings": [
    { "tool_name": "nuclei", "finding_type": "xss", "severity": "high", "...": "..." }
  ],
  "event_seq": { "index": 1, "total": 1 }
}
```

**Findings persistence (per ADR-017).** The `findings` array carries the RawFinding rows produced by the engine for this job; the Python `CompletionsConsumer` (M4 Task 4.2) inserts them in the same transaction as the `ScanJob.status` update. `findings` is REQUIRED on terminal events (`status` ∈ {`completed`, `partial`}) and absent on `failed`/`canceled` events.

**Sequencing for large batches (per ADR-017).** The engine caps each event at **1000 findings** (`MaxFindingsPerEvent`). Jobs producing >1000 findings emit multiple `job_completed` events with `event_seq.total > 1`:

- Intermediate events: `status: "partial_findings"`, `event_seq: {index: N, total: T}`, `N < T`. No `finding_count`/`duration_ms`.
- Terminal event: canonical `status` (`completed`/`partial`), `event_seq: {index: T, total: T}`, authoritative `finding_count` covering all batches, `duration_ms`.

Single-event jobs (≤1000 findings) carry `event_seq: {index: 1, total: 1}`. The `event_seq` field is REQUIRED on every `job_completed` event for forward-compatibility — consumers always read it rather than branching on presence.

### 7.4 Scan Cancellation (Python → Redis → Go)

Channel: `shieldscan:cancel:{scan_id}`

```json
{
  "event_type": "cancel_requested",
  "scan_id": "scn_x1y2z3",
  "reason": "user_requested",
  "timestamp": "2026-04-18T14:33:00Z"
}
```

Go worker subscribes to this channel per-scan and calls `ctx.Cancel()` on receipt. See v3 Section 2 for full propagation flow.

### 7.5 Job Idempotency

Every job has an `idempotency_key` format `{scan_id}:{engine}:{unix_timestamp}`. Redis stores this key with 24h TTL. Workers check before processing — duplicates are silently dropped. Safe to retry or replay.

---

## 8. AI Analysis Pipeline

### 8.1 Pipeline Stages

```
Raw findings (PostgreSQL)
        ↓
[1] Embed findings → OpenAI text-embedding-3-small
        ↓
[2] Deduplicate → Qdrant cosine similarity (threshold 0.92)
        ↓
[3] Correlate → Cross-layer DAST↔SAST matching (threshold 0.75)
        ↓
[4] Score → Weighted formula (CVSS + corroboration + exploitability)
        ↓
[5] Generate fixes → Claude Sonnet per vulnerability, stack-aware
        ↓
[6] Generate executive summary → Claude Sonnet, business language
        ↓
Vulnerabilities (PostgreSQL) + Report (R2)
```

### 8.2 Cross-Layer Correlation Algorithm

```python
CORRELATION_WEIGHTS = {
    "cwe_exact":      0.40,  # Same CWE ID
    "cwe_parent":     0.25,  # Parent/child CWE
    "url_path":       0.20,  # DAST URL matches SAST file route
    "finding_type":   0.30,  # Same finding_type
    "parameter_name": 0.15,  # Same parameter in URL and code
}

def correlation_score(dast, sast, route_map) -> float:
    score = 0.0
    if dast.cwe_id == sast.cwe_id:
        score += WEIGHTS["cwe_exact"]
    elif is_cwe_parent_child(dast.cwe_id, sast.cwe_id):
        score += WEIGHTS["cwe_parent"]
    if dast.finding_type == sast.finding_type:
        score += WEIGHTS["finding_type"]
    if route_map.get(urlparse(dast.evidence.url).path) == sast.evidence.code_file:
        score += WEIGHTS["url_path"]
    if extract_params(dast.evidence.url) & extract_code_params(sast.evidence.code_snippet):
        score += WEIGHTS["parameter_name"]
    return score

# Correlated if score ≥ 0.75 → merged into "corroborated" vulnerability
```

### 8.3 Severity Scoring Formula

```
final_score = base_cvss × corroboration_multiplier × exploitability_multiplier

corroboration_multiplier:
  1.0  if found by 1 engine
  1.3  if corroborated by 2+ engines (DAST + SAST)
  1.5  if found in production-facing asset

exploitability_multiplier:
  0.8  if in admin-only / authenticated path
  1.0  default
  1.2  if proof-of-concept exploit generated by Nuclei/SQLMap
  1.5  if publicly accessible + unauthenticated
```

### 8.4 Mobile-Specific AI Fix Generation

When generating a fix for a mobile finding, the prompt to Claude includes:

```
Target platform: {mobile_os}  // android | ios
Language context: {java/kotlin | swift/objc}
Component type: {activity | service | broadcast_receiver | fragment}
Permission context: {permission_name}
Code location: {file}:{line}

Finding: {title} — {description}
Raw evidence: {raw_output}

Generate a secure code fix following platform best practices (Android Security Guidelines / Apple App Transport Security).
```

### 8.5 Multi-Provider AI Strategy

| Task | Model | Rationale |
|---|---|---|
| Embeddings | OpenAI `text-embedding-3-small` | 10x cheaper than Claude embeddings |
| Deduplication | Local (Qdrant vector math) | No AI call needed |
| Correlation | Local (weighted formula) | Deterministic |
| Scoring | Local formula | Deterministic |
| Fix generation | Claude Sonnet | Code quality + reasoning |
| Executive summary | Claude Sonnet | Business language |
| Attack chain analysis (Phase 2) | Claude Opus | Deep reasoning |
| Chat/Q&A (future) | Claude Haiku | Fast + cheap |

**Target AI cost per scan:**
- Quick scan: $0.08
- Full web scan: $0.25
- Full spectrum scan: $0.55

### 8.6 Error Recovery

AI pipeline must be resilient to partial failures:

| Failure | Fallback |
|---|---|
| OpenAI embedding API down | Skip dedup, use rule-based fingerprint matching |
| Qdrant unavailable | Same — rule-based fingerprinting |
| Claude API rate limited | Queue fix generation for retry (up to 3x) |
| Claude generates invalid code | Flag for human review, still show description |
| Correlation route map missing | Skip URL-path signal, fall back to CWE + type only |

---

## 9. Pricing & Business Model

### 9.1 Pricing Tiers

| Feature | Free | Starter | Growth | Business | Enterprise |
|---|---|---|---|---|---|
| Price (monthly) | $0 | $29 | $99 | $299 | Custom |
| Team size | 1 user | 3 users | 10 users | Unlimited | Unlimited |
| Projects | 1 | 5 | 25 | Unlimited | Unlimited |
| Quick scans/mo | 5 | 50 | 500 | Unlimited | Unlimited |
| Full web scans/mo | 0 | 10 | 100 | Unlimited | Unlimited |
| Mobile scans/mo | 0 | 0 | 2 | Unlimited | Unlimited |
| Full spectrum/mo | 0 | 0 | 0 | 50 | Unlimited |
| Recon (subdomain) | ✓ (10 max) | ✓ (50 max) | ✓ (100 max) | ✓ (500 max) | ✓ (unlimited) |
| AI fix generation | ✗ | ✓ | ✓ | ✓ | ✓ |
| Cross-layer correlation | ✗ | ✗ | ✓ | ✓ | ✓ |
| Compliance reports | ✗ | ✗ | ISO 27001 | +SOC2 +PCI | All frameworks |
| On-prem agent | ✗ | ✗ | ✗ | ✓ | ✓ |
| White-label | ✗ | ✗ | ✗ | ✗ | ✓ |
| SLA | ✗ | ✗ | 99.5% | 99.9% | 99.95% |

### 9.2 Add-Ons

| Add-On | Price | Target |
|---|---|---|
| Mobile scan pack | $99 / 10 scans | Starter users with occasional mobile needs |
| Full Spectrum one-time scan | $49 | Non-Business users needing full coverage once |
| Compliance PDF report (external audit format) | $49 | Companies in compliance cycle |
| Extended vulnerability history (> 90 days) | $29/mo | Compliance-heavy industries |
| Priority scanning queue | $49/mo | Business tier and below |
| Custom Nuclei template dev | $199/template | Specific business logic needs |

### 9.3 Phase 2 Revenue Streams

- **Insurance partnership kickbacks** — percentage of premium reduction delivered
- **Acquisition due diligence audits** — $2,000–5,000 per audit
- **White-label reseller program** — 20% wholesale discount for MENA IT firms
- **Marketplace revenue share** — 30% of paid template sales

### 9.4 Target AI Cost vs Revenue

| Tier | Monthly avg scans | AI cost | Revenue | Margin |
|---|---|---|---|---|
| Starter | 50 quick + 10 full | $6.50 | $29 | 78% |
| Growth | 500 quick + 100 full + 2 mobile | $65 | $99 | 34% |
| Business | Unlimited mix + 50 spectrum | $140 | $299 | 53% |

Costs optimized via caching (50–70% reduction on repeated scans of same code), batch embeddings, and tiered AI depth.

---

## 10. Security Hardening

### 10.1 Platform Security (Eating Our Own Dogfood)

- **ShieldScan scans ShieldScan** — continuous self-scan in CI/CD
- All secrets in Vault (not env vars in production)
- Fernet encryption at rest for project credentials
- Credentials never logged, memory-only decryption
- Key rotation support for encryption keys

### 10.2 Tenant Isolation

- PostgreSQL row-level security (RLS) on all tenant-scoped tables
- `organization_id` enforced at query level, not application layer
- Zero cross-tenant data leakage possible at the database level
- Audit log records every cross-tenant access attempt (should be zero in production)

### 10.3 Scan Authorization

- **Mandatory domain verification** before scanning any target
- DNS TXT record or meta tag verification
- On-prem agent requires manual approval token before scanning internal networks
- ToS requires target ownership attestation — users take legal responsibility

### 10.4 Container Security (Go Workers)

- Rootless Docker containers for tool execution
- Seccomp profiles limiting syscalls
- Resource limits per container (memory, CPU, PIDs)
- Network isolation per scan (each scan gets its own network namespace)
- No privileged containers — ever

### 10.5 File Upload Security (Mobile)

- Magic byte validation (not just extension check)
- ClamAV malware scan on every APK/IPA upload
- Sandboxed analysis — MobSF runs in isolated container
- Files auto-deleted from R2 after 30 days post-scan

### 10.6 API Security

- JWT tokens in httpOnly cookies (no localStorage)
- CSRF protection on mutation endpoints
- Rate limiting per organization
- API key rotation (90-day warning + 180-day forced rotation)
- Separate test and production key prefixes to prevent accidental misuse

---

## 11. Performance Requirements

| Metric | Target | Measurement |
|---|---|---|
| API response time (p95) | < 200ms | Non-scan endpoints |
| API response time (p99) | < 500ms | Non-scan endpoints |
| Scan start latency | < 3s | POST /scans → first job dispatched |
| Mobile upload processing | < 5s | File received → R2 → DB record |
| Concurrent scans per worker | 5 | Go worker instances |
| DAST scan throughput | 500 req/s | Per worker, across all active scans |
| AI pipeline duration (p95) | < 120s | For scans with < 100 unique findings |
| SSE event latency | < 500ms | Worker progress → browser render |
| Dashboard page load | < 2s | First contentful paint |
| PDF report generation | < 30s | For reports with < 100 vulnerabilities |
| Mobile scan duration (p95) | < 10 min | Single APK, static analysis |
| Recon phase | < 60s | Subfinder + httpx combined |
| API uptime | 99.9% | Monthly, excluding scheduled maintenance |

**Load test scenarios (run before every release):**
1. 50 concurrent API users browsing dashboards
2. 10 concurrent full-spectrum scans across 5 projects
3. 1,000 API key requests/minute (CI/CD simulation)
4. SSE with 20 simultaneous scan progress streams
5. 5 concurrent mobile uploads (APK/IPA — 100MB each)

---

## 12. Risk Register

| # | Risk | Impact | Probability | Mitigation |
|---|---|---|---|---|
| 1 | AI costs exceed revenue at scale | High | Medium | Aggressive caching, tiered depth, usage caps, batch processing |
| 2 | False positives erode user trust | High | Medium | AI dedup, ZAP confirmation, corroborated badges, user feedback training |
| 3 | Target sites block our scanners | Medium | High | Configurable rate limiting (default 50 req/s), user-agent rotation, robots.txt awareness |
| 4 | Go scan engine complexity slows development | Medium | Medium | Start with Nuclei + MobSF, incremental tool addition, keep Docker runner generic |
| 5 | Competitors copy AI features | Medium | Low | Execution speed, data flywheel from marketplace, continuous model improvement |
| 6 | Legal liability from scanning | High | Low | Mandatory domain verification, ToS requiring target ownership, scan logging |
| 7 | Redis failure causes data loss | High | Low | Redis Sentinel HA, job idempotency, scan state in PostgreSQL |
| 8 | Claude API rate limits during peak | Medium | Medium | Queue-based throttling, request batching, fallback to cached fixes, circuit breaker |
| 9 | Docker container escapes on shared workers | High | Low | Rootless containers, seccomp, resource limits, network isolation |
| 10 | Credential leakage from project configs | Critical | Low | Fernet encryption at rest, never logged, memory-only decryption, key rotation |
| 11 | MobSF high memory on small workers | Medium | High | 8GB worker minimum, mobile jobs on dedicated queue |
| 12 | Malicious APK upload infects worker | Critical | Low | Magic byte validation, ClamAV scan, MobSF sandboxed container, no execution |
| 13 | Recon discovers domains outside client ownership | Medium | Medium | Recon limited to apex domain + verified subdomains, max 100 subdomains per scan |
| 14 | MobSF version drift between workers | Medium | Medium | Pinned Docker image version, health check on startup |
| 15 | Large APK/IPA files slow queue | Medium | Medium | 500MB file limit, separate mobile queue with lower concurrency |

---

## 13. Architectural Decision Records

### ADR-001: FastAPI + Go Hybrid Architecture
**Status:** Accepted
**Decision:** Python (FastAPI) handles API + AI + business logic. Go handles scan execution + agent. Redis is the only coupling point.
**Rationale:** Python excels at AI/ML and rapid API development. Go excels at concurrent network I/O. Clean language boundary via Redis.

### ADR-002: PostgreSQL + Qdrant (Not pgvector)
**Status:** Accepted
**Decision:** PostgreSQL as source of truth. Qdrant as derived vector index.
**Rationale:** Vector search 10–50x faster than pgvector at scale. Qdrant rebuildable from PG if lost. Graceful degradation.

### ADR-003: Redis for Job Queues (Not Kafka/RabbitMQ)
**Status:** Accepted
**Decision:** Redis with asynq library. Redis Streams for pub/sub.
**Rationale:** Simpler ops than Kafka/RabbitMQ. Redis Sentinel provides HA. Queue interface abstracted — swappable later.

### ADR-004: Multi-Provider AI Strategy
**Status:** Accepted
**Decision:** Claude for reasoning. OpenAI for embeddings. Local rules for deterministic ops.
**Rationale:** Best-in-class per task type. ~$0.25/scan for Growth tier (vs $0.80 all-Claude).

### ADR-005: Per-Developer + Flat Pricing Hybrid
**Status:** Accepted
**Decision:** Flat tiers for small teams, per-developer scaling for larger.
**Rationale:** Captures solo devs who balk at per-seat. Revenue scales with customer value on upper tiers.

### ADR-006: Hybrid Native + Persistent Docker (Refined from v2)
**Status:** Accepted (updated 2026-04-18)
**Original decision:** All tools in per-scan Docker containers.
**Updated decision:** Lightweight native binaries + persistent Docker services for heavy tools (ZAP, MobSF, Trivy, SQLMap).
**Rationale:** Eliminates 2–3s per-scan container startup for heavy tools. MobSF is designed as a persistent service. Native binaries fastest for CLI tools.

### ADR-007: Recon-First Pipeline
**Status:** Accepted (2026-04-18)
**Decision:** Every web scan starts with Subfinder + httpx. Discovered live subdomains automatically added to scan target list (up to subdomain_limit).
**Rationale:** Massive differentiator — scans assets clients didn't know existed. Zero extra cost because Subfinder/httpx run in seconds.

### ADR-008: MobSF for Mobile Security
**Status:** Accepted (2026-04-18)
**Decision:** MobSF as persistent Docker service for all mobile scanning (APK, IPA, source).
**Rationale:** Best open-source mobile security tool. Covers both platforms and both static + binary analysis. Zero licensing cost. REST API first-class integration.

### ADR-009: Recon-First + Full 9-Category Coverage vs. Specialization
**Status:** Accepted (2026-04-18)
**Decision:** Build full 9-category platform rather than specialize in one area.
**Rationale:** Every major competitor specializes (Snyk: SAST+SCA, Burp: DAST, MobSF: mobile only). Unified full-spectrum platform is unique market position at SME price point.

### ADR-010: Password Hashing and JWT Signing Choices
**Status:** Accepted (2026-04-21)

**Context:**
- IMPLEMENTATION-PLAN.md §2.1 originally specified `passlib.context.CryptContext` as the password-hashing wrapper.
- At Task 2.1 implementation we discovered `passlib 1.7.4` (its last release, Oct 2020) is incompatible with `bcrypt >= 4.1`: it probes `bcrypt.__about__.__version__`, an attribute the newer bcrypt library has removed. The failure path silently raises a misleading "password cannot be longer than 72 bytes" error on arbitrary inputs.
- `passlib` has had no release in over 5 years and is effectively unmaintained.

**Decision (password hashing):**
- Use the PyPA-maintained `bcrypt` library directly; remove `passlib`.
- Cost factor **12**, pinned explicitly via `bcrypt.gensalt(rounds=12)`.
- Abstraction boundary is `hash_password(pwd)` / `verify_password(plain, hashed)` in `app.services.auth`. Internals can be swapped (e.g. to Argon2id) later without changing call sites.

**Decision (JWT signing):**
- **HS256** for now. Acceptable because:
  - Single signing key, held only by `shieldscan-api`.
  - No inter-service verification requirement yet.
- Revisit **RS256** (asymmetric) when any of: (a) a service outside `shieldscan-api` must verify tokens, (b) enterprise tier launches with multi-region key distribution, or (c) a compliance requirement demands it.

**Consequences:**
- `passlib` removed from `VERSIONS.md` and `shieldscan-api/pyproject.toml`.
- `bcrypt` pinned directly (`^5.0`).
- The bcrypt 72-byte password-length limit becomes an API-layer concern: the register endpoint (Task 2.2) must explicitly reject passwords >72 bytes with HTTP 400. `hash_password`/`verify_password` do not enforce it.
- Crypto-agility is preserved — migrating to Argon2id later touches only the two service-layer functions.

### ADR-011: Refresh Token Revocation via Redis jti-Set
**Status:** Accepted (2026-04-21)

**Context:**
- JWT tokens are self-contained; once issued, they are cryptographically valid until `exp`. We need a way to invalidate specific tokens *before* `exp` — triggered by logout, password change, suspected compromise, or admin action.
- Options considered: stateful session store, DB-backed revocation list, Redis-backed revocation list, signed-token-version (global bump).

**Decision:**
- **Primary mechanism (targeted, per-jti):** Redis `SET revoked_jti:{jti} 1 EX (exp - now)`. Auth path checks `is_revoked(jti)` after signature verification in `get_auth_identity` (and in `/auth/refresh` for refresh tokens). TTL auto-matches the token's own lifetime — no GC / reaper needed.
- **Refresh-rotation ordering: revoke OLD jti BEFORE minting NEW tokens.** If the mint fails after a successful revoke, the worst case is the client re-logs-in. If we minted first and then lost the revoke, the client would briefly hold two valid refresh tokens (security issue). Fail-closed.
- **User-level revocation (shipped in Task 2.X):** Redis `SET user_revoked_before:{user_id} <unix_ts> EX (JWT_REFRESH_TOKEN_EXPIRE_DAYS × 86400)`. Auth path compares the token's `iat` claim against the stored timestamp; `iat < revoked_before` → 401. Used by `POST /auth/password/change` to invalidate every token issued before the change without enumerating jtis. Consumer hooks for future admin-force-logout / account-recovery / compromise-response are now in place. Short-circuits on missing key (Option C — sub-millisecond Redis GET on the 99.9% no-revocation path).

**Consequences:**
- Redis becomes a hard dependency of the auth path (cross-ref OPERATIONS-RUNBOOK §11.6).
- Fail-closed: Redis unreachable → all auth fails → readiness probe returns 503 → LB stops routing. Preferred over "serve with auth broken."
- O(1) revocation lookup — `EXISTS` on a small key.
- Revocation-key space grows with revocations but auto-expires with TTL; no unbounded growth.
- Multi-region deployment requires Redis replication OR per-region revocation (deferred — single-region in M2).

**Reuse-detection behavior (Task 2.3.4 + Task 2.X amendment):**
- If a client presents a refresh token whose `jti` is already in the revocation list, we treat it as a **compromise signal** — legitimate clients never replay an already-rotated refresh token.
- The `/refresh` endpoint audits `auth.token.revoked` with `details={"reason": "reuse_detected", "token_jti": <jti>}` and returns 401.
- **Auto-trigger of user-level revocation on reuse is deliberately NOT enabled** even though the mechanism is now available (Task 2.X). Considerations: legitimate network-blip retries could trigger false positives, the signal is observability-poor (no undo path), and revocation storms from buggy client libraries would be self-amplifying. A future ADR will revisit with a deliberate policy decision; until then, reuse-detection remains targeted-jti-only. See DRIFT-LOG 2026-04-23.

**Alternatives rejected:**
- **DB-backed revocation list:** auth-path DB hit on every authenticated request. Unacceptable performance ceiling — O(1) Redis is strictly better.
- **Short-lived tokens only (no refresh revocation):** still need a revocation path for refresh tokens (which are long-lived by design); pushes UX problems onto users (frequent re-login) for no security win.
- **Signed-token-version (global bump):** cannot revoke individual sessions without revoking every token in circulation. Too coarse.

**Cross-references:**
- `OPERATIONS-RUNBOOK.md` §11.6 — Redis-hard-dep posture
- `shieldscan-api` commit `f6b9bbf` — `jti` claim added (revocation hook)
- `shieldscan-api` commit `4d32fcc` — `iat` claim added (user-level hook)
- `shieldscan-api` commit `88b6966` — `token_revocation.py` primitive
- `shieldscan-api` commit `711dfbc` — revocation integrated into `/auth/refresh` + `/auth/logout`
- `shieldscan-api` commit `07d0189` — user-level revocation primitive + auth-path check (Task 2.X Commit 1)
- `shieldscan-api` commit `1bb0b66` — `/auth/password/change` endpoint (Task 2.X Commit 2)

### ADR-012: Credential-Indexed Tables Use App-Layer Tenant Scoping
**Status:** Accepted (2026-04-21)

**Context:**
- M1 established RLS at the DB layer for all tenant-scoped tables (see M1.6, `policies.py`, `TENANT_TABLES`).
- RLS policies require `current_setting('app.current_org_id')` to be set before queries return rows.
- API-key authentication creates a chicken-and-egg problem: the lookup by `key_hash` MUST precede establishing tenant context, because the org is unknown until the row is read.
- This problem is not unique to `api_keys`. Any "lookup by opaque credential" pattern has the same shape (webhook-signature verification, invitation tokens, password-reset tokens if DB-backed, etc.).

**Decision:**
- Tables where the row identifier IS the credential bypass RLS.
- For these tables, tenant scoping is enforced at the **application layer**: every query path that reads these tables either
  (a) looks up by the opaque credential (hash match = proof of ownership, no further scoping needed), or
  (b) explicitly filters `WHERE organization_id = :authenticated_org`.
- Tables in this category must be enumerated in a `CREDENTIAL_INDEXED_TABLES` constant in `policies.py` with a justification string.
- The first table in this category: `api_keys`.

**Why this is not a security regression:**
- `api_keys` rows can only be read via:
  - `key_hash` lookup (reading requires the plaintext, which IS the credential — reader is already authenticated for that org), or
  - list/delete by org (explicit org filter at application layer).
- RLS provides defense-in-depth when app-layer scoping might be forgotten. Here, forgetting the org filter would be equally visible in either regime — both a missing RLS policy and a missing `WHERE` clause fail noisy code review.
- The alternative (`SECURITY DEFINER` function) introduces a permanent bypass primitive with worse properties: future engineers may reuse it incorrectly, and the defense becomes "the function is written correctly" rather than "the lookup pattern is cryptographically self-protecting."

**Consequences:**
- `api_keys` has no RLS policy, no `FORCE ROW LEVEL SECURITY`, and is NOT in `TENANT_TABLES`.
- New `CREDENTIAL_INDEXED_TABLES: Final[dict[str, str]]` constant in `policies.py` lists `api_keys` with a justification comment.
- **Adding any future table to `CREDENTIAL_INDEXED_TABLES` requires an ADR amendment** — not just a code change. The justification string is the forcing function for code review.
- App-layer cross-tenant isolation tests replace DB-layer tests for `api_keys`; the list endpoint test and delete endpoint test MUST include cross-tenant scenarios with an assertion that other orgs' keys are invisible.
- `audit_logs` append-only trigger still applies — `api_keys` row changes still go to `audit_logs`.

**Alternatives considered:**
- **`SECURITY DEFINER` function:** rejected. Introduces a permanent bypass primitive; defense becomes function-correctness rather than lookup-pattern-correctness; pattern may be misused later.
- **GUC carve-out (`app.api_key_lookup = 'true'`):** rejected. Session-level GUC that bypasses RLS is a security footgun; forgetting to unset leaves the entire session bypassing.
- **Prefix-encoded org id:** rejected. Breaks Task 2.2's `env`-prefix convention; leaks org identity through the key.

**Cross-references:**
- ADR-011 (revocation strategy for JWT, also Redis-backed)
- `shieldscan-api` commit `9a4e2c1b7d3f` Alembic rev + `policies.py` refactor (Task 2.4 Commit 0)
- `shieldscan-api` regression tests: `tests/models/test_credential_indexed_rls.py`

### ADR-013: Python is the Sole Writer for Scan State
**Status:** Accepted (2026-04-30, Task 4.2)

**Context:**
Scans and scan_jobs have lifecycle state (`queued → reconning → running → analyzing → completed | partial | failed | canceled`). Two services touch these flows: the Python API (creates scans, dispatches jobs, lists results, cancels) and the Go workers (process jobs, run tools, emit progress + completion events).

The question: who is the canonical writer for `Scan.status` + `ScanJob.status` columns?

**Decision:**
**Python is the sole writer.** Go workers communicate every state change via Redis events; they never write to PostgreSQL.

Concretely:
- Python `ScanOrchestrator.dispatch()` (Task 4.2) inserts `Scan` (`status=queued`) + `ScanJob` rows.
- Python `CompletionsConsumer` (Task 4.2) subscribes to `shieldscan:completions` and UPDATEs `scan_jobs` per event; aggregates `Scan.status` when all sibling jobs reach a terminal state.
- Python cancel endpoint (Task 4.5) UPDATEs `Scan.status = canceled` and emits `shieldscan:cancel:{scan_id}` to signal workers.
- Go workers consume `shieldscan:queue:{priority}`, run tools, publish progress to `shieldscan:progress:{scan_id}` Stream + completion to `shieldscan:completions` channel. **No PostgreSQL connection.**

**Consequences:**
- One codepath per state transition. Eliminates concurrent-update races between services.
- Worker outage = events lag, jobs stay `queued`/`running` until events resume. No DB drift, no inconsistent partial-update state.
- Go workers don't need PostgreSQL credentials in their config (M5 task 5.6 — see cross-references). One less attack surface.
- Python becomes the bottleneck for state transitions — but at MVP scale this is fine; the completions consumer is a single async task processing one event at a time, which keeps ordering simple.
- Python is the source of truth for "what state is this scan in?" — Redis is signaling, PostgreSQL is state.

**Anti-patterns this prevents:**
- **Go workers writing partial-completion state to PG to "help" Python aggregate.** Workers send events; Python interprets. Even if a Go engineer is convinced the round-trip is "wasteful," the single-writer discipline is what makes the design tractable.
- **Python re-fetching scan state from Redis as a "cache" before PG.** PG is truth; Redis is signaling. Reading state from Redis in Python is a category error.
- **Periodic reconciliation jobs that "verify Redis matches PG."** No reconciliation needed — state lives in one place. If you find yourself writing one, the design has been violated upstream.

**Alternatives considered and rejected:**
- **Dual writers (Python + Go).** Would require cross-service distributed locking (advisory locks, Redlock) on every state-bearing column. Concurrency complexity not justified by performance gain at MVP scale.
- **Go writes ScanJob, Python writes Scan.** Clean column-level ownership but introduces a hard cross-service handoff on the "all jobs done → Scan.status = completed" transition. Complexity not justified.
- **Event-sourced architecture (events as source of truth, projections to PG).** Bigger architectural commitment; deferred.

**Forcing functions:**
- Workers (M5 task 5.6) are configured WITHOUT PostgreSQL credentials. Attempting to write would fail at the network/auth layer — failure is unmissable.
- Tests in `tests/services/test_completions_consumer.py` verify the Redis event → Python UPDATE round-trip. Specifically `test_consumer_aggregates_scan_status_when_all_jobs_terminal` asserts Python UPDATEs aggregate status from event consumption, not from any external trigger. If a future "optimization" tries to have Go workers write aggregate directly, this test breaks immediately.
- ADR-014 (Streams over Pub/Sub) is a precondition: progress events flow through Streams, completion events through Pub/Sub. Same single-direction signaling discipline.

**Open follow-ups:**
- ADR-015 (decrypted credentials in Redis transit): defer until the orchestrator's `auth` block is enabled in job payloads (currently `null`).
- Multi-region scale: completions consumer is currently one task per API process. At 100+ concurrent scans, may need consumer-group dispatch. Not MVP.

**Cross-references:**
- ADR-014 (Streams over Pub/Sub for progress events).
- SPECIFICATION §7.3 (Job Completion channel format).
- IMPLEMENTATION-PLAN.md M5 task 5.6 (worker startup) — MUST exclude PG credentials. This ADR is referenced there as the reason.
- `shieldscan-api` commit `cf3b30a` — `app.services.orchestrator` + `app.services.completions_consumer`.

### ADR-014: Redis Streams (not Pub/Sub) for scan progress events
**Status:** Accepted (2026-04-30, Task 4.1)

**Context:**
Scan progress events flow Go worker → Redis → Python API → SSE → web client. SPECIFICATION §6.4 requires the SSE endpoint to honor `Last-Event-ID` headers and replay events from the last 60 seconds. The plan literal in IMPLEMENTATION-PLAN.md §4.1 + §4.4 sketched a Pub/Sub-only design that cannot satisfy the replay requirement — Pub/Sub is fire-and-forget with no native replay. On a multi-process uvicorn worker pool, an in-process ring-buffer alternative is process-local: a client reconnecting to a different worker process loses history.

**Decision:**
Use Redis Streams (`XADD` for produce, `XREAD BLOCK` for live tail, `XRANGE` for replay) as the single primitive for `shieldscan:progress:{scan_id}`. Bound retention with `XADD ... MAXLEN ~ 1000` (approximate trim) — covers ~60s of replay at peak emit rate.

Mixed-primitive use is intentional:
- `shieldscan:progress:{scan_id}` → **Streams** (replay required).
- `shieldscan:cancel:{scan_id}` → **Pub/Sub** (one-shot live-only; a worker not subscribed when cancel emits cannot usefully consume a stale cancel — replay would be misleading).
- `shieldscan:completions` → **Pub/Sub** (one-shot completion broadcast; the orchestrator's completions consumer either receives the live event or rebuilds state from the database row at startup — replay adds no value).

**Consequences:**
- Multi-process API server reconnects work correctly: any uvicorn worker reads the same Stream, replay is consistent.
- Stream keys persist after scan completion; cleanup requires either (a) ops-milestone janitor with TTL on completed scans, or (b) periodic `XADD MAXLEN` keeping the per-scan key bounded but accumulating empty/inactive keys. Carry-forward to OPS milestone.
- One write per event. Equivalent fan-out cost to Pub/Sub at MVP load (1–3 concurrent scans × <10 viewers each); `XREAD BLOCK` is Redis's Streams-equivalent of `SUBSCRIBE`.
- SPEC §7.2 wording patched in the same docs commit ("Channel:" → "Stream:" + XADD/XREAD/XRANGE primitives).

**Alternatives rejected:**
- **Pure Pub/Sub.** Cannot satisfy §6.4 replay on multi-process server. In-process ring buffer is process-local; reconnects routed to a different worker lose history. Disqualifying.
- **Hybrid (Streams + Pub/Sub).** Two writes per event, two consumer codepaths, two failure modes (Stream wrote but Pub/Sub failed → live consumers miss; Pub/Sub wrote but Stream failed → replay misses). Justifiable only if Streams alone shows latency/fan-out concerns at scale. At MVP MENA-SMB scale, `XREAD BLOCK` is an equivalent fan-out primitive to `SUBSCRIBE`. Reject hybrid for MVP; revisit if sustained-load profiling later shows fan-out is hot. The wrapper isolation in `app/services/scan_queue.py` makes that swap mechanical.

**Forcing functions:**
Tests in `tests/services/test_scan_queue.py` exercise XADD + XREAD round-trip + XRANGE replay. Switching to Pub/Sub would break those tests. Specifically: `test_subscriber_replay_returns_history` requires `XRANGE` semantics — Pub/Sub has no equivalent. The test docstring explicitly references ADR-014 so an engineer debugging a switchback finds the reasoning immediately.

**Open follow-ups:**
- Stream-key cleanup TTL → OPS milestone (track scan completion time, purge stream keys for completed scans after 24–48h).
- ADR-013 (state-machine ownership) — docked at Task 4.2 with the orchestrator commit.

**Cross-references:**
- SPECIFICATION §6.4 (SSE replay requirement).
- SPECIFICATION §7.2 (channel → stream patch landed in same docs commit).
- `shieldscan-api` commit `349fc5e` — `app.services.scan_queue`.
- `tests/services/test_scan_queue.py::test_subscriber_replay_returns_history` — regression guard.

### ADR-016: Raw Redis (not Asynq) for Go-side queue protocol
**Status:** Accepted (2026-05-01, Task 5.1)

**Context:**
SPECIFICATION §3.3 + IMPLEMENTATION-PLAN.md preamble both originally named "asynq" in the stack. Plan Task 5.1's `go.mod` block (lines 1510-1525) imports `github.com/hibiken/asynq v0.24.1`. **But:**
- M4 Python orchestrator (Task 4.1, shipped) dispatches via raw `LPUSH shieldscan:queue:{priority}`.
- Plan Task 5.4 step 2 consumes via raw `BRPOP shieldscan:queue:{priority}` — matches the Python side.
- Asynq has its own queue protocol (`asynq:{queue}`, `asynq:active`, JSON envelope with own fields including `type`, `payload`, `id`, `retry`, `unique_key`). **The two sides cannot co-exist on the same queue with raw + asynq mixed.**

The asynq dependency in plan §5.1 is therefore dead weight as written — imported but unused, with the actual queue logic in §5.4 using raw Redis primitives.

The question: drop asynq, or refactor M4 to adopt it end-to-end?

**Decision:**
**Drop asynq from the stack.** Use raw Redis `LPUSH` + `BRPOP` end-to-end on `shieldscan:queue:{priority}` queues. The Go consumer in Task 5.4 reads the exact JSON payload Python's M4 orchestrator already writes (per SPEC §7.1 schema) — no envelope, no library coupling.

Concretely:
- Remove `github.com/hibiken/asynq` from `shieldscan-engine/go.mod` (Checkpoint 4 commit `39e1e5e`, already done).
- Update SPECIFICATION §3.3 stack list to drop "asynq" (this commit).
- Update IMPLEMENTATION-PLAN.md preamble stack list to drop "asynq" (this commit).
- Plan §5.1 go.mod literal stays as written (state-at-time discipline); Task 5.1 implementation authors go.mod from VERSIONS.md per CLAUDE.md hierarchy.
- Plan §5.4 raw `BRPOP` literal is the canonical implementation shape — no plan correction needed for §5.4's queue side (Pub/Sub vs Streams progress is a separate concern; see ADR-018).

**Consequences:**
- Smaller dependency graph: `go.mod` drops asynq + transitive deps. Less attack surface, faster builds, fewer security-advisory PRs.
- Custom retry logic: ADR-013 already moved retry/state-machine ownership to Python (commit-then-dispatch + completions consumer). Asynq's retry primitives would have been redundant.
- Custom dedup: idempotency_key + Redis SETNX (Task 5.5) is what we ship. Asynq's `unique_key` would have been redundant.
- Lose Asynq dashboard — observability provided by Sentry (`getsentry/sentry-go ^v0.29.0` in VERSIONS.md §2.4) + Prometheus (`prometheus/client_golang v1.20.0`).
- Lose scheduled-jobs primitive — no MVP need; cron-equivalents land at OPS milestone if needed.
- Single source of queue-protocol truth: SPEC §7.1 schema. Both sides reference it directly; no library version drift to manage.

**Anti-patterns this prevents:**
- **"Should we adopt Asynq?" rediscovery cycles** every 6 months. ADR pins the call so future engineers find the reasoning before re-litigating.
- **Hybrid migration ("we'll refactor to Asynq later")** that never happens but rots in TODOs and dead imports.

**Alternatives considered and rejected:**
- **Adopt Asynq end-to-end (refactor M4 Python).** Requires either a Python Asynq-compatible client library (third-party with weaker maintenance posture) or hand-rolling Asynq's wire-format encoding in Python. Cost: M4 contract churn + new third-party dep + test rewrites. Benefit: dashboard + retry/scheduling primitives already replaced by ADR-013 + Sentry. Cost > benefit.
- **Adopt Asynq on the Go side only, consuming from raw queues.** Doesn't work — Asynq is not a generic queue wrapper; its consumer expects asynq-encoded entries in asynq's own keyspace.
- **Adopt Asynq for a future job category (e.g., scheduled scans).** Defer-on-trigger pattern: revisit if/when scheduled-scan use case lands and the cron-equivalent build cost exceeds Asynq adoption cost. Carry-forward, not blocker.

**Forcing functions:**
- `shieldscan-engine/go.mod` MUST NOT contain `github.com/hibiken/asynq`. Verified by Task 5.1 buildguard test:
  ```go
  func TestGoMod_ExcludesAsynq(t *testing.T) {
      out, err := exec.Command("go", "list", "-m", "all").Output()
      require.NoError(t, err)
      assert.NotContains(t, string(out), "hibiken/asynq",
          "asynq must not appear in dependency graph (ADR-016)")
  }
  ```
  This pins the rule at the build-graph layer, not just by code review.
- VERSIONS.md §2.4 inline comment at the top of the require block: `// asynq dropped per ADR-016 — raw Redis matches M4 dispatch contract`. Anyone considering re-adding asynq sees the rationale before opening this ADR.

**Open follow-ups:**
- If scheduled-scan use case lands (Phase 2), revisit Asynq adoption as a targeted dependency for that job category only — would not affect the M4-shipped raw-queue path.

**Cross-references:**
- ADR-013 (Python sole writer) — the load-bearing reason Asynq's retry/state primitives are redundant.
- ADR-014 (Streams over Pub/Sub) — same plan-correction shape; both ADRs override stale plan literals.
- SPECIFICATION §7.1 (Job Dispatch schema) — canonical contract; both sides reference it directly.
- IMPLEMENTATION-PLAN.md §5.1 lines 1510-1525 (stale go.mod literal) — left as written per state-at-time discipline.
- IMPLEMENTATION-PLAN.md §5.4 lines 1644-1663 (raw BRPOP literal) — canonical implementation shape, retained.
- M4 commit `349fc5e` (`app.services.scan_queue.ScanQueue.push`) — the LPUSH-side contract this Go consumer mirrors.
- Checkpoint 4 commit `39e1e5e` (asynq removed from VERSIONS.md §2.4).

### ADR-017: Findings inline in `job_completed` events (with sequencing)
**Status:** Accepted (2026-05-01, Task 5.1)

**Context:**
ADR-013 (Python sole writer) forbids Go workers from writing to PostgreSQL. But scan findings are produced in Go (tool runners parse stdout, build `RawFinding` structs) and must end up persisted in the `vulnerabilities`/`raw_findings` PG tables for downstream queries (M9 AI pipeline, M10 read-side endpoints, M11 dashboard).

Plan Task 5.5 step 2 has:
```go
if err := w.storage.StoreFindings(ctx, findings); err != nil { return err }
publisher.Publish(ctx, "job_completed", ...)
```
That's a direct Go→PG write, contradicting ADR-013. Plan was authored before ADR-013; the contradiction is plan-staleness, not architectural intent.

The question: how do findings cross the Go→Python boundary, given the sole-writer rule?

**Decision:**
**Inline findings in the `job_completed` event payload.** The Python `CompletionsConsumer` (Task 4.2, lifespan-managed, runs under the sole-writer DB role) inserts the findings rows in the same transaction as the `ScanJob.status` update. SPEC §7.3 schema patched in this commit to add `findings` array + `event_seq` object.

**Soft cap + sequencing.** Workers cap each event at **1000 findings** (`MaxFindingsPerEvent`, defined in `shieldscan-engine/internal/events/events.go`). Jobs producing >1000 findings split into multiple `job_completed` events with `event_seq.total > 1`:
- Intermediate batches: `event_seq: {index: N, total: T}`, `N < T`, `status: "partial_findings"`.
- Terminal batch: `event_seq: {index: T, total: T}`, canonical `status`/`finding_count`/`duration_ms`.

Python `CompletionsConsumer` semantics:
- On `event_seq.total == 1`: existing single-event path — insert findings + update ScanJob status in one txn.
- On `event_seq.total > 1`: collect batches keyed by `(scan_id, job_id)` in an in-memory accumulator. On terminal batch, insert the union of accumulated batches + terminal-batch findings + update ScanJob status — single txn at the end.

**Trigger for Option C migration (R2 staging).** Promote findings persistence to R2-staged batch when **any** of the following is observed in production:
1. Sustained `job_completed` event payloads exceeding **5MB** (operational metric on Pub/Sub message size).
2. Pub/Sub message size approaching the Redis 32MB hard limit (early-warning at 16MB sustained).
3. M9 AI pipeline shows ingest-time problems with batch sizes — e.g., deep Nuclei or Trivy scans producing 5000+ findings per job, where the Python consumer's per-event txn time exceeds 30s and starts blocking subsequent events.
4. **Any production occurrence** of "ScanJob ghost-queued due to mid-sequence crash" (see Consequences below) — once is the trigger, not sustained.

Migration shape (deferred): worker writes findings batch to R2 as JSONL blob (`findings/{org_id}/{scan_id}/{job_id}.jsonl`), `job_completed` event carries `findings_ref: "r2://..."` instead of inline `findings`, Python consumer downloads + streams insert. Decouples size + Redis pressure. ADR amendment + new ADR on landing.

**Consequences:**
- Single Python transaction per event — preserves ADR-013 sole-writer atomicity. ScanJob.status update + findings insert commit together.
- No new Redis primitive surfaces. `shieldscan:completions` Pub/Sub channel grows in payload, not in primitive complexity.
- Existing `CompletionsConsumer` shape (Task 4.2, lifespan-managed, session_factory DI) needs minimal extension: parse `findings` field, parse `event_seq`, accumulate when needed, persist on terminal batch. **Extension is Task 5.5 scope, not Task 5.1.**
- Pub/Sub size pressure: 1000 findings × ~2KB/finding = ~2MB worst case. Comfortably under the 32MB Redis Pub/Sub default. If average finding size grows (rich evidence blobs, base64 screenshots), the trigger conditions activate before saturation.
- M5 implementation note: the 1000-finding cap + sequencing pattern lands at **Task 5.5** (where the worker emits completion events). M5 itself emits zero findings (no tool runners until M6.1), so the cap is exercised structurally (via tests with synthetic findings) but not behaviorally until M6.1 Nuclei.
- **Accumulator failure-mode.** Python CompletionsConsumer holds an in-memory accumulator keyed by `(scan_id, job_id)` for sequenced events. Accumulator survives only the single API process; on crash mid-sequence, partial findings are lost AND the `ScanJob.status` remains `running` (terminal event never lands). Recovery: M5+ ghost-queued janitor (Task 4.2 carry-forward) sweeps stuck ScanJob rows and either re-dispatches or marks failed. **Loss-rate threshold for triggering ADR-017→Option-C migration: any production occurrence of "ScanJob ghost-queued due to mid-sequence crash" — once is the trigger, not sustained.** Operationally: this is a sharp-edge case at MVP scale (deep scans + API restart simultaneously) but the recovery path must work.

**Anti-patterns this prevents:**
- **Go workers opening a PG connection "just for findings."** ADR-013 violation; this ADR closes the loophole.
- **A separate `shieldscan:findings:{scan_id}` Stream.** Adds a third Redis primitive shape, third Python consumer codepath, third failure mode. Considered in M5 landscape (Option B); rejected as premature complexity.
- **Per-finding events on the progress Stream.** Progress Stream is bounded `MAXLEN ~ 1000` for replay; finding events would either pollute replay or get evicted before consumption. Wrong primitive.

**Alternatives considered and rejected:**
- **Option B: separate `shieldscan:findings:{scan_id}` Stream.** Three Redis primitives instead of two. Streams give replay safety if the consumer is briefly down — but Pub/Sub is not lossy at MVP scale (single API process subscribed continuously) and the in-memory-accumulator crash-recovery concern is the same shape either way. Rejected as added complexity without proportionate benefit.
- **Option C: R2 staging.** Decouples size + Redis pressure, but adds R2 round-trip latency to every job completion + R2 cleanup lifecycle + Python consumer download path. Right answer at scale; wrong answer at MVP. Deferred per trigger conditions above.
- **Synchronous PG-write from Python during job dispatch (pre-allocate finding rows).** Doesn't work — findings are unknown until tools run.
- **Worker writes to a worker-local sqlite + Python pulls on demand.** Operational nightmare; rejected outright.

**Forcing functions:**
- 1000-finding soft cap enforced in worker emitter code (Task 5.5):
  ```go
  // shieldscan-engine/internal/events/events.go
  const MaxFindingsPerEvent = 1000  // ADR-017 soft cap
  ```
  Constant lives in `internal/events/` (created at Task 5.1) so 5.5's emitter and 5.5's tests both import the canonical value. Test pin (Task 5.5): `TestProcessor_SplitsLargeFindingBatches` synthesizes 2500 findings, asserts emitter produces 3 events with `event_seq.total = 3`.
- Python CompletionsConsumer test pin (Task 5.5 cross-repo): `test_consumer_aggregates_sequenced_findings` synthesizes 3 sequenced events, asserts single-txn commit on terminal batch with all 2500 findings persisted.
- Inline comment on the constant cites ADR-017 so engineers tempted to bump it find the trigger conditions first.
- The Option C migration trigger metrics (5MB / 16MB / 30s consumer txn time / 1× ghost-queued occurrence) become Prometheus alerts at OPS milestone — not just doc text but operational signals that surface when the trigger actually fires.

**Open follow-ups:**
- **R2-staging ADR.** Drafted on trigger-fire; not preempted now.
- **Schema versioning of `RawFinding`.** Worker and consumer must agree on field names. Either (a) shared schema doc (SPEC §7.3 + this ADR), (b) generated bindings, or (c) JSON-schema validation in consumer. Carry-forward to Task 5.5 scope proposal.

**Cross-references:**
- ADR-013 (Python sole writer) — the load-bearing constraint that forced this design.
- IMPLEMENTATION-PLAN.md §5.5 lines 1742-1748 (stale `w.storage.StoreFindings` literal) — Task 5.5 implementation replaces this with publish-via-completions.
- M4 commit `cf3b30a` (`app.services.completions_consumer`) — the consumer this ADR extends.
- SPECIFICATION §7.3 (Job Completion schema) — patched in this commit to add `findings` + `event_seq` fields.
- Task 4.2 DRIFT-LOG entry "long-lived background tasks need session_factory DI" — the consumer's session lifecycle pattern, reused for sequenced-event accumulator.

### ADR-018: Plan §5.4 correction — Streams (not Pub/Sub) for progress events
**Status:** Accepted (2026-05-01, Task 5.1)

**Context:**
This is a plan-correction ADR (precedent: ADR-014 itself was a plan correction over §4.1's Pub/Sub-only sketch). The plan literal at IMPLEMENTATION-PLAN.md §5.4 lines 1665-1677:

```go
// internal/redis/pubsub.go
func (p *ProgressPublisher) Publish(ctx context.Context, eventType string, payload map[string]interface{}) error {
    // ...
    return p.client.Publish(ctx, "shieldscan:progress:"+p.scanID, data).Err()
}
```

That's `client.Publish` — Pub/Sub. **Contradicts ADR-014 + SPECIFICATION §7.2** (already patched in M4 docs commit) which mandate Streams: `XADD shieldscan:progress:{scan_id} MAXLEN ~ 1000 * event <json>`. Plan was authored pre-ADR-014.

The question: which side wins, plan §5.4 or ADR-014?

**Decision:**
**ADR-014 wins. Plan §5.4 progress publisher implementation is corrected to Streams.** Task 5.4 ships `XAdd` against `shieldscan:progress:{scan_id}` matching the M4 Python `ProgressSubscriber` (Task 4.1, commit `349fc5e`) exactly:

```go
// shieldscan-engine/internal/redis/stream.go (note: NOT pubsub.go for progress)
const ProgressMaxLen = 1000  // matches Python ProgressSubscriber/XADD MAXLEN

func (p *ProgressPublisher) Publish(ctx context.Context, eventType string, payload map[string]any) error {
    payload["event_type"] = eventType
    payload["scan_id"] = p.scanID
    payload["timestamp"] = time.Now().UTC().Format(time.RFC3339)
    data, err := json.Marshal(payload)
    if err != nil { return fmt.Errorf("progress marshal: %w", err) }
    return p.client.XAdd(ctx, &redis.XAddArgs{
        Stream: "shieldscan:progress:" + p.scanID,
        MaxLen: ProgressMaxLen,
        Approx: true,  // MAXLEN ~ 1000 (approximate trim — matches Python)
        Values: map[string]any{"event": string(data)},
    }).Err()
}
```

Plan §5.4's filename `internal/redis/pubsub.go` is split: progress goes to `internal/redis/stream.go`; cancel-subscriber + completions-publisher remain in `internal/redis/pubsub.go` (those primitives stay Pub/Sub per ADR-014 mixed-primitive rationale).

**Consequences:**
- Multi-process API server SSE replay works correctly (the load-bearing reason for ADR-014).
- M4 Python `ProgressSubscriber` (Task 4.1) reads what Go writes without modification — single contract surface in SPEC §7.2.
- Plan §5.4 lines 1665-1677 stay as written (state-at-time discipline) but Task 5.4 implementation diverges per this ADR.
- File layout split (`stream.go` for progress, `pubsub.go` for cancel/completions) makes the primitive boundary visible at the package level.

**Anti-patterns this prevents:**
- **"Quick fix" reverting to Pub/Sub** because XADD has slightly more ceremony than Publish. The forcing-function test below catches it immediately.
- **Hybrid (XADD + Publish) for "compatibility"** — the M4 SSE consumer reads Streams; Pub/Sub publishes are simply unread. Wasteful + confusing.

**Alternatives considered and rejected:**
- **Pub/Sub as plan literal specifies.** Cannot satisfy SPEC §6.4 SSE replay on multi-process server (full reasoning in ADR-014). Disqualifying.
- **Hybrid Streams+Pub/Sub.** Same rejection rationale as ADR-014 — two writes per event, two failure modes, no MVP benefit.
- **Update plan §5.4 instead of overriding via ADR.** Would require editing the plan document literal; violates state-at-time discipline (precedent: ADR-014 also overrode plan §4.1 without editing the plan literal).

**Forcing functions:**
- Task 5.4 progress-publisher tests use `miniredis/v2` Streams primitives (XADD + XRANGE), not Pub/Sub. A regression switching to `client.Publish` would cause `XLen` reads to return zero — test failure is unmissable.
- Cross-repo round-trip test at Task 5.4: M4 Python `ProgressSubscriber.replay()` reads what Go's `ProgressPublisher.Publish()` writes. Pub/Sub regression breaks this end-to-end.
- File location: progress-publisher in `stream.go`, not `pubsub.go`. Engineer adding progress logic looks at `stream.go` first.

**Open follow-ups:**
- Plan §5.4 stays with stale Pub/Sub literal per state-at-time. Task 5.1 commit body adds explicit warning so 5.4 implementation kickoff doesn't trip on the literal.
- Stream-key cleanup TTL — already a carry-forward from ADR-014 to OPS milestone. ADR-018 inherits.

**Cross-references:**
- ADR-014 (originating decision) — ADR-018 enforces it on the Go side.
- SPECIFICATION §7.2 (already patched in M4 docs commit `8a21704`) — the canonical contract.
- IMPLEMENTATION-PLAN.md §5.4 lines 1665-1677 (stale Pub/Sub literal) — left as written; this ADR overrides at implementation time.
- M4 commit `349fc5e` (`app.services.scan_queue.ProgressSubscriber`) — the consumer this publisher mirrors.
- ADR-016 (asynq drop) — same shape: post-plan ADR overrides plan literal, plan stays per state-at-time.

### ADR-021: ctx-discipline — context propagation, goroutine lifecycle, subprocess control
**Status:** Accepted (2026-05-01, Task 5.1)

**Context:**
Go's idiomatic primitive for cancellation, deadlines, and request-scoped values is `context.Context`. Cancel signals from M4 Python `CancelPublisher` (`shieldscan:cancel:{scan_id}` Pub/Sub) need to map to `ctx.Done()` in worker code. Tool subprocesses (Nuclei, Semgrep, etc., via M6 runners) must terminate when the scan is canceled — not just signal a cancel and let the subprocess finish naturally. Background goroutines spawned by the worker (cancel watcher, progress emitter, idempotency reaper) must exit when the worker shuts down — not survive across SIGTERM and leak into the next worker process.

These are not optional Go conventions; they are load-bearing M5 architecture concerns. Patterns established at Task 5.5 (processor) recur across every M6 tool runner, every M7 Docker service runner, and M8's recon-first executor. **Setting the rule once at M5.1 prevents N tool integrations from each making slightly-different choices that drift into bug clusters.**

**Decision:**
**Mandatory ctx-discipline.** Three rules apply to all Go code in `shieldscan-engine`:

**Rule 1 — Every blocking call takes ctx.** No `time.Sleep(d)` without `select { case <-ctx.Done(): ...; case <-time.After(d): ... }`. No `BRPOP` without `BRPopWithContext`. No HTTP request without `req = req.WithContext(ctx)`. No subprocess execution without `exec.CommandContext(ctx, ...)` (never `exec.Command(...)`).

**Rule 2 — Every goroutine has a ctx-aware exit.** Every `go func() { ... }` spawned in worker code must either:
- Receive a ctx parameter (typically `scanCtx` for job-scoped goroutines, root `workerCtx` for worker-lifetime goroutines).
- `select` on `<-ctx.Done()` in any blocking loop.
- Be wrapped in a structured-concurrency pattern (`errgroup.Group`, `sync.WaitGroup` + ctx-aware children).

No fire-and-forget goroutines. No `go logSomething()` that survives shutdown.

**Rule 3 — ctx flows down, never back up.** Goroutines may derive child contexts (`context.WithCancel`, `context.WithTimeout`) from their parent ctx and pass them deeper. Goroutines never construct `context.Background()` mid-call — that severs the cancellation chain.

**Legitimate `context.Background()` locations:**
- `main()` (process entry — root context)
- Top-level test functions (test entry — fresh context per test)
- Worker-lifetime services that intentionally outlive request-scoped ctxs (e.g., worker heartbeat, idempotency-reaper). These derive a worker-root ctx in `main()` and pass it down; they do NOT construct `context.Background()` at the goroutine spawn site — `main()` does it, then passes ctx down to the service constructor.

**Anti-pattern:** spawning a goroutine in a request handler that uses `context.Background()` to "ensure it survives the request." This is always a bug. If the work needs to outlive the request, hand it to a worker-lifetime service via a channel; the service's worker-root ctx governs.

Applied to M5 task surfaces:
- **Task 5.5 processor:** `ProcessJob(ctx, job)` derives `scanCtx, cancel := context.WithCancel(ctx)`; cancel-watcher goroutine gets `scanCtx`; tool runner gets `scanCtx`; `defer cancel()` ensures cleanup on all return paths.
- **Task 5.6 startup:** health-check polling loops `select` on `<-ctx.Done()` between attempts; service-wait timeouts derive from parent ctx, not constructed independently. Heartbeat goroutine receives `workerRootCtx` from `main()`, never constructs `context.Background()` itself.
- **Task 5.4 BRPOP loop:** `BRPopWithContext` returns `redis.Nil` cleanly on ctx cancel; loop exits.
- **M6 tool runners (every one):** subprocess via `exec.CommandContext(scanCtx, binary, args...)`; ctx cancel kills subprocess via SIGKILL after grace period.
- **M7 Docker service runners:** HTTP polling loops respect `<-ctx.Done()`; long polls (`http.Client.Do(req.WithContext(ctx))`) cancel cleanly.

**Per-worker job concurrency.** The BRPOP loop runs N concurrent jobs via `chan struct{}` semaphore; N is **env-var configurable** (`SHIELDSCAN_WORKER_CONCURRENCY`, default 5), not hard-coded — so M8 per-job tool-fanout can tune without restructuring 5.5. Config field lives in 5.1's `internal/config/config.go`; 5.5 reads it.

**Consequences:**
- Cancel propagation works correctly end-to-end: Python cancel endpoint (Task 4.5) → Pub/Sub `shieldscan:cancel:{scan_id}` → Go cancel-watcher → `cancel()` → ctx-aware tool runner → `exec.CommandContext` SIGKILLs subprocess. No code path can swallow cancel by accident.
- Goroutines cannot leak. Worker shutdown is clean — `main`'s SIGTERM handler cancels root ctx; all derived ctxs cancel; all goroutines exit. `goleak.VerifyTestMain(m)` catches violations in tests.
- Tests can simulate cancel deterministically: `ctx, cancel := context.WithCancel(context.Background()); cancel()` reproduces production cancel exactly.
- Code review burden: every new goroutine needs to answer "what cancels this?" and "what waits for it?" Sets a clear bar for M6+ contributions.
- Trade-off: more ceremony than Python's "let it leak; GC will sort it." Worth it because tool subprocesses don't get GC'd — a leaked goroutine watching a leaked subprocess is a real memory + CPU drain on the worker.

**Anti-patterns this prevents:**
- **Bare `time.Sleep`** in retry loops or health checks. Sleep ignores ctx; cancellation hangs until the sleep elapses.
- **`go logProgress()` fire-and-forget.** Progress publishers (Task 5.4) are Stream-write goroutines. If they outlive job ctx, they may emit progress for a job that's already canceled — causing UI confusion ("scan canceled, but I see new progress events").
- **`exec.Command` instead of `exec.CommandContext`.** Subprocess survives ctx cancel; worker process leaks subprocesses on cancel.
- **`context.Background()` in tool runner code.** Severs cancel chain; test-time cancel cannot reach the runner.
- **Deferring `cancel()` to "shutdown handler"** instead of inline. Misses the per-job lifetime; cancels accumulate until shutdown.

**Alternatives considered and rejected:**
- **Channel-only coordination (no ctx).** Pre-Go-1.7 idiom; legacy. Loses deadline semantics + the standard-library convention every Go HTTP/DB/exec library expects. Rejected.
- **Per-package cancellation managers** (e.g., a package-level `done chan struct{}`). Doesn't compose across packages; tool runners would need their own cancel mechanism plus ctx; double-state-machine bug surface. Rejected.
- **Manual goroutine accounting** (every goroutine increments/decrements a counter; shutdown waits for zero). Ad-hoc; reinvents `sync.WaitGroup` poorly; doesn't catch leaks at test time. Rejected; use WaitGroup or errgroup directly when structure is needed.
- **`golang.org/x/sync/errgroup` everywhere.** Considered as a stronger forcing function than ctx-only. Adds a dep + idiom for marginal ergonomic gain over plain ctx + WaitGroup at M5 scale. Defer adoption to M8's recon-first executor where parallel-tool fan-out makes errgroup's structured-concurrency value clearer.

**Forcing functions:**
- **`goleak.VerifyTestMain(m)`** in every test package that spawns goroutines. Goroutine leaks fail the test suite at runtime — not silently accumulate. Added in Checkpoint 4 commit `39e1e5e` to VERSIONS.md §2.4 (`go.uber.org/goleak v1.3.0`).
- **`go vet`** runs `lostcancel` analyzer by default — catches `cancel := context.WithCancel(...)` without a `defer cancel()` or unconditional call. Wired into Task 5.1's CI workflow `.github/workflows/engine.yml`.
- **`golangci-lint`** with `containedctx` + `noctx` linters enabled — flags struct fields holding ctx (anti-pattern) and HTTP requests built without ctx. Lands in Task 5.1 lint config.
- **Engine `CLAUDE.md` gotchas section** (Task 5.1) explicitly documents Rules 1-3 with example violations + corrections.

**Open follow-ups:**
- **errgroup adoption at M8** if recon-first parallel-tool fan-out shows that plain WaitGroup + ctx is insufficient for error aggregation across N parallel tool calls.
- **Subprocess SIGKILL grace period.** Some tools may benefit from SIGTERM-first with a 2-3s grace period for clean shutdown. Defer to M6.1 Nuclei integration.
- **Subprocess output streaming under cancel.** When a tool is canceled mid-run, partial stdout may be useful (some findings already emitted). Default: discard. Override per-tool if a runner explicitly opts in. Decide at M6.1.

**Cross-references:**
- IMPLEMENTATION-PLAN.md §5.5 lines 1710-1733 (`ProcessJob` skeleton with `scanCtx`) — already ctx-aware in plan literal; ADR-021 formalizes the discipline beyond §5.5 to all engine code.
- M4 commit `cf3b30a` (Python `CancelPublisher`) — the cancel signal source this discipline subscribes to via Pub/Sub.
- SPECIFICATION §7.4 (cancel channel format).
- Task 5.4 (cancel-subscriber landing) + Task 5.5 (processor).
- M6 tasks 6.1-6.7 (every tool runner inherits the discipline via `exec.CommandContext`).
- M7 tasks 7.1-7.6 (every Docker service runner inherits via `req.WithContext`).
- VERSIONS.md §2.4 `goleak v1.3.0` (forcing function dep) — Checkpoint 4 commit `39e1e5e`.

> **ADR numbering note.** §13 jumps from ADR-018 to ADR-021. ADR-019 (cancel Pub/Sub confirmation) and ADR-020 (worker concurrency model) numbers are reserved — not missing. Both decisions are currently captured as DRIFT-LOG entries; promote to full ADR with the reserved number when the underlying decision becomes load-bearing across multiple tasks.
>
> ADR-022 (recon-as-pre-scan-helpers) lands at M6.3 — see below.

### ADR-022: Recon tools as pre-scan helpers, not ToolRunner-registered

**Status:** Accepted at M6.3 (2026-05-03).

**Context.** Subfinder and httpx are pre-scan discovery tools, not finding-producers. Subfinder emits subdomain strings; httpx emits LiveHost metadata (URL, status, tech stack, etc.). Neither produces `events.RawFinding` — the canonical output of `tools.ToolRunner.Run`.

**Decision.** Ship recon as helper functions in `internal/tools/recon/`, NOT as ToolRunners registered with `worker.Registry`:

```go
// internal/tools/recon/recon.go
type ReconResult struct {
    Subdomains []string
    LiveHosts  []LiveHost
}
type LiveHost struct {
    URL string; StatusCode int; Title string; Tech []string;
    Webserver string; ContentType string
}
func RunRecon(ctx context.Context, domain string, limit int,
    publisher *redis.ProgressPublisher, log zerolog.Logger) (*ReconResult, error)
```

M8 (Recon-First Pipeline) imports `internal/tools/recon` and invokes `RunRecon` as a pre-scan phase before dispatching per-target scan jobs to the standard processor.

`cmd/worker/run.go` (M6.8 wiring) explicitly does NOT register Subfinder or httpx with `worker.Registry`; an explicit code comment references this ADR.

**Rationale.**

Three alternatives evaluated and rejected:

| Alternative | Why rejected |
|---|---|
| Synthetic `subdomain_discovered` findings via ToolRunner | Semantically wrong — subdomain discovery is not a vulnerability; would pollute the findings stream + confuse the M9 AI pipeline that operates on findings |
| Defer recon entirely to M8 (no M6 work) | Wastes M6 capacity; recon parsers + helpers are valuable + testable in isolation |
| Per-task hack at M8 | M8 invocation needs a canonical, testable shape; per-task hacks fragment |

The recon-as-helpers shape is documented in TOOL-ARCH §8 (Recon-First Pipeline) and plan §6.3's literal. ADR-022 codifies the architectural commitment.

**Cross-reference: ADR-023 asymmetric-cost meta-principle.** ADR-023 (NativeRunner OutputFile mode at M6.7) overrode the three-instance promotion threshold because the cost of NOT abstracting (race conditions, debugging burden) exceeded the cost of premature abstraction. ADR-022 is a different shape of architectural commitment — not threshold-override but type-system carve-out. The shared meta-principle: **architectural commitments are made when the alternative is operationally worse, not when a generic threshold is met.** Future ADRs in the M6/M7 corpus may invoke similar asymmetric-cost reasoning.

**Consequences.**

Positive:
- `internal/tools/recon` is testable in isolation (parsers + RunRecon orchestration).
- M8 inherits a canonical signature; integration is import + invoke.
- The findings stream stays semantically clean (only vulnerabilities flow to M9 AI pipeline).

Negative:
- The "tool" abstraction is bimodal: most are ToolRunners; recon are helpers. New readers may expect uniformity.
- 6.8 wiring needs an explicit code comment explaining why recon is absent from the Registry.

**Alternatives considered (and rejected).** See Rationale table above.

**⚠️ SPECULATIVE M8 invocation pattern.** The example below is a **best-effort prediction**, NOT a binding contract. M8 implementation may refine the invocation API based on requirements that emerge during M8 design. The recon-as-helpers principle holds regardless of how the call site evolves.

```go
// M8 ScanExecutor (SPECULATIVE — subject to M8 implementation refinement)
result, _ := recon.RunRecon(ctx, scanReq.Domain, scanReq.SubdomainLimit, publisher, log)
targets := buildTargetList(scanReq.RootURL, result.LiveHosts)
for _, target := range targets {
    dispatcher.Enqueue(targetJob(scanReq, target))
}
```

M8 may instead introduce a `ReconCoordinator` type, batch multiple recon invocations across scan-job batches, or apply additional filtering layers before target dispatch. The above is illustrative.

**Triggers to revisit.**

- **A new "tool" emits non-finding output** (e.g., a fuzzer producing wordlists). At that point, the recon-helpers shape may generalize to "pre-scan-utilities" and warrant a DEVELOPMENT-PATTERNS entry.
- **M8 implementation surfaces a need for ToolRunner-shaped recon access** (unlikely; speculative). Revisit if integration tension emerges.
- **Customer demand for treating subdomain discoveries as findings** (e.g., compliance reporting). At that point, add a synthetic-finding emitter alongside `RunRecon`; don't replace the helper.

### ADR-023: Extend NativeRunner with file-output mode for tools that don't write findings to stdout

**Status:** Accepted at M6.7 (2026-05-02).

**Context.** OWASP Dependency-Check writes findings to a file via `--out <path>` rather than stdout (which carries logging chatter). NativeRunner (5.2) is stdout-only — `BuildArgs` returns args, the subprocess writes to stdout, `ParseOutput` receives the bytes. Dep-Check breaks this assumption.

**Decision.** Add three fields to `NativeRunner`:

- `OutputFile bool` — opt into file-output mode.
- `OutputFilePlaceholder string` — literal substring in BuildArgs args replaced with per-Run tempfile path.
- `ParseOutputFile func(outputFilePath string) ([]events.RawFinding, error)` — file-output variant of ParseOutput.

Per Run, when `OutputFile=true`, NativeRunner creates a unique tempfile via `os.CreateTemp` (with `os.TempDir()` honoring `TMPDIR`), substitutes its path into BuildArgs's args slice (replacing every occurrence of `OutputFilePlaceholder`), invokes the subprocess, calls `ParseOutputFile` with the path, and removes the tempfile via `defer` regardless of success/failure.

Stdout content is discarded in file-output mode (typical file-output tool's stdout is logging chatter, not findings).

**Rationale.**

Hack alternatives all evaluated and rejected at M6.7 pre-prep:

| Alternative | Why rejected |
|---|---|
| Closure-shared mutable state | Concurrency-unsafe under concurrent Run() calls on the same runner |
| Known-fixed path (e.g., `/tmp/depcheck.json`) | Cross-cuts NativeRunner contract; needs scan/worker context injection |
| Per-PID nanosec timestamp | Race-prone if Run() called twice within same nanosec |
| Per-Run factory | NativeRunner is held in Registry as singleton; no per-call factory |
| Filename grep from stdout | Brittle; depends on tool verbosity flags |

Cost of NOT promoting (race conditions in production, debugging burden) exceeds cost of premature abstraction (~50 LoC framework + 5 framework tests + this ADR). Three-instance threshold is OVERRIDDEN per the asymmetric-cost reasoning documented in `shieldscan-engine/DEVELOPMENT-PATTERNS.md` preamble. Future readers should understand: the three-instance heuristic is not a universal rule; it is overridable when the cost asymmetry inverts.

**Consequences.**

Positive:
- Dep-Check ships at M6.7 without race-prone hacks.
- Future file-output tools (M7 Trivy filesystem-scan output, etc.) inherit cleanly.
- Framework abstraction confined to 3 fields + 1 lifecycle hook in `Run()`.

Negative:
- NativeRunner's contract is now bimodal (stdout-mode default, OutputFile-mode opt-in). Slight cognitive cost for new readers.
- Tempfile lifecycle adds I/O overhead per Run (one `CreateTemp` + one `Remove`). Negligible vs scan duration (Dep-Check scans take 5-10 minutes; tempfile I/O is sub-millisecond).

**Alternatives considered (and rejected).** See Rationale hack-alternatives table above.

**Triggers to revisit.**

- **5+ tools using OutputFile mode.** Consider a separate `FileOutputRunner` type vs continued NativeRunner extension; bimodal contract may grow into multiple narrow types.
- **Tools that write to stderr** (not stdout, not file) — distinct shape; would need separate accommodation rather than extending OutputFile.
- **Performance regression** (>5% scan-duration overhead from tempfile I/O at high throughput / very-many-target portfolios). At that point, evaluate in-memory pipe alternatives or reuse a persistent tempfile across Runs (still concurrency-safe via locking).
- **Tempfile-location operator concern** (e.g., ops needs tempfiles on tmpfs for performance, or persistent disk for forensics). At that point, add `SHIELDSCAN_TMPDIR` env var; for now, `os.TempDir()` honors stdlib + `TMPDIR` transparently.

---

## 14. Glossary

| Term | Definition |
|---|---|
| **DAST** | Dynamic Application Security Testing — testing a running application from outside |
| **SAST** | Static Application Security Testing — analyzing source code without running it |
| **SCA** | Software Composition Analysis — scanning dependencies for known vulnerabilities |
| **IaC** | Infrastructure as Code — Terraform, Kubernetes manifests, CloudFormation |
| **MobSF** | Mobile Security Framework — open-source mobile SAST + binary analysis platform |
| **APK** | Android Package — compiled Android app binary |
| **IPA** | iOS App Package — compiled iOS app binary |
| **OWASP** | Open Web Application Security Project — security standards organization |
| **CWE** | Common Weakness Enumeration — vulnerability classification system |
| **CVSS** | Common Vulnerability Scoring System — severity rating 0.0–10.0 |
| **SARIF** | Static Analysis Results Interchange Format — standard format for code scanning results |
| **RLS** | Row-Level Security — PostgreSQL feature for tenant isolation |
| **BOLA** | Broken Object Level Authorization — OWASP API Top 10 #1 |
| **PDPL** | Personal Data Protection Law — Saudi Arabia's data protection regulation |
| **SSE** | Server-Sent Events — one-way streaming from server to browser |
| **Corroborated finding** | A vulnerability confirmed by 2+ independent scan engines |
| **Attack surface** | All discovered assets (domains, subdomains, ports) of a target |
| **Recon-first** | Scanning pipeline that runs reconnaissance before vulnerability scanning |
| **Cross-layer correlation** | Matching DAST runtime findings to SAST source code findings |

---

*End of specification. See `TOOL-ARCHITECTURE.md` for scan engine details.*
