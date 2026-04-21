# ShieldScan — Critical 5 Feature Addendum

**Version:** 1.0
**Date:** 2026-04-20
**Status:** Authoritative — extends SPECIFICATION.md and IMPLEMENTATION-PLAN.md
**Timeline Impact:** +3 weeks (10 → 13 weeks to launch)

> **For Claude Code:** This document adds 5 critical launch features plus 4 schema changes that preserve future moat-building options. Read this AFTER the base 8 docs. When conflicts arise, this document wins for anything it specifies; the original docs win for everything else.

---

## Table of Contents

1. [Summary of Changes](#1-summary-of-changes)
2. [Feature 1 — Free Public Security Scan](#2-feature-1--free-public-security-scan)
3. [Feature 2 — Peer Benchmarking](#3-feature-2--peer-benchmarking)
4. [Feature 3 — Scheduled Scans + Email Digest](#4-feature-3--scheduled-scans--email-digest)
5. [Feature 4 — 2-Minute Onboarding Flow](#5-feature-4--2-minute-onboarding-flow)
6. [Feature 5 — Verified Badge System](#6-feature-5--verified-badge-system)
7. [Moat-Preparation Schema Changes](#7-moat-preparation-schema-changes)
8. [Updated Execution Timeline](#8-updated-execution-timeline)
9. [Cross-References to Updated Docs](#9-cross-references-to-updated-docs)

---

## 1. Summary of Changes

### 1.1 New Bounded Contexts

Adds 3 new bounded contexts to the original 7 in SPECIFICATION.md §4:

| # | Context | Purpose |
|---|---|---|
| 8 | **Public Scanning** | Unauthenticated scan endpoint, rate-limited, top-of-funnel |
| 9 | **Scheduling** | Automated recurring scans, email digests |
| 10 | **Trust Signals** | Badges, verification pages, public trust infrastructure |

Benchmarking and Onboarding are enhancements to existing contexts (AI Analysis and Identity, respectively).

### 1.2 New Database Tables (7 additions)

| Table | Purpose |
|---|---|
| `public_scans` | Unauthenticated scan records with IP-based rate limiting |
| `scan_schedules` | Recurring scan configurations per project |
| `scan_digests` | Email digest history + unsubscribe tracking |
| `verification_badges` | Signed, expiring trust badges |
| `industry_benchmarks` | Nightly-aggregated per-industry/country/size averages |
| `onboarding_progress` | Per-user onboarding funnel state |
| `aggregate_findings` | Anonymized finding rollups for threat intel (moat prep) |

### 1.3 Schema Modifications to Existing Tables (4)

| Table | Column Added | Purpose |
|---|---|---|
| `organizations` | `industry`, `country`, `company_size` | Benchmarking + moat prep |
| `projects` | `scan_schedule_id` (FK) | Link to schedule |
| `scans` | `is_public`, `public_slug` | Public scan marker |
| `compliance_frameworks` | MENA entries added | Saudi PDPL, Egypt DPL, UAE DPL |

### 1.4 New API Endpoints (14 additions)

```
Public (no auth):
POST   /v1/public/check                     — trigger free public scan
GET    /v1/public/check/{slug}              — view public scan results
GET    /v1/public/badge/{token}             — verify + render badge
GET    /v1/public/badge/{token}/embed.svg   — SVG badge for embed

Onboarding:
GET    /v1/onboarding/progress              — fetch user progress
POST   /v1/onboarding/progress              — update progress
POST   /v1/onboarding/demo-scan             — trigger demo scan

Schedules:
GET    /v1/orgs/{org_id}/projects/{pid}/schedule
POST   /v1/orgs/{org_id}/projects/{pid}/schedule
PATCH  /v1/orgs/{org_id}/projects/{pid}/schedule
DELETE /v1/orgs/{org_id}/projects/{pid}/schedule

Badges:
GET    /v1/orgs/{org_id}/projects/{pid}/badges
POST   /v1/orgs/{org_id}/projects/{pid}/badges/issue

Benchmarks:
GET    /v1/orgs/{org_id}/scans/{scan_id}/benchmark
```

### 1.5 New Implementation Milestones (5 additions)

| Milestone | Duration | Position |
|---|---|---|
| **M4.5** — Scheduling Infrastructure | 3 days | After M4 |
| **M9.5** — Peer Benchmarking | 2 days | After M9 |
| **M10.5** — Public Scan + Badge System | 4 days | After M10 |
| **M11.13** — Onboarding Flow | 3 days | Inside M11 |
| **M12.5** — Email Digest System | 2 days | After M12 |

**Total additional time:** 14 working days = ~3 weeks.

---

## 2. Feature 1 — Free Public Security Scan

### 2.1 User Story

> Anyone on the internet can visit `shieldscan.io/check/example.com` and get a lightweight security report within 60 seconds, without creating an account. Results are shareable via short URL and include a prompt to sign up for the full scan.

### 2.2 Goals

- **G1 (acquisition):** Primary top-of-funnel hook. SEO-friendly, shareable, embeddable.
- **G4 (compounding):** Every public check contributes anonymized baseline data.

### 2.3 What The Scan Covers

Lightweight, passive-only checks (< 60 seconds total):

| Check | Tool | Data Collected |
|---|---|---|
| SSL/TLS quality | SSLyze (fast mode) | Protocol versions, cipher strength, cert validity |
| Security headers | httpx + custom checks | HSTS, CSP, X-Frame-Options, Referrer-Policy |
| Subdomain exposure | Subfinder (passive only, 30s cap) | Count of discovered subdomains |
| DNS configuration | Built-in | SPF, DMARC, DNSSEC presence |
| Technology fingerprint | httpx tech-detect | Server, framework, known-vulnerable versions |
| Cookie security | httpx | Secure, HttpOnly, SameSite flags |

**NOT included in free scan (reserved for paid):**
- Active fuzzing (Nuclei, Wapiti, ZAP)
- Deep subdomain scanning (>30s Subfinder)
- SAST / secret scanning (requires source code)
- Mobile analysis (requires upload)
- Authenticated testing

### 2.4 Data Model

```sql
CREATE TABLE public_scans (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    slug VARCHAR(20) UNIQUE NOT NULL,  -- short URL ID, e.g., "X7K2mQ"
    domain VARCHAR(255) NOT NULL,
    normalized_domain VARCHAR(255) NOT NULL,  -- lowercase, no protocol
    ip_address INET NOT NULL,           -- for rate limiting
    user_agent TEXT,
    referrer TEXT,

    status VARCHAR(20) NOT NULL DEFAULT 'queued',  -- queued|running|completed|failed
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    duration_ms INTEGER,

    -- Results
    grade CHAR(1),                       -- A|B|C|D|F
    score INTEGER,                       -- 0-100
    findings JSONB,                      -- top 5 findings only
    raw_results JSONB,                   -- full tool output
    tech_stack JSONB,                    -- detected technologies
    subdomain_count INTEGER,

    -- Conversion tracking
    viewed_count INTEGER DEFAULT 0,
    shared_count INTEGER DEFAULT 0,
    converted_to_signup BOOLEAN DEFAULT FALSE,
    converted_org_id UUID REFERENCES organizations(id),

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE DEFAULT (NOW() + INTERVAL '30 days')
);

CREATE INDEX idx_public_scans_slug ON public_scans(slug);
CREATE INDEX idx_public_scans_domain ON public_scans(normalized_domain, created_at DESC);
CREATE INDEX idx_public_scans_ip_created ON public_scans(ip_address, created_at DESC);

CREATE TABLE public_scan_rate_limits (
    ip_address INET PRIMARY KEY,
    hourly_count INTEGER DEFAULT 0,
    daily_count INTEGER DEFAULT 0,
    hourly_reset_at TIMESTAMP WITH TIME ZONE,
    daily_reset_at TIMESTAMP WITH TIME ZONE,
    blocked_until TIMESTAMP WITH TIME ZONE,
    block_reason VARCHAR(100)
);
```

### 2.5 Rate Limiting Rules

| Tier | Limit | Enforcement |
|---|---|---|
| Per IP / hour | 5 scans | Redis sliding window |
| Per IP / day | 20 scans | Redis + DB |
| Per domain / day | 3 scans | DB lookup (prevents abuse) |
| Global / second | 50 scans | Redis counter |
| Suspicious patterns | Auto-block 24h | Rate of >10/min = bot |

**Bypass for logged-in users:** Authenticated Business+ tier users get 100/hour, no domain limit.

### 2.6 Abuse Prevention

1. **hCaptcha** required after 3 scans/hour from same IP
2. **No scanning of IP ranges** — must be a resolvable domain
3. **Blocklist of known scanner-evasion domains** (competitor honeypots)
4. **Ownership disclaimer** on the public form: "You confirm you are authorized to scan this domain"
5. **Scan results redacted** for findings that could enable attack (e.g., exact version numbers hidden behind "sign up to see full details")

### 2.7 API Specification

**POST `/v1/public/check`**

```json
Request:
{
  "domain": "example.com",
  "captcha_token": "optional-if-above-threshold",
  "email": "optional-for-notification"
}

Response (202 Accepted):
{
  "slug": "X7K2mQ",
  "status": "queued",
  "estimated_seconds": 45,
  "result_url": "https://shieldscan.io/check/X7K2mQ",
  "share_url": "https://shieldscan.io/check/X7K2mQ"
}
```

**GET `/v1/public/check/{slug}`**

```json
Response:
{
  "slug": "X7K2mQ",
  "domain": "example.com",
  "status": "completed",
  "grade": "B",
  "score": 72,
  "summary": "Good SSL/TLS. Missing security headers. 2 subdomains discovered.",
  "findings": [
    {
      "severity": "medium",
      "title": "Missing Content-Security-Policy header",
      "description": "Your site doesn't set a CSP...",
      "cwe": "CWE-693"
    }
    // ... up to 5 top findings
  ],
  "tech_stack": ["nginx", "React", "Cloudflare"],
  "subdomain_count": 2,
  "peer_comparison": {
    "industry": "technology",
    "percentile": 65,
    "message": "Better than 65% of tech companies scanned this month"
  },
  "signup_prompt": {
    "message": "Sign up for a free account to see 7 more findings and get monthly scans",
    "cta_url": "https://app.shieldscan.io/signup?from_check=X7K2mQ"
  }
}
```

### 2.8 Implementation Tasks

#### Task M10.5.1 — Public scan endpoint + rate limiting

**Files:**
- Create: `src/app/routes/public_scans.py`
- Create: `src/app/services/public_scan.py`
- Create: `src/app/services/rate_limit.py`
- Test: `tests/routes/test_public_scans.py`

**Step 1: Failing tests**

```python
async def test_public_scan_rate_limit_per_hour(client, mock_redis):
    for i in range(5):
        r = await client.post("/v1/public/check", json={"domain": f"test{i}.com"})
        assert r.status_code == 202
    # 6th request from same IP should fail
    r = await client.post("/v1/public/check", json={"domain": "test6.com"})
    assert r.status_code == 429
    assert "hourly limit" in r.json()["error"]["message"].lower()

async def test_public_scan_rejects_ip_addresses(client):
    r = await client.post("/v1/public/check", json={"domain": "192.168.1.1"})
    assert r.status_code == 400

async def test_public_scan_normalizes_domain(client):
    r = await client.post("/v1/public/check", json={"domain": "HTTPS://Example.COM/"})
    assert r.status_code == 202
    scan = await fetch_scan(r.json()["slug"])
    assert scan.normalized_domain == "example.com"

async def test_public_scan_returns_partial_results_during_run(client):
    r = await client.post("/v1/public/check", json={"domain": "example.com"})
    slug = r.json()["slug"]
    # Immediate check should show running state
    r2 = await client.get(f"/v1/public/check/{slug}")
    assert r2.json()["status"] in ["queued", "running", "completed"]

async def test_public_scan_redacts_exact_versions_in_findings(client, completed_public_scan):
    r = await client.get(f"/v1/public/check/{completed_public_scan.slug}")
    findings_json = json.dumps(r.json()["findings"])
    # Exact version numbers should be masked for anonymous users
    assert "1.21.6" not in findings_json  # nginx version
    assert "version number hidden" in findings_json.lower() or "sign up" in findings_json.lower()
```

**Step 2: Implementation**

```python
# src/app/services/rate_limit.py
import ipaddress
from datetime import timedelta
from redis.asyncio import Redis

class PublicScanRateLimiter:
    def __init__(self, redis: Redis):
        self.redis = redis

    async def check_and_increment(self, ip: str, domain: str) -> dict:
        """Returns {allowed: bool, reason: str, retry_after: int}"""
        hour_key = f"pubscan:hour:{ip}"
        day_key = f"pubscan:day:{ip}"
        domain_key = f"pubscan:domain:{domain}:{ip}"

        hour_count = await self.redis.incr(hour_key)
        if hour_count == 1:
            await self.redis.expire(hour_key, 3600)
        if hour_count > 5:
            ttl = await self.redis.ttl(hour_key)
            return {"allowed": False, "reason": "hourly limit (5/hour)", "retry_after": ttl}

        day_count = await self.redis.incr(day_key)
        if day_count == 1:
            await self.redis.expire(day_key, 86400)
        if day_count > 20:
            ttl = await self.redis.ttl(day_key)
            return {"allowed": False, "reason": "daily limit (20/day)", "retry_after": ttl}

        domain_count = await self.redis.incr(domain_key)
        if domain_count == 1:
            await self.redis.expire(domain_key, 86400)
        if domain_count > 3:
            return {"allowed": False, "reason": "domain scanned too recently", "retry_after": 3600}

        return {"allowed": True}

# src/app/routes/public_scans.py
from fastapi import APIRouter, Request, HTTPException
from app.services.public_scan import PublicScanService
from app.services.rate_limit import PublicScanRateLimiter

router = APIRouter(prefix="/public", tags=["public"])

@router.post("/check", status_code=202)
async def create_public_scan(
    request: Request,
    req: PublicCheckRequest,
    svc: PublicScanService = Depends(get_public_scan_service),
    limiter: PublicScanRateLimiter = Depends(get_rate_limiter),
):
    # Get real IP (respects X-Forwarded-For from Cloudflare)
    client_ip = request.headers.get("CF-Connecting-IP") or request.client.host

    # Validate domain
    normalized = normalize_domain(req.domain)
    if not is_valid_domain(normalized):
        raise HTTPException(400, "Invalid domain. Please provide a resolvable domain name, not an IP address.")

    # Check rate limit
    rate_check = await limiter.check_and_increment(client_ip, normalized)
    if not rate_check["allowed"]:
        raise HTTPException(429, {
            "message": f"Rate limit exceeded: {rate_check['reason']}",
            "retry_after_seconds": rate_check["retry_after"]
        })

    # CAPTCHA required after 3 hourly scans
    if await limiter.needs_captcha(client_ip):
        if not req.captcha_token or not await verify_captcha(req.captcha_token):
            raise HTTPException(403, "CAPTCHA required. Complete verification to continue.")

    # Enqueue scan
    scan = await svc.create_scan(
        domain=normalized,
        ip=client_ip,
        user_agent=request.headers.get("User-Agent"),
        referrer=request.headers.get("Referer"),
        email=req.email,
    )

    return {
        "slug": scan.slug,
        "status": "queued",
        "estimated_seconds": 45,
        "result_url": f"{settings.FRONTEND_URL}/check/{scan.slug}",
        "share_url": f"{settings.FRONTEND_URL}/check/{scan.slug}",
    }
```

**Step 3: Commit**
```bash
git commit -m "feat(public): add free public scan endpoint with rate limiting"
```

#### Task M10.5.2 — Public scan executor (dedicated queue)

**Files:**
- Create: `shieldscan-engine/internal/orchestrator/public_executor.go`
- Test: same file `_test.go`

**Implementation notes:**
- Runs on `shieldscan:queue:public` Redis queue (separate from regular scans)
- Uses fast-path tool config: Subfinder --max-time 30, SSLyze --regular, httpx (headers only)
- Max duration: 60 seconds total — hard cap
- No retries on failure (public scan, users can retry themselves)
- Results published to Redis pub/sub: `shieldscan:public_scan:{slug}`
- On completion, calculates grade using existing scoring logic

**Step 3: Commit**
```bash
git commit -m "feat(engine): add public scan executor with 60s hard cap"
```

#### Task M10.5.3 — Grade calculation

```python
# src/app/services/public_scan.py

SCORE_WEIGHTS = {
    "ssl_critical": -30,     # TLS 1.0/SSL 3.0
    "ssl_high": -15,         # Weak ciphers
    "cert_expired": -25,
    "hsts_missing": -10,
    "csp_missing": -10,
    "xfo_missing": -5,
    "cookie_insecure": -8,
    "dnssec_missing": -5,
    "spf_missing": -5,
    "dmarc_missing": -5,
    "tech_outdated_critical": -20,
    "subdomain_excess": -5,  # >20 subdomains = attack surface
}

def calculate_public_score(findings: list) -> tuple[int, str]:
    """Returns (score 0-100, letter grade A-F)."""
    score = 100
    for finding in findings:
        score += SCORE_WEIGHTS.get(finding["type"], 0)
    score = max(0, min(100, score))

    if score >= 90: return score, "A"
    if score >= 75: return score, "B"
    if score >= 60: return score, "C"
    if score >= 40: return score, "D"
    return score, "F"
```

#### Task M10.5.4 — Public results page + share cards

**Files:**
- Create: `shieldscan-web/src/pages/PublicCheck.tsx`
- Create: `shieldscan-web/src/components/GradeDisplay.tsx`
- Create: `src/app/routes/public_og.py` (Open Graph image generation)

**Features:**
- Beautifully animated grade reveal
- Real-time progress during scan (polls `/v1/public/check/{slug}` every 2s)
- Top 5 findings shown, remaining hidden behind "Sign up to see 7 more"
- Twitter/LinkedIn/WhatsApp share buttons with pre-filled message
- Dynamic OG image generation: `/og/check/{slug}.png` returns 1200x630 branded card
- Embed code: `<iframe src="shieldscan.io/embed/check/{slug}" width="400" height="200">`

#### Task M10.5.5 — Conversion tracking

**Files:**
- Modify: `src/app/routes/auth.py`
- Modify: `src/app/services/public_scan.py`

**Behavior:**
- If user signs up with `from_check=slug` query param, link `public_scans.converted_org_id`
- Dashboard shows: "Welcome! Your free scan found 12 issues. Run a full scan to see all of them."
- Pre-fills project creation with the scanned domain

**Step 3: Commit**
```bash
git commit -m "feat(public): add conversion tracking from public scan to signup"
```

---

## 3. Feature 2 — Peer Benchmarking

### 3.1 User Story

> After each scan, customers see: "Your security posture is in the top 30% of fintechs in Egypt" with a chart showing how their score compares to peers in their industry, country, and company size.

### 3.2 Goals

- **G2 (premium pricing):** Justifies subscription — you're not just giving them a number, you're giving them context
- **G4 (compounding):** Dataset improves with every customer; competitors can't replicate without a customer base

### 3.3 Data Model

```sql
-- Added to organizations table
ALTER TABLE organizations ADD COLUMN industry VARCHAR(50);
-- Values: technology | finance | healthcare | ecommerce | government | education | other
ALTER TABLE organizations ADD COLUMN country CHAR(2);  -- ISO 3166-1 alpha-2
ALTER TABLE organizations ADD COLUMN company_size VARCHAR(20);
-- Values: solo | small (2-10) | medium (11-50) | large (51-200) | enterprise (200+)

CREATE TABLE industry_benchmarks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    industry VARCHAR(50) NOT NULL,
    country CHAR(2),                  -- nullable for global benchmark
    company_size VARCHAR(20),         -- nullable for any-size benchmark

    org_count INTEGER NOT NULL,       -- count of orgs in this bucket
    scan_count INTEGER NOT NULL,      -- total scans aggregated

    -- Score statistics
    avg_score FLOAT NOT NULL,
    median_score FLOAT NOT NULL,
    p25_score FLOAT NOT NULL,         -- 25th percentile
    p75_score FLOAT NOT NULL,
    p90_score FLOAT NOT NULL,
    stddev_score FLOAT NOT NULL,

    -- Common findings
    top_cwes JSONB,                   -- [{cwe: "CWE-79", percent: 45}, ...]
    critical_rate FLOAT,              -- % of orgs with >0 critical findings
    high_rate FLOAT,

    computed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    valid_until TIMESTAMP WITH TIME ZONE DEFAULT (NOW() + INTERVAL '24 hours'),

    UNIQUE(industry, country, company_size)
);

CREATE INDEX idx_benchmarks_lookup ON industry_benchmarks(industry, country, company_size);
```

### 3.4 Privacy Rules

**Minimum bucket size: 10 orgs.** If fewer than 10 orgs match a specific (industry, country, size) triple, fall back progressively:

```
Tier 1: (industry, country, size)  — need 10 orgs
Tier 2: (industry, country, *)      — need 10 orgs
Tier 3: (industry, *, *)            — need 10 orgs
Tier 4: (*, *, *)                   — global baseline
```

This prevents customer identification in small buckets (e.g., "there are only 2 Egyptian banks using ShieldScan, so the 'Egyptian finance' bucket would leak info").

**Aggregate data only.** Benchmarks never include specific findings from specific customers. Only statistical rollups.

### 3.5 Computation

**Nightly cron job:**
- Runs at 3am UTC in a dedicated worker
- Reads all scans completed in last 90 days
- Buckets by (industry, country, size)
- Computes statistics per bucket
- Writes to `industry_benchmarks` table with 24-hour TTL
- Previous buckets with <10 orgs are deleted

### 3.6 API Specification

**GET `/v1/orgs/{org_id}/scans/{scan_id}/benchmark`**

```json
Response:
{
  "your_score": 72,
  "your_grade": "B",
  "bucket": {
    "industry": "finance",
    "country": "EG",
    "company_size": "medium",
    "org_count": 23
  },
  "benchmark": {
    "avg_score": 58,
    "median_score": 61,
    "p25": 45,
    "p75": 75,
    "p90": 88
  },
  "your_percentile": 72,
  "message": "Your security posture is better than 72% of medium-sized finance companies in Egypt",
  "top_peer_issues": [
    {"cwe": "CWE-79", "name": "Cross-Site Scripting", "percent_affected": 45},
    {"cwe": "CWE-200", "name": "Information Disclosure", "percent_affected": 38},
    {"cwe": "CWE-352", "name": "CSRF", "percent_affected": 30}
  ],
  "improvements_vs_peers": [
    "Better than peers on SSL/TLS configuration",
    "Below peer average on dependency management"
  ]
}
```

### 3.7 Implementation Tasks

#### Task M9.5.1 — Benchmarking aggregation job

**Files:**
- Create: `src/app/services/benchmarking/aggregator.py`
- Create: `scripts/compute_benchmarks.py`
- Test: `tests/services/benchmarking/test_aggregator.py`

**Step 1: Failing tests**

```python
@pytest.mark.asyncio
async def test_aggregator_buckets_by_industry_country_size(db_session):
    # Create 10 orgs in (finance, EG, medium) with scans
    for i in range(10):
        org = await create_org(industry="finance", country="EG", company_size="medium")
        await create_scan(org_id=org.id, score=70 + i)

    aggregator = BenchmarkAggregator()
    await aggregator.compute_all(db_session)

    bucket = await fetch_benchmark(db_session, "finance", "EG", "medium")
    assert bucket is not None
    assert bucket.org_count == 10
    assert 69 <= bucket.avg_score <= 80

@pytest.mark.asyncio
async def test_aggregator_skips_small_buckets(db_session):
    # Only 3 orgs — below threshold
    for i in range(3):
        org = await create_org(industry="healthcare", country="SA", company_size="small")
        await create_scan(org_id=org.id, score=50)

    aggregator = BenchmarkAggregator()
    await aggregator.compute_all(db_session)

    bucket = await fetch_benchmark(db_session, "healthcare", "SA", "small")
    assert bucket is None  # Not enough orgs to compute

@pytest.mark.asyncio
async def test_aggregator_computes_percentiles_correctly(db_session):
    scores = [10, 20, 30, 40, 50, 60, 70, 80, 90, 100]
    for score in scores:
        org = await create_org(industry="technology", country="EG", company_size="medium")
        await create_scan(org_id=org.id, score=score)

    aggregator = BenchmarkAggregator()
    await aggregator.compute_all(db_session)

    bucket = await fetch_benchmark(db_session, "technology", "EG", "medium")
    assert bucket.median_score == 55  # midpoint
    assert bucket.p25_score == 32.5
    assert bucket.p75_score == 77.5
    assert bucket.p90_score == 91.0
```

**Step 2: Implementation**

```python
# src/app/services/benchmarking/aggregator.py
from datetime import datetime, timedelta
import statistics
from sqlalchemy import select, func, text
from app.models.organizations import Organization
from app.models.scans import Scan
from app.models.benchmarks import IndustryBenchmark

MINIMUM_BUCKET_SIZE = 10
LOOKBACK_DAYS = 90

class BenchmarkAggregator:
    async def compute_all(self, db):
        # Find all unique (industry, country, size) tuples
        tuples = await db.execute(text("""
            SELECT industry, country, company_size, COUNT(DISTINCT o.id) as org_count
            FROM organizations o
            INNER JOIN scans s ON s.organization_id = o.id
            WHERE s.completed_at > NOW() - INTERVAL ':days days'
              AND o.industry IS NOT NULL
              AND o.country IS NOT NULL
              AND o.company_size IS NOT NULL
            GROUP BY industry, country, company_size
            HAVING COUNT(DISTINCT o.id) >= :min_size
        """), {"days": LOOKBACK_DAYS, "min_size": MINIMUM_BUCKET_SIZE})

        for row in tuples:
            await self._compute_bucket(db, row.industry, row.country, row.company_size)

        # Also compute fallback buckets (no country, no size)
        await self._compute_fallback_buckets(db)

    async def _compute_bucket(self, db, industry, country, size):
        # Get latest scan per org in bucket
        latest_scans = await db.execute(select(Scan.score).where(
            Scan.organization_id.in_(
                select(Organization.id).where(
                    Organization.industry == industry,
                    Organization.country == country,
                    Organization.company_size == size,
                )
            ),
            Scan.status == "completed",
            Scan.completed_at > datetime.utcnow() - timedelta(days=LOOKBACK_DAYS),
        ))
        scores = sorted([s.score for s in latest_scans if s.score is not None])

        if len(scores) < MINIMUM_BUCKET_SIZE:
            return

        # Compute statistics
        bench = IndustryBenchmark(
            industry=industry,
            country=country,
            company_size=size,
            org_count=len(scores),
            scan_count=len(scores),
            avg_score=statistics.mean(scores),
            median_score=statistics.median(scores),
            p25_score=self._percentile(scores, 25),
            p75_score=self._percentile(scores, 75),
            p90_score=self._percentile(scores, 90),
            stddev_score=statistics.stdev(scores) if len(scores) > 1 else 0,
            top_cwes=await self._compute_top_cwes(db, industry, country, size),
            critical_rate=await self._compute_severity_rate(db, "critical", industry, country, size),
            high_rate=await self._compute_severity_rate(db, "high", industry, country, size),
        )
        await db.merge(bench)  # upsert

    @staticmethod
    def _percentile(sorted_scores, p):
        k = (len(sorted_scores) - 1) * p / 100
        f = int(k)
        c = f + 1 if f + 1 < len(sorted_scores) else f
        return sorted_scores[f] + (k - f) * (sorted_scores[c] - sorted_scores[f])
```

**Step 3: Commit**
```bash
git commit -m "feat(benchmarking): add nightly aggregation job with privacy safeguards"
```

#### Task M9.5.2 — Benchmark lookup API

**Files:**
- Create: `src/app/services/benchmarking/lookup.py`
- Create: `src/app/routes/benchmarks.py`

**Implementation:** Progressive fallback logic — try specific bucket, then relax constraints.

```python
async def get_benchmark_for_scan(db, scan: Scan) -> dict:
    org = await db.get(Organization, scan.organization_id)

    # Tier 1: exact match
    bench = await _lookup(db, org.industry, org.country, org.company_size)
    if bench: return _format_benchmark(scan, bench, tier=1)

    # Tier 2: drop size
    bench = await _lookup(db, org.industry, org.country, None)
    if bench: return _format_benchmark(scan, bench, tier=2)

    # Tier 3: drop country
    bench = await _lookup(db, org.industry, None, None)
    if bench: return _format_benchmark(scan, bench, tier=3)

    # Tier 4: global
    bench = await _lookup(db, None, None, None)
    return _format_benchmark(scan, bench, tier=4)
```

**Step 3: Commit**
```bash
git commit -m "feat(benchmarking): add benchmark lookup API with progressive fallback"
```

#### Task M9.5.3 — Benchmark UI component

**Files:**
- Create: `shieldscan-web/src/components/BenchmarkCard.tsx`

**Includes:**
- Horizontal bar showing your score on the distribution
- Percentile message in customer's language (Arabic available)
- Top peer issues you've already fixed (positive reinforcement)
- Top peer issues you haven't fixed yet (upsell opportunity)

---

## 4. Feature 3 — Scheduled Scans + Email Digest

### 4.1 User Story

> Customers configure a project to scan weekly or monthly. Results are automatically scanned, and a digest email is sent with diffs from the previous scan. One click from the email opens the full scan report.

### 4.2 Goals

- **G1:** Turns one-time scanners into recurring users without manual action
- **G2:** Creates habit/stickiness — reason not to cancel

### 4.3 Data Model

```sql
CREATE TABLE scan_schedules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,

    frequency VARCHAR(20) NOT NULL,       -- daily | weekly | monthly
    day_of_week INTEGER,                  -- 0-6 (for weekly)
    day_of_month INTEGER,                 -- 1-28 (for monthly)
    hour_of_day INTEGER NOT NULL,         -- 0-23 (in timezone)
    timezone VARCHAR(50) NOT NULL DEFAULT 'Africa/Cairo',

    scan_type VARCHAR(30) NOT NULL,       -- which scan type to run
    recipients JSONB NOT NULL,            -- [{email, digest_format}]

    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_run_at TIMESTAMP WITH TIME ZONE,
    next_run_at TIMESTAMP WITH TIME ZONE NOT NULL,
    consecutive_failures INTEGER DEFAULT 0,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),

    UNIQUE(project_id)  -- one schedule per project
);

CREATE INDEX idx_schedules_next_run ON scan_schedules(next_run_at) WHERE is_active = TRUE;

ALTER TABLE scan_schedules ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON scan_schedules
    USING (organization_id = current_setting('app.current_org_id')::uuid);

CREATE TABLE scan_digests (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    schedule_id UUID REFERENCES scan_schedules(id) ON DELETE SET NULL,
    scan_id UUID NOT NULL REFERENCES scans(id),
    previous_scan_id UUID REFERENCES scans(id),

    sent_to TEXT NOT NULL,                -- email address
    sent_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    opened_at TIMESTAMP WITH TIME ZONE,
    clicked_at TIMESTAMP WITH TIME ZONE,

    -- Content snapshot
    new_critical INTEGER DEFAULT 0,
    new_high INTEGER DEFAULT 0,
    resolved_critical INTEGER DEFAULT 0,
    resolved_high INTEGER DEFAULT 0,

    unsubscribe_token VARCHAR(64) UNIQUE NOT NULL
);

CREATE INDEX idx_digests_scan ON scan_digests(scan_id);
CREATE INDEX idx_digests_unsubscribe ON scan_digests(unsubscribe_token);
```

### 4.4 Scheduling Rules

**Business hours avoidance:**
- Default scan time: 03:00 local timezone (low traffic)
- MENA customers: avoid Friday 12:00-14:00 (Jummah prayer)
- Configurable per project

**Failure handling:**
- 3 consecutive failures → auto-disable schedule, email owner
- Failed scan does not count against scan quota
- Retry next scheduled time, not immediately

### 4.5 API Specification

**POST `/v1/orgs/{org_id}/projects/{pid}/schedule`**

```json
Request:
{
  "frequency": "weekly",
  "day_of_week": 1,           // Monday
  "hour_of_day": 3,
  "timezone": "Africa/Cairo",
  "scan_type": "full_web",
  "recipients": [
    {"email": "ceo@company.com", "digest_format": "executive"},
    {"email": "dev@company.com", "digest_format": "technical"}
  ]
}

Response: 201 Created
{
  "id": "sch_xxx",
  "next_run_at": "2026-04-27T00:00:00Z",
  ...
}
```

### 4.6 Email Digest Template

**Subject:** "ShieldScan Weekly — example.com (2 new issues this week)"

**Content (HTML):**
```
Hi {name},

Here's your weekly security summary for example.com:

🔴 2 new critical issues     (was 0 last week)
🟠 3 new high issues         (was 5 last week) ⬇ improved
🟡 4 medium issues           (was 4 last week)
✅ 7 issues resolved

Top new issues:
- Critical: Cross-Site Scripting on /api/search
- High: Outdated jQuery with known CVE
- High: Missing HSTS header on admin.example.com

Your security posture is in the top 45% of finance companies in Egypt.

[Read full report]

--
Unsubscribe from these emails: {unsubscribe_link}
```

### 4.7 Implementation Tasks

#### Task M4.5.1 — Schedule management endpoints

**Files:**
- Create: `src/app/routes/schedules.py`
- Create: `src/app/services/schedule.py`
- Test: `tests/routes/test_schedules.py`

**Step 1-5:** Standard CRUD with TDD. Commit.

#### Task M4.5.2 — Scheduler worker (Go)

**Files:**
- Create: `shieldscan-engine/internal/scheduler/scheduler.go`
- Test: same file `_test.go`

**Implementation:**
- Runs on one worker (leader election via Redis)
- Checks `next_run_at <= NOW()` every 60 seconds
- Dispatches to regular scan queue
- Updates `last_run_at` and computes new `next_run_at`

```go
func (s *Scheduler) Tick(ctx context.Context) error {
    rows, err := s.db.QueryContext(ctx, `
        SELECT id, organization_id, project_id, scan_type, timezone,
               frequency, day_of_week, day_of_month, hour_of_day
        FROM scan_schedules
        WHERE is_active = TRUE
          AND next_run_at <= NOW()
        FOR UPDATE SKIP LOCKED
    `)
    if err != nil { return err }
    defer rows.Close()

    for rows.Next() {
        var sched Schedule
        rows.Scan(&sched.ID, &sched.OrgID, ...)

        // Dispatch scan
        err := s.scanQueue.Dispatch(ctx, CreateScanJob{
            OrgID: sched.OrgID,
            ProjectID: sched.ProjectID,
            ScanType: sched.ScanType,
            ScheduleID: sched.ID,
        })
        if err != nil {
            s.markScheduleFailure(sched.ID)
            continue
        }

        // Compute next run time
        nextRun := sched.ComputeNextRun(time.Now())
        s.db.ExecContext(ctx, `
            UPDATE scan_schedules
            SET last_run_at = NOW(), next_run_at = $1, consecutive_failures = 0
            WHERE id = $2
        `, nextRun, sched.ID)
    }
    return nil
}
```

#### Task M12.5.1 — Digest email generator

**Files:**
- Create: `src/app/services/digest.py`
- Create: `src/app/templates/emails/digest.html`
- Test: `tests/services/test_digest.py`

**Behavior:**
- After every scan completion, check if scan has a `schedule_id`
- If yes, generate digest comparing to previous scan for same project
- Send via SendGrid with unique unsubscribe token
- Record in `scan_digests` table

**Step 3: Commit**
```bash
git commit -m "feat(digest): add email digest generator after scheduled scans"
```

#### Task M12.5.2 — Email open/click tracking + unsubscribe

**Files:**
- Create: `src/app/routes/email_tracking.py`

**Endpoints:**
- `GET /v1/email/open/{digest_id}.gif` — 1x1 pixel tracker
- `GET /v1/email/click/{digest_id}?url=...` — click redirect
- `GET /v1/email/unsubscribe/{token}` — unsubscribe page
- `POST /v1/email/unsubscribe/{token}` — confirm unsubscribe

**Commit:**
```bash
git commit -m "feat(email): add open/click tracking + unsubscribe"
```

---

## 5. Feature 4 — 2-Minute Onboarding Flow

### 5.1 User Story

> A new user signs up and completes their first scan in under 2 minutes. They see a guided 4-step flow, never touch a complex form, and end with an actual scan result showing real findings.

### 5.2 Goals

- **G1:** Activation rate is the #1 metric for SME SaaS. Without this, >95% of signups go inactive.

### 5.3 Onboarding Steps

**Step 1 — Welcome (15 seconds)**
- "What are you protecting?" → 4 options: Web app / Mobile app / API / "I'll decide later"
- Selection tunes demo scan target

**Step 2 — Demo Scan (60 seconds)**
- Pre-selected vulnerable target (OWASP Juice Shop for web; Damn Insecure Mobile App for mobile)
- Animated progress: recon → scanning → analyzing
- No auth, no config, runs to completion
- Shows "This is a practice scan on a known-vulnerable site"

**Step 3 — Results Reveal (20 seconds)**
- Animated counter: "X vulnerabilities found"
- Top 3 findings highlighted with severity
- Big button: "Now scan your real website"

**Step 4 — First Real Scan (25 seconds)**
- Enter domain (prefilled from public scan if applicable)
- Choose scan type (recommended option auto-selected)
- Skip domain verification for first scan (marked as "preview scan")
- Click "Run scan"
- User lands on scan progress page

**Target:** 2 minutes total from click-of-signup to viewing-real-findings.

### 5.4 Data Model

```sql
CREATE TABLE onboarding_progress (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,

    step_1_welcome_at TIMESTAMP WITH TIME ZONE,
    step_1_choice VARCHAR(20),            -- web | mobile | api | deferred

    step_2_demo_started_at TIMESTAMP WITH TIME ZONE,
    step_2_demo_completed_at TIMESTAMP WITH TIME ZONE,
    step_2_demo_scan_id UUID,

    step_3_results_viewed_at TIMESTAMP WITH TIME ZONE,

    step_4_real_scan_at TIMESTAMP WITH TIME ZONE,
    step_4_real_scan_id UUID,

    completed_at TIMESTAMP WITH TIME ZONE,
    skipped_at TIMESTAMP WITH TIME ZONE,
    total_duration_seconds INTEGER,

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### 5.5 Implementation Tasks

#### Task M11.13.1 — Onboarding backend

**Files:**
- Create: `src/app/routes/onboarding.py`
- Create: `src/app/services/onboarding.py`
- Create: `src/app/services/demo_scan.py`

**Key behavior:**
- Demo scan runs on preset target (configurable in .env)
- Demo scan bypasses domain verification
- Demo scan uses lowest-tier resources (fastest possible)
- Demo scan results marked `is_demo = true` and excluded from benchmarks

#### Task M11.13.2 — Onboarding UI

**Files:**
- Create: `shieldscan-web/src/pages/Onboarding.tsx`
- Create: `shieldscan-web/src/components/onboarding/Step{1,2,3,4}.tsx`

**Design requirements:**
- Full-screen modal (no dashboard chrome)
- Smooth animations between steps
- Skip button always visible (don't trap users)
- Progress indicator: dots showing current step
- Copy in Arabic + English (detect from signup)

#### Task M11.13.3 — Activation analytics

**Files:**
- Create: `src/app/services/analytics.py`

**Events tracked:**
- `onboarding_started`
- `onboarding_step_{1-4}_completed`
- `onboarding_completed`
- `onboarding_skipped_at_step_X`
- `first_real_scan_triggered`
- `time_to_first_scan_seconds`

**Dashboard for founder:** `/admin/activation` shows conversion funnel.

**Step 3: Commit**
```bash
git commit -m "feat(onboarding): add 4-step guided onboarding with demo scan"
```

---

## 6. Feature 5 — Verified Badge System

### 6.1 User Story

> Customers who pass a clean scan (no critical/high findings) can embed a ShieldScan Verified badge on their website. The badge links to a public verification page showing when verification was issued and expires. Badges renew automatically if customer stays clean.

### 6.2 Goals

- **G1:** Every badge on every customer site = marketing ROI
- **G2:** Badges expire → creates lock-in (customer can't leave without losing badge)
- **G4:** Network effect — badges become recognized trust signal in MENA

### 6.3 Badge Lifecycle

```
Scan completed with 0 critical + 0 high findings
          ↓
Badge automatically issued, valid 30 days
          ↓
Customer embeds SVG/iframe on their site
          ↓
Next scheduled scan (or manual)
          ↓
 Still clean? → Badge auto-renewed, 30 more days
 New critical/high? → Badge marked "expired" on verification page
          ↓
If expired >14 days without remediation → badge revoked
```

### 6.4 Fraud Prevention

Every badge token is cryptographically signed:

```python
token = signed("{org_id}|{project_id}|{issued_at}|{expires_at}", key=BADGE_SIGNING_KEY)
```

Attempted tampering = signature validation fails. Token embedded in URL.

**Preventing copy/paste fraud:**
- Every badge page checks current validity (live DB lookup, not cached)
- Expired badges show "NOT CURRENTLY VERIFIED" clearly
- Revoked badges trigger email alert to affected org (in case of legitimate remediation)

### 6.5 Data Model

```sql
CREATE TABLE verification_badges (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    project_id UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    scan_id UUID NOT NULL REFERENCES scans(id),

    token VARCHAR(64) UNIQUE NOT NULL,    -- URL-safe, signed

    status VARCHAR(20) NOT NULL DEFAULT 'active',  -- active | expired | revoked
    issued_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    revoked_at TIMESTAMP WITH TIME ZONE,
    revoke_reason VARCHAR(100),

    -- Badge content
    grade CHAR(1) NOT NULL,               -- A|B (no badges for C or worse)
    display_name VARCHAR(255) NOT NULL,   -- shown on badge

    -- Analytics
    view_count INTEGER DEFAULT 0,
    embed_count INTEGER DEFAULT 0,        -- unique domains embedding

    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_badges_token ON verification_badges(token);
CREATE INDEX idx_badges_project ON verification_badges(project_id);
CREATE INDEX idx_badges_active ON verification_badges(organization_id)
    WHERE status = 'active';

ALTER TABLE verification_badges ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON verification_badges
    USING (organization_id = current_setting('app.current_org_id')::uuid
           OR current_setting('app.is_public_badge_view', true) = 'true');
```

### 6.6 Public Verification Page

**`https://shieldscan.io/verify/{token}`**

Shows:
- Organization name + logo (if provided)
- Grade (A or B)
- Issue date + expiration date
- "Currently valid" ✓ or "Expired" ✗
- Link to company website
- "This badge was issued based on a security scan completed on {date}. It does not guarantee absolute security. Expires on {date}."

### 6.7 Embed Code

Given to customer as copy-paste:

```html
<!-- Shieldscan Verified badge -->
<a href="https://shieldscan.io/verify/abc123xyz" target="_blank" rel="noopener">
  <img
    src="https://shieldscan.io/badge/abc123xyz.svg"
    alt="ShieldScan Verified"
    width="150"
    height="50"
  />
</a>
```

SVG is dynamically generated — if badge becomes expired/revoked, SVG visually shows "expired" state.

### 6.8 API Specification

**POST `/v1/orgs/{org_id}/projects/{pid}/badges/issue`**

```json
Response (201):
{
  "id": "bdg_xxx",
  "token": "abc123xyz",
  "status": "active",
  "grade": "A",
  "issued_at": "...",
  "expires_at": "...",
  "verify_url": "https://shieldscan.io/verify/abc123xyz",
  "embed_code": "<a href=\"...",
  "badge_svg_url": "https://shieldscan.io/badge/abc123xyz.svg"
}
```

Requires:
- Latest scan score: grade A or B
- Zero critical + zero high findings
- Domain verified
- Project not archived

### 6.9 Implementation Tasks

#### Task M10.5.6 — Badge issuance service

**Files:**
- Create: `src/app/services/badges.py`
- Test: `tests/services/test_badges.py`

**Step 1: Failing tests**

```python
async def test_badge_issued_only_if_clean(db, scan_with_critical):
    svc = BadgeService(db)
    with pytest.raises(BadgeNotEligibleError):
        await svc.issue(scan_with_critical.id)

async def test_badge_auto_renews_if_still_clean(db, active_badge, new_clean_scan):
    svc = BadgeService(db)
    renewed = await svc.check_renewal(active_badge.project_id, new_clean_scan.id)
    assert renewed.expires_at > active_badge.expires_at

async def test_badge_expires_on_new_critical_finding(db, active_badge, scan_with_critical):
    svc = BadgeService(db)
    await svc.check_renewal(active_badge.project_id, scan_with_critical.id)
    refreshed = await db.get(VerificationBadge, active_badge.id)
    assert refreshed.status == "expired"

async def test_badge_token_is_cryptographically_signed(db, active_badge):
    # Tampering with token should fail validation
    tampered = active_badge.token[:-5] + "XXXXX"
    with pytest.raises(InvalidBadgeTokenError):
        validate_badge_token(tampered)
```

**Step 2: Implementation**

```python
# src/app/services/badges.py
import hmac, hashlib, base64
from datetime import datetime, timedelta
from app.config import settings
from app.models.badges import VerificationBadge

class BadgeService:
    BADGE_VALIDITY_DAYS = 30

    async def issue(self, scan_id: UUID) -> VerificationBadge:
        scan = await self.db.get(Scan, scan_id)
        if not scan or scan.status != "completed":
            raise BadgeNotEligibleError("Scan not completed")

        # Check eligibility
        critical_count = await self._count_findings(scan.id, "critical")
        high_count = await self._count_findings(scan.id, "high")

        if critical_count > 0 or high_count > 0:
            raise BadgeNotEligibleError(f"Scan has {critical_count} critical, {high_count} high findings")

        grade = "A" if scan.score >= 90 else "B" if scan.score >= 75 else None
        if grade is None:
            raise BadgeNotEligibleError(f"Score {scan.score} doesn't qualify for badge")

        # Check domain verified
        project = await self.db.get(Project, scan.project_id)
        if not project.domain_verified:
            raise BadgeNotEligibleError("Domain must be verified before badge issuance")

        # Generate signed token
        org = await self.db.get(Organization, scan.organization_id)
        issued_at = datetime.utcnow()
        expires_at = issued_at + timedelta(days=self.BADGE_VALIDITY_DAYS)
        token = self._generate_token(scan.organization_id, scan.project_id, issued_at, expires_at)

        badge = VerificationBadge(
            organization_id=scan.organization_id,
            project_id=scan.project_id,
            scan_id=scan.id,
            token=token,
            status="active",
            issued_at=issued_at,
            expires_at=expires_at,
            grade=grade,
            display_name=org.name,
        )
        self.db.add(badge)
        await self.db.commit()
        return badge

    def _generate_token(self, org_id, project_id, issued_at, expires_at) -> str:
        payload = f"{org_id}|{project_id}|{issued_at.isoformat()}|{expires_at.isoformat()}"
        signature = hmac.new(
            settings.BADGE_SIGNING_KEY.encode(),
            payload.encode(),
            hashlib.sha256
        ).digest()
        return base64.urlsafe_b64encode(signature[:18]).decode().rstrip("=")

    async def check_renewal(self, project_id: UUID, new_scan_id: UUID):
        """Called after each completed scan. Renews or expires existing badge."""
        existing = await self.db.execute(select(VerificationBadge).where(
            VerificationBadge.project_id == project_id,
            VerificationBadge.status == "active",
        ))
        badge = existing.scalar_one_or_none()
        if not badge:
            # No active badge — try to issue a new one
            try:
                return await self.issue(new_scan_id)
            except BadgeNotEligibleError:
                return None

        # Existing badge — check if still clean
        scan = await self.db.get(Scan, new_scan_id)
        crit = await self._count_findings(scan.id, "critical")
        high = await self._count_findings(scan.id, "high")

        if crit > 0 or high > 0:
            # Expire badge
            badge.status = "expired"
            badge.revoke_reason = f"{crit} critical / {high} high findings detected"
            await self.db.commit()
            await self._notify_badge_expired(badge)
            return badge

        # Renew badge
        badge.expires_at = datetime.utcnow() + timedelta(days=self.BADGE_VALIDITY_DAYS)
        await self.db.commit()
        return badge
```

**Step 3: Commit**
```bash
git commit -m "feat(badges): add verification badge issuance + renewal service"
```

#### Task M10.5.7 — Public badge verification page + SVG

**Files:**
- Create: `src/app/routes/public_badges.py`
- Create: `shieldscan-web/src/pages/BadgeVerify.tsx`

**Endpoints:**
- `GET /v1/public/badge/{token}` — JSON or HTML verification page
- `GET /v1/public/badge/{token}/embed.svg` — dynamically generated SVG

**SVG generation:**
```python
def generate_badge_svg(badge: VerificationBadge) -> str:
    color = "#10B981" if badge.status == "active" else "#EF4444"
    status_text = "Verified" if badge.status == "active" else "Expired"

    return f"""<svg xmlns="..." width="150" height="50" viewBox="0 0 150 50">
        <rect width="150" height="50" fill="{color}" rx="6"/>
        <text x="12" y="20" fill="white" font-family="sans-serif" font-size="12" font-weight="600">
            ShieldScan
        </text>
        <text x="12" y="38" fill="white" font-family="sans-serif" font-size="14" font-weight="700">
            {status_text} {badge.grade}
        </text>
        <text x="148" y="44" fill="white" font-family="sans-serif" font-size="9" text-anchor="end" opacity="0.8">
            {badge.expires_at.strftime("%b %Y")}
        </text>
    </svg>"""
```

**Commit:**
```bash
git commit -m "feat(badges): add public badge verification page + dynamic SVG"
```

---

## 7. Moat-Preparation Schema Changes

**These are additions to preserve the option of building data-moat features later.** Don't build the features yet — but add the schema so future migration is painless.

### 7.1 Organization Metadata for Benchmarking

```sql
-- Add to organizations table
ALTER TABLE organizations ADD COLUMN industry VARCHAR(50);
ALTER TABLE organizations ADD COLUMN country CHAR(2);
ALTER TABLE organizations ADD COLUMN company_size VARCHAR(20);

-- Require during registration (see onboarding)
```

**Collected during signup:** "What industry are you in?" + country (auto-detected from IP, confirmed) + "How many people in your team?"

### 7.2 Anonymized Finding Aggregation

```sql
-- New table for threat intel data (moat feature to be built Month 3+)
CREATE TABLE aggregate_findings (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    period_start DATE NOT NULL,           -- weekly rollup
    period_end DATE NOT NULL,

    industry VARCHAR(50),
    country CHAR(2),
    company_size VARCHAR(20),

    cwe_id VARCHAR(20) NOT NULL,
    finding_type VARCHAR(100) NOT NULL,
    severity VARCHAR(20) NOT NULL,

    org_count INTEGER NOT NULL,           -- orgs with this finding
    instance_count INTEGER NOT NULL,      -- total instances across orgs

    computed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(period_start, period_end, industry, country, company_size, cwe_id)
);

-- No PII, no org_id — pure aggregation
-- Populated by nightly cron alongside benchmarks
-- Privacy rule: minimum 10 orgs per row
```

This enables future **MENA Security Report** (quarterly publication) without schema migration.

### 7.3 MENA Regulatory Framework Mapping

```sql
-- Add rows to existing compliance_frameworks table
INSERT INTO compliance_frameworks (code, name, region, description) VALUES
  ('egypt_dpl_2020', 'Egypt Data Protection Law (Law 151 of 2020)', 'MENA', '...'),
  ('saudi_pdpl_2023', 'Saudi Personal Data Protection Law', 'MENA', '...'),
  ('uae_dpl_2021', 'UAE Federal Data Protection Law', 'MENA', '...');

-- Add CWE → control mappings for MENA frameworks
-- Populated via data migration
```

### 7.4 Certificate Eligibility Tracking

```sql
-- Add to organizations table
ALTER TABLE organizations ADD COLUMN certificate_eligible_since TIMESTAMP WITH TIME ZONE;
ALTER TABLE organizations ADD COLUMN continuous_clean_days INTEGER DEFAULT 0;

-- Updated by post-scan hook:
--   If scan clean → increment continuous_clean_days
--   If scan has critical/high → reset to 0
--   If continuous_clean_days >= 365 → eligible for "ShieldScan Verified Secure" annual certificate
```

Foundation for annual certificate program (Month 6+).

---

## 8. Updated Execution Timeline

### 8.1 Milestone Sequencing

```
Original:
Week 1-2:  M1, M2, M3 (API foundation)
Week 2-3:  M4 (orchestration)
Week 3:    M5 (Go worker)
Week 3-4:  M6 (native runners)
Week 4-5:  M7 (Docker runners)
Week 5:    M8 (recon pipeline)
Week 5-6:  M9 (AI pipeline)
Week 6:    M10 (vuln APIs)
Week 6-8:  M11 (React dashboard)
Week 8-9:  M12 (billing)
Week 9:    M13 (CLI)
Week 10:   M14, M15 (agent + launch)

With Critical 5:
Week 1-2:  M1, M2, M3 (including schema additions for benchmarking)
Week 2-3:  M4, M4.5 (scheduling infrastructure) ← NEW 3 days
Week 3:    M5
Week 3-4:  M6
Week 4-5:  M7
Week 5:    M8
Week 5-6:  M9, M9.5 (benchmarking) ← NEW 2 days
Week 6-7:  M10, M10.5 (public scan + badges) ← NEW 4 days
Week 7-9:  M11 (dashboard including M11.13 onboarding) ← +3 days inline
Week 9-10: M12, M12.5 (billing + digest) ← NEW 2 days
Week 10:   M13
Week 11:   M14
Week 12-13: M15 (testing + launch)

Total: 13 weeks (originally 10)
```

### 8.2 Critical 5 Task Summary

| Task ID | Feature | Duration | Dependencies |
|---|---|---|---|
| M4.5.1 | Schedule management endpoints | 1 day | M4 |
| M4.5.2 | Scheduler worker (Go) | 2 days | M5 |
| M9.5.1 | Benchmark aggregation job | 1 day | M9 |
| M9.5.2 | Benchmark lookup API | 0.5 day | M9.5.1 |
| M9.5.3 | Benchmark UI component | 0.5 day | M9.5.2, M11 |
| M10.5.1 | Public scan endpoint + rate limiting | 1 day | M10 |
| M10.5.2 | Public scan executor (Go) | 1 day | M10.5.1 |
| M10.5.3 | Grade calculation | 0.5 day | M10.5.1 |
| M10.5.4 | Public results page + share cards | 1 day | M10.5.1 |
| M10.5.5 | Conversion tracking | 0.5 day | M10.5.4 |
| M10.5.6 | Badge issuance service | 1 day | M10 |
| M10.5.7 | Public badge page + SVG | 1 day | M10.5.6 |
| M11.13.1 | Onboarding backend | 1 day | M11 |
| M11.13.2 | Onboarding UI | 1.5 days | M11.13.1 |
| M11.13.3 | Activation analytics | 0.5 day | M11.13.1 |
| M12.5.1 | Digest email generator | 1 day | M12 |
| M12.5.2 | Email tracking + unsubscribe | 1 day | M12.5.1 |

**Total: 15 engineer-days ≈ 3 weeks.**

### 8.3 Priority if Timeline Slips

If you run short on time before launch, **cut in this order** (lowest impact first):

1. Cut first: M9.5.3 (benchmark UI) — ship backend, add UI week 1 post-launch
2. Cut second: M11.13.3 (activation analytics) — manually check DB for first month
3. Cut third: M10.5.4 (OG share cards) — ship without pretty previews
4. Cut fourth: M12.5.2 (email tracking) — ship without open/click metrics
5. DO NOT CUT: M10.5.1-3 (public scan), M4.5 (scheduling), M11.13.1-2 (onboarding)

The public scan, scheduling, and onboarding are the acquisition/retention backbone. Everything else is enhancement.

---

## 9. Cross-References to Updated Docs

### 9.1 Changes to SPECIFICATION.md

**Section §4 (Bounded Contexts):** Add contexts 8, 9, 10 — Public Scanning, Scheduling, Trust Signals.

**Section §5 (Data Model):** Add 7 new tables, 4 column additions per §1.2.

**Section §6 (API Endpoints):** Add 14 endpoints per §1.4.

**Section §9 (Pricing):** No changes to pricing tiers, but add: "All public scans free. Verified badge included in all paid tiers. Peer benchmarks included in Starter+."

**Section §13 (ADRs):** Add:
- **ADR-010:** Why rate limiting on public scans uses Redis sliding window (vs token bucket)
- **ADR-011:** Why minimum bucket size of 10 for benchmarks (privacy vs utility tradeoff)
- **ADR-012:** Why badges expire after 30 days (lock-in vs customer freedom balance)

### 9.2 Changes to TOOL-ARCHITECTURE.md

**Section §10 (Scan Type Matrix):** Add `public` scan type:
```go
"public": {
    Tools:         []string{"subfinder_fast", "httpx_headers", "sslyze_quick"},
    MaxDuration:   60 * time.Second,
    MinTier:       "none",  // unauthenticated
},
```

### 9.3 Changes to IMPLEMENTATION-PLAN.md

**Insert between M4 and M5:** Milestone M4.5 (Scheduling Infrastructure)
**Insert between M9 and M10:** Milestone M9.5 (Benchmarking)
**Insert between M10 and M11:** Milestone M10.5 (Public Scan + Badges)
**Insert in M11 after Task 11.12:** Task 11.13 (Onboarding)
**Insert between M12 and M13:** Milestone M12.5 (Email Digest)

### 9.4 Changes to OPERATIONS-RUNBOOK.md

**Section §2 (Architecture):** Add public scan queue to topology diagram.

**Section §4 (Worker Pools):** Add:
| Pool | Workers | Capability |
|---|---|---|
| `public` | 2× (4GB) | Lightweight tools only, rate-limit-aware |

**Section §7 (Monitoring):** Add alerts:
- `PublicScanQueueBackedUp` — queue > 50 scans
- `BadgeSigningKeyRotationNeeded` — every 180 days
- `BenchmarkComputationFailed` — nightly job failure
- `DigestEmailDeliveryRate` — bounces > 5%

**Section §12 (Cost):** Add:
- Public scan infrastructure: ~$30/month at 1000 scans/day
- Email digest delivery: ~$15/month for 10K emails/month

### 9.5 Changes to CONSTITUTION.md

**Section §6.1 (Customer Acceptance):** Add: "Public scans are unauthenticated but logged. Users confirm ownership before submitting. Abuse detection automatically blocks suspicious IP patterns."

**Section §8.1 (Service Commitments):** Add: "Verified badges are issued in good faith based on scan results at time of issuance. They are not warranties. Revocation occurs automatically on new critical/high findings."

### 9.6 Changes to VERSIONS.md

No version changes required. All features use existing pinned libraries.

### 9.7 Changes to CLAUDE.md

**New Gotcha:** Add Gotcha 9:

> **Gotcha 9: Public vs Authenticated Scans Diverge**
>
> Public scans (unauthenticated, rate-limited) go on `shieldscan:queue:public`. They use a restricted tool set (no active fuzzing, no SAST, no mobile). Results are intentionally less detailed than paid scans. Do not let public-scan code paths touch the authenticated scan pipeline — they have different security implications, different error handling, and different rate-limiting logic. Keep them cleanly separated in `services/public_scan.py` vs `services/scans.py`.

---

## Appendix A: Success Metrics for Critical 5

### First 30 days post-launch

| Metric | Target | How to Measure |
|---|---|---|
| Public scans/day | 100+ | `SELECT COUNT(*) FROM public_scans WHERE created_at > NOW() - INTERVAL '24 hours'` |
| Public → signup conversion | 5%+ | `SELECT COUNT(*) FROM public_scans WHERE converted_to_signup` / total |
| Onboarding completion rate | 60%+ | `onboarding_progress.completed_at IS NOT NULL` |
| Time-to-first-scan (median) | < 180 sec | From signup to first real scan |
| Scheduled scan adoption | 30%+ of paid | `SELECT COUNT(*) FROM scan_schedules WHERE is_active` / paid orgs |
| Badge issuance rate | 10%+ of scans | Clean scans / total scans |
| Badge embed rate | 40%+ of issued | `embed_count > 0` |
| Benchmark view rate | 80%+ of scans | Benchmark API calls / scans |

### First 90 days post-launch

| Metric | Target |
|---|---|
| Paying customers | 20+ |
| MRR | $2K+ |
| Public scan → paying customer | 2%+ |
| Churn rate | < 10%/month |

If metrics miss targets, the **first place to look** is activation funnel in onboarding — the data shows whether users get stuck at a specific step.

---

*End of Addendum. This document authorizes the Critical 5 launch features and 4 moat-preparation schema changes. After Week 13, return to the original IMPLEMENTATION-PLAN.md for post-launch roadmap.*
