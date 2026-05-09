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
│   │   ├── docker/                # M7.5a + ADR-026
│   │   │   ├── container.go       # Docker SDK Container abstraction
│   │   │   ├── client_adapter.go  # productionClient adapter
│   │   │   ├── warmpool.go        # WarmPool primitive (lazy spin-up; ContainerFactory hook)
│   │   │   ├── dockerrunner.go    # DockerRunner framework type
│   │   │   └── service/           # DockerServiceRunner (HTTP-API tools; ZAP, MobSF)
│   │   │                          # (M7.5b commit 1306ca8; replaces docker_service.go)
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
│   │   └── tool_router.go         # Scan type → tools mapping
│   ├── worker/
│   │   ├── startup.go             # Tool health checks on boot
│   │   └── processor.go           # Job processing loop
│   └── redis/
│       ├── queue.go
│       └── pubsub.go
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

**RawFinding wire shape (canonical).** Each entry in the `findings` array is a `RawFinding` object. The canonical source of truth for the field set is the Go struct `events.RawFinding` in `shieldscan-engine/internal/events/events.go` and its Python mirror `app.models.raw_findings.RawFinding` in `shieldscan-api`. The two MUST stay in sync; ADR-017's "Schema versioning of `RawFinding`" follow-up identifies SPEC §7.3 as the shared schema doc, and ADR-024 establishes the workflow for synchronized cross-repo extensions.

The full per-field listing as of M6-close-followup (post-ADR-024):

| Group | Field | JSON name | Type | Required | Notes |
|---|---|---|---|---|---|
| Identity | ToolName | `tool_name` | string | yes | e.g., `"nuclei"`, `"semgrep"` |
| Identity | EngineCategory | `engine_category` | string | yes | enum per §5.3 (dast, sast, sca, ...) |
| Identity | ScanID | `scan_id` | string | optional | populated by processor; runner leaves empty |
| Identity | OrgID | `org_id` | string | optional | populated by processor |
| Classification | Title | `title` | string | yes | short human label |
| Classification | Description | `description` | string | optional | longer context |
| Classification | Severity | `severity` | string | yes | enum: critical/high/medium/low/info |
| Classification | FindingType | `finding_type` | string | yes | tool-specific (e.g., `"xss"`, `"hardcoded-secret"`) |
| Classification | CWEID | `cwe_id` | string | optional | e.g., `"CWE-89"` (primary CWE; multi-CWE → AdditionalCWEs) |
| Classification | OWASP | `owasp` | string | optional | e.g., `"A03:2021"` |
| Classification | CVSSScore | `cvss_score` | float | optional | numeric severity 0.0–10.0 |
| Evidence (web) | TargetURL | `target_url` | string | optional | DAST/API tools |
| Evidence (web) | Parameter | `parameter` | string | optional | DAST/API tools |
| Evidence (web) | Payload | `payload` | string | optional | DAST/API tools |
| Evidence (web) | Request | `request` | string | optional | 4 KB truncated by tool |
| Evidence (web) | Response | `response` | string | optional | DAST/API tools |
| Evidence (source) | CodeFile | `code_file` | string | optional | SAST/SCA tools |
| Evidence (source) | CodeLine | `code_line` | int | optional | SAST/SCA tools |
| Evidence (source) | CodeSnippet | `code_snippet` | string | optional | SAST/SCA tools |
| Evidence (mobile) | MobileOS | `mobile_os` | string | optional | enum: android/ios |
| Evidence (mobile) | Permission | `permission` | string | optional | MobSF |
| Evidence (mobile) | ComponentName | `component_name` | string | optional | MobSF |
| Evidence (SSL) | CipherSuite | `cipher_suite` | string | optional | SSLyze |
| Evidence (SSL) | CertSubject | `cert_subject` | string | optional | SSLyze |
| Categorical (ADR-024) | References | `references` | []string | optional | URLs / advisory identifiers (M6.4 trigger; landed M6-followup) |
| Categorical (ADR-024) | Tags | `tags` | []string | optional | fine-grained tags; MUST NOT duplicate `engine_category` |
| Categorical (ADR-024) | CVSSVector | `cvss_vector` | string | optional | canonical CVSS 3.1 vector (`"CVSS:3.1/AV:N/..."`) |
| Categorical (ADR-024) | AdditionalCWEs | `additional_cwes` | []string | optional | secondary CWE strings beyond primary `cwe_id` |
| Categorical (ADR-027) | Metadata | `metadata` | map[string]string | optional | per-tool structured payload as key-value pairs; per-tool key contracts documented in consumer package docstrings; Nmap canonical first consumer (Task 7.2): 6 required keys (host, port, protocol, state, service, target) + 5 conditional (product, version, extra_info, cpe, tunnel); future Trivy/SQLMap/ZAP/MobSF consumers inherit pattern |
| Metadata | RawOutputRef | `raw_output_ref` | string | optional | future R2-staged output reference |
| Metadata | DiscoveredAt | `discovered_at` | string | optional | RFC3339 timestamp |
| Metadata | Fingerprint | `fingerprint` | string | optional | SHA-256 deterministic dedup key |

All fields are encoded with `omitempty` on the Go side and `Optional` on the Python side; absent/empty values do not appear on the wire and do not break decoding. Both sides enforce strict-schema validation (Go: `DisallowUnknownFields`; Python: `extra="forbid"` per ADR-017's M4 Pydantic discipline transfer) — adding a new field requires synchronized extension on both sides per ADR-024's coordination workflow.

**Findings persistence (per ADR-017).** The `findings` array carries the RawFinding rows produced by the engine for this job; the Python `CompletionsConsumer` (M4 Task 4.2) is responsible for inserting them in the same transaction as the `ScanJob.status` update. `findings` is REQUIRED on terminal events (`status` ∈ {`completed`, `partial`}) and absent on `failed`/`canceled` events.

> **Implementation status (M6-close-followup, ADR-024).** As of the M6-close-followup landing the SQLAlchemy `RawFinding` model has all columns required to persist the wire shape above (including the four ADR-024 categorical fields), but `CompletionsConsumer` does not yet insert `findings[]` rows — at M4 it was scoped to `ScanJob.status` + `ScanJob.finding_count` updates only. Findings ingest (Pydantic schema + bulk-insert path + ingest tests) is a separate deliverable tracked under ADR-024's "Triggers to revisit". The columns-ready posture means the schema change can land without churn when the ingest path is implemented.

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

**Addendum (2026-05-04, per ADR-026):** Tool classification refined. Trivy and SQLMap are CLI-shaped tools (single execution; output to stdout/file) and use the warm pool framework (DockerRunner) rather than persistent service framework (DockerServiceRunner). Persistent Docker services per this ADR are now scoped to ZAP and MobSF only. See ADR-026 for the full warm-pool primitive + DockerRunner framework architecture.

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

### ADR-024: RawFinding schema extension — References, Tags, CVSSVector, AdditionalCWEs

**Status:** Accepted at M6-close-followup (2026-05-03).

**Context.**
The reductions counter — tracking M6 tools whose parsers fold or drop fields from upstream tool output — fired its trigger at M6.4 (SSLyze) when 4 of 5 implemented tools showed reductions. Per the landscape decision at 6.4, schema extension was deferred to M6 close (Path A in the M6.4 scope proposal). At M6 close (post-6.8), the counter sits at **8/9 tools (89%) with reductions**, totalling ~38 folded/dropped fields across the corpus.

The reductions are not problems in themselves — folding is legitimate parser strategy when the canonical schema lacks appropriate slots. The decision is whether to extend the canonical RawFinding schema with new categorical fields, which categorical patterns warrant first-class fields, and how those fields couple to the M9 AI pipeline that consumes RawFindings.

Per ADR-017's "Schema versioning of `RawFinding`" follow-up: SPEC §7.3 is the canonical shared schema doc between Engine (Go) and Python orchestrator. Extension requires synchronized changes across the Engine struct, Python SQLAlchemy model, Python Pydantic schema (when it lands), Alembic migration, and wire-format fixtures.

A structured brainstorming session preceded this ADR (design doc archived at `plans/2026-05-03-spec73-schema-extension-design.md`). The brainstorming locked moderate scope (4 fields) over conservative (2) and comprehensive (6) options. The 4 fields capture the highest-value categorical patterns; comprehensive scope's `Evidence` flexible-map field was rejected as anti-pattern (dumping ground for inconsistent per-tool usage).

A Phase 0 verification pass before implementation surfaced one load-bearing deviation from the design doc — see "Python ingest scope" subsection below.

**Decision.**

Extend `events.RawFinding` (Engine Go struct) and the SQLAlchemy model `app.models.raw_findings.RawFinding` (Python persistence mirror) with 4 new optional fields. Pydantic ingest schema is out of scope at this task; see "Python ingest scope" subsection below.

```go
// internal/events/events.go
type RawFinding struct {
    // ... existing fields ...
    References     []string `json:"references,omitempty"`
    Tags           []string `json:"tags,omitempty"`
    CVSSVector     string   `json:"cvss_vector,omitempty"`
    AdditionalCWEs []string `json:"additional_cwes,omitempty"`
}
```

Field semantics:

- **References** — array of URLs / advisory identifiers (CVE links, remediation guidelines, OWASP WSTG identifiers, vendor advisories). Free-form strings; no URL validation at SPEC §7.3 time.
- **Tags** — array of fine-grained categorical tags. Distinct from existing `engine_category` (broad: dast/sast/sca/etc., 13 enum values per §5.3). Tags is finer-grained per-tool sub-categorization. **MUST NOT duplicate `engine_category` values** — parsers emitting tags that overlap engine_category should drop them.
- **CVSSVector** — canonical CVSS 3.1 vector string (`"CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H"`). Stored as-emitted; no parsing into 8-dimension subfields at SPEC §7.3 time. Parsing into subfields is reserved for M9 §8.3 implementation if the algorithm evolves to consume the vector directly.
- **AdditionalCWEs** — array of CWE identifier strings beyond the primary `CWEID` field (`["CWE-89", "CWE-20"]`). Empty/absent when finding is single-CWE.

All four fields are `omitempty` on the Go side and Optional on the Python side; backward-compatible with existing engine emissions.

**Python ingest scope.**

This ADR adds the 4 fields to the canonical schema (SPEC §7.3) and
the Engine emission path. Python-side ingest (CompletionsConsumer
extension to insert RawFinding rows from event["findings"]) is
**deferred** to a future findings-ingest task.

Phase 0 verification (M6-close-followup, 2026-05-03) discovered
that CompletionsConsumer currently only updates ScanJob.status +
finding_count counter; it does NOT insert RawFinding rows from
event["findings"]. This is an unfinished M4 deliverable per
ADR-013 (Python sole writer) + ADR-017 (findings inline in
job_completed events).

The schema-extension work proceeds in **columns-ready posture**:
- Engine emits findings with new fields populated
- SQLAlchemy model + Alembic migration land columns
- When findings-ingest task lands, Python side reads existing
  columns; no schema change required at ingest time

This deferral is documented explicitly so future engineers don't
mistake "schema columns exist" for "ingest path operational."
The Engine commit + Docs commit deliver immediate value (richer
wire format; M9 §8.2 forward-pin); ingest landing is a separate
milestone task.

Trigger to revisit: findings-ingest task lands (likely M4-completion
or dedicated M-followup before M9). At that point, add Pydantic
schema mirroring SQLAlchemy model + extend CompletionsConsumer
+ add ingest tests + fixtures. The schema columns are already in
place from this ADR.

**Rationale.**

Three scope alternatives considered and rejected (per the design doc):

| Alternative | Why rejected |
|---|---|
| **Conservative (2 fields):** References + Tags only | Defers CVSSVector + AdditionalCWEs to a future task. AdditionalCWEs is load-bearing for §8.2 correlation; deferring delays the M9 forward-pin. |
| **Comprehensive (6 fields including Evidence map):** | The Evidence map is a flexible-map anti-pattern: tends to accumulate inconsistent usage across tools (one stores `"hash"`, another `"sha256"`, M9 has to handle both). Better to add specific fields when patterns warrant than to preempt with a dumping ground. |
| **Status quo (no extension):** | 38 folded/dropped fields across M6 is the trigger fire point. Continuing without extension means M7+ tools accumulate folds against the current schema; cleanup cost grows non-linearly. |

The 4-field selection is grounded in a SPEC §8 read:
- **References** — used by M11 dashboard cross-linking + AI fix-generation prompt context + future cross-tool dedup signal.
- **Tags** — used by M11 fast-filtering + future M9 AI categorization input.
- **CVSSVector** — reserved for §8.3 `exploitability_multiplier` derivation (currently uses derived `base_cvss` numeric + separate publicly-accessible detection logic).
- **AdditionalCWEs** — **load-bearing for §8.2 correlation algorithm** extension to handle multi-CWE findings (especially Dep-Check, which routinely emits 2–4 CWEs per CVE).

**Cross-reference: asymmetric-cost meta-principle (3rd ADR invocation).** ADR-022 (recon-as-pre-scan-helpers) and ADR-023 (NativeRunner OutputFile mode) both invoked the asymmetric-cost meta-principle to justify architectural commitments. ADR-024 applies the same principle: extension cost (~4.5–5h cross-repo work post-Path-A reduction) is asymmetrically smaller than alternative cost (38 folds compounding across M7+ tools, schema-extension trigger remaining fired without incremental progress, M9 missing the multi-CWE correlation upgrade entirely). The shared meta-principle, now invoked across three consecutive M6 ADRs, is project corpus norm: **architectural commitments are made when the alternative is operationally worse, not when a generic threshold is met.**

**M9 forward-pin (load-bearing).**

SPEC §8.2 currently uses `cwe_id` singular for cross-layer correlation:

```python
if dast.cwe_id == sast.cwe_id:
    score += WEIGHTS["cwe_exact"]
elif is_cwe_parent_child(dast.cwe_id, sast.cwe_id):
    score += WEIGHTS["cwe_parent"]
```

**M9 implementation MUST extend these checks to consider intersection with `additional_cwes`:**

```python
# Extended logic (M9 implementation):
all_dast_cwes = {dast.cwe_id} | set(dast.additional_cwes or [])
all_sast_cwes = {sast.cwe_id} | set(sast.additional_cwes or [])

if all_dast_cwes & all_sast_cwes:
    score += WEIGHTS["cwe_exact"]
elif any_parent_child(all_dast_cwes, all_sast_cwes):
    score += WEIGHTS["cwe_parent"]
```

Without this extension, M9 correlation misses multi-CWE matches (especially Dep-Check). This is the load-bearing M9 forward-pin of the entire SPEC §7.3 extension — the reason the ADR exists at this scope rather than the conservative 2-field scope.

**M9 forward-pins (non-load-bearing).**

- **References** — SPEC §8 algorithms do not currently use this field; reserved for UI cross-linking (M11), AI fix-generation prompt grounding, and future cross-tool dedup signal (findings citing the same CVE may be dedup candidates beyond vector similarity).
- **Tags** — SPEC §8 algorithms do not currently use this field; reserved for M11 fast-filtering and future AI categorization input. Constraint: parsers MUST NOT duplicate `engine_category` values into Tags.
- **CVSSVector** — SPEC §8.3 currently uses `base_cvss` numeric. Reserved for future `exploitability_multiplier` derivation: the §8.3 multiplier value 1.5 ("publicly accessible + unauthenticated") could be derived from `AV:N` + `PR:N` + `UI:N` without separate detection logic. M9 implementation may evolve §8.3 to parse CVSSVector when available; current spec leaves this as a future enhancement.

**Consequences.**

Positive:
- ~12 of ~38 folded fields (~32%) rescued via per-tool parser retrofits in 6 of 9 M6 tools (Nuclei, Semgrep, Gitleaks, Dep-Check, Checkov, Wapiti).
- M9 correlation algorithm has a documented forward-pin for the multi-CWE upgrade, discoverable by an M9 implementation engineer searching for `additional_cwes`.
- M11 dashboard and future AI features have schema slots ready (columns-ready posture).
- Cross-repo schema-coordination workflow exercised (synchronized Engine + Python + Docs ordering); future schema extensions inherit this commit shape.

Negative:
- **Reductions counter does NOT clear post-§7.3.** 8/9 tools still have reductions (count per tool reduced but not eliminated). The trigger remains fired; future incremental schema extensions may address tool-specific metadata. ADR-024 is one incremental step, not a complete resolution.
- **3 tools (SSLyze, Nikto, CORStest) get zero retrofit at this scope** — their reductions stay folded. SSLyze's plugin-rules pattern doesn't carry References/Tags/CVSSVector/multi-CWE per-finding; Nikto XML emits description-only; CORStest text-with-ANSI parser extracts URL/origin/header values only.
- **Python ingest deferred to future task.** Schema columns exist
  + ready for ingest, but actual ingest path (CompletionsConsumer
  extension + Pydantic schema) requires a separate findings-ingest
  task. Scope discipline matters: SPEC §7.3 extension is canonical
  schema work, not ingest implementation.
- Cross-repo schema-coordination cost (~4.5–5h cross-repo) is real; future schema extensions will incur similar costs unless batched with M7+ tool data.
- Two new tracked patterns at 1st instance (track-only, not promoted at this scope):
  - Multi-repo schema-coordination commits (Engine + Python + Docs, strict ordering).
  - Optional-field additive migrations (Alembic upgrade with backward-compat).

**Alternatives considered (and rejected).** See Rationale table above. The "Python ingest scope" subsection above documents the deferral of ingest-path implementation to a separate findings-ingest task; ADR-024 itself is scoped to canonical schema work + Engine emission, not ingest implementation.

**Anti-patterns this prevents.**
- Evidence map accumulating inconsistent per-tool usage.
- Schema extension via per-tool subfields (NucleiTemplateID, GitleaksRule, etc.) proliferating the schema.
- Premature DB indexes on fields that §8 algorithms don't query.
- Premature parsing of CVSSVector into 8-dimension subfields when §8 doesn't need them yet.

**Triggers to revisit.**

1. **M9 §8.2 implementation.** When M9 lands the cross-layer correlation algorithm, verify the multi-CWE intersection extension is implemented per the load-bearing forward-pin above.
2. **M9 §8.3 implementation.** Decide whether to derive `exploitability_multiplier` from CVSSVector AV/PR/UI fields (replacing separate detection logic) or maintain separate logic. Either is acceptable; the CVSSVector schema is forward-compat.
3. **Trigger remains fired (8/9 tools post-§7.3).** Future incremental schema extensions may address remaining tool-specific metadata. Likely candidates: NucleiTemplateID + GitleaksRuleID (per-tool identifiers); the Evidence map IF a 3rd+ instance of "tool-specific binary artifact storage" emerges across M7 tools and the anti-pattern risk is empirically bounded.
4. **M7 tool reductions accumulate.** If M7 surfaces new categorical patterns (e.g., Trivy SBOM data, Nmap port-scan structure), a second SPEC §7.3 extension task may be warranted.
5. **DB query patterns surface.** If M11 dashboard or M9 pipeline shows query patterns that benefit from indexes on Tags/References/etc., add indexes via an additive migration. Premature at this scope.
6. **Findings-ingest task lands.** Pydantic schema mirroring
   SQLAlchemy model + CompletionsConsumer ingest extension +
   ingest tests + fixtures land at that point. Schema columns
   from this ADR are already in place; no schema change needed.

**Forcing functions.**
- Per-tool retrofit checklist documented in the design doc; each retrofit has explicit field-mapping assertion in code review.
- Engine `events.RawFinding` JSON encoding uses `DisallowUnknownFields`; new fields landing on the Engine side without the Python schema mirror would (eventually, when ingest exists) trip strict-validation.
- ADR-024 forward-pin text references SPEC §8.2 explicitly; an M9 implementation engineer searching for `additional_cwes` finds the algorithm-extension requirement adjacent to the schema rationale.
- Reductions counter post-§7.3 status documented in shieldscan-engine `DRIFT-LOG.md`; future engineers see ongoing trigger status.

**Open follow-ups.**
- Findings-ingest task (Pydantic schema + `CompletionsConsumer.handle_findings()` + ingest tests + fixtures). Per Path A, scoped separately.
- M9 §8.2 algorithm extension per the load-bearing forward-pin.
- Reductions-counter tracking continues; future SPEC §7.3 extensions may land at M7 close or later if M7 tools add categorical patterns warranting additional first-class fields.
- Multi-repo commit-ordering pattern documentation if a 3rd+ instance emerges (currently 1st instance at SPEC §7.3; M6.3 was a precedent but lighter-weight, no DB migration involved).

**Cross-references.**
- ADR-013 — Python sole writer constraint (load-bearing for cross-repo coordination).
- ADR-017 — Findings inline in `job_completed` events; identifies SPEC §7.3 as canonical RawFinding schema doc; "Schema versioning of `RawFinding`" follow-up resolved by ADR-024.
- ADR-022 — Recon-as-pre-scan-helpers (asymmetric-cost meta-principle, 1st invocation).
- ADR-023 — NativeRunner OutputFile mode (asymmetric-cost meta-principle, 2nd invocation).
- SPEC §7.3 — Canonical schema doc (updated in this commit with explicit per-field listing + the four ADR-024 categorical fields + Python-ingest implementation-status callout).
- SPEC §8.2 — Cross-layer correlation algorithm (load-bearing forward-pin).
- SPEC §8.3 — Severity scoring formula (CVSSVector forward-pin, non-load-bearing).
- Design doc — `plans/2026-05-03-spec73-schema-extension-design.md` (brainstorming output + per-tool retrofit checklist + scope rationale; Path A deviation documented in the M6-close-followup engine DRIFT-LOG entry).

### ADR-026: DockerRunner framework + lazy warm pool — M7 container lifecycle architecture
**Status:** Accepted (2026-05-04, Task 7.5a)

**Context.**
M7 introduces 5 Docker-based scanning tools (Trivy, Nmap, MobSF, ZAP, SQLMap) extending the M6-shipped 9 native CLI tools. ADR-006 (Hybrid Native + Persistent Docker, refined 2026-04-18) established the broad architectural decision to use persistent Docker services for heavy tools, eliminating 2-3s per-scan container startup. ADR-008 (MobSF for Mobile Security) refined this for MobSF specifically as a persistent HTTP-API service.

The M7.5 brainstorming surfaced that ADR-006's "persistent Docker services" framing implicitly conflated two distinct architectural shapes:
- **HTTP-API persistent services:** ZAP and MobSF run as long-lived servers; runner makes HTTP requests; container lifecycle decoupled from individual scans.
- **CLI-shaped warm-pool tools:** Trivy, Nmap, SQLMap exit after each scan; per-scan container startup is the cost driver; reuse via warm pool eliminates the 2-3s startup without persistent service complexity.

These are different concurrency and lifecycle contracts. HTTP services need session/state management, version drift mitigation per scan-run, and HTTP-specific health checks. CLI tools need cleanup-between-checkouts for tenant isolation, lazy spin-up to keep resource floor at zero, and exit-code-aware execution.

The original ADR-006 text grouped Trivy and SQLMap with ZAP and MobSF as "persistent Docker services," which is architecturally correct in cost-driver terms (avoiding cold start) but operationally wrong (CLI tools don't run as services). Option β was locked in brainstorming: two distinct framework types, one per shape.

**Decision.**
Establish two M7 framework types in shieldscan-engine:

1. **DockerRunner** (`internal/tools/docker/dockerrunner.go`, this ADR): warm-pool wrapper for short-lived CLI-shaped Docker tools. Wraps WarmPool primitive (`internal/tools/docker/warmpool.go`) which provides lazy spin-up + max-bound + cleanup-between-checkouts + optional health check. Symmetric with NativeRunner (per ADR-023 OutputFile mode framework precedent).
   - Consumers: Trivy (M7.1), Nmap (M7.2), SQLMap (M7.6).

2. **DockerServiceRunner** (`internal/tools/docker/service/`, Task 7.5b commit 1306ca8 — replaces the original `internal/tools/docker_service.go` from M5.3 + ADR-006 + ADR-008 per V8 Option (c) Replace; see ContainerFactory Extension addendum below): HTTP-API wrapper for persistent Docker services. Long-lived service containers; runner makes HTTP requests; image pinning per ADR-006 risk #14.
   - Consumers: ZAP (M7.3), MobSF (M7.4).

ADR-006's "persistent Docker services" classification is refined: the term applies to ZAP and MobSF only post-ADR-026; Trivy and SQLMap reclassify to DockerRunner warm pool.

**Lazy warm pool semantics.**
WarmPool starts empty. First Checkout triggers spin-up of a single container; subsequent Checkouts reuse warmed containers if available, otherwise spin up to max-bound. Resource floor is zero for unused tools (customers running only SAST scans pay zero cost for Trivy/Nmap/SQLMap pools). Cleanup hook runs between Checkouts to ensure tenant isolation (no state leak between scans of different tenants/targets). Stateless tools use the exported NoCleanup helper explicitly — statelessness is architecturally visible.

**Concurrency contract.**
WarmPool is safe for concurrent use across goroutines. Internal mechanism: channel-based available pool with sync.Mutex for size accounting; separate done channel for shutdown signal (channel-close-as-signal anti-pattern explicitly avoided to prevent send-to-closed-channel races). Health-check replacement is size-neutral on success (stop -1 + spinUp +1 = 0); size decrement only on spinUp failure. Blocked Checkouts observe both ctx.Done() and the shutdown signal to prevent silent nil-Container hazards.

These concurrency invariants are pinned by:
- `TestWarmPool_Concurrent_NoRaceCondition` — race-detector pin (10 goroutines × 50 cycles)
- `TestWarmPool_Shutdown_UnblocksWaitingCheckout` — done-channel signal pin
- `TestWarmPool_Return_DecrementsSizeAllowsFutureSpinUp` — size accounting pin

**ToolRunner contract symmetry.**
DockerRunner satisfies the ToolRunner interface (Run/Name/Category) symmetric with NativeRunner. Per-tool config (image, BuildArgs, ParseOutput, ExitCodeLenient, Timeout); framework handles lifecycle + observability + finding enrichment (ToolName/EngineCategory/DiscoveredAt/Fingerprint populated by runner not ParseOutput per `internal/tools/runner.go:65-69` docstring). Compile-time interface assertion (`var _ tools.ToolRunner = (*DockerRunner)(nil)`) prevents signature drift.

3-tier timeout precedence symmetric with NativeRunner: `cfg.Timeout (per-scan) → runner.Timeout (per-tool default) → DefaultDockerTimeout (30 min framework floor)`.

Cleanup runs via `context.WithoutCancel(parent) + 30s timeout` — detached from Run's primary context to ensure cleanup completes even when Run hits its timeout. Cleanup-uses-parent-context anti-pattern explicitly avoided (matches the M6.7 Wapiti file-output cleanup pattern at 2nd instance).

**Rationale.**
Three framework alternatives considered:

| Alternative | Why rejected |
|---|---|
| Single DockerRunner type for all 5 M7 tools | Conflates HTTP-API session management with CLI execution; complicates per-tool configuration; cleanup contracts genuinely differ between persistent services and short-lived CLI tools. |
| Per-tool runner type (no shared framework) | Reinvents lifecycle management per tool; loses the ToolRunner interface uniformity ADR-023 established for native tools; dramatically multiplies test surface area. |
| Persistent containers for CLI tools (ADR-006 verbatim) | Resource floor non-zero for unused tools; no cleanup-between-checkouts contract for tenant isolation; CLI tools that exit cleanly fight against persistent-service framework. |

The two-framework-types decision (Option β) provides:
- Clear separation of concerns: HTTP session management vs CLI execution
- Symmetric framework patterns (NativeRunner / DockerRunner / DockerServiceRunner all satisfy ToolRunner with appropriate per-tool config)
- Cost-driver elimination via warm pool for CLI tools (matches ADR-006's original cost-driver intent)
- Tenant isolation via cleanup-between-checkouts contract
- Resource floor at zero for unused tools

**Cross-reference:** ADR-022, ADR-023, ADR-024 invoke the asymmetric-cost meta-principle (architectural commitment cost vs alternative cost). ADR-026 is the project corpus's 5th ADR invoking this meta-principle, well past 3-instance threshold; promotion candidate to DEVELOPMENT-PATTERNS at next architectural decision-point.

**Consequences.**

Positive:
- Two M7 framework types with clean separation of concerns; ADR-006's broad architecture refined with operationally accurate classification
- Warm pool primitive enables zero-floor resource cost for unused tools; matches the ADR-006 cost-driver elimination intent without persistent-service complexity
- Tenant isolation via cleanup-between-checkouts contract is architecturally explicit
- Consumer tasks (M7.1 Trivy, M7.2 Nmap, M7.6 SQLMap) inherit ready-to-use DockerRunner framework
- Test coverage 43/43 (7 Container + 19 WarmPool + 17 DockerRunner); race detector clean

Negative:
- Two framework types vs one increases the M7 framework surface area; each consumer task requires deciding which framework applies (though Option β resolution makes this assignment unambiguous per tool category)
- DockerServiceRunner framework needs Task 7.5b expansion before ZAP + MobSF consumer tasks (M7.3, M7.4) can land
- ADR-006 retains its broad framing but now requires reading the addendum + ADR-026 for accurate tool classification — small discoverability cost mitigated by inline addendum + cross-reference

**Anti-patterns this prevents.**
- Channel-close-as-signal pattern in concurrent code: WarmPool uses dedicated done channel for shutdown signaling; available channel never closed.
- Decrement-without-increment size accounting: Health-check replacement is size-neutral on success.
- Cleanup-uses-parent-context pattern: defer Pool.Return uses `context.WithoutCancel(ctx) + 30s timeout` to survive primary-context cancellation.
- ParseOutput populating identity fields: Framework owns identity enrichment per ToolRunner docstring contract.
- Single-type-fits-all framework: Two framework types align with two operational shapes (CLI vs HTTP-API).

**Triggers to revisit.**
1. **Task 7.5b DockerServiceRunner framework expansion.** ~~When framework lands, verify HTTP session management + version drift mitigation + health checks satisfy ADR-006 risk #14 + ADR-008 MobSF requirements.~~ **RESOLVED** via shieldscan-engine commit 1306ca8 + ContainerFactory Extension addendum above (Phase 5.B; this commit). HTTP session management (`Client.Get/Post/PollUntil` + `AuthFunc`); version drift mitigation (digest pinning per consumer per Q9 lock); health checks (readiness probe at spin-up via `waitForReady` per Q6 lock; HealthCheck field nil v1 per Q9 lock with promotion path to checkout-time probe forward-pinned).
2. **First DockerRunner consumer (M7.1 Trivy).** Validate the framework abstraction against real tool integration. If consumer task surfaces framework gaps (e.g., output streaming for large SBOMs, exit-code-leniency edge cases), iterate via additive framework changes.
3. **Cleanup-uses-parent-context anti-pattern at 3rd instance.** Currently 2nd instance (M6.7 Wapiti + Phase 3 DockerRunner). 3rd instance triggers DEVELOPMENT-PATTERNS promotion.
4. **Compile-time interface assertion at 3rd instance.** Currently 2nd instance (NativeRunner native.go:147 + DockerRunner). 3rd instance triggers promotion.
5. **dockerClient adapter pattern at 3rd instance.** Currently 1st instance (productionClient). If reused for AWS SDK, GCP SDK, or other SDK abstractions in future tasks, track for promotion.

**Forcing functions.**
- Compile-time `var _ tools.ToolRunner = (*DockerRunner)(nil)` assertion: signature drift fails build before any test runs.
- `TestWarmPool_Concurrent_NoRaceCondition`: 500-cycle race detector pin; CI fails if any concurrency regression introduced.
- `TestWarmPool_Shutdown_UnblocksWaitingCheckout`: done-channel signal contract pin; future engineer reverting the done-channel pattern triggers test failure.
- ADR-026 cross-references in shieldscan-engine `internal/tools/runner.go` docstring + commit f5d77c8 message body: future engineers searching for runner-type assignments find this ADR via either docstring read or git log search.

**ContainerFactory Extension (Task 7.5b; shieldscan-engine commit 1306ca8).**

Task 7.5b DockerServiceRunner framework expansion surfaced Phase 0 verification finding V2: WarmPool's `spinUp` was hardcoded to exec-shape containers (`Cmd: [sleep infinity]`; no `PortBindings`; private function). Service-shape consumers (ZAP, MobSF) need port mapping + readiness probing without `Cmd` override. Two architectural options were considered:

- **Option (α) — Framework extension via ContainerFactory hook (~30-50 LoC additive):** add `ContainerFactoryFunc` type + `Config.ContainerFactory` field + `DefaultContainerFactory` exported function; `spinUp` routes through the factory. Preserves Task 7.2 Nmap consumer backward-compat (nil ContainerFactory → DefaultContainerFactory → existing newContainer behavior). Selected.
- **Option (β) — Parallel service-shape pool primitive (~150 LoC):** new ServicePool type alongside WarmPool; duplicates lazy-spin-up + max-bound + cleanup-between-checkouts logic. Rejected: violates ADR-026's single-WarmPool-primitive contract; multiplies test surface; framework-asymmetric.

Option (α) preserves the WarmPool primitive as the canonical container-pool abstraction. A single primitive serves three container shapes:

- **Exec-shape** (`DefaultContainerFactory`): DockerRunner consumers — Nmap (Task 7.2 commit 872b2b0); future Trivy (Task 7.1) and SQLMap (Task 7.6).
- **Service-shape** (`ServiceContainerFactory` in `internal/tools/docker/service/spinup.go`): DockerServiceRunner consumers — future ZAP (Task 7.3) and MobSF (Task 7.4); configures `PortBindings` (127.0.0.1:dynamic) + `ContainerInspect` for dynamic host-port discovery + readiness probing.
- **Future shapes:** additional factories accommodate gRPC-shaped, batch-shaped, or sidecar-paired containers without further WarmPool changes.

Implementation surface (per shieldscan-engine commit 1306ca8):

- `ContainerFactoryFunc func(ctx context.Context, cli dockerClient, image string, log zerolog.Logger) (*Container, error)` — new exported type in `internal/tools/docker/warmpool.go`.
- `Config.ContainerFactory ContainerFactoryFunc` — new field on WarmPool config; nil → DefaultContainerFactory.
- `DefaultContainerFactory` — new exported function; replicates existing newContainer behavior.
- `spinUp` body change — routes through `p.factory(ctx, p.cli, p.image, p.log)` (~3 LoC).
- `DockerClient = dockerClient` exported type alias added to `internal/tools/docker/container.go` to enable cross-package factory function literals (Task 7.5b D1 deviation).
- `ContainerInspect` method added to the `dockerClient` interface — required for dynamic host-port discovery in ServiceContainerFactory (Task 7.5b D2 deviation).
- `NewServiceContainer(id, image, baseURL, cli, log) *Container` exported constructor + `BaseURL` exported field on Container struct — enables service-shape Container construction from external packages (Task 7.5b D3 deviation).

Backward-compat verification (per shieldscan-engine commit 1306ca8 quality gate):

- All 21+ existing Task 7.5a + Task 7.2 tests pass unchanged.
- 4 new ContainerFactory-specific tests added (nil-routing-to-default; custom-factory-invoked; factory-error-propagates; DefaultContainerFactory delegation).
- DockerRunner consumers (Nmap shipped; Trivy/SQLMap forthcoming) require zero changes; default-routes through DefaultContainerFactory implicitly.

This extension is additive and backward-compatible; framework-symmetric. WarmPool remains the single container-pool primitive; specialization happens at the factory layer rather than the pool layer. Per asymmetric-cost-aware reasoning: extension cost (~30-50 LoC additive + 4 new tests + 3 D1/D2/D3 cross-package interface deviations) is bounded; alternative cost (parallel pool primitive at ~150 LoC + duplicated concurrency contract + asymmetric framework surface) compounds across future container shapes.

**Open follow-ups.**
- ~~Task 7.5b: DockerServiceRunner framework expansion (HTTP-API persistent services).~~ **RESOLVED** via shieldscan-engine commit 1306ca8 + ContainerFactory Extension addendum above (Phase 5.B; this commit).
- Per-tool consumer tasks: M7.1 Trivy, M7.2 Nmap, M7.3 ZAP, M7.4 MobSF, M7.6 SQLMap.
- Asymmetric-cost meta-principle DEVELOPMENT-PATTERNS promotion candidate (5th instance well past threshold).
- Cleanup-uses-parent-context anti-pattern tracking (2nd instance; promotion at 3rd).
- Compile-time interface assertion tracking (2nd instance; promotion at 3rd).

**Cross-references.**
- ADR-006: Hybrid Native + Persistent Docker (broad architectural framing; refined per addendum).
- ADR-008: MobSF for Mobile Security (DockerServiceRunner consumer precedent).
- ADR-013: Python sole writer for scan state (atomicity preserved; runners return findings to caller).
- ADR-017: Findings inline in job_completed events (per-runner findings flow into shared pipeline).
- ADR-021: ctx-discipline (context propagation honored throughout WarmPool + DockerRunner).
- ADR-022: Recon-as-pre-scan-helpers (asymmetric-cost meta-principle precedent).
- ADR-023: NativeRunner OutputFile mode (framework precedent; DockerRunner symmetric).
- ADR-024: RawFinding schema extension (asymmetric-cost meta-principle precedent).
- shieldscan-engine commit f5d77c8: warm pool primitive + DockerRunner framework implementation.
- shieldscan-engine commit 1306ca8: Task 7.5b DockerServiceRunner framework + ContainerFactory hook extension; ContainerFactory Extension addendum above documents the framework expansion.
- shieldscan-engine `internal/tools/runner.go`: ToolRunner contract docstring + three-runner-type architecture documentation.
- shieldscan-docs `docs/plans/2026-05-03-warm-pool-primitive-design.md`: design doc artifact (Phase 2 corrections pending).
- shieldscan-docs `plans/2026-05-09-task-7.5b-docker-service-runner-design.md` (commit 3067c92 design doc revision; commit bb09f4e Phase 5.A post-implementation alignment): Task 7.5b design doc; V2 Option (α) ContainerFactory hook resolution lock.
- IMPLEMENTATION-PLAN Task 7.5: warm pool primitive obligation.
- SPEC §3.2: shieldscan-engine repository structure (path drift corrected per this commit).

### ADR-027: RawFinding.Metadata field for per-tool structured payload
**Status:** Accepted (2026-05-06, Task 7.2)

**Context.**
ADR-024 (RawFinding schema extension) established the canonical RawFinding shape with typed evidence fields (TargetURL, Parameter, Payload, Request, Response for web/API; CodeFile, CodeLine, CodeSnippet for source; MobileOS, Permission, ComponentName for mobile; CipherSuite, CertSubject for SSL/TLS) plus four ADR-024 categorical fields (References, Tags, CVSSVector, AdditionalCWEs).

Task 7.2 (Nmap as first DockerRunner consumer) brainstorming surfaced that the typed-evidence-fields pattern doesn't fit Nmap's per-port structured output (host, port, protocol, service, product, version, extra_info, CPE identifiers, tunnel attribute). Three accommodation options were considered:

- **Option (a) — Schema extension via dedicated Metadata field:** add `Metadata map[string]string` to RawFinding; per-tool structured payload as key-value pairs. Type-safe map access; extensible without schema change per consumer. Selected.
- **Option (b) — Tags namespacing:** use existing `Tags []string` with namespaced keys (`"service:ssh"`, `"product:OpenSSH"`, etc.). Zero schema change. Rejected: stringly-typed; downstream parsing burden for M9 CVE matching; namespace convention breaks if service names contain `:`.
- **Option (c) — Repurpose typed fields:** TargetURL=`tcp://host:port`; Parameter=port number; service/product/version into Description only. Rejected: semantic misuse of TargetURL (documented for HTTP URLs); structured-data-loss for M9 downstream consumption.

The asymmetric-cost meta-principle applies: cost of "schema extension via ADR" is one ADR + small SPEC §7.3 update + one map field across consumers (per ADR-024 precedent: cheap). Cost of "every M7 consumer reinvents Tags namespace conventions" compounds across 5 consumers (Nmap, Trivy, SQLMap, ZAP, MobSF) + creates downstream parsing burden + loses type safety.

**Decision.**
Add `Metadata map[string]string` field to RawFinding (`internal/events/events.go`). JSON tag `json:"metadata,omitempty"` ensures backward-compatibility for shieldscan-api findings-ingest consumer (per ADR-025, lives in shieldscan-api): nil/empty Metadata gets omitted from serialized output; existing consumer code unmarshalling RawFinding to a struct without the new field will ignore it (Go's encoding/json default behavior).

Per-tool consumer pattern: each tool documents its Metadata key contract in package docstrings. Required keys (always present) + conditional keys (omitted when empty). Naming convention: lowercase snake_case keys; tool-namespaced if cross-consumer collision risk (e.g., `"nmap.target"` vs `"trivy.target"` — though most keys are tool-naturally-distinct so namespacing is per-key judgment, not blanket prefix).

**Nmap consumer (Task 7.2; first canonical consumer):**
- Required keys: `host`, `port`, `protocol`, `state`, `service`, `target`
- Conditional keys: `product`, `version`, `extra_info`, `cpe`, `tunnel`

**Future M7 consumers (forward-pinned per Task 7.2 design doc §8):**
- **Trivy (Task 7.1):** SBOM components — `component_name`, `component_version`, `package_manager`, `license`, `installed_path`
- **SQLMap (Task 7.6):** SQLi technique details — `injection_type`, `dbms`, `parameter_position`, `payload_technique`
- **ZAP (Task 7.3):** HTTP context beyond TargetURL — `http_method`, `request_headers_hash`, `response_code`, `attack_vector`
- **MobSF (Task 7.4):** APK/IPA analysis — `manifest_permission`, `component_type`, `intent_filter`, `obfuscation_score`

These are illustrative; canonical key contracts land in each consumer task's design doc + package docstring.

**M9 AI Pipeline downstream consumption** (forward-pin): CVE matching consumes `cpe` + `product` + `version` from Nmap; `component_name` + `component_version` from Trivy; etc. Severity adjustment based on CVE matches. Cross-tool finding correlation via shared keys (e.g., `host` matches Nmap finding to ZAP finding for same target).

**M8 Recon-First Pipeline downstream consumption** (forward-pin): Nmap structured port/service/version data feeds downstream scanners' target lists. Recon pipeline reads Metadata map directly; no parsing required.

**Rationale.**
ADR-024 establishes the additive-RawFinding-schema-extension pattern. ADR-027 invokes the same pattern for the 6th time across the project corpus (after ADR-022 recon-as-pre-scan-helpers; ADR-023 NativeRunner OutputFile mode; ADR-024 RawFinding schema extension; ADR-025 Findings-ingest direct DB-write [shieldscan-api]; ADR-026 DockerRunner framework). Per the canonical asymmetric-cost meta-principle established by ADR-022 and ADR-024: **architectural commitments are made when the alternative is operationally worse, not when a generic threshold is met.** The 6-instance pattern is well past 3-instance threshold; DEVELOPMENT-PATTERNS promotion candidate evaluated in Phase 5.D (forthcoming this session).

Schema-evolution precedent: small, additive, backward-compatible field additions per ADR are the established pattern. ADR-027 extends it to per-tool structured payloads without breaking the typed-evidence-fields pattern that ADR-024 established. Both patterns coexist: typed fields for cross-tool common evidence (TargetURL, Severity, etc.); Metadata map for tool-specific structured data.

**Consequences.**

Positive:
- Per-tool structured payload preserved end-to-end (Nmap → RawFinding → ADR-025 ingest → M9 AI Pipeline → M8 Recon)
- Schema evolution remains additive; new M7 consumers (Trivy, SQLMap, ZAP, MobSF) inherit the field without per-consumer schema changes
- Type-safe map access (Go's `map[string]string`); not stringly-typed
- Backward-compatibility via `omitempty` JSON tag; existing shieldscan-api findings-ingest consumer unaffected
- M9 CVE matching downstream consumption gets first-class data access

Negative:
- Map keys are stringly-typed within the map (no compile-time enforcement of key contracts)
- Per-tool key contracts must be documented in package docstrings; documentation discipline becomes load-bearing
- Cross-consumer key conflict risk (mitigated by per-key namespace judgment but not enforced)
- Schema-evolution-via-map can mask design pressure that should land as new typed fields

**Anti-patterns this prevents.**
- Typed-fields-for-everything inflation: ADR-024's 4 schema extensions covered cross-tool common evidence; ADR-027 prevents pressure to add typed fields per per-tool concept (e.g., would ADR-024-pattern require adding `NmapPort int`, `NmapService string`, `TrivyComponent string` etc. to RawFinding? Map field eliminates that pressure).
- Tags-as-structured-data abuse: ADR-027 prevents reinventing namespace conventions in `Tags []string` for structured per-tool payloads. ADR-024 documents Tags as fine-grained tags MUST NOT duplicate `engine_category`; using Tags for `"service:ssh"` etc. would conflict with that contract.
- Stringly-typed-everything: ADR-027 confines stringly-typing to keys-within-the-map; the field itself is type-safe `map[string]string`.
- Per-consumer-schema-churn: ADR-027 prevents each new M7 consumer task from triggering its own schema-extension ADR.

**Triggers to revisit.**
1. **First M7 consumer hits Metadata key contract conflict.** When Trivy + SQLMap + ZAP + MobSF land, verify no two consumers use same key for different semantics. If conflict, namespace prefixes may need to become enforced convention (e.g., `"nmap.host"` always; never bare `"host"`).
2. **M9 AI Pipeline implementation surfaces schema gap.** If CVE matching needs a typed field that map representation can't cleanly support (e.g., versioned CPE comparison requiring native type semantics), evaluate schema extension via new ADR.
3. **Per-tool key contract documentation drift.** If consumer package docstrings drift from actual emitted keys, surface for remediation.
4. **Asymmetric-cost meta-principle promotion.** Evaluated in Phase 5.D (6-instance threshold met across ADR-022/023/024/025/026/027).

**Forcing functions.**
- **JSON `omitempty` tag on Metadata field:** ensures nil/empty Metadata serializes cleanly; backward-compatibility regression test would catch breakage.
- **Per-tool consumer package docstring documents Metadata key contract:** if docstring drifts from emitted keys, the discrepancy is visible during code review.
- **Phase 5.A design doc post-implementation alignment (commit aa0d034):** documents the 6-required + 5-conditional Nmap key contract; precedent for future consumer design docs.
- **SPEC §7.3 schema extension (this commit):** authoritative source for Metadata field semantics; future consumer ADRs cross-reference this.

**Open follow-ups.**
- Phase 5.C (forthcoming this session): DEVELOPMENT-PATTERNS entry for cleanup-uses-parent-context anti-pattern (3rd instance threshold met; M6.7 Wapiti + ADR-026 DockerRunner.Run + Task 7.2 runMain).
- Phase 5.D (forthcoming this session): asymmetric-cost meta-principle promotion evaluation (6-instance threshold).
- Future Trivy/SQLMap/ZAP/MobSF consumer tasks: each lands per-tool Metadata key contract.
- M9 AI Pipeline implementation: consumes Metadata for CVE matching + cross-tool finding correlation.
- M8 Recon-First Pipeline implementation: consumes Metadata for downstream scanner target list population.
- shieldscan-api SQLAlchemy `RawFinding` model + Pydantic schema: synchronized extension (per ADR-024 coordination workflow); the columns-ready-vs-ingest-implementation distinction documented in §7.3 callout applies symmetrically.

**Cross-references.**
- ADR-022: Recon-as-pre-scan-helpers (asymmetric-cost meta-principle, 1st invocation).
- ADR-023: NativeRunner OutputFile mode (asymmetric-cost meta-principle, 2nd invocation; framework precedent for tool-specific config).
- ADR-024: RawFinding schema extension (asymmetric-cost meta-principle, 3rd invocation; direct schema-extension pattern precedent for ADR-027).
- ADR-025: Findings-ingest direct DB-write (asymmetric-cost meta-principle, 4th invocation; consumer-side; lives in shieldscan-api repo, not in this SPEC).
- ADR-026: DockerRunner framework + lazy warm pool (asymmetric-cost meta-principle, 5th invocation; framework that ADR-027 extends).
- shieldscan-engine commit 872b2b0: Task 7.2 engine implementation; lands Metadata field at internal/events/events.go.
- shieldscan-docs commit f87e923: Task 7.2 design doc; documents per-tool key contract in §3.4.
- shieldscan-docs commit aa0d034: Task 7.2 design doc post-implementation alignment.
- SPECIFICATION.md §7.3: RawFinding schema documentation (Metadata field added per this ADR).
- shieldscan-docs Phase 5.C (forthcoming this session): DEVELOPMENT-PATTERNS entry for cleanup-uses-parent-context.
- shieldscan-docs Phase 5.D (forthcoming this session): asymmetric-cost meta-principle promotion evaluation.

---

## 14. Meta-Principles

Meta-principles are reasoning frames that recur across architectural decisions. They are structurally distinct from ADRs (which are individual decisions; one-way doors) and from DEVELOPMENT-PATTERNS entries (which are concrete code patterns; tactical). Meta-principles are the *frame* a decision is made through — they shape *how* ADRs reason, not *what* ADRs decide.

This section documents project-corpus-wide meta-principles. Promotion to this section happens when a meta-principle has been invoked across 3+ ADRs without contradiction; the cross-instance accumulation evidences a stable reasoning frame, not just a single ADR's local logic.

Future ADRs may invoke documented meta-principles by reference (e.g., "Per §14.1 asymmetric-cost meta-principle, ..."). Inverse pin: future ADRs that depart from a documented meta-principle should explicitly acknowledge the departure + reason in their text.

### 14.1 Asymmetric-cost meta-principle

**Promoted:** 2026-05-06 (Task 7.2 Phase 5.D; 6-instance threshold met)

**Canonical phrasing.**

> Architectural commitments are made when the alternative is operationally worse, not when a generic threshold is met.

**Frame.** Architectural decisions sometimes face symmetric framing: "we should add this thing only if N instances accumulate." Symmetric framing is appropriate when the decision's cost (cognitive load, schema growth, framework expansion) is roughly equal to the cost of NOT making it (continued ad-hoc handling, accumulated technical debt). When the costs are asymmetric — i.e., when continued ad-hoc handling becomes operationally worse than the architectural commitment — the threshold is overrideable. The decision is made on cost-asymmetry, not on threshold-reaching.

This frame is invoked when an architectural decision deserves attention before reaching a generic threshold OR when a generic threshold is reached but the decision deserves *not* being made because the alternative isn't operationally worse.

**Invocation enumeration.** As of 2026-05-06, the meta-principle has been invoked in 6 ADRs across the project corpus:

| # | ADR | Repo | Phrasing |
|---|---|---|---|
| 1 | ADR-022 (Recon-as-pre-scan-helpers) | shieldscan-docs SPEC §13 | Canonical |
| 2 | ADR-023 (NativeRunner OutputFile mode) | shieldscan-docs SPEC §13 | Variation: emphasizes threshold-override mechanism explicitly |
| 3 | ADR-024 (RawFinding schema extension) | shieldscan-docs SPEC §13 | Canonical |
| 4 | ADR-025 (Findings-ingest direct DB-write) | shieldscan-api DRIFT-LOG | Canonical |
| 5 | ADR-026 (DockerRunner framework + lazy warm pool) | shieldscan-docs SPEC §13 | Variation: short parenthetical "architectural commitment cost vs alternative cost" |
| 6 | ADR-027 (RawFinding.Metadata field) | shieldscan-docs SPEC §13 | Canonical |

4 of 6 invocations use canonical phrasing; ADR-023 + ADR-026 use semantically-equivalent variations. Variations are preserved as-written in their respective ADRs; future ADRs invoking the meta-principle should use canonical phrasing OR cross-reference §14.1 directly.

**Authorship note.** ADR-022 hosts the canonical phrasing as a forward-pin to the M6/M7 corpus, but explicitly self-disclaims being the threshold-override application — that role is filled by ADR-023 (first APPLIED). ADR-024 retroactively counted ADR-022 as 1st invocation. The corpus has internal ambiguity about authorship vs application; §14.1 supersedes the authorship question by establishing the meta-principle at section-level, separate from any individual ADR.

**When to invoke.** ADR drafters should invoke §14.1 when:

1. An architectural decision is being made at lower-than-typical-threshold count (e.g., 2 instances) because the alternative cost is high
2. An architectural decision is being deferred at higher-than-typical-threshold count (e.g., 5+ instances) because the alternative cost is low
3. Cost-asymmetry is the primary justification for the decision direction; symmetric framing would mislead readers about the reasoning

**When NOT to invoke.** §14.1 is NOT a generic "make architectural decisions wisely" frame. Routine decisions made on standard threshold-counting do not invoke §14.1; they apply standard project-corpus thresholds (e.g., DEVELOPMENT-PATTERNS' 3-instance promotion). §14.1 specifically applies when threshold-counting would mislead the decision direction.

**Tension acknowledgment.** Meta-principles risk becoming checklist items ("did you invoke §14.1?") rather than genuine reasoning frames. The frame's value is in articulating *why* a decision direction was chosen against threshold-counting, not in providing rhetorical cover for any decision direction. Drafters should invoke §14.1 only when cost-asymmetry is genuinely load-bearing for the decision; otherwise, standard threshold-counting reasoning suffices.

**Cross-references.**
- ADR-022 (originating ADR; canonical phrasing host; self-disclaimed application)
- ADR-023 (first APPLIED instance; phrasing variation)
- ADR-024 (3rd invocation; established schema-extension precedent)
- ADR-025 (consumer-side; lives in shieldscan-api DRIFT-LOG)
- ADR-026 (DockerRunner framework; phrasing variation)
- ADR-027 (RawFinding.Metadata; canonical phrasing; Phase 5.B Task 7.2)
- shieldscan-engine DEVELOPMENT-PATTERNS.md entry #5 (cleanup-uses-detached-context; 3-instance promotion via different threshold mechanism; not a §14.1 invocation but illustrates the standard threshold path)

### 14.x Future Meta-Principles

Reserved for future promotions. Candidates that have not yet reached 3-ADR threshold:

- *To be enumerated as candidates emerge.*

---

## 15. Glossary

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
