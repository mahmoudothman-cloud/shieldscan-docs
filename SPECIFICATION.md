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
│   │   │       │                   # (M7.5b commit 1306ca8; replaces docker_service.go)
│   │   │       ├── zap/           # ZAP DAST consumer (M7.3; commit e905afe)
│   │   │       └── mobsf/         # MobSF MAST consumer (M7.4; commit c15a60d)
│   │   ├── nuclei.go
│   │   ├── semgrep.go
│   │   ├── nmap.go
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

**Phase 5.C update (2026-05-19; Task 7.1):** Trivy directory layout shipped as `internal/tools/docker/trivy/` (package directory; 7 files: `types.go` + `parser.go` + `scan.go` + `trivy.go` + `parser_test.go` + `scan_test.go` + `integration_test.go` + `testdata/`) per Q4 plan §3.5 concretization + Phase 1 implementation (shieldscan-engine commit `d4028d0`). Same directory-layout drift pattern as Task 7.4 MobSF (`mobsf.go` singular → `mobsf/` package per Phase 5.C `767a99a`). M5-era SPEC §3.2 singular `trivy.go` framing (line 248 above) preserved per historical-authority discipline; current implementation reality is package-based per DockerRunner-shape consumer convention.

**Phase 5.C update (2026-05-23; Task 7.6):** SQLMap directory layout shipped as `internal/tools/docker/sqlmap/` (package directory; 7 source files + 3 testdata: `parser.go` + `scan.go` + `sqlmap.go` + `parser_test.go` + `scan_test.go` + `sqlmap_test.go` + `integration_test.go` + `testdata/{dvwa-sqli-scan,empty-no-injection,malformed}.txt`) per Task 7.6 design `d8e25b5` §4 Q4 lock + Phase 1 implementation (shieldscan-engine commit `723426d`; cross-repo pair partner shieldscan-api `2cd4065`). Third repetition of singular-`.go` → package-directory drift pattern (Trivy + MobSF + SQLMap); same DockerRunner-shape consumer convention. M5-era SPEC §3.2 singular `sqlmap.go` framing (line 247 above) preserved per historical-authority discipline. Per-section adaptor 2nd-instance pattern empirically grounded across MobSF (1st; 7 adaptors at `service/mobsf/sections.go`) + SQLMap (2nd; 2 adaptors `adaptInjectionFindings` + `adaptDBMSFingerprint` at `sqlmap/parser.go`); pattern stays at 2 instances pending 3rd-instance evaluation per Task 7.1 P5.D `aa3fb5f` scope-mismatch methodology.

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

### 7.6 AttackSurface Event (Go → Redis Pub/Sub → Python) (2026-05-30 Addendum)

**`EventAttackSurface` wire shape canonical authority.**

Engine recon pipeline publishes `EventAttackSurface` events to the completions Pub/Sub channel (`shieldscan:completions`) for `AttackSurface` row persistence. Channel routing per ADR-014 mixed-primitives lock (Pub/Sub for persistence-targeted events; per-scan Streams reserved for SSE).

**Wire shape (JSON):**

```json
{
  "event_type": "attack_surface",
  "scan_id": "uuid",
  "organization_id": "uuid",
  "root_domain": "example.com",
  "subdomains": [
    {
      "url": "https://api.example.com",
      "status": "live",
      "status_code": 200,
      "tech_stack": ["nginx", "Node.js", "React"],
      "last_probed_at": "2026-05-30T14:32:00Z"
    }
  ],
  "timestamp": "2026-05-30T14:32:01Z"
}
```

**Field semantics:**

- `event_type`: literal `"attack_surface"` (dispatch key at api consumer)
- `scan_id` + `organization_id`: UUID FK references; `organization_id` used for RLS GUC `SET LOCAL app.current_org_id` pre-write at api consumer
- `root_domain`: canonical project root domain string
- `subdomains`: array of per-subdomain rich rows
- `subdomain.url`: full HTTP(S) URL string; api consumer parses subdomain label via `urlparse(...).hostname`
- `subdomain.status`: `"live"` / `"dead"` / `"timeout"` string; maps to `SubdomainStatus` enum at api
- `subdomain.status_code`: HTTP status code int (optional; omitted for dead/timeout)
- `subdomain.tech_stack`: tech fingerprint string array (optional; empty/null for unfingerprinted)
- `subdomain.last_probed_at`: RFC3339 UTC timestamp (optional; api fills with event `timestamp` if absent)
- `timestamp`: emission RFC3339 UTC timestamp

**Channel routing:** `shieldscan:completions` Pub/Sub per ADR-014. Consumer is api `completions_consumer.py` with new `_handle_attack_surface` dispatch. Per-scan Streams are reserved for SSE consumption (Task 4.4 / `ProgressSubscriber`).

**Sole-writer continuity:** api `completions_consumer` remains the sole writer of `AttackSurface` ORM rows per ADR-013; events drive persistence; engine NEVER directly writes `AttackSurface`.

**Multi-process posture:** UPSERT-idempotency-by-construction via `uq_scan_subdomain` UNIQUE constraint + `ON CONFLICT (scan_id, subdomain) DO UPDATE` (Q-MULTI-PROCESS-POSTURE b). ADR-017 sequencing is NOT invoked for `AttackSurface` UPSERT — the unique-constraint + CONFLICT-DO-UPDATE pattern makes duplicate delivery safe without sequence ordering.

**Drift #58 Layer B root-cause repair:** Prior to this addendum, no api service consumed `EventAttackSurface` events nor wrote `AttackSurface` rows — the ORM existed at `src/app/models/recon.py` but no writer existed (orphaned-table catch-class lineage from Drift #54). This wire shape addendum plus Stage 3 Commit 3 api consumer implementation repair the api consumer absence. Two-layer manifestation: Layer A engine emission shape repaired at TOOL-ARCHITECTURE.md §8.5 addendum + engine Stage 3 Commit 2; Layer B api consumer absence repaired here on the canonical-authority side + at Stage 3 Commit 3 (implementation side).

**Cross-references:** TOOL-ARCHITECTURE.md §8.5 addendum (engine emission canonical); Task 8.3α design doc `plans/2026-05-30-attack-surface-consumer-design.md` + implementation plan `plans/2026-05-30-attack-surface-consumer-implementation.md`; §13 ADR-013 (sole-writer) + ADR-014 (mixed-primitives) + ADR-017 (sequencing; not invoked) + ADR-018 (forward-pinned).

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
**Status:** Accepted (2026-04-18); Amended (2026-05-11; ephemeral-default addendum per Task 7.4 + Task 7.5c V4 precedent)
**Decision:** MobSF as persistent Docker service for all mobile scanning (APK, IPA, source).
**Rationale:** Best open-source mobile security tool. Covers both platforms and both static + binary analysis. Zero licensing cost. REST API first-class integration.

**Ephemeral-Default Refinement (Task 7.4; shieldscan-engine commit c15a60d).** Task 7.5c V4 verification (shieldscan-engine commit bfccef8) empirically demonstrated that ZAP `newSession` is session-state-only; user definitions + custom scan policies + per-policy attack-strength tuning persist across cleanup attempts (S7 PARTIAL_RESET orphan user records; S8 NOT_RESET custom scan policies; S9 NOT_RESET per-policy tuning). MobSF V5 forward-pin from Task 7.5b design (shieldscan-docs commit 3067c92) explicitly flagged analogous "Suppression / user / settings table persistence UNCLEAR" — same architectural failure mode possible.

Per §14.1 asymmetric-cost meta-principle (this invocation enumerated as §14.1 row 7): bounded ephemeral container startup cost (~60–120s amortized poorly for low-frequency mobile scans per pricing tier §9.1) vs unbounded multi-tenant configuration-state-leakage risk where cleanup contract is UNCLEAR. Calculus tilts hard toward ephemeral default until empirical verification of MobSF `delete_scan` cleanup contract completeness.

**Refined v1 architectural commitment:** MobSF persistent-service framework infrastructure preserved (per ADR-026 ContainerFactory hook); v1 consumer DEFAULT is `cfg.EphemeralContainer = true` (Task 7.4 Q5 lock). Warm-pool path forward-pinned to Task 7.5d empirical verification of MobSF `delete_scan` cleanup contract (analogous to Task 7.5c V4 plan-then-execute pattern). When Task 7.5d empirically verifies cleanup contract COMPLETE across configuration-scoped state (suppression registry + user accounts + instance settings + per-scan plugin tuning), warm-pool path becomes viable; flip Default to false. Until then, ephemeral is the architecturally-correct default (NOT transitional).

**Backward-compat verification:** Framework-level commitment (ADR-026 ContainerFactory + DockerServiceRunner) unchanged; only consumer-default refined. Other future MobSF deployment topologies (e.g., dedicated single-tenant deployments where multi-tenant leak surface doesn't apply) may continue persistent-service path per their own architectural justification.

**Asymmetric-cost-aware reasoning:** Ephemeral cost is bounded — measured in seconds; pricing tier §9.1 caps mobile scans. Cleanup-contract-uncertainty cost is unbounded — would ship multi-tenant data leakage if Task 7.5d empirically validates the same failure mode Task 7.5c V4 surfaced for analogous consumer. Calculus prefers verified-cost over unverified-cost in safety-relevant territory.

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

### ADR-013 Addendum: Payload Contract Auth-Field Extension (ADR-015 Enablement; 2026-05-21)

Extends payload contract per ADR-015 enablement: `auth` field semantics — nullable (no credential provided) → typed when present (`{type: <auth_type>, data: <decrypted-blob>, fields?: <form-fields-map>}`). Orchestrator is sole writer of credential-bearing payloads per ADR-013 canonical authority. Cross-reference ADR-015 §13 for security model + mitigations + threat surface.

**Selected.**

### ADR-013 Addendum: Payload Contract Pre-Signed URL Extension (MobSF R2 Pre-Signed URL Task; 2026-05-23)

Extends payload contract per MobSF R2 pre-signed URL task: `JobMobileConfig` gains `signed_fetch_url` field alongside existing `upload_ref`. Both fields populated by orchestrator during migration window per Q1 (β) sibling-field decision. Orchestrator-as-sole-writer-of-time-bounded-access-tokens responsibility per ADR-013 canonical authority preserved (architectural analog to ADR-015 credential decryption: orchestrator generates pre-signed URL via R2 SDK at scan-dispatch time). Engine consumes `signed_fetch_url` via plain HTTP GET when present + non-empty; falls back to `upload_ref` (`r2://<key>`) via worker-side R2 SDK per Q3 (a) backward-compat during migration window. Migration close (forward-pinned to ***"Begin MobSF R2 migration-close task"*** per Q7-refined) removes `upload_ref` emission. Cross-reference R2 design doc shieldscan-docs `b25e9ba` §3 for threat model + mitigations + Q-chain locks.

**Selected.**

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

### ADR-014 Addendum: Credential Transit Posture (ADR-015 Enablement; 2026-05-21)

Extends Redis Streams transit medium per ADR-015 enablement: credential-bearing payloads carry decrypted credentials at-rest in Redis for queue-residence duration. Mitigations enumerated in ADR-015 §13 — Redis authenticated access + TLS-in-transit + short queue TTL + no-persistence config (`appendonly no`) for credential-bearing queues. v1.1+ adds per-queue ACL + AOF cipher if persistence enabled.

**Selected.**

### ADR-014 Addendum: Pre-Signed URL Transit Posture (MobSF R2 Pre-Signed URL Task; 2026-05-23)

Extends Redis JobDispatch transit medium per MobSF R2 pre-signed URL task: `JobMobileConfig.signed_fetch_url` payloads carry pre-signed R2 GET URLs at-rest in Redis for queue-residence duration (typically seconds-to-minutes per BullMQ semantics; cross-reference ADR-014 transit characterization). Mandatory v1 mitigations: 600s URL expiry per Q2 (a) lock (10x buffer over expected queue-residence; bounded stolen-URL exposure window) + Redis authenticated access per ADR-014 canonical + TLS-in-transit between api/engine ↔ Redis + short queue TTL. Architectural analog to ADR-015 credential transit posture (symmetric orchestrator-as-sole-writer-of-time-bounded-access-tokens forensics; symmetric ephemeral-access-token discipline). v1.1+ enhancements forward-pinned: per-scan-deadline-derived expiry (Q2 (c)) + URL refresh-on-expiry (Q2 (d)) at scale-up motivation. Cross-reference R2 design doc shieldscan-docs `b25e9ba` §3.4 for threat model details.

**Selected.**

### ADR-015: Decrypted Credentials in Redis Transit
**Status:** Accepted (2026-05-21, ADR-015 enablement task)

**Context:**
Scan targets often require authentication (cookies, bearer tokens, basic-auth, custom headers, form-based login) to reach injectable surfaces. Credentials are stored encrypted (Fernet) in `ProjectCredential.encrypted_data`. Workers (engine) execute scans against authenticated targets and need decrypted credentials at runtime. The architectural question is: where does decryption happen — orchestrator-side (transit decrypted via Redis) or worker-side (transit encrypted; worker decrypts)?

ADR-015 was reserved at Task 4.2 (`cf3b30a`) pending the orchestrator's `auth` block enablement in job payloads (which carried `null` per `test_dispatch_payload_auth_is_null_pending_adr_015` regression pin). Task 7.6 SQLMap consumer surfaced this empirically at Phase 1 Drift #35 (architectural-reconciliation: integration test could not reach DVWA's auth-gated SQLi endpoint without cookies; wiring-validation reframing landed pending this ADR).

**Decision:**
Orchestrator decrypts `ProjectCredential` at scan-dispatch time and emits decrypted credentials in the Redis job payload (`JobDispatch.Auth` field). Workers consume pre-decrypted credentials and thread them to `target.AuthConfig` for consumer-side use. Alternative "fetch-by-reference" (worker fetches ciphertext + decrypts) was considered and rejected per (a) inverting current pre-built infrastructure; (b) requiring worker-side Vault integration + DB ACLs; (c) longer attack window (DB + key distribution).

**Architectural pattern:** Decrypted-in-transit with bounded threat surface.

- **Threat surface:** decrypted credentials at-rest in Redis for queue-residence duration (typically seconds-to-minutes per BullMQ semantics).
- **Mandatory v1 mitigations:** (1) Redis authenticated access; (2) TLS-in-transit between api/engine ↔ Redis; (3) short queue TTL (bounded queue-residence-duration); (4) Redis no-persistence config (`appendonly no`; no AOF write of credential-bearing payloads).
- **Recommended v1.1+ mitigations:** (5) Redis ACL per-queue (credential-bearing queues read-restricted to worker-pool identity); (6) encryption-at-rest at Redis layer (TLS + AOF cipher if persistence enabled).

**AuthType coverage:** ADR-015 v1 enables all 5 typed values per existing `AuthType` enum (`cookie`, `bearer`, `basic`, `custom_header`, `form`). Orchestrator decrypt+emit code is auth-type-agnostic. Per-consumer handling is per-consumer concern (SQLMap v1 cookie-only; bearer/basic/custom_header/form land per consumer need).

**Discriminator translation:** `ProjectCredential` model keeps `auth_type` DB column (DB schema unchanged). Orchestrator translates DB `auth_type` → wire `type` at payload-emit time. Two-contract separation (DB schema vs engine wire) is canonical separation-of-concerns; translation is orchestrator's sole-writer responsibility per ADR-013.

**Audit logging:** Orchestrator emits audit row at decrypt-time: `{timestamp, project_id, credential_id, scan_id, dispatcher_user_id}` per scan-job dispatch consuming a credential. Audit table (TBD; can land as `credential_access_audit`). v1 includes audit; revocation flow forward-pinned to separate task.

**Out of scope (forward-pinned):**
- (a) Credential revocation flow — separate task (trigger: *"Begin credential revocation flow task"*)
- (b) Per-credential rate limits
- (c) Credential rotation (use existing application-level patterns)
- (d) R2 pre-signed URL pattern (forward-pinned to MobSF V10 task; orthogonal threat model)

**Cross-references:**
- ADR-013 (sole-writer + payload contract; addendum extends `auth` field semantics)
- ADR-014 (Redis Streams transit; addendum captures credential-transit posture mitigations)
- ADR-024 (RawFinding schema; unaffected)
- ADR-027 (Metadata; unaffected)
- Task 4.2 `cf3b30a` (orchestrator deferral; lifted by this ADR)
- Task 7.3 (ZAP design forward-pin; consumer-side cookie pass-through)
- Task 7.6 (SQLMap Drift #35 architectural-reconciliation; integration test auto-upgrade target)
- `shieldscan-engine` `DRIFT-LOG.md` lines 175 + 581-583 (reservation status; updated at Stage 3 Commit 3)
- `shieldscan-engine` commits `723426d` (Task 7.6 cross-repo pair Commit 2; Drift #35 origin) + `ba860a5` (Task 7.5e Phase 1+2; framework Mounts capability + Entrypoint fix)
- `shieldscan-docs` commits `15d1ac5` (Task 7.5e ADR-026 Mounts Extension addendum precedent shape) + `c28a5de` (Task 7.6 P5.C latest docs state pre-ADR-015) + `b344d0c` (ADR-015 design doc; §4 verbatim authority for this section) + `00dd2d1` (ADR-015 implementation plan; Stage 3 sub-step canonical)
- `shieldscan-api` commits `2cd4065` (Task 7.6 SCAN_TYPE_TOOLS; SQLMap routing; `auth:null` currently — lifted at Stage 3 Commit 2) + `e6fb0a5` (Task 7.1; orchestrator scaffold)

**Selected.** Decrypted-in-Redis-transit is the canonical pattern for job-queue + worker architectures; matches existing pre-built infrastructure; bounded threat surface mitigatable via existing Redis security practices.

### ADR-015 Addendum: Credential Lifecycle Extension (Credential Revocation Flow Task; 2026-05-28)

Extends ADR-015 with lifecycle revocation semantics, settling the Q6 (a) revocation forward-pin operationally. Revocation = hard-delete + cascade-cancel per Q2 (γ) (preserves `ProjectCredential` per-project UNIQUE invariant — Drift #41 M1; migration `d4f6b1e9a527` — without partial-index relaxation), Q3 (β) (extend existing DELETE `/orgs/:org_id/projects/:project_id/credentials` endpoint with cascade-cancel; no new verb), and Q4 (a) (orchestrator-side fan-out via `find_in_flight_scans_by_project(project_id)` + loop `CancelPublisher.publish_cancel`; reuses Task 4.5 `cancel_scan` infrastructure 1:1 including PG state flip → `CANCELED` + `SCAN_CANCELED` audit + Pub/Sub signal). Audit emission per Q6 (b): single new `ProjectAction.PROJECT_CREDENTIAL_REVOKED` at revoke-action; cascade-canceled scans emit existing `SCAN_CANCELED` (Task 4.5 reuse). Forensic chain reconstructed via time-window join on (`PROJECT_CREDENTIAL_REVOKED`, `SCAN_CANCELED`) audit rows by project + actor + timestamp. Engine ZERO scope per Q1 (b) axis 4 + Q5 (a) + V-FG empirical infrastructure-already-shipped: `shieldscan-engine` `internal/worker/processor.go:181-322` per-job `jobCtx` + `spawnCancelWatcher` + `ctx.Done` + 3 goleak-clean exit paths per ADR-021 Rule 2 handles credential-revocation-cancel identically to user-initiated cancel; no `cancel_reason` field, no `Scan.cancel_reason` DB column, no engine repo changes. Cross-reference Task 4.5 cancel infrastructure (`shieldscan-api` `routes/scans.py:246-340` `cancel_scan` + `services/scan_queue.py:224` `CancelPublisher`; V-FF canonical authority) + revocation design doc `shieldscan-docs` commit `0e55a4f` + revocation implementation plan commit `fdad021`.

**Selected.**

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

**→ ADR-028 canonical authority (2026-06-09).** M8.1β.2 architectural decision supersedes this speculative invocation pattern. See ADR-028 for canonical two-phase dispatch architecture (phase-1 recon ScanJob via `engine="recon"` + phase-2 per-(target, tool) ScanJobs via `orchestrator.dispatch_phase2`). The speculative example below is preserved as historical context but is INFORMATIONAL, not BINDING.

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

#### ADR-022 Addendum: Drift #60 Name-Mismatch Reconciliation (2026-06-08)

**Drift #60 catalogued:** `SCAN_TYPE_TOOLS` at `shieldscan-api/src/app/services/orchestrator.py` lists 6 engine names not registered at the engine `worker.Registry` (stored-design-intent-with-unimplemented-mechanism catch-class; 3rd instance of pattern after Drift #54 source-ingestion + Drift #58 AttackSurface consumer). Surfaced at M81_PV pre-verification (Outcome 3 architectural-decision territory survey) + expanded to 6-engine surface at M81A_PV pre-verification.

**6-engine surface across 3 sub-categories:**

| Sub-category | Engines | Disposition |
|---|---|---|
| Name-mismatch | `dependency_check` (api) ↔ `depcheck` (engine) | **RESOLVED at M8.1α** (this addendum + api Commit 2) |
| Engine-variant naming | `nuclei_fast`, `nuclei_api`, `zap_api` | FORWARD-PINNED to M8.1β |
| Recon-orphan | `subfinder`, `httpx` | FORWARD-PINNED to M8.1β |

**Name-mismatch resolution:** Y-CANONICAL-NAME-DIRECTION (α) — api adapts to engine canonical name. The engine `worker.Registry` registers the runner as `depcheck` (per `cmd/worker/registry_wiring.go` L107-128 spec table; engine-binary path `dependency-check.sh`). The api `SCAN_TYPE_TOOLS` entries at `FULL_WEB_SOURCE` + `FULL_SPECTRUM` are renamed `dependency_check` → `depcheck` to match. Behavior-preservation invariant: ScanJob row count + dispatch ordering unchanged; engine `registry.Get("depcheck")` now resolves rather than `emitFailure`-ing.

**Pre-existing ADR-022 architecture lock PRESERVED:** Recon (`subfinder`, `httpx`) remains pre-scan-helper at `internal/tools/recon/`; NEVER ToolRunner-registered. Resolution of the recon-orphan sub-category (M8.1β) must align with this lock: target-expansion mechanism invokes `RunRecon` from outside the per-ScanJob dispatch flow (NOT register subfinder/httpx as ToolRunners). The engine-variant sub-category (M8.1β) is architecturally orthogonal to ADR-022 — its resolution is bundled with the M8.1β scan-executor architectural decision (engine=nuclei + `config.fast=true` vs engine=nuclei_fast as separate engine).

**Forward-pin chain (5/6 engines pending M8.1β):**

- Engine-variant resolution decision: tied to scan-executor architecture (M8.1β Stage 1 design doc)
- Recon-orphan resolution decision: tied to scan-executor architecture (M8.1β Stage 1 design doc) per ADR-022 architecture lock
- **Discipline-level forward-pin (rule-of-three trigger fired):** "audit-driven model+spec orphan check" becomes a standard pre-verification step for future tasks. Drift #60 establishes pattern durability (3rd instance of stored-design-intent-with-unimplemented-mechanism catch-class); meta-discipline pattern integration into pre-verification template warranted.

**Compressed-lifecycle disposition (Approach B per cancel-helper extraction commit `40ce2f1` precedent):** Bounded refactor + 3-commit cross-repo trio (docs → api → engine) + commit bodies serve as canonical authority artifacts + ZERO design doc + plan ceremony. Aggregate LoC scope ~60-120; matches forecast envelope.

**Cross-references:** M81_PV + M81A_PV pre-verification surface reports (prior sessions); Task 8.3α design doc `0030319` + plan `dba6a7c` §6 (M8.1 forward-pin context); ADR-013 sole-writer canonical + ADR-014 mixed-primitives canonical + ADR-022 (recon-helper canonical preserved); Drift #54 source-ingestion fix (`ac82d48` P5.A close); Drift #58 Task 8.3α AttackSurface consumer (`05023f4` Stage 3 C3 close).

#### ADR-022 Addendum Continuation: M8.1β.1 Engine-Variant Resolution (2026-06-09)

**Sub-category 2 of Drift #60** (engine-variant naming; 3 engines: `nuclei_fast` / `nuclei_api` / `zap_api`) **RESOLVED** at this lifecycle per Y-ENGINE-VARIANT-RESOLUTION (a) config-flag (M81B_PV V-UE empirically grounded; prior session).

**Resolution lock:**

| Variant engine name | Resolved to | Variant axis mechanism |
|---|---|---|
| `nuclei_fast` (QUICK ScanType) | `engine=nuclei` | `ScanConfig.Depth="quick"` injected at `_build_job_payload` via `_DEPTH_FOR_SCAN_TYPE` map |
| `nuclei_api` (API ScanType) | `engine=nuclei` | `target.target_type="api"` already auto-set per existing `_target_type_for(ScanType.API) → "api"` |
| `zap_api` (API ScanType) | `engine=zap` | `target.target_type="api"` already auto-set per same mechanism |

**Rationale per V-UE + V-VC + V-VD empirical analysis:** Pre-existing canonical `ScanConfig` infrastructure (`Depth` knob at `runner.go:206-225` + `TemplateCategories` + per-target `target_type` discriminator via `_target_type_for`) supports all three engine-variants natively. `nuclei_fast`/`nuclei_api`/`zap_api` are artificial engine names for semantic `engine + config` differentiation. Engine `ScanConfig` is the canonical variant-axis mechanism; the api orchestrator threads variant defaults via the `config` block already present in the wire payload.

**Wire-shape preservation:** ZERO wire schema changes. `target.target_type` discriminator already wired (Source-Ingestion Fix orchestrator preserved); `config` block already produces `dict(scan.config or {})` (existing). M8.1β.1 only extends the orchestrator's `config` defaults injection — no `JobDispatch` field additions.

**Architectural continuity:**
- **ADR-022** (recon-as-pre-scan-helper) preserved — no engine registry changes for variant resolution
- **ADR-013** (sole-writer) preserved — api remains canonical `SCAN_TYPE_TOOLS` dispatch authority; engine receives `engine + config + target_type` payload
- **ADR-026** (NativeRunner / DockerRunner consumers) preserved — `nuclei` (NativeRunner) and `zap` (DockerServiceRunner) consumers unchanged at engine

**Drift #60 progress after M8.1β.1:** **4/6 engines resolved** (1 name-mismatch at M8.1α + 3 engine-variants here). Remaining 2/6 (`subfinder` + `httpx` recon-orphans) forward-pinned to **M8.1β.2 scan-executor ADR-style decision document** (Outcome 3 architectural-decision territory per M81_PV).

**Cross-references:** M81B_PV pre-verification surface report (prior session; V-UE grounded engine-variant resolution + Pattern A 2-task decomposition lock); M8.1α lifecycle CLOSED at `fb8cff9` + `2b36d62` + `64b8421`; Pattern A 2-task decomposition (M8.1β → M8.1β.1 + M8.1β.2) locked at M81B_PV; ADR-022 canonical preserved; ADR-026 NativeRunner/DockerRunner consumer assignments preserved.

#### ADR-022 Addendum Continuation: M8.1β.2 Recon-Orphan Resolution + Drift #60 6/6 Closure (2026-06-09)

**Sub-category 3 of Drift #60** (recon-orphan; 2 engines: `subfinder` + `httpx`) **RESOLVED** at M8.1β.2 ADR-028 per Y-RECON-ENGINE-NAME (a) + (a.ii) implicit orchestrator dispatch.

**Resolution lock:** `subfinder` + `httpx` REMOVED from `SCAN_TYPE_TOOLS` entirely (4 web-ScanType entries: QUICK + FULL_WEB + FULL_WEB_SOURCE + FULL_SPECTRUM). Phase-1 recon dispatch is orchestrator-implicit per web-ScanType category (dedicated `engine="recon"` ScanJob; `SCAN_TYPE_TOOLS` stays semantically pure as tools-only list).

**ADR-022 architectural lock preserved:** Recon stays pre-scan-helper at `internal/tools/recon/`. Engine `processor.Process` invokes `RunRecon` directly (NEW dispatch case `engine="recon"` at Stage 3 Commit 2); NOT via registry. ADR-022 explicit rejection of "register subfinder/httpx as ToolRunners" preserved by construction.

**Drift #60 6/6 closure end-to-end:**

- Name-mismatch (1/1): `depcheck` (RESOLVED at M8.1α; commits `fb8cff9` + `2b36d62` + `64b8421`)
- Engine-variant (3/3): `nuclei_fast` + `nuclei_api` + `zap_api` (RESOLVED at M8.1β.1; commits `bb3e75f` + `d773776` + `9ccde1a`)
- Recon-orphan (2/2): `subfinder` + `httpx` (RESOLVED at this ADR M8.1β.2 commit 1 docs canonical; engine processor dispatch case at C2; api `SCAN_TYPE_TOOLS` rename at C3)

**Discipline-level forward-pin preserved:** "audit-driven model+spec orphan check should become standard pre-verification step" (rule-of-three trigger fired at #60; 3 instances of stored-design-intent-with-unimplemented-mechanism catch-class — #54 source-ingestion + #58 AttackSurface consumer + #60 SCAN_TYPE_TOOLS orphans).

**Cross-references:** ADR-028 (M8.1β.2 canonical authority for recon-orphan resolution); M81B_PV pre-verification (prior session; engine-variant + recon-orphan sub-category framing); Task 8.3α architecture (foundation for hybrid follow-up dispatch composition).

#### ADR-020 Closure Note (2026-06-09): Promotion-Trigger Empirically Didn't Fire at M8.1β.2

ADR-020 (worker concurrency model) was reserved with explicit promotion-trigger documented at DRIFT-LOG L349: "M5.5 ships per-worker BRPOP-loop concurrency only; M8's recon-first executor ships per-job tool-fanout. Promote to ADR-020 if M8.1 surfaces a load-bearing trade-off."

**Empirical outcome at M8.1β.2.** Promotion-trigger DID NOT fire. Per Q2 Y-DISPATCH-MODEL (a) per-(target, tool) ScanJobs + Q3 Y-CONCURRENCY-MODEL (a) per-worker only: phase-2 dispatch creates N×M ScanJob rows at api orchestrator (concurrency happens at dispatch-fanout layer at api side); existing BRPOP-loop + per-worker semaphore consumes them naturally; no new concurrency primitives introduced at engine side.

**ADR-020 disposition.** Stays DEFERRED indefinitely. Future tasks may revisit if load-bearing concurrency trade-offs surface empirically. M8.1 hypothesized promotion-trigger is closed at this ADR.

**Cross-references.** ADR-028 (M8.1β.2 canonical authority); Q2 + Q3 brainstorming locks at Stage 1 design doc `3f07611` §3.2 + §3.3; M81B_PV V-UI pre-verification (worker concurrency primitives + ADR-020 reserved territory analysis).

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
M7 introduces 5 Docker-based scanning tools (Trivy, Nmap, MobSF, ZAP, SQLMap) extending the M6-shipped 9 native CLI tools. ADR-006 (Hybrid Native + Persistent Docker, refined 2026-04-18) established the broad architectural decision to use persistent Docker services for heavy tools, eliminating 2-3s per-scan container startup. ADR-008 (MobSF for Mobile Security) refined this for MobSF specifically as a service-shape DockerServiceRunner consumer (default ephemeral v1; warm-pool path forward-pinned to Task 7.5d cleanup verification per ADR-008 ephemeral-default addendum).

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
1. **Task 7.5b DockerServiceRunner framework expansion.** ~~When framework lands, verify HTTP session management + version drift mitigation + health checks satisfy ADR-006 risk #14 + ADR-008 MobSF requirements.~~ **RESOLVED** via shieldscan-engine commit 1306ca8 + ContainerFactory Extension addendum above (Phase 5.B; this commit). HTTP session management (`Client.Get/Post/PollUntil` + `AuthFunc`); version drift mitigation (digest pinning per consumer per Q9 lock); health checks (readiness probe at spin-up via `waitForReady` per Q6 lock; HealthCheck field nil v1 per Q9 lock with promotion path to checkout-time probe forward-pinned). (Framework-level infrastructure satisfied; consumer-default refined per ADR-008 ephemeral-default addendum — see Task 7.4 + Task 7.5c V4 precedent.)
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

### ADR-026 Addendum: Mounts Extension (Task 7.5e; 2026-05-20)

**Context.** First DockerRunner integration test (Trivy Task 7.1 P1.8) surfaced D-PLAN-3 framework gap — `DefaultContainerFactory` lacked Mounts/socket/bind-mount support. Trivy fs-mode real-container scans require host bind-mount (host scan-target path → container scan path); integration tests `t.Skip`'d pending framework extension. Task 7.5e Phase 0 v2 empirical verification (V1.s-ii OCI-sufficient verdict) narrowed framework-extension scope: image-mode works via OCI registry pull without Docker socket; only fs-mode bind-mount required.

**Decision.** Extend `WarmPool` `Config` with optional `Mounts []mount.Mount` field; `WarmPool.New()` internally constructs closure threading `cfg.Mounts` to `newContainer` when `cfg.ContainerFactory` is nil AND `cfg.Mounts` is non-empty. `DefaultContainerFactory` signature unchanged (passes nil mounts to `newContainer`; backward-compat preserved); `ContainerFactoryFunc` signature unchanged (service-shape `ServiceContainerFactory` + custom factories unaffected). `newContainer` signature extends to accept `mounts []mount.Mount` parameter; `HostConfig.Mounts` populated when non-nil.

**Architectural pattern (Q1 α.ii closure capture).** Three-branch decision tree at `WarmPool.New()`:

1. `cfg.ContainerFactory != nil` → custom factory used as-is (service-shape consumers + custom factories preserved)
2. `cfg.ContainerFactory == nil && cfg.Mounts != nil` → internal closure wraps `newContainer` with captured `cfg.Mounts`
3. `cfg.ContainerFactory == nil && cfg.Mounts == nil` → `DefaultContainerFactory` (pre-7.5e behavior preserved)

Closure capture-by-value (`mounts := cfg.Mounts`) ensures slice header isolation; caller mutation of `cfg.Mounts` post-`New()` does not affect closure execution.

**Scope (Q5 (a) DockerRunner-only).** Extension applies to DockerRunner exec-shape framework only. Service-shape DockerServiceRunner consumers (ZAP, MobSF) unaffected — `ServiceContainerFactory` implements `ContainerFactoryFunc` independently without reading `Config.Mounts`. Parallel DockerServiceRunner Mounts extension forward-pinned: trigger phrase ***"Begin DockerServiceRunner Mounts parallel extension task"*** if service-shape consumer surfaces empirical bind-mount need.

**Consumer-side guidance — Trivy Task 7.5e adoption pattern.** Single bind-mount with env-overridable source path:

```go
scanBasePath := os.Getenv("TRIVY_SCAN_BASE_PATH")
if scanBasePath == "" { scanBasePath = "/tmp" }
mounts := []mount.Mount{{Type: mount.TypeBind, Source: scanBasePath, Target: "/scan", ReadOnly: true}}
docker.New(docker.Config{Image, MaxSize, Mounts: mounts, ...}, ...)
```

Future consumers populate `Config.Mounts` with their specific bind-mount needs. `ReadOnly:true` recommended for read-only scan workloads (defense-in-depth + tenant isolation).

**Bonus correctness fix (D-PLAN-7.5e-Phase2-Entrypoint).** Phase 2 integration testing surfaced latent framework correctness bug — `newContainer` did NOT override image `ENTRYPOINT`, causing containers using tools with declared `ENTRYPOINT` (Trivy, Nmap) to fail warm-pool `sleep infinity` initialization. Fix: `Entrypoint: []string{}` added to `container.Config` (canonical Docker SDK pattern). Universal benefit across all DockerRunner consumers; not Mounts-specific. 26th cumulative framing-drift catch this session-tail (latent correctness bug surfaced by empirical real-Docker integration testing).

**Operational validation.** Trivy integration tests (`TestIntegration_TrivyContainer_Alpine` + `TestIntegration_TrivyFs_TestData`) lifted from `t.Skip` and pass real-Docker end-to-end (~12.3s + ~10.8s respectively). D-PLAN-3 operationally CLOSED.

**Cross-references.** shieldscan-engine commit `<7.5e-engine-hash>` (cross-repo pair Commit 2; forthcoming); Task 7.5b ContainerFactory Extension addendum precedent (engine commit `1306ca8`); Task 7.1 D-PLAN-3 framework gap forward-pin origin (engine commit `d4028d0`); Task 7.5e plan (this docs repo commit `829fe4b`); `aquasec/trivy:0.70.0@sha256:be1190af...961a41e` + `instrumentisto/nmap:7.94` (consumer images affected by entrypoint fix); `github.com/docker/docker v28.5.2+incompatible` (`mount.Mount` API source).

**Selected.** Additive framework change preserving backward-compat for Task 7.2 Nmap + Task 7.4 MobSF + Task 7.3 ZAP consumers.

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

### ADR-028: Scan-Executor Recon-First Architecture

**Status:** Accepted (2026-06-09; M8.1β.2 Stage 3 Commit 1 landing).

**Context.** Pre-launch ShieldScan needed scan-executor architecture reconciling:

- (a) Speculative engine-side multi-target scaffolding at TOOL-ARCH §8.2+§10.3+§12.3 + SPEC §1885+ (pre-dates ADR-013 sole-writer; M81B_PV V-UB INFORMATIONAL classification)
- (b) Per-(scan, tool) ScanJob dispatch at api orchestrator with single `target_url` (M81_PV V-RC empirical state)
- (c) Drift #60 6-engine surface across 3 sub-categories: name-mismatch resolved at M8.1α (`fb8cff9` + `2b36d62` + `64b8421`); engine-variant resolved at M8.1β.1 (`bb3e75f` + `d773776` + `9ccde1a`); recon-orphan (`subfinder` + `httpx`) remaining

M8.1 milestone audit V-K-F classified as arc-evolution-pivot territory; M81_PV + M81B_PV pre-verification grounded Outcome 3 architectural-decision territory.

**Decision.** Two-phase recon-first dispatch architecture per 10-Q-chain brainstorming locks (Stage 1 design doc `3f07611` §1 + §3):

**Phase-1 — Recon dispatch:** ScanCreate → orchestrator dispatches single RECON ScanJob (`engine="recon"`) for web-ScanTypes (QUICK + FULL_WEB + FULL_WEB_SOURCE + FULL_SPECTRUM). `SCAN_TYPE_TOOLS` stays semantically pure (tools only); recon is orchestrator-implicit dispatch per web-ScanType category. Engine `processor.Process` invokes `RunRecon` directly (NOT via registry per ADR-022; recon stays helper not ToolRunner). `RunRecon` emits `EventAttackSurface` to completions Pub/Sub channel (Task 8.3α infrastructure operational at `fc75a98`).

**Phase-2 — Tool dispatch:** api `completions_consumer._handle_attack_surface` UPSERTs `AttackSurface` rows (Task 8.3α infrastructure at `05023f4`) AND invokes `orchestrator.dispatch_phase2(scan_id)` in NEW session after UPSERT commit (sequential sessions per Q7 c.ii). `orchestrator.dispatch_phase2` queries `AttackSurface` inline + dispatches per-(target, tool) ScanJobs for non-recon tools (per Q2 a per-(target, tool) shape; idempotency-key `{scan_id}:{tool}:{sha256(target_url)[:16]}:{ts}` per Q2 a.ii; empty-AttackSurface = zero phase-2 dispatches per Q2 i recon-first semantics). Phase-2 identity context: audit-log lookup pattern (Q7.4 V-WD refinement; queries most recent SCAN_DISPATCHED `AuditLog` for `scan_id`; reconstructs partial `AuthIdentity` from `actor_id`). Fail-loud-audit on dispatch failure (Q7 c.ii.B; marks scan FAILED with diagnostic-rich audit per Y-PHASE2-DISPATCH-FAILURE-AUDIT-SHAPE b).

**Audit trail.** Two-phase emission (Q5 b). Phase-1 `SCAN_DISPATCHED` preserved unchanged (backward-compat). Phase-2 NEW event type `SCAN_DISPATCHED_PHASE2 = "scan.dispatched_phase2"` (per V-WC convention) with rich `details` JSONB: `{scan_type, priority, phase: "tools", target_count, tool_count, job_count, recon_event_id}`. Always-emit even when `target_count=0` (Q5 b.i; operator visibility into "scan didn't dispatch tools because recon found no targets").

**Concurrency.** Per-worker only (Q3 a); existing BRPOP-loop + per-worker semaphore handles N×M ScanJobs naturally; concurrency at dispatch-fanout layer not in-engine; ADR-020 promotion-trigger empirically didn't fire (see ADR-020 Closure Note in the ADR numbering note region).

**Migration.** NO migration (Q4 a); `ScanJob` schema preserved; idempotency-key extension dispatch-logic-only; no alembic files. Pre-launch context: bounded-staleness disposition (Q10 c); new behavior activates at deploy-time.

**Rationale.** Strategic reuse of Task 8.3α infrastructure (engine `RunRecon` emission + api `completions_consumer` + `AttackSurface` UPSERT) — Q1 hybrid follow-up dispatch composes existing primitives elegantly. ADR-013 sole-writer + ADR-014 mixed-primitives + ADR-022 recon-as-pre-scan-helper all preserved by construction. Multi-target fanout happens at architectural layer designed for it (api orchestrator extension; no new layer needed). Cancel-fanout natural per existing per-scan cancel channel infrastructure.

**Rejected alternatives.**

- (a) api synchronous pre-dispatch — IMPRACTICAL (FastAPI doesn't own recon binaries)
- (b) engine multi-target per ScanJob — VIABLE but architectural debt (`target_urls` JSONB migration + new wire-shape + per-target completion-event semantics)
- (d) ReconCoordinator at engine — SPECULATIVE (writer responsibility uncertain; no advantage over c)
- (β) System identity for phase-2 — loses original-actor attribution at `SCAN_DISPATCHED_PHASE2` audit
- (γ) Redis-cached identity snapshot — adds stateful Redis primitive outside ADR-014 scope
- (δ) Reopen Q4 + add `Scan.created_by_user_id` migration — rejected per brainstorming-chain coherence + Q4 lock-preservation

**Consequences.**

- ✅ Strategic reuse of Task 8.3α infrastructure
- ✅ ADR-013 + ADR-022 preserved by construction
- ✅ Cancel-fanout natural per existing infrastructure
- ✅ No migration scope
- ⚠️ Per-(target, tool) row count amplification for high-target scans (per-target rate-limiting forward-pinned)
- ⚠️ Implicit ordering dependency at audit-log lookup (synchronous `SCAN_DISPATCHED` emission required pre-phase-2-trigger; documented constraint)
- ⚠️ Empty-AttackSurface case: scan completes with phase-1-only (correct per recon-first semantics)
- ⚠️ Phase-2 dispatch failure handling fail-loud-audit (retry-logic forward-pinned post-M8.1β.2)

**Composition with prior architecture.**

- **ADR-013 sole-writer:** api remains canonical writer of `Scan.status` + `ScanJob.status` + `AttackSurface`; engine emits events only
- **ADR-014 mixed-primitives:** phase-2 dispatch uses completions Pub/Sub channel via Task 8.3α composition
- **ADR-017 sequencing:** not invoked at this ADR (`AttackSurface` UPSERT idempotency-by-construction per `uq_scan_subdomain`)
- **ADR-022 recon-as-pre-scan-helper:** preserved (recon stays helper; engine `processor.Process` invokes `RunRecon` directly NOT via registry)
- **ADR-020 worker-concurrency:** reserved-now-closed at this ADR (promotion-trigger hypothesis empirically didn't fire; see ADR-020 Closure Note)

**Drift #60 6/6 closure.** Recon-orphan sub-category (`subfinder` + `httpx`) RESOLVED STRUCTURALLY at this ADR (engines removed from `SCAN_TYPE_TOOLS` entirely; dedicated `engine="recon"` ScanJob at phase-1 dispatch). Combined with M8.1α (name-mismatch 1/1) + M8.1β.1 (engine-variant 3/3) + this ADR (recon-orphan 2/2) = **6/6 Drift #60 closure end-to-end**. Discipline-level "audit-driven model+spec orphan check" forward-pin preserved (rule-of-three trigger fired at #60).

**Drift #61 V-WD refinement.** Q7.4 brainstorming-chain lock "Identity from `Scan.created_by_user_id`" invalidated by V-WD pre-Stage-1 verification (field absent); refined to Option α audit-log lookup. 4th-instance plan/design-vs-empirical-precision catch-class; discipline-level meta-pattern "DEFERRED-EMPIRICAL marking for concrete-empirical-field Q-decisions" added forward-pin.

**Cross-references.** Stage 1 design doc `plans/2026-06-09-scan-executor-recon-first-design.md` (commit `3f07611`); Stage 2 implementation plan `plans/2026-06-09-scan-executor-recon-first-implementation.md` (commit `fb61129`); M81_PV + M81B_PV + V-W + V-X pre-verification surface reports; Task 8.3α design doc + plan + Stage 3 trio (`0030319` + `dba6a7c` + `721ba02` + `fc75a98` + `05023f4` + `0e5249e`); M8.1α (`fb8cff9` + `2b36d62` + `64b8421`) + M8.1β.1 (`bb3e75f` + `d773776` + `9ccde1a`) lifecycle closures; SPEC §13 ADR-013 + ADR-014 + ADR-017 + ADR-020 (closed at this ADR) + ADR-022 + addendums + this ADR-028.

---

### ADR-029: AI Analysis Pipeline Foundation (M9.0)

**Status:** Accepted (2026-06-12; M9.0 Stage 3 Commit 0 docs landing). Sub-milestone decomposition declared per Q11 — M9.0 (this foundation) → M9.A (embed/dedup; Tasks 9.1+9.2) → M9.B (correlate/score; Tasks 9.3+9.4) → M9.C (fix-gen/summary; Tasks 9.5+9.6) → M9.D (orchestrator; Task 9.7); strict linear sequencing.

**Context.** M9 AI Analysis Pipeline scope per IMPLEMENTATION-PLAN.md M9 milestone + SPEC §8 canonical authority (§8.1 pipeline stages diagram + §8.2 correlation weights + §8.3 scoring formula + §8.4 mobile fix prompt + §8.5 multi-provider cost targets + §8.6 error-recovery fallback matrix). V-EE pre-verification (M9.0 entry) surfaced 3 empirical gaps: (1) `ai_api_calls` cost-tracking table absent vs CLAUDE.md Gotcha 5 hard mandate; (2) `Scan.executive_summary` column absent vs M9.7 plan-literal pseudo-code; (3) pipeline trigger seam undefined at `completions_consumer._maybe_complete_scan`. M9.0 brainstorming chain (12 Y-decisions + 25+ sub-decisions; Mode 1 sequential conversational) resolved foundational architectural decisions for sub-milestone decomposition.

**Decision.** AI Analysis Pipeline Foundation architecture per 12-Q-chain brainstorming locks (Stage 1 design doc `a46fedd` §1 + §3):

**Phase-1 — Pipeline Execution Model (Q1):** `completions_consumer._maybe_complete_scan` detects all-jobs-terminal → invokes `ScanOrchestrator.dispatch_ai_pipeline` in NEW session (sequential per Q7 c.ii M8.1β.2 precedent) → orchestrator transitions Scan ANALYZING + LPUSH `shieldscan:ai_pipeline` queue with `scan_id` → in-api ai-pipeline-consumer task (BRPOP-loop; mirrors completions_consumer pattern) drains queue → runs M9 pipeline (embed → dedup → correlate → score → fix-gen → summary per SPEC §8.1) → transitions Scan COMPLETED/FAILED. ADR-014 mixed-primitives extension (new Redis queue `shieldscan:ai_pipeline`). Recon-invocation architectural seam forward-pin (Drift #59 + #62 adjacent-layer) extends to ai-pipeline-dispatch seam.

**Phase-2 — Cost Tracking (Q2):** `ai_api_calls` table (TenantMixin RLS isolation) records per-call rows (`scan_id` + `provider` + `model` + `operation_type` + `tokens_in` + `tokens_out` + `cost_usd` + `created_at`). Budget enforcement: hybrid pre-call check for high-cost operations (Claude fix-generation + executive summary); post-call check for low-cost (embeddings + correlation). Per-scan circuit breaker; global circuit breaker forward-pinned for production-readiness. Budget source: hardcoded ScanType constants per SPEC §8.5; database-configurable per-org tiers forward-pinned. **Closes Drift #64 (Drift #60 catch-class 4th-instance: stored-design-intent-with-unimplemented-mechanism per CLAUDE.md Gotcha 5 mandate).**

**Phase-3 — Storage (Q3):** `Scan.executive_summary` Text nullable=True column added at M9.0 migration; full Report architecture (PDF/HTML rendering + R2 + API surface) forward-pinned to M10 lifecycle. **Closes Drift #65 (Drift #61 catch-class 5th-instance: concrete-empirical-field absence sub-category 2nd-instance).**

**Phase-4 — Qdrant Topology (Q4):** Collection-per-organization `findings_{org_id}` with lazy creation at first AI pipeline call. Hard isolation at collection level (ADR-013 sole-writer architectural-layer analog; ShieldScan regulated-industry security positioning). Cross-customer trending opt-in via separate `findings_trending_consented` collection forward-pinned for onboarding maturity.

**Phase-5 — Dedup Persistence (Q5):** Per-scan dedup primary; cross-scan dedup with regression detection + fix-verification forward-pinned to M11+. Points persistent with 90-day TTL retention policy (cleanup-job forward-pinned). Deterministic point identity: `hash(tool + target_url + cwe_id + raw_finding signature)`. `Vulnerability.qdrant_point_id` UUID nullable column added at M9.0 migration; enables correlation explainability + future cross-scan dedup linkage.

**Phase-6 — Vulnerability Promotion Shape (Q6):** One Vulnerability per dedup-cluster (≥0.92 similarity); preserves per-raw-finding traceability via `Vulnerability.raw_finding_ids` UUID[] back-references. raw_finding promotion state: `raw_finding.promoted_at` timestamp + `raw_finding.vulnerability_id` FK. Existing `Vulnerability.fingerprint` column reused per Q6 (C.c) lock (V-FFC empirically verified). Algorithm-level details (cluster representative selection + correlation weighting + promotion threshold logic) deferred to M9.A activation.

**Phase-7 — Provider Client Lifecycle (Q7):** AsyncOpenAI + AsyncAnthropic + AsyncQdrantClient singletons at `src/app/services/ai/clients.py` initialized at FastAPI app lifespan startup. FastAPI Depends pattern for endpoint dependency injection; module-level singleton access for ai-pipeline-consumer background task (mirrors completions_consumer + redis_client precedent). Hard-fail at startup if API keys missing/invalid (configuration errors loud at deploy time).

**Phase-8 — Error Recovery Composition (Q8):** SPEC §8.6 fallback matrix honored: embedding/Qdrant down → rule-based fingerprint fallback; Claude rate-limit → retry 3× with exponential backoff; Claude failure → fallback per matrix. `Scan.ai_pipeline_degraded` boolean column added at M9.0 migration (default False; flipped True on any stage fallback); COMPLETED status preserved when pipeline produces final outputs via fallback. Structured logging at M9 entry; dedicated `fallback_events` table forward-pinned for production-readiness.

**Phase-9 — Migration (Q9):** Single Alembic migration revision file containing all 7 schema changes (`ai_api_calls` table + `Scan.executive_summary` + `Scan.ai_pipeline_degraded` + `Vulnerability.qdrant_point_id` + `Vulnerability.raw_finding_ids` + `raw_finding.promoted_at` + `raw_finding.vulnerability_id`) lands at M9.0 Stage 3 C1. Forward-only at M9 per pre-launch context Q10 M8.1β.2 bounded-staleness precedent; explicit `downgrade()` forward-pinned for production-readiness audit.

**Sub-milestone decomposition (Q11):** M9.0 → M9.A → M9.B → M9.C → M9.D strict linear sequencing. M9.0 closure includes no-op smoke test exercising architectural pipes (real AI provider calls deferred to M9.A activation per Q11 C.c-lite).

**Rationale.** Strategic composition with ADR-028 phase-1+phase-2 architecture (ADR-028 provides AttackSurface + Vulnerability foundation; ADR-029 builds AI pipeline atop). ADR-013 sole-writer + ADR-014 mixed-primitives + ADR-022 recon-as-pre-scan-helper all preserved by construction. completions_consumer + dispatch_phase2 patterns reused at ai-pipeline-consumer + dispatch_ai_pipeline. Pre-launch context dominant: hardcoded budgets, single-collection-per-org, no cross-scan dedup, no global circuit breaker, no fallback_events table — all forward-pinned to production-readiness audit.

**Rejected alternatives.**

- (Q1.a) Inline-consumer execution — couples completions channel to AI latency
- (Q1.b) Asyncio-background — fails reliability invariant (no persistence + no observability)
- (Q1.c) Dispatched engine ai-job — language boundary architectural awkwardness (Go engine → Python AI pipeline)
- (Q1.d) Dedicated Python ai-worker process — operational overhead at pre-launch
- (Q2.A.b) Hierarchical `ai_api_calls` + `ai_scan_budget` — over-engineered at pre-launch
- (Q3.B) Separate Report model + R2 object at M9 — absorbs M10 architecture into M9
- (Q3.D) JSONB `scan_results` column — conflates domains
- (Q4.B) Single multi-tenant Qdrant collection + payload filter — soft isolation security weakness
- (Q6.A.a) One-Vulnerability-per-cluster without per-raw-finding traceability — loses verification confidence
- (Q7.B) Per-call clients — defeats connection pooling
- (Q8.A.a) Fail-fast per-stage — loses partial pipeline value
- (Q9.A.b) Sub-milestone-aligned migrations — activation-order dependency complexity

**Consequences.**

- ✅ Strategic reuse of ADR-028 phase-1+phase-2 + completions_consumer + dispatch_phase2 patterns
- ✅ ADR-013 + ADR-014 + ADR-022 preserved by construction
- ✅ Drift #64 (ai_api_calls absence; CLAUDE.md Gotcha 5 mandate) closed at this ADR
- ✅ Drift #65 (Scan.executive_summary absence; M9.7 plan pseudo-code) closed at this ADR
- ✅ Sub-milestone decomposition declared (M9.0 → M9.A → M9.B → M9.C → M9.D)
- ⚠️ Hardcoded ScanType budgets — production-readiness audit forward-pin (database-configurable per-org tiers at enterprise pricing)
- ⚠️ Per-scan circuit breaker only — production-readiness audit forward-pin (global circuit breaker safety net)
- ⚠️ Collection-per-org operational scale — production-readiness audit forward-pin (revisit at >100 customers)
- ⚠️ 90-day TTL without enforcement mechanism — production-readiness audit forward-pin (cleanup-job implementation)
- ⚠️ Implicit ordering dependency at ai-pipeline-consumer lifespan startup ordering (documented constraint)
- ⚠️ M10 Report architecture deferred — possible `Scan.executive_summary` → `Report.summary_text` migration at M10 if architecturally warranted

**Composition with prior architecture.**

- **ADR-013 sole-writer:** api remains canonical writer of `Scan.status` + `ScanJob.status` + `AttackSurface` + `Vulnerability` + `ai_api_calls`; engine emits events only
- **ADR-014 mixed-primitives:** `shieldscan:ai_pipeline` Redis queue extension; consistent with `shieldscan:completions` Pub/Sub + per-scan idempotency-key patterns
- **ADR-017 sequencing:** not invoked at this ADR (AI pipeline operations are idempotent-by-construction per Q5 deterministic fingerprint identity)
- **ADR-022 recon-as-pre-scan-helper:** not invoked (M9 is post-scan AI analysis territory; recon stays at engine; ADR-022 architectural lock preserved by construction)
- **ADR-028 scan-executor-recon-first:** COMPOSITION (M9 consumes ADR-028 output — AttackSurface + Vulnerability rows populated by phase-1+phase-2 dispatch — and produces AI-enriched outputs)

**Drift #64 + #65 catalog references.** Stage 1 design doc `plans/2026-06-12-m9-ai-pipeline-foundation-design.md` (commit `a46fedd`) §9 dual catalogue. Persistent engine/api DRIFT-LOG entries forward-pinned to Stage 3 C1 (timing for #64/#65 resolution canonical) OR Stage 4 P5.A per M8.1β.2 V-CC reconciliation precedent.

**Cross-references.** Stage 1 design doc `plans/2026-06-12-m9-ai-pipeline-foundation-design.md` (commit `a46fedd`; 240 LoC §1-§9); Stage 2 implementation plan `plans/2026-06-12-m9-ai-pipeline-foundation-implementation.md` (commit `55dbe32`; 293 LoC §1-§8 + 4 plan-level Y-decisions); V-EE + V-FF + V-GG pre-verification surface reports; CLAUDE.md Gotcha 5 cost-tracking mandate; SPEC §8 canonical M9 authority + SPEC §13 ADR-013 + ADR-014 + ADR-017 + ADR-022 + ADR-028 + this ADR-029.

---

### ADR-030: AI Pipeline: Embedding + Deduplication (M9.A)

**Status:** Accepted at M9.A Stage 1 design doc landing (`aaf7ea0`; Y-PROMOTION-TIMING Path C gate-decision + Q1-Q12 + 35+ sub-decisions ratified); Stage 2 plan landing (`a8ad52c`; PY1-PY4 plan-level Y-decisions + Stage 3 3-commit breakdown); operational at M9.A Stage 3 C1 + C2 implementation execution.

**Context.** M9.A is the first real-AI sub-milestone of the M9 lifecycle per Q11-M9.0 strict linear sequencing (M9.0 ✅ → M9.A → M9.B → M9.C → M9.D). Composes ADR-029 (AI Analysis Pipeline Foundation; M9.0 architectural authority) by activating the no-op `run_no_op` placeholder into a real-AI pipeline: stage [1] embedding via OpenAI text-embedding-3-small (SPEC §8.1) + stage [2] deduplication via Qdrant cosine similarity ≥0.92 threshold (SPEC §8.1) + Path C cluster→Vulnerability promotion at clustering time. Activates Y2 Task 8.3β attack-surface endpoint `vulnerability_count` forward-pin (Vulnerability rows populate at M9.A; attack-surface join query returns actual counts post-pipeline execution).

**Decision (Y-PROMOTION-TIMING Path C + Q1-Q12 locks).**

**Gate-decision Y-PROMOTION-TIMING (Path C — promote-at-M9.A with incremental fields):** First finding of each dedup cluster creates Vulnerability row with cluster-representative fields (organization_id + project_id [derived from Scan] + scan_id + title + finding_type + severity + engine_category + target_url + cwe_id + fingerprint + raw_finding_ids = [first_id] + qdrant_point_id); subsequent cluster matches append raw_finding_id to existing Vulnerability.raw_finding_ids via `_merge_evidence` helper. Matches M9.0 Q6 (A.c) hybrid clustering + (B.a) raw_finding promotion state lock literally.

**Q1 Y-PIPELINE-WIRING (A' + A'.i):** Replace `run_no_op` with `run(*, db: AsyncSession, scan_id: UUID) -> None` + module-private stage helpers (`_embed_findings` + `_dedup_and_promote` + `_merge_evidence` + `_create_vulnerability_from_finding`); AIPipelineConsumer call-site updates `run_no_op(db, scan_id)` → `run(db, scan_id)`; M9.B/C/D extend `run()` composition + add additional helpers without architectural restructuring.

**Q2 Y-EMBEDDING-INPUT-CONSTRUCTION (A.b + B.c + C.c):** Extended field set (title + finding_type + cwe_id + target_url + description); field-label structure with newline separators; None-marker "N/A". Construction: `f"Title: {title}\nType: {finding_type}\nCWE: {cwe_id or 'N/A'}\nTarget: {target_url}\nDescription: {description or 'N/A'}"`. target_url inclusion prevents endpoint-level vulnerability conflation (XSS at /api/users vs /api/products → distinct Vulnerabilities).

**Q3 Y-EMBEDDING-BATCH-SHAPE (A.a + B.a + C.c + D.b):** batch=100 sequential chunking; hybrid retry (AsyncOpenAI max_retries=3 transient + manual exponential backoff 1s/2s/4s for 429 per SPEC §8.6); rule-based fingerprint fallback at M9.A on 429 exhaustion → Scan.ai_pipeline_degraded=True per Q8-M9.0.

**Q4 Y-EMBEDDING-CACHE-STRATEGY (C'):** No cache at M9.A; cache strategy forward-pinned to production-readiness audit (~$10+/customer/month threshold OR ~5+ seconds embedding latency per scan); embedding cost telemetry at M9.A C2 smoke tests for data-driven revisit (~$0.10/scan threshold).

**Q5 Y-QDRANT-OPERATIONS-API (A.c + B.c + C.a' + D.b):** Explicit `_ensure_collection_exists` helper at `_dedup_and_promote` entry; search vs query_points API choice deferred to V-PP/V-QQ Stage 3 C1 verification; deterministic UUID via `uuid.uuid5(NAMESPACE_FINGERPRINT, fingerprint)` preserves Q5-M9.0 deterministic-identity lock with UUID API compatibility; extended payload (raw_finding_id + scan_id + organization_id + cwe_id + target_url + fingerprint + vulnerability_id). NAMESPACE_FINGERPRINT = `uuid.UUID("00000000-0000-0000-0000-000000000001")`. VectorParams(size=1536, distance=COSINE).

**Q6 Y-DEDUP-ALGORITHM-DETAILS (A.a + B.a + C.a + D.b):** limit=1 + score_threshold=0.92 server-side filter per SPEC §8.1; filter to current scan only (Q5-M9.0 per-scan dedup lock literal); first-emitted cluster representative (M9.B scoring may update Vulnerability.severity per SPEC §8.3); two-phase embed-batch then dedup-sequential; Q5 payload refinement adding vulnerability_id field.

**Q7 Y-MERGE-EVIDENCE-SHAPE (A.a + B.a + C.a + D.b + Resolution γ):** Append-only field update (only Vulnerability.raw_finding_ids updated at M9.A merge; M9.B/C own field refinements); atomic same-flush at end of `_dedup_and_promote` loop; Python read-modify-write via reassignment pattern (`vuln.raw_finding_ids = [*existing, new_id]` forces SQLAlchemy dirty flag); `_merge_evidence(*, db, vulnerability, raw_finding)` module-private receiving Vulnerability instance; Resolution γ pre-generated UUID (`vulnerability_id = uuid.uuid4()`) before instantiation avoids intermediate flush. Idempotency safeguard: `if raw_finding.id in existing_ids: return`.

**Q8 Y-COST-TRACKING-INTEGRATION-POINTS (A.a' + B.a + C.a):** No pre-call check at M9.A embedding stage (honors Q2-M9.0 B.c hybrid lock — embedding classified low-cost = post-call only); per-batch log_ai_call (operation_type="embedding"; tokens_in = batch.usage.prompt_tokens; tokens_out=None; cost_usd via `Decimal(tokens) * Decimal("0.00000002")` per SPEC §8.5); API response usage field authoritative.

**Q9 Y-TEST-FIXTURE-STRATEGY (A.b + B.a + C.a + D.b):** unittest.mock patching client.embeddings.create returning canned 1536-dim vectors + usage stub; in-memory Qdrant via `AsyncQdrantClient(":memory:")` (V-PP/V-QQ Stage 3 C1 verification dependency); function-scoped fixtures; shared parametrized fixtures (two_clustering_findings, two_distinct_findings).

**Q10 Y-MIGRATION-NEEDED:** NO new schema at M9.A. M9.0 C1 schema (`b7e4a1f93c2d`) fully satisfies all M9.A Y-locks per empirical audit. Sub-milestone-specific migrations forward-pinned for M9.B/C/D.

**Q11 Y-ADR-NUMBER:** ADR-030 "AI Pipeline: Embedding + Deduplication (M9.A)" — this ADR. Composes ADR-029 architectural foundation.

**Q12 Y-STAGE3-DECOMPOSITION-FOR-M9.A (A.a + B.a):** 3-commit Stage 3 + top-down docs-first. (1) C0 docs ADR-030 (this) ~80-120 LoC; (2) C1 api pipeline.py rewrite + call-site + M9.0 C3 test conversions ~300-500 LoC; (3) C2 api new tests + M9.A smoke ~300-500 LoC. Aggregate ~680-1120 LoC.

**Plan-level Y-decisions (PY1-PY4; Stage 2 `a8ad52c`).**

**PY1 Y-RAW-FINDINGS-LOAD-QUERY:** Eager load all raw_findings for scan at `run()` entry via `select(RawFinding).where(RawFinding.scan_id == scan_id)`; sequential processing per Q6 (D.b) two-phase pattern.

**PY2 Y-VULNERABILITY-FIELDS-AT-CREATION (per V-NNE pre-grounding):** Path C cluster-representative fields with project_id derivation from Scan.project_id (RawFinding lacks project_id; Scan has it; M9.0 P5.A averted-prediction lineage).

**PY3 Y-PIPELINE-RUN-RETURN-SHAPE:** `run()` returns None per Q1 (A'.i) functional pattern; side-effects only; consumer queries database for state.

**PY4 Y-PIPELINE-ERROR-PROPAGATION:** `run()` raises on unrecoverable errors (Qdrant unavailable + OpenAI 429 retries exhausted + database errors); AIPipelineConsumer M9.0 C2 (`1c98330`) exception handler catches + Scan.ai_pipeline_degraded=True + FAILED transition; Q3 D.b rule-based fingerprint fallback handles 429 within `run()` before propagation.

**Rationale.**

- (a) Path C aligns Q6-M9.0 (A.c) hybrid clustering + (B.a) raw_finding promotion state lock literally; resolves V-MM-surfaced merge_evidence undefined-helper gap; activates Y2 Task 8.3β vulnerability_count forward-pin cleanly at M9.A.
- (b) Function-style stage helpers (Q1) match arc functional-style precedent (cost_tracking + clients are function-style); class-based orchestration deferred to M9.D (Pipeline class if needed).
- (c) Extended embedding-input field set (Q2) including target_url prevents endpoint-level vulnerability conflation; field-label structure with newline separators provides explicit semantic boundaries; None-marker "N/A" consistent across nullable fields.
- (d) Conservative batch size (Q3) honors plan-literal + matches pre-launch scale; hybrid retry strategy uses SDK strengths (transient) + manual control (429 per SPEC §8.6 exponential backoff).
- (e) No cache at M9.A (Q4) matches pre-launch context discipline (Q2/Q4/Q5/Q8/Q9 forward-pin chain at M9.0); cost telemetry enables data-driven revisit at production-readiness audit.
- (f) Qdrant deterministic UUID via uuid5 (Q5) preserves M9.0 deterministic-fingerprint lock with API compatibility; extended payload (D.b) enables debugging + cluster-hit lookup without separate JOIN.
- (g) limit=1 + 0.92 threshold (Q6) matches SPEC §8.1 specification; per-scan filter honors Q5-M9.0 per-scan dedup lock literal; first-emitted cluster representative (C.a) consistent with Path C; M9.B scoring may update severity per SPEC §8.3.
- (h) Append-only merge_evidence (Q7) preserves M9.B/C field-update authority; atomic same-flush + reassignment pattern + pre-generated UUID Resolution γ avoid SQLAlchemy intermediate-flush complexity.
- (i) Post-call cost-tracking at M9.A (Q8) honors Q2-M9.0 B.c hybrid lock for low-cost operations; pre-call discipline reserved for M9.C high-cost operations.
- (j) In-memory Qdrant test fixture (Q9 B.a) preserves real similarity behavior; novel testing territory at first-real-AI sub-milestone establishes precedent for M9.B/C/D test coverage.
- (k) No new schema (Q10) demonstrates M9.0 C1 architectural foundation completeness; sub-milestone-specific migrations preserved for M9.B/C/D if needed.
- (l) 3-commit Stage 3 + top-down docs-first (Q12) mirrors M9.0 Stage 3 4-commit pattern adjusted for no-schema-migration scope.

**Rejected alternatives.**

- (1) Y-PROMOTION-TIMING Path A — full Vulnerability creation at clustering; semantically heavier; rejected for Path C hybrid alignment with Q6-M9.0 (A.c) lock.
- (2) Y-PROMOTION-TIMING Path B — defer promotion to M9.C/D; Y2 activation delayed; M9.A scope becomes Qdrant-only; rejected for Y2 forward-pin alignment.
- (3) Q1 Pipeline class with stage methods — deviates from arc functional-style precedent (cost_tracking + clients); rejected for functional consistency.
- (4) Q2 Plan-literal field set (excludes target_url) — risks endpoint-level vulnerability conflation; rejected for architectural correctness.
- (5) Q3 Larger batch size (500-1000) — marginal benefit at ShieldScan finding counts; rejected for conservative pre-launch alignment.
- (6) Q3 Async concurrent batching — rate-limit risk; rejected for sequential simplicity.
- (7) Q4 Qdrant existence check / separate Redis cache / Plan-literal cache mechanism — premature optimization at pre-launch scale; rejected for production-readiness forward-pin.
- (8) Q5 raw_finding.id as point ID — violates Q5-M9.0 deterministic-fingerprint lock; rejected for fingerprint-based identity preservation.
- (9) Q6 Whole-collection org-scope query — de-facto cross-scan dedup; deviates from Q5-M9.0 per-scan-primary lock; rejected for Q5-M9.0 literal adherence.
- (10) Q7 PostgreSQL array_append SQL / SQLAlchemy MutableList tracking — adds complexity for marginal benefit; rejected for reassignment pattern simplicity.
- (11) Q8 Pre-call check at embedding stage — violates Q2-M9.0 B.c low-cost classification; rejected for Q2-M9.0 lock alignment.
- (12) Q9 respx HTTPX-level mock / Docker testcontainers / custom in-memory fake — dependency burden OR fidelity loss; rejected for unittest.mock + qdrant-client :memory: simplicity.

**Consequences.**

- Activates Y2 Task 8.3β attack-surface endpoint vulnerability_count forward-pin cleanly at M9.A (Vulnerability rows populate at clustering time; join query returns actual counts post-pipeline execution).
- Enables real-AI pipeline execution against M9.0 architectural foundation (AIPipelineConsumer + ai_api_calls cost-tracking + clients singletons all operational); foundation operational state empirically re-verified at V-MM pre-verification.
- Establishes test fixture precedent for M9.B/C/D (unittest.mock + in-memory Qdrant) at first-real-AI sub-milestone; pattern propagates to subsequent sub-milestones.
- Pre-launch context discipline preserved: no cache + no tiktoken + no pre-call embedding budget check + sub-milestone-specific schema migrations; production-readiness forward-pin chain documents path-to-scale-readiness.
- Recon-invocation-seam-extension discipline operational: pipeline-rewrite seam at M9.A C1; M9.0 C3 test-conversion territory anticipated (Drift #63 catch-class extension; V-PP/V-QQ Stage 3 C1 pre-verification covers).

**Composition with prior architecture.**

- **ADR-013 (RLS sole-writer):** `_create_vulnerability_from_finding` + `_merge_evidence` + raw_finding state-transition all respect tenant boundaries via established RLS GUC pattern; AIPipelineConsumer sets GUC before `pipeline.run()` invocation per M9.0 C2.
- **ADR-014 (mixed-primitives):** Vulnerability + raw_finding + ai_api_calls + Qdrant collections all per-org-isolated via established tenant pattern; collection-per-organization `findings_{org_id}` (Q4-M9.0) extends primitive.
- **ADR-022 (cost-tracking authority):** ai_api_calls operations (log_ai_call) at per-batch granularity per Q8 lock; check_budget reserved for M9.C high-cost operations per Q2-M9.0 B.c.
- **ADR-028 (scan-executor recon-first):** Pipeline runs after Phase-2 (completions) phase per M9.0 C2 dispatch architecture (completions_consumer DQ1 raw_findings-gated dispatch → orchestrator.dispatch_ai_pipeline → AIPipelineConsumer BRPOP → run()).
- **ADR-029 (AI Analysis Pipeline Foundation):** Phase-1 execution model (in-api ai-pipeline-consumer) + Phase-2 cost-tracking schema (ai_api_calls) + Phase-5 dedup persistence (Qdrant collection-per-org + Vulnerability.qdrant_point_id) + Phase-6 vulnerability promotion shape (Vulnerability.raw_finding_ids + raw_finding.promoted_at + vulnerability_id) all consumed at M9.A real-implementation activation.

**Cross-references.** Stage 1 design doc `plans/2026-06-16-m9a-embedding-dedup-design.md` (commit `aaf7ea0`; canonical reference for Y-locks + Path C resolution + V-MM/V-NNE pre-grounding); Stage 2 implementation plan `plans/2026-06-17-m9a-embedding-dedup-implementation.md` (commit `a8ad52c`; canonical reference for PY1-PY4 plan-level Y-decisions + Stage 3 sub-step breakdown + D-deviation forecasts); M9.0 lifecycle (commits `a46fedd` + `55dbe32` + `45dcabe` + `51b26ea` + `1c98330` + `8410df4` + `62499a3` + `4616672` + `6254849`; architectural foundation composed); SPEC §8.1 pipeline stages canonical authority (embedding + dedup at M9.A; correlate + score at M9.B; fix + summary at M9.C; orchestrate at M9.D); SPEC §8.5 cost targets ($0.02/1M tokens for text-embedding-3-small); SPEC §8.6 error-recovery fallback matrix (rule-based fingerprint fallback per Q3 D.b); CLAUDE.md Gotcha 5 cost-tracking mandate (activates at M9.A first real-cost logging).

---

### ADR-031: AI Pipeline: Correlation + Scoring (M9.B)

**Status:** Operational at M9.B Stage 3 C0 docs landing (this commit). Composes ADR-013 (Python sole writer) + ADR-014 (Redis Streams progress) + ADR-022 (recon as pre-scan helpers) + ADR-028 (Phase-1 execution model) + ADR-029 (M9.0 AI pipeline foundation) + ADR-030 (M9.A embedding + deduplication).

**Date:** 2026-06-26.

#### Context

Per SPEC §8.2 Cross-Layer Correlation Algorithm + §8.3 Severity Scoring Formula: post-deduplication (M9.A operational; ADR-030 Path C cluster→Vulnerability promotion), Vulnerabilities require cross-layer correlation (DAST↔SAST + cross-engine_category generalized per Q2 lock) + composite severity scoring (final_score = base_cvss × corroboration × exploitability per SPEC §8.3).

M9.B (Tasks 9.3 + 9.4) implements: (1) rule-based weighted correlation scoring per SPEC §8.2 weights (cwe_exact 0.40 + cwe_parent 0.25 + url_path 0.20 + finding_type 0.30 + parameter_name 0.15; threshold ≥0.75); (2) deterministic severity scoring per SPEC §8.3 multipliers (corroboration 1.0/1.3/1.5 + exploitability 0.8/1.0/1.2/1.5). Both are mechanistically distinct from M9.A's Qdrant cosine dedup — correlation evaluates structural finding attributes, not semantic text similarity.

Path B "link + corroborate" (Y-CORRELATION-MERGE-VS-LINK gate-decision) preserves ADR-030 Path C cluster→Vulnerability promotion architectural lock literally; correlation creates link relationships (Vulnerability.correlation_cluster_id UUID + union-find clustering) rather than row deletion; SPEC §8.2 "merged into corroborated vulnerability" interpreted as "linked into corroboration relationship."

#### Decision

**Path B + Q1-Q18 + ~45+ sub-decisions ratified per Stage 1 design doc `d10be51` + Stage 2 plan `a172fd1`:**

**Gate-decision Y-CORRELATION-MERGE-VS-LINK: Path B — Link + Corroborate.** Two Vulnerabilities scoring ≥0.75 per SPEC §8.2 → both rows preserved; correlation creates link relationships via Vulnerability.correlation_cluster_id (UUID; union-find); corroborated_count + severity refined per SPEC §8.3; no row deletion.

**Stage 3 4-commit decomposition per Q18 lock:**
- C0 docs ADR-031 at SPEC §13 (this commit; ~80-120 LoC)
- C1 api schema + modules (alembic migration + cwe_hierarchy.py + correlation.py + scoring.py + vulnerabilities.py; ~400-550 LoC)
- C2 api pipeline integration (pipeline.py + test_pipeline.py extensions; ~200-300 LoC)
- C3 api tests + smoke (test_correlation.py + test_scoring.py + test_cwe_hierarchy.py + conftest.py + test_m9b_smoke.py; ~430-660 LoC)

**Correlation algorithm (Q1-Q4 + Q6-Q7):** correlation_score sync function takes Vulnerability instances + loads raw_finding fields via raw_finding_ids join; SPEC §8.2 canonical weights; ≥0.75 threshold + correlation_score stored; SQL pairwise full-scan within scan (per-scan correlation per Q3; cross-engine_category generalized per Q2 B.b; skip same-engine pairs per Q2 C.a); itertools.combinations + neutral naming + asymmetric url_path tried both directions with max-not-sum.

**Correlation storage (Q8):** Vulnerability.correlation_cluster_id nullable UUID column + union-find clustering at correlation pass; transitive A↔B↔C automatically merged into a single cluster; Y2 vulnerability_count adapter via DISTINCT(correlation_cluster_id).

**Undefined-helper resolutions (Q5/Q6/Q7):**
- is_cwe_parent_child: CWE_PARENT_CHILD hardcoded dict at src/app/services/ai/cwe_hierarchy.py + CWEHierarchy class + module-level singleton (top-~50 CWE coverage per MITRE CWE-1000 Research View)
- route_map: heuristic 1-to-1 URL-path↔code_file matching helper (last segment match against code_file basename; conservative ambiguous-skip)
- extract_params: hybrid raw_finding.parameter primary + URL query param parsing + multi-language regex (Flask + Django + FastAPI + Express + Spring) + length ≥3 filter + normalized comparison

**Scoring formula (Q9-Q12):** compute_severity_score sync function at src/app/services/ai/scoring.py; final_score = base_cvss × corroboration_multiplier × exploitability_multiplier; cap at 10.0; default base_cvss=5.0 when null; PoC-derivation per RawFinding.tool_name ∈ {"nuclei", "sqlmap"} → 1.2 multiplier (per V-UUE DEFERRED-EMPIRICAL pre-grounding; the field is tool_name, not engine_name); auth/public default 1.0 at M9.B (forward-pinned to M9.C/D + production-readiness); add Vulnerability.severity_score Float column; standard CVSS mapping (≥9.0 CRITICAL / 7.0-8.9 HIGH / 4.0-6.9 MEDIUM / <4.0 LOW); engine-distinct corroborated_count within correlation cluster.

**Pipeline composition (Q13):** Sequential extension of M9.A pipeline.run(): embed → dedup_and_promote → _correlate_vulnerabilities → _score_vulnerabilities → flush; defensive skip when <2 distinct engine_categories present.

**Cost-tracking (Q14):** No cost tracking at M9.B (deterministic computation; no AI calls); ai_api_calls infrastructure operational from M9.0 but unused at M9.B; forward-pin Claude tie-break for near-threshold correlations (0.65-0.85) at production-readiness.

**Schema migration (Q16):** Single alembic migration at C1 + correlation_cluster_id indexed (Y2 DISTINCT queries) + severity_score not-indexed (M10 forward-pin) + explicit downgrade() implementation; revision after b7e4a1f93c2d (M9.0 C1 head).

#### Rationale

**Path B over Path A (Merge) or Path C (Hybrid):** Preserves M9.A Path C architectural lock literally (Vulnerabilities created at clustering remain; M9.B composes + extends, doesn't replace). Path A would actively contradict M9.A by deleting rows M9.A created. Path C adds architectural surface without clear value at M9.B (phased complexity defers merge to M9.D without justification).

**Rule-based weighted scoring (not embedding cosine):** SPEC §8.2 is mechanistically distinct from M9.A's Qdrant cosine dedup. Cross-layer correlation evaluates structural finding attributes (CWE + endpoint + parameter + code location), not semantic text similarity. Determinism + audit-trail value vs embedding similarity at this layer.

**Per-scan correlation (Q3):** Honors Q5-M9.0 per-scan dedup lock literally. Cross-scan correlation forward-pinned to M11+ alongside cross-scan dedup + regression-detection + fix-verification.

**Hardcoded CWE dict (Q5):** Top-~50 web-app CWE coverage sufficient at pre-launch; embedded MITRE library forward-pinned to production-readiness when CWE diversity exceeds.

**Heuristic route_map (Q6):** Cost-zero + conservative 1-to-1 matching minimizes false positives; AI/uploaded/static-analysis alternatives forward-pinned to production-readiness.

**Deterministic scoring (Q14):** SPEC §8.3 formula is mechanically deterministic; no AI call needed at M9.B; Claude tie-break for ambiguous cases forward-pinned to production-readiness.

#### Rejected Alternatives

**Path A (Merge into single Vulnerability row):** Two Vulnerabilities scoring ≥0.75 merged into one row with raw_finding_ids union + row deletion. Rejected: contradicts ADR-030 Path C architectural lock (M9.A creates separate Vulnerabilities per cluster; Path A would delete some at correlation time); FK migration burden (raw_finding.vulnerability_id → merged ID); lost engine_category accuracy at merged row.

**Path C (Hybrid Link at M9.B + Merge at M9.D):** Phased complexity defers merge semantics to M9.D orchestrator. Rejected: adds architectural surface (two algorithms to maintain); SPEC §8.2 "merged" wording can be interpreted as "linked" at M9.B without requiring a later merge stage.

**(A.a) Operates on Vulnerability rows only at Q1 / (A.b) raw_finding rows only:** Rejected. (A.a) lacks raw_finding fields (parameter + code_file + code_snippet) for correlation evaluation. (A.b) conflicts with M9.A Path C (raw_findings already promoted to Vulnerabilities at M9.A).

**(B) Cross-scan correlation at Q3:** Rejected at M9.B. Premature complexity at pre-launch; M11+ regression-detection territory.

**(B) Embed CWE data file / (C) pip library at Q5:** Rejected at M9.B. ~2-5MB repo size addition + parsing overhead (B); external dependency burden (C). Top-~50 hardcoded coverage sufficient at pre-launch.

**(A) AI route_map via Claude / (C) static analysis / (D) empty map at Q6:** Rejected. (A) cost-incurring + complexity. (C) very-high complexity multi-framework support. (D) loses url_path 0.20 weight + edge case correlation value.

**(A) regex identifiers / (C) common-name dictionary at Q7:** Rejected. (A) too noisy, high false-positive. (C) misses uncommon parameter names.

**(B) junction table only at Q8:** Rejected at M9.B. Cluster membership requires graph traversal; transitive clustering not automatic. Junction table forward-pinned for per-pair audit at production-readiness.

**(A) full URL heuristic derivation including auth/public at Q10:** Rejected. URL heuristics 30-70% accuracy insufficient; introduces more noise than signal. Default 1.0 + forward-pin to M9.C/D when scope metadata captured.

**(B) Optional Claude tie-break at Q14:** Rejected at M9.B. Introduces M9.B AI cost + complexity for marginal benefit. Forward-pinned to production-readiness with threshold-tuning analysis.

#### Consequences

**Positive:**
- ADR-030 Path C architectural lock preserved (composition + extension, not replacement)
- SPEC §8.2 + §8.3 canonical algorithms implemented deterministically
- Y2 Task 8.3β vulnerability_count semantics preserved via DISTINCT(correlation_cluster_id) adapter
- corroborated_count + severity_score updates enable M10 Report ranking
- Cross-layer evidence (DAST + SAST cross-engine corroboration) operational at M9.B
- Zero AI cost at M9.B (deterministic computation per Q14)
- Forward-pin chain established for production-readiness optimizations (per Stage 1 design doc §7)

**Negative / Trade-offs:**
- Hardcoded CWE dict requires manual extension as CWE diversity grows (mitigated: top-~50 covers ~95% of web-app vulnerabilities)
- Heuristic route_map accuracy variable (conservative 1-to-1 matching minimizes false positives)
- Auth/public exploitability inputs default to 1.0 at M9.B (mitigated: PoC derivation reliable; auth/public forward-pinned to M9.C/D)
- N² pairwise iteration complexity (mitigated: pre-launch scale ~10-50 Vulnerabilities/scan; cwe_id pre-filter optimization forward-pinned)
- Union-find transitive clustering may over-cluster weak A↔C correlations (mitigated: forward-pin investigation at production-readiness scan corpus)

#### Composition

- **ADR-013** Python sole writer for scan state: M9.B writes Vulnerability.correlation_cluster_id + severity_score + corroborated_count from the api side only (engine emits events; never writes pipeline state)
- **ADR-014** Redis Streams progress: M9.B is downstream of progress emission; correlation + scoring run in the in-api ai-pipeline consumer (no new wire contract)
- **ADR-022** Recon as pre-scan helpers: unaffected; M9.B operates on promoted Vulnerabilities, not recon dispatch
- **ADR-028** Phase-1 execution model: M9.B inherits Phase-1 single-provider semantics; deterministic computation requires no provider routing
- **ADR-029** M9.0 AI pipeline foundation: M9.B extends pipeline.run() composition established at M9.0 C2 (ai_pipeline_consumer dispatching pipeline.run()); ai_api_calls cost-tracking schema preserved-but-unused per Q14
- **ADR-030** M9.A embedding + deduplication: M9.B extends Path C cluster→Vulnerability promotion architectural lock; correlation operates on Vulnerability rows post-M9.A; preserves M9.A operational state

#### Cross-references

- **Specification:** §8.2 Cross-Layer Correlation Algorithm (canonical weights + threshold) + §8.3 Severity Scoring Formula (canonical multipliers) + §13 ADR-029/030/031 (architectural lineage)
- **Implementation plan:** Tasks 9.3 (Cross-layer correlation) + 9.4 (Severity scoring)
- **Design doc:** plans/2026-06-25-m9b-correlation-scoring-design.md (commit `d10be51`; Y-locks + Path B + V-UUE pre-grounding)
- **Implementation plan doc:** plans/2026-06-25-m9b-correlation-scoring-implementation.md (commit `a172fd1`; PY1-PY8 + 4-commit breakdown)
- **Composition:** ADR-013 + ADR-014 + ADR-022 + ADR-028 + ADR-029 + ADR-030
- **CLAUDE.md Gotcha 5:** Cost-tracking mandate (operational from M9.0; ai_api_calls unused at M9.B per Q14)

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

**Invocation enumeration.** As of 2026-05-11, the meta-principle has been invoked in 7 ADRs across the project corpus:

| # | ADR | Repo | Phrasing |
|---|---|---|---|
| 1 | ADR-022 (Recon-as-pre-scan-helpers) | shieldscan-docs SPEC §13 | Canonical |
| 2 | ADR-023 (NativeRunner OutputFile mode) | shieldscan-docs SPEC §13 | Variation: emphasizes threshold-override mechanism explicitly |
| 3 | ADR-024 (RawFinding schema extension) | shieldscan-docs SPEC §13 | Canonical |
| 4 | ADR-025 (Findings-ingest direct DB-write) | shieldscan-api DRIFT-LOG | Canonical |
| 5 | ADR-026 (DockerRunner framework + lazy warm pool) | shieldscan-docs SPEC §13 | Variation: short parenthetical "architectural commitment cost vs alternative cost" |
| 6 | ADR-027 (RawFinding.Metadata field) | shieldscan-docs SPEC §13 | Canonical |
| 7 | ADR-008 (MobSF ephemeral-default addendum) | shieldscan-docs SPEC §13 | Canonical (bounded ephemeral cost vs unbounded cleanup-contract-uncertainty risk; Task 7.4 + Task 7.5c V4 precedent; engine commits c15a60d + bfccef8) |

5 of 7 invocations use canonical phrasing; ADR-023 + ADR-026 use semantically-equivalent variations. Variations are preserved as-written in their respective ADRs; future ADRs invoking the meta-principle should use canonical phrasing OR cross-reference §14.1 directly.

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
