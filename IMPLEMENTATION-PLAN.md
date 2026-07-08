# ShieldScan — Implementation Plan

> **For Claude Code:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build the ShieldScan v4.0 platform — a 9-category, 19-tool AI-powered security testing SaaS with mobile security (MobSF), recon-first subdomain auto-expansion, AI cross-layer correlation, React dashboard, REST API, CLI, and Stripe billing.

**Architecture:** FastAPI (Python) API + AI layer, Go scan workers, PostgreSQL with RLS, Redis queues + pub/sub, Qdrant vector dedup, React + TypeScript dashboard, hybrid tool deployment (native binaries + persistent Docker services).

**Tech Stack:** FastAPI, SQLAlchemy 2.0, Alembic, Pydantic v2, React 18, TypeScript, Tailwind, Vite, Go 1.22, Nuclei CLI, Docker SDK, MobSF REST API, PostgreSQL 16, Redis 7, Qdrant, Claude API, OpenAI Embeddings, Stripe, Cloudflare R2.

**Companion docs:** `SPECIFICATION.md`, `TOOL-ARCHITECTURE.md`, `OPERATIONS-RUNBOOK.md`.

**Repositories:** `shieldscan-api` (Python + React) and `shieldscan-engine` (Go).

---

## Table of Contents

- [Milestone 1: Project Scaffold & Database](#milestone-1-project-scaffold--database-week-1)
- [Milestone 2: Auth & Identity](#milestone-2-auth--identity-week-1-2)
- [Milestone 3: Projects & Mobile Uploads](#milestone-3-projects--mobile-uploads-week-2)
- [Milestone 4: Scan Orchestration & Redis Contracts](#milestone-4-scan-orchestration--redis-contracts-week-2-3)
- [Milestone 5: Go Worker Foundation](#milestone-5-go-worker-foundation-week-3)
- [Milestone 6: Native Tool Runners](#milestone-6-native-tool-runners-week-3-4)
- [Milestone 7: Persistent Docker Service Runners](#milestone-7-persistent-docker-service-runners-week-4-5)
- [Milestone 8: Recon-First Pipeline](#milestone-8-recon-first-pipeline-week-5)
- [Milestone 9: AI Analysis Pipeline](#milestone-9-ai-analysis-pipeline-week-5-6)
- [Milestone 10: Vulnerability & Report APIs](#milestone-10-vulnerability--report-apis-week-6)
- [Milestone 11: React Dashboard](#milestone-11-react-dashboard-week-6-8)
- [Milestone 12: Billing & Integrations](#milestone-12-billing--integrations-week-8-9)
- [Milestone 13: CLI & CI/CD](#milestone-13-cli--cicd-week-9)
- [Milestone 14: On-Prem Agent](#milestone-14-on-prem-agent-week-10)
- [Milestone 15: Testing, Hardening & Launch](#milestone-15-testing-hardening--launch-week-10)

---

## Milestone 1: Project Scaffold & Database (Week 1)

### Task 1.1: Scaffold shieldscan-api repo

**Files:**
- Create: `shieldscan-api/pyproject.toml`
- Create: `shieldscan-api/src/app/main.py`
- Create: `shieldscan-api/src/app/config.py`
- Create: `shieldscan-api/.env.example`
- Create: `shieldscan-api/docker-compose.dev.yml`
- Create: `shieldscan-api/.github/workflows/ci.yml`

**Step 1: Create pyproject.toml with Poetry**

```toml
[tool.poetry]
name = "shieldscan-api"
version = "0.1.0"
description = "ShieldScan API and AI analysis platform"

[tool.poetry.dependencies]
python = "^3.12"
fastapi = "^0.110.0"
uvicorn = {extras = ["standard"], version = "^0.27.0"}
sqlalchemy = "^2.0.25"
alembic = "^1.13.1"
asyncpg = "^0.29.0"
pydantic = "^2.6.0"
pydantic-settings = "^2.1.0"
redis = "^5.0.1"
qdrant-client = "^1.7.0"
anthropic = "^0.18.0"
openai = "^1.12.0"
stripe = "^8.0.0"
boto3 = "^1.34.0"
python-jose = {extras = ["cryptography"], version = "^3.3.0"}
passlib = {extras = ["bcrypt"], version = "^1.7.4"}
cryptography = "^42.0.0"
weasyprint = "^60.2"
jinja2 = "^3.1.3"
httpx = "^0.26.0"
email-validator = "^2.1.0"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0.0"
pytest-asyncio = "^0.23.0"
pytest-cov = "^4.1.0"
httpx = "^0.26.0"
ruff = "^0.2.0"
mypy = "^1.8.0"
```

**Step 2: Create src/app/main.py**

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings
from app.db import engine, init_db

@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()
    yield
    await engine.dispose()

app = FastAPI(
    title="ShieldScan API",
    version="4.0.0",
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/health")
async def health():
    return {"status": "ok", "version": "4.0.0"}
```

**Step 3: Create src/app/config.py**

```python
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    DATABASE_URL: str
    REDIS_URL: str
    QDRANT_URL: str
    ANTHROPIC_API_KEY: str
    OPENAI_API_KEY: str
    STRIPE_SECRET_KEY: str
    STRIPE_WEBHOOK_SECRET: str
    R2_ACCOUNT_ID: str
    R2_ACCESS_KEY: str
    R2_SECRET_KEY: str
    R2_BUCKET: str
    JWT_SECRET_KEY: str
    JWT_ALGORITHM: str = "HS256"
    JWT_ACCESS_TOKEN_EXPIRE_MINUTES: int = 15
    JWT_REFRESH_TOKEN_EXPIRE_DAYS: int = 7
    FERNET_KEY: str
    CORS_ORIGINS: List[str] = ["http://localhost:5173", "https://app.shieldscan.io"]
    ENVIRONMENT: str = "development"

    class Config:
        env_file = ".env"

settings = Settings()
```

**Step 4: Run health check**
```bash
cd shieldscan-api && poetry install && poetry run uvicorn app.main:app --reload
# Visit http://localhost:8000/health → {"status":"ok","version":"4.0.0"}
```

**Step 5: Commit**
```bash
git init && git add -A
git commit -m "feat: scaffold shieldscan-api with FastAPI + Poetry"
```

---

### Task 1.2: Database base models + TenantMixin

**Files:**
- Create: `shieldscan-api/src/app/db.py`
- Create: `shieldscan-api/src/app/models/base.py`
- Create: `shieldscan-api/alembic.ini`
- Create: `shieldscan-api/alembic/env.py`

**Step 1: Write failing test**

```python
# tests/models/test_base.py
from app.models.base import Base, TenantMixin

def test_tenantmixin_has_org_id_column():
    from sqlalchemy import inspect
    class TestModel(Base, TenantMixin):
        __tablename__ = "test_tenant_model"
    cols = {c.name for c in inspect(TestModel).columns}
    assert "organization_id" in cols
```

**Step 2: Run test — should fail**
```bash
pytest tests/models/test_base.py -v
```

**Step 3: Implement src/app/models/base.py**

```python
from datetime import datetime
from uuid import UUID, uuid4
from sqlalchemy import DateTime, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy.dialects.postgresql import UUID as PgUUID

class Base(DeclarativeBase):
    pass

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=datetime.utcnow, nullable=False
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=datetime.utcnow, onupdate=datetime.utcnow, nullable=False
    )

class TenantMixin:
    organization_id: Mapped[UUID] = mapped_column(
        PgUUID(as_uuid=True),
        ForeignKey("organizations.id", ondelete="CASCADE"),
        nullable=False,
        index=True,
    )
```

**Step 4: Run test — passes**

**Step 5: Commit**
```bash
git add src/app/models/base.py src/app/db.py alembic.ini alembic/
git commit -m "feat(db): add Base model with TimestampMixin and TenantMixin"
```

---

### Task 1.3: Core identity models

**Files:**
- Create: `src/app/models/identity.py`
- Test: `tests/models/test_identity.py`

**Step 1: Write failing tests**

```python
# tests/models/test_identity.py
import pytest
from app.models.identity import Organization, User, Membership, APIKey, MembershipRole

@pytest.mark.asyncio
async def test_create_organization(db_session):
    org = Organization(name="Test Org", slug="test-org")
    db_session.add(org)
    await db_session.commit()
    assert org.id is not None
    assert org.name == "Test Org"

@pytest.mark.asyncio
async def test_create_user(db_session):
    user = User(email="user@test.com", hashed_password="...", full_name="Test User")
    db_session.add(user)
    await db_session.commit()
    assert user.id is not None
    assert user.email_verified is False

@pytest.mark.asyncio
async def test_membership_links_user_to_org(db_session, sample_user, sample_org):
    m = Membership(user_id=sample_user.id, organization_id=sample_org.id, role=MembershipRole.ADMIN)
    db_session.add(m)
    await db_session.commit()
    assert m.role == MembershipRole.ADMIN
```

**Step 2: Implement**

```python
# src/app/models/identity.py
from enum import Enum as PyEnum
from uuid import UUID, uuid4
from sqlalchemy import String, Boolean, Enum, ForeignKey, DateTime, UniqueConstraint, Index
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from app.models.base import Base, TimestampMixin

class MembershipRole(str, PyEnum):
    ADMIN = "admin"
    MEMBER = "member"
    VIEWER = "viewer"

class Organization(Base, TimestampMixin):
    __tablename__ = "organizations"
    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True, default=uuid4)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    slug: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    stripe_customer_id: Mapped[str | None] = mapped_column(String(255), nullable=True)
    members = relationship("Membership", back_populates="organization", cascade="all, delete-orphan")

class User(Base, TimestampMixin):
    __tablename__ = "users"
    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True, default=uuid4)
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False, index=True)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    full_name: Mapped[str] = mapped_column(String(255), nullable=False)
    email_verified: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
    github_id: Mapped[str | None] = mapped_column(String(100), nullable=True, index=True)
    memberships = relationship("Membership", back_populates="user", cascade="all, delete-orphan")

class Membership(Base, TimestampMixin):
    __tablename__ = "memberships"
    __table_args__ = (UniqueConstraint("user_id", "organization_id", name="uq_user_org"),)
    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True, default=uuid4)
    user_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), ForeignKey("users.id", ondelete="CASCADE"))
    organization_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), ForeignKey("organizations.id", ondelete="CASCADE"))
    role: Mapped[MembershipRole] = mapped_column(Enum(MembershipRole), nullable=False, default=MembershipRole.MEMBER)
    user = relationship("User", back_populates="memberships")
    organization = relationship("Organization", back_populates="members")

class APIKey(Base, TimestampMixin):
    __tablename__ = "api_keys"
    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True, default=uuid4)
    organization_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), ForeignKey("organizations.id", ondelete="CASCADE"), index=True)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    key_prefix: Mapped[str] = mapped_column(String(20), nullable=False)
    key_hash: Mapped[str] = mapped_column(String(255), unique=True, nullable=False, index=True)
    last_used_at: Mapped["DateTime | None"] = mapped_column(DateTime(timezone=True), nullable=True)
    revoked_at: Mapped["DateTime | None"] = mapped_column(DateTime(timezone=True), nullable=True)
```

**Step 3: Generate Alembic migration**
```bash
alembic revision --autogenerate -m "create identity tables"
alembic upgrade head
```

**Step 4: Run tests**
```bash
pytest tests/models/test_identity.py -v
```

**Step 5: Commit**
```bash
git add src/app/models/identity.py alembic/versions/
git commit -m "feat(db): add Organization, User, Membership, APIKey models"
```

---

### Task 1.4: Project, Scan, Mobile, and Attack Surface models

**Files:**
- Create: `src/app/models/projects.py`
- Create: `src/app/models/scans.py`
- Create: `src/app/models/mobile.py`
- Create: `src/app/models/recon.py`
- Test: `tests/models/test_scans.py`

**Step 1: Write failing tests**

```python
# tests/models/test_scans.py
import pytest
from app.models.scans import Scan, ScanStatus, ScanType
from app.models.mobile import MobileUpload, MobilePlatform
from app.models.recon import AttackSurface, SubdomainStatus

@pytest.mark.asyncio
async def test_create_mobile_scan(db_session, sample_project, sample_mobile_upload):
    scan = Scan(
        organization_id=sample_project.organization_id,
        project_id=sample_project.id,
        scan_type=ScanType.MOBILE,
        mobile_upload_id=sample_mobile_upload.id,
        status=ScanStatus.QUEUED,
    )
    db_session.add(scan)
    await db_session.commit()
    assert scan.scan_type == ScanType.MOBILE

@pytest.mark.asyncio
async def test_attack_surface_unique_per_scan(db_session, sample_scan):
    a1 = AttackSurface(
        organization_id=sample_scan.organization_id,
        scan_id=sample_scan.id,
        root_domain="example.com",
        subdomain="api.example.com",
        full_url="https://api.example.com",
        status=SubdomainStatus.LIVE,
    )
    db_session.add(a1)
    await db_session.commit()
    assert a1.id is not None
```

**Step 2: Implement all 4 model files**

```python
# src/app/models/projects.py
from enum import Enum as PyEnum
from uuid import UUID, uuid4
from sqlalchemy import String, Boolean, Enum, ForeignKey, JSON, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID as PgUUID
from app.models.base import Base, TimestampMixin, TenantMixin

class Project(Base, TimestampMixin, TenantMixin):
    __tablename__ = "projects"
    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True, default=uuid4)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    target_url: Mapped[str] = mapped_column(String(500), nullable=False)
    root_domain: Mapped[str] = mapped_column(String(255), nullable=False, index=True)
    source_repo_url: Mapped[str | None] = mapped_column(String(500), nullable=True)
    container_image: Mapped[str | None] = mapped_column(String(255), nullable=True)
    domain_verified: Mapped[bool] = mapped_column(Boolean, default=False)
    verification_token: Mapped[str] = mapped_column(String(100), nullable=False, default=lambda: uuid4().hex)
    archived_at: Mapped["DateTime | None"] = mapped_column(nullable=True)

class ProjectCredential(Base, TimestampMixin, TenantMixin):
    __tablename__ = "project_credentials"
    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True, default=uuid4)
    project_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), ForeignKey("projects.id", ondelete="CASCADE"), index=True)
    auth_type: Mapped[str] = mapped_column(String(50), nullable=False)  # cookie|bearer|basic|form
    encrypted_data: Mapped[bytes] = mapped_column(nullable=False)  # Fernet-encrypted

# src/app/models/scans.py
class ScanType(str, PyEnum):
    QUICK = "quick"
    FULL_WEB = "full_web"
    FULL_WEB_SOURCE = "full_web_source"
    API = "api"
    MOBILE = "mobile"
    CONTAINER = "container"
    FULL_SPECTRUM = "full_spectrum"

class ScanStatus(str, PyEnum):
    QUEUED = "queued"
    RECONNING = "reconning"
    RUNNING = "running"
    ANALYZING = "analyzing"
    COMPLETED = "completed"
    PARTIAL = "partial"
    FAILED = "failed"
    CANCELED = "canceled"

class Scan(Base, TimestampMixin, TenantMixin):
    __tablename__ = "scans"
    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True, default=uuid4)
    project_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), ForeignKey("projects.id", ondelete="CASCADE"), index=True)
    scan_type: Mapped[ScanType] = mapped_column(Enum(ScanType), nullable=False)
    status: Mapped[ScanStatus] = mapped_column(Enum(ScanStatus), nullable=False, default=ScanStatus.QUEUED)
    mobile_upload_id: Mapped[UUID | None] = mapped_column(PgUUID(as_uuid=True), ForeignKey("mobile_uploads.id"), nullable=True)
    recon_enabled: Mapped[bool] = mapped_column(Boolean, default=True)
    subdomain_limit: Mapped[int] = mapped_column(default=100)
    config: Mapped[dict] = mapped_column(JSON, default=dict)
    priority: Mapped[str] = mapped_column(String(20), default="normal")
    started_at: Mapped["DateTime | None"] = mapped_column(nullable=True)
    completed_at: Mapped["DateTime | None"] = mapped_column(nullable=True)
    error_message: Mapped[str | None] = mapped_column(Text, nullable=True)

class ScanJob(Base, TimestampMixin, TenantMixin):
    __tablename__ = "scan_jobs"
    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True, default=uuid4)
    scan_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), ForeignKey("scans.id", ondelete="CASCADE"), index=True)
    engine: Mapped[str] = mapped_column(String(50), nullable=False)  # nuclei|zap|mobsf|...
    target_url: Mapped[str] = mapped_column(String(500), nullable=False)
    status: Mapped[str] = mapped_column(String(20), default="queued")
    idempotency_key: Mapped[str] = mapped_column(String(255), unique=True, nullable=False)
    finding_count: Mapped[int] = mapped_column(default=0)
    duration_ms: Mapped[int | None] = mapped_column(nullable=True)
    error_message: Mapped[str | None] = mapped_column(Text, nullable=True)

# src/app/models/mobile.py
class MobilePlatform(str, PyEnum):
    ANDROID = "android"
    IOS = "ios"
    UNKNOWN = "unknown"

class MobileUpload(Base, TimestampMixin, TenantMixin):
    __tablename__ = "mobile_uploads"
    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True, default=uuid4)
    project_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), ForeignKey("projects.id", ondelete="CASCADE"), index=True)
    uploaded_by: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    filename: Mapped[str] = mapped_column(String(255), nullable=False)
    file_extension: Mapped[str] = mapped_column(String(10), nullable=False)
    file_size_bytes: Mapped[int] = mapped_column(nullable=False)
    platform: Mapped[MobilePlatform] = mapped_column(Enum(MobilePlatform), nullable=False)
    r2_key: Mapped[str] = mapped_column(String(500), nullable=False)
    mobsf_file_hash: Mapped[str | None] = mapped_column(String(100), nullable=True)
    deleted_at: Mapped["DateTime | None"] = mapped_column(nullable=True)

# src/app/models/recon.py
class SubdomainStatus(str, PyEnum):
    LIVE = "live"
    DEAD = "dead"
    TIMEOUT = "timeout"

class AttackSurface(Base, TimestampMixin, TenantMixin):
    __tablename__ = "attack_surface"
    __table_args__ = (UniqueConstraint("scan_id", "subdomain", name="uq_scan_subdomain"),)
    id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), primary_key=True, default=uuid4)
    scan_id: Mapped[UUID] = mapped_column(PgUUID(as_uuid=True), ForeignKey("scans.id", ondelete="CASCADE"), index=True)
    root_domain: Mapped[str] = mapped_column(String(255), nullable=False)
    subdomain: Mapped[str] = mapped_column(String(255), nullable=False)
    full_url: Mapped[str] = mapped_column(String(500), nullable=False)
    status: Mapped[SubdomainStatus] = mapped_column(Enum(SubdomainStatus), nullable=False)
    status_code: Mapped[int | None] = mapped_column(nullable=True)
    tech_stack: Mapped[list | None] = mapped_column(JSON, nullable=True)
    last_probed_at: Mapped["DateTime | None"] = mapped_column(nullable=True)
```

**Step 3: Generate migration and run tests**
```bash
alembic revision --autogenerate -m "add projects scans mobile recon tables"
alembic upgrade head
pytest tests/models/test_scans.py -v
```

**Step 4: Commit**
```bash
git add src/app/models/ alembic/versions/
git commit -m "feat(db): add Project, Scan, ScanJob, MobileUpload, AttackSurface models"
```

---

### Task 1.5: Vulnerability, evidence, and audit log models

**Files:**
- Create: `src/app/models/vulnerabilities.py`
- Create: `src/app/models/audit.py`
- Create: `src/app/models/raw_findings.py`

**Step 1: Failing tests** — verify Vulnerability has all classification and evidence fields, raw_findings.engine_category enum is valid, audit_logs is append-only.

**Step 2: Implement** — full models per Specification §5.

Key additions for mobile:
```python
class RawFinding(Base, TimestampMixin, TenantMixin):
    __tablename__ = "raw_findings"
    # ... standard fields ...
    engine_category: Mapped[str] = mapped_column(
        Enum("dast", "sast", "sca", "mobile", "infrastructure", "recon",
             "ssl", "api", "iac", "secrets", "container", name="engine_category_enum"),
        nullable=False,
    )
    mobile_os: Mapped[str | None] = mapped_column(Enum("android", "ios", name="mobile_os_enum"), nullable=True)
    mobile_permission: Mapped[str | None] = mapped_column(String(255), nullable=True)
    mobile_component_name: Mapped[str | None] = mapped_column(String(255), nullable=True)
```

**Step 3: Migration + tests + commit**
```bash
alembic revision --autogenerate -m "add vulnerabilities raw_findings audit_logs"
alembic upgrade head
git add src/app/models/ alembic/versions/
git commit -m "feat(db): add Vulnerability, RawFinding, AuditLog models with mobile fields"
```

---

### Task 1.6: Enable Row-Level Security on tenant tables

**Files:**
- Create: `alembic/versions/xxx_enable_rls.py` (manual migration)

**Step 1: Write the RLS migration**

```python
"""enable RLS on tenant tables

Revision ID: xxxxx
Revises: yyyyy
"""
from alembic import op

def upgrade():
    tenant_tables = [
        "projects", "project_credentials", "scans", "scan_jobs",
        "raw_findings", "vulnerabilities", "evidence", "vulnerability_history",
        "mobile_uploads", "attack_surface", "audit_logs", "api_keys",
        "subscriptions", "usage_records",
    ]
    for table in tenant_tables:
        op.execute(f"ALTER TABLE {table} ENABLE ROW LEVEL SECURITY;")
        op.execute(f"""
            CREATE POLICY tenant_isolation ON {table}
            USING (organization_id = current_setting('app.current_org_id')::uuid);
        """)

def downgrade():
    tenant_tables = [...]
    for table in tenant_tables:
        op.execute(f"DROP POLICY IF EXISTS tenant_isolation ON {table};")
        op.execute(f"ALTER TABLE {table} DISABLE ROW LEVEL SECURITY;")
```

**Step 2: Write tests proving RLS works**

```python
async def test_rls_prevents_cross_tenant_access(db_session, org_a_scan, org_b):
    await db_session.execute(text(f"SET LOCAL app.current_org_id = '{org_b.id}'"))
    result = await db_session.execute(
        text("SELECT COUNT(*) FROM scans WHERE id = :id"),
        {"id": org_a_scan.id}
    )
    assert result.scalar() == 0  # org_b cannot see org_a's scan
```

**Step 3: Run migration, run tests, commit**
```bash
alembic upgrade head
pytest tests/test_rls.py -v
git add alembic/versions/
git commit -m "feat(db): enable row-level security on all tenant tables"
```

---

## Milestone 2: Auth & Identity (Week 1-2)

### Task 2.1: Password hashing + JWT utilities

**Files:**
- Create: `src/app/services/auth.py`
- Test: `tests/services/test_auth.py`

**Step 1: Failing test**

```python
def test_hash_and_verify_password():
    from app.services.auth import hash_password, verify_password
    pwd = "SecureP@ssw0rd123"
    hashed = hash_password(pwd)
    assert hashed != pwd
    assert verify_password(pwd, hashed) is True
    assert verify_password("wrong", hashed) is False

def test_create_and_decode_jwt():
    from app.services.auth import create_access_token, decode_token
    from uuid import uuid4
    user_id = uuid4()
    token = create_access_token(sub=str(user_id))
    payload = decode_token(token)
    assert payload["sub"] == str(user_id)
```

**Step 2: Implement**

```python
# src/app/services/auth.py
from datetime import datetime, timedelta
from passlib.context import CryptContext
from jose import jwt, JWTError
from app.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(sub: str, extra: dict | None = None) -> str:
    expire = datetime.utcnow() + timedelta(minutes=settings.JWT_ACCESS_TOKEN_EXPIRE_MINUTES)
    payload = {"sub": sub, "exp": expire, "type": "access"}
    if extra: payload.update(extra)
    return jwt.encode(payload, settings.JWT_SECRET_KEY, algorithm=settings.JWT_ALGORITHM)

def create_refresh_token(sub: str) -> str:
    expire = datetime.utcnow() + timedelta(days=settings.JWT_REFRESH_TOKEN_EXPIRE_DAYS)
    payload = {"sub": sub, "exp": expire, "type": "refresh"}
    return jwt.encode(payload, settings.JWT_SECRET_KEY, algorithm=settings.JWT_ALGORITHM)

def decode_token(token: str) -> dict:
    try:
        return jwt.decode(token, settings.JWT_SECRET_KEY, algorithms=[settings.JWT_ALGORITHM])
    except JWTError as e:
        raise ValueError(f"Invalid token: {e}")
```

**Step 3-5: Run tests, commit**

---

### Task 2.2: API key generation and validation

**Files:**
- Create: `src/app/services/api_keys.py`
- Test: `tests/services/test_api_keys.py`

**Step 1: Failing test**

```python
def test_generate_api_key_returns_plain_and_hash():
    from app.services.api_keys import generate_api_key
    plain, key_hash, prefix = generate_api_key(env="live")
    assert plain.startswith("ss_live_")
    assert len(plain) == 48  # "ss_live_" + 40 random chars
    assert prefix == plain[:12]  # first 12 chars shown to user
    assert key_hash != plain
```

**Step 2: Implement**

```python
# src/app/services/api_keys.py
import secrets, hashlib

def generate_api_key(env: str = "live") -> tuple[str, str, str]:
    """Returns (plain_key, sha256_hash, prefix_for_display)."""
    random_part = secrets.token_urlsafe(30)[:40]
    plain = f"ss_{env}_{random_part}"
    key_hash = hashlib.sha256(plain.encode()).hexdigest()
    prefix = plain[:12]  # "ss_live_abcd"
    return plain, key_hash, prefix

def verify_api_key(plain: str, stored_hash: str) -> bool:
    return hashlib.sha256(plain.encode()).hexdigest() == stored_hash
```

**Step 3-5: Tests + commit**

---

### Task 2.3: Auth endpoints (register, login, refresh, logout)

**Files:**
- Create: `src/app/routes/auth.py`
- Create: `src/app/schemas/auth.py`
- Create: `src/app/dependencies.py`
- Test: `tests/routes/test_auth.py`

**Step 1: Failing tests**

```python
async def test_register_creates_user_and_org(client):
    r = await client.post("/auth/register", json={
        "email": "new@test.com", "password": "SecureP@ss123", "full_name": "New User",
        "organization_name": "New Co"
    })
    assert r.status_code == 201
    data = r.json()
    assert "access_token" in data
    assert data["user"]["email"] == "new@test.com"

async def test_login_success(client, existing_user):
    r = await client.post("/auth/login", json={
        "email": existing_user.email, "password": "TestPass123"
    })
    assert r.status_code == 200
    assert "access_token" in r.json()

async def test_login_invalid_credentials(client):
    r = await client.post("/auth/login", json={
        "email": "nobody@test.com", "password": "wrong"
    })
    assert r.status_code == 401

async def test_me_requires_auth(client):
    r = await client.get("/auth/me")
    assert r.status_code == 401

async def test_me_returns_user(client, auth_headers):
    r = await client.get("/auth/me", headers=auth_headers)
    assert r.status_code == 200
    assert "email" in r.json()
```

**Step 2: Implement routes**

```python
# src/app/routes/auth.py
from fastapi import APIRouter, Depends, HTTPException, Response
from sqlalchemy.ext.asyncio import AsyncSession
from app.dependencies import get_db, get_current_user
from app.models.identity import User, Organization, Membership, MembershipRole
from app.services.auth import hash_password, verify_password, create_access_token, create_refresh_token
from app.schemas.auth import RegisterRequest, LoginRequest, TokenResponse, UserResponse

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post("/register", status_code=201, response_model=TokenResponse)
async def register(req: RegisterRequest, db: AsyncSession = Depends(get_db)):
    # Check email not taken
    existing = await db.execute(select(User).where(User.email == req.email))
    if existing.scalar_one_or_none():
        raise HTTPException(400, "Email already registered")

    # Create org + user + membership in transaction
    org = Organization(name=req.organization_name, slug=slugify(req.organization_name))
    user = User(email=req.email, hashed_password=hash_password(req.password), full_name=req.full_name)
    db.add_all([org, user])
    await db.flush()
    db.add(Membership(user_id=user.id, organization_id=org.id, role=MembershipRole.ADMIN))
    await db.commit()

    access = create_access_token(sub=str(user.id))
    refresh = create_refresh_token(sub=str(user.id))
    return TokenResponse(access_token=access, refresh_token=refresh, user=UserResponse.from_orm(user))

@router.post("/login", response_model=TokenResponse)
async def login(req: LoginRequest, db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(User).where(User.email == req.email))
    user = result.scalar_one_or_none()
    if not user or not verify_password(req.password, user.hashed_password):
        raise HTTPException(401, "Invalid credentials")

    return TokenResponse(
        access_token=create_access_token(sub=str(user.id)),
        refresh_token=create_refresh_token(sub=str(user.id)),
        user=UserResponse.from_orm(user),
    )

@router.get("/me", response_model=UserResponse)
async def me(current: User = Depends(get_current_user)):
    return current
```

**Step 3: Wire into main.py**
```python
from app.routes import auth
app.include_router(auth.router, prefix="/v1")
```

**Step 4: Run tests, commit**
```bash
pytest tests/routes/test_auth.py -v
git add src/app/routes/auth.py src/app/schemas/auth.py src/app/dependencies.py
git commit -m "feat(auth): add register, login, refresh, me endpoints"
```

---

### Task 2.4: API key endpoints + auth dependency

**Files:**
- Create: `src/app/routes/api_keys.py`
- Modify: `src/app/dependencies.py` (add API key auth)

**Step 1: Failing tests** — create key, list keys, revoke key, auth with API key.

**Step 2: Implement** — include rate limiting check per org tier on creation.

**Step 3: Commit**
```bash
git commit -m "feat(auth): add API key management + API key authentication"
```

---

### Task 2.5: Email verification flow

**Files:**
- Create: `src/app/services/email.py`
- Modify: `src/app/routes/auth.py` (add verify-email, resend-verification)

**Step 1: Failing tests** — verification token generated, token expires after 24h, verify endpoint updates email_verified.

**Step 2: Implement** — use SendGrid/Resend. Send verification link on register. Token stored in Redis with 24h TTL.

**Step 3: Commit**
```bash
git commit -m "feat(auth): add email verification via SendGrid"
```

---

## Milestone 3: Projects & Mobile Uploads (Week 2)

### Task 3.1: Project CRUD endpoints

**Files:**
- Create: `src/app/routes/projects.py`
- Create: `src/app/schemas/projects.py`
- Create: `src/app/services/projects.py`
- Test: `tests/routes/test_projects.py`

**Step 1: Failing tests**

```python
async def test_create_project(client, auth_headers, sample_org):
    r = await client.post(f"/v1/orgs/{sample_org.id}/projects",
        json={"name": "My Site", "target_url": "https://example.com"},
        headers=auth_headers)
    assert r.status_code == 201
    assert r.json()["domain_verified"] is False

async def test_list_projects_scoped_to_org(client, auth_headers, sample_org):
    r = await client.get(f"/v1/orgs/{sample_org.id}/projects", headers=auth_headers)
    assert r.status_code == 200

async def test_get_project_other_org_fails(client, auth_headers, other_org_project):
    r = await client.get(f"/v1/orgs/{other_org_project.organization_id}/projects/{other_org_project.id}",
                         headers=auth_headers)
    assert r.status_code == 403
```

**Step 2-5: Implement + commit**

---

### Task 3.2: Domain verification

**Files:**
- Create: `src/app/services/domain_verify.py`
- Modify: `src/app/routes/projects.py`

**Step 1: Failing tests**

```python
async def test_verify_via_dns_txt(client, auth_headers, sample_project, mock_dns):
    mock_dns.set_txt(f"_shieldscan-verify.{sample_project.root_domain}",
                     sample_project.verification_token)
    r = await client.post(f"/v1/orgs/{sample_project.organization_id}/projects/{sample_project.id}/verify",
                          json={"method": "dns_txt"}, headers=auth_headers)
    assert r.status_code == 200
    assert r.json()["verified"] is True

async def test_verify_via_meta_tag(client, auth_headers, sample_project, mock_http):
    mock_http.set_html(sample_project.target_url,
                       f'<meta name="shieldscan-verification" content="{sample_project.verification_token}">')
    r = await client.post(f"/v1/orgs/{sample_project.organization_id}/projects/{sample_project.id}/verify",
                          json={"method": "meta_tag"}, headers=auth_headers)
    assert r.status_code == 200
```

**Step 2: Implement**

```python
# src/app/services/domain_verify.py
import dns.asyncresolver, httpx, re

async def verify_dns_txt(domain: str, expected_token: str) -> bool:
    try:
        answers = await dns.asyncresolver.resolve(f"_shieldscan-verify.{domain}", "TXT")
        for rdata in answers:
            if expected_token in str(rdata):
                return True
    except Exception:
        pass
    return False

async def verify_meta_tag(url: str, expected_token: str) -> bool:
    try:
        async with httpx.AsyncClient() as client:
            resp = await client.get(url, timeout=10.0)
            match = re.search(r'<meta[^>]+name=["\']shieldscan-verification["\'][^>]+content=["\']([^"\']+)', resp.text)
            return match and match.group(1) == expected_token
    except Exception:
        return False
```

**Step 3: Commit**
```bash
git commit -m "feat(projects): add DNS TXT + meta tag domain verification"
```

---

### Task 3.3: R2 storage client + Mobile upload endpoint

**Files:**
- Create: `src/app/services/r2.py`
- Create: `src/app/services/mobile.py`
- Create: `src/app/routes/mobile.py`
- Create: `src/app/schemas/mobile.py`
- Test: `tests/routes/test_mobile.py`

**Step 1: Failing tests**

```python
async def test_upload_apk_returns_upload_ref(client, auth_headers, sample_project, mock_r2, fake_apk_bytes):
    files = {"file": ("myapp.apk", fake_apk_bytes, "application/vnd.android.package-archive")}
    r = await client.post(
        f"/v1/orgs/{sample_project.organization_id}/projects/{sample_project.id}/mobile/upload",
        files=files, headers=auth_headers
    )
    assert r.status_code == 200
    data = r.json()
    assert data["upload_ref"].startswith("r2://")
    assert data["platform_detected"] == "android"

async def test_upload_rejects_wrong_extension(client, auth_headers, sample_project):
    files = {"file": ("document.pdf", b"PDF content", "application/pdf")}
    r = await client.post(
        f"/v1/orgs/{sample_project.organization_id}/projects/{sample_project.id}/mobile/upload",
        files=files, headers=auth_headers
    )
    assert r.status_code == 400
    assert "not supported" in r.json()["error"]["message"].lower()

async def test_upload_rejects_oversized(client, auth_headers, sample_project):
    huge = b"0" * (501 * 1024 * 1024)
    files = {"file": ("big.apk", huge, "application/vnd.android.package-archive")}
    r = await client.post(
        f"/v1/orgs/{sample_project.organization_id}/projects/{sample_project.id}/mobile/upload",
        files=files, headers=auth_headers
    )
    assert r.status_code == 413

async def test_upload_validates_magic_bytes(client, auth_headers, sample_project):
    # Fake .apk with wrong content
    files = {"file": ("fake.apk", b"not a real apk", "application/vnd.android.package-archive")}
    r = await client.post(
        f"/v1/orgs/{sample_project.organization_id}/projects/{sample_project.id}/mobile/upload",
        files=files, headers=auth_headers
    )
    assert r.status_code == 400
    assert "magic bytes" in r.json()["error"]["message"].lower()
```

**Step 2: Implement**

```python
# src/app/services/r2.py
import boto3
from botocore.config import Config
from app.config import settings

def get_r2_client():
    return boto3.client(
        "s3",
        endpoint_url=f"https://{settings.R2_ACCOUNT_ID}.r2.cloudflarestorage.com",
        aws_access_key_id=settings.R2_ACCESS_KEY,
        aws_secret_access_key=settings.R2_SECRET_KEY,
        config=Config(signature_version="s3v4"),
        region_name="auto",
    )

async def put_object(key: str, content: bytes, content_type: str):
    client = get_r2_client()
    client.put_object(Bucket=settings.R2_BUCKET, Key=key, Body=content, ContentType=content_type)

async def delete_object(key: str):
    get_r2_client().delete_object(Bucket=settings.R2_BUCKET, Key=key)

# src/app/services/mobile.py
from pathlib import Path
from uuid import uuid4
from app.models.mobile import MobilePlatform

ALLOWED_EXTENSIONS = {".apk", ".ipa", ".zip"}
MAX_FILE_SIZE_MB = 500
MAGIC_BYTES = {
    ".apk": [b"PK\x03\x04", b"PK\x05\x06"],  # ZIP (APK is a ZIP)
    ".ipa": [b"PK\x03\x04"],  # IPA is also ZIP
    ".zip": [b"PK\x03\x04", b"PK\x05\x06"],
}

def detect_platform(filename: str, content: bytes) -> MobilePlatform:
    ext = Path(filename).suffix.lower()
    if ext == ".apk": return MobilePlatform.ANDROID
    if ext == ".ipa": return MobilePlatform.IOS
    # For .zip: try to detect from content
    if b"AndroidManifest.xml" in content[:50000]: return MobilePlatform.ANDROID
    if b"Info.plist" in content[:50000]: return MobilePlatform.IOS
    return MobilePlatform.UNKNOWN

def validate_magic_bytes(ext: str, content: bytes) -> bool:
    expected = MAGIC_BYTES.get(ext, [])
    return any(content.startswith(sig) for sig in expected)

# src/app/routes/mobile.py
from fastapi import APIRouter, UploadFile, HTTPException, Depends
from app.services import r2, mobile as mobile_svc
from app.models.mobile import MobileUpload

router = APIRouter(prefix="/orgs/{org_id}/projects/{project_id}/mobile", tags=["mobile"])

@router.post("/upload")
async def upload_mobile_file(
    org_id: UUID, project_id: UUID, file: UploadFile,
    current_user: User = Depends(get_current_user),
    db: AsyncSession = Depends(get_db),
):
    # Validate project access (RLS check)
    project = await db.get(Project, project_id)
    if not project or project.organization_id != org_id:
        raise HTTPException(404, "Project not found")

    # Validate extension
    ext = Path(file.filename).suffix.lower()
    if ext not in mobile_svc.ALLOWED_EXTENSIONS:
        raise HTTPException(400, f"File type {ext} not supported. Use .apk, .ipa, or .zip")

    # Read and validate size
    content = await file.read()
    size_mb = len(content) / (1024 * 1024)
    if size_mb > mobile_svc.MAX_FILE_SIZE_MB:
        raise HTTPException(413, f"File exceeds {mobile_svc.MAX_FILE_SIZE_MB}MB limit")

    # Validate magic bytes
    if not mobile_svc.validate_magic_bytes(ext, content):
        raise HTTPException(400, "File magic bytes don't match extension")

    # Detect platform
    platform = mobile_svc.detect_platform(file.filename, content)

    # Upload to R2
    r2_key = f"mobile/{org_id}/{project_id}/{uuid4()}{ext}"
    await r2.put_object(r2_key, content, file.content_type or "application/octet-stream")

    # Create DB record
    upload = MobileUpload(
        organization_id=org_id, project_id=project_id, uploaded_by=current_user.id,
        filename=file.filename, file_extension=ext, file_size_bytes=len(content),
        platform=platform, r2_key=r2_key,
    )
    db.add(upload)
    await db.commit()
    await db.refresh(upload)

    return {
        "upload_ref": f"r2://{r2_key}",
        "platform_detected": platform.value,
        "file_size_mb": round(size_mb, 2),
        "filename": file.filename,
        "upload_id": str(upload.id),
    }
```

**Step 3: Run tests**
```bash
pytest tests/routes/test_mobile.py -v
```

**Step 4: Commit**
```bash
git add src/app/services/r2.py src/app/services/mobile.py src/app/routes/mobile.py
git commit -m "feat(mobile): add APK/IPA/ZIP upload endpoint with R2 storage + magic byte validation"
```

---

## Milestone 4: Scan Orchestration & Redis Contracts (Week 2-3)

### Task 4.1: Redis connection + queue wrapper

**Files:**
- Create: `src/app/services/redis.py`
- Test: `tests/services/test_redis.py`

**Step 1: Failing test**

```python
@pytest.mark.asyncio
async def test_dispatch_job_to_queue():
    queue = ScanQueue()
    job = {"id": "job_123", "scan_id": "scn_456", "engine": "nuclei"}
    await queue.dispatch("normal", job)
    popped = await queue.pop("normal", timeout=1)
    assert popped["id"] == "job_123"

@pytest.mark.asyncio
async def test_publish_progress_event():
    publisher = ProgressPublisher(scan_id="scn_123")
    subscriber = publisher.subscribe()
    await publisher.publish({"event_type": "job_progress", "progress": 50})
    msg = await subscriber.get(timeout=1)
    assert msg["progress"] == 50
```

**Step 2: Implement**

```python
# src/app/services/redis.py
import json
import redis.asyncio as aioredis
from app.config import settings

redis_client = aioredis.from_url(settings.REDIS_URL, decode_responses=True)

class ScanQueue:
    QUEUE_KEY_PREFIX = "shieldscan:queue:"

    async def dispatch(self, priority: str, job: dict):
        await redis_client.lpush(f"{self.QUEUE_KEY_PREFIX}{priority}", json.dumps(job))

    async def pop(self, priority: str, timeout: int = 0):
        result = await redis_client.brpop(f"{self.QUEUE_KEY_PREFIX}{priority}", timeout=timeout)
        return json.loads(result[1]) if result else None

class ProgressPublisher:
    def __init__(self, scan_id: str):
        self.channel = f"shieldscan:progress:{scan_id}"

    async def publish(self, event: dict):
        await redis_client.publish(self.channel, json.dumps(event))

class CancelPublisher:
    async def publish_cancel(self, scan_id: str, reason: str = "user_requested"):
        event = {"event_type": "cancel_requested", "scan_id": scan_id, "reason": reason}
        await redis_client.publish(f"shieldscan:cancel:{scan_id}", json.dumps(event))
```

**Step 3-5: Tests + commit**

---

### Task 4.2: Scan orchestrator service

**Files:**
- Create: `src/app/services/orchestrator.py`
- Test: `tests/services/test_orchestrator.py`

**Step 1: Failing tests**

```python
async def test_orchestrator_creates_jobs_for_scan_type_mobile(db_session, sample_scan_mobile):
    orch = ScanOrchestrator()
    jobs = await orch.dispatch(sample_scan_mobile)
    assert len(jobs) == 1
    assert jobs[0].engine == "mobsf"

async def test_orchestrator_creates_jobs_for_full_web(db_session, sample_scan_full_web):
    orch = ScanOrchestrator()
    jobs = await orch.dispatch(sample_scan_full_web)
    job_engines = [j.engine for j in jobs]
    # Recon runs first as a single coordinator job
    assert "nuclei" in job_engines
    assert "zap" in job_engines
    assert "sslyze" in job_engines
    assert "wapiti" in job_engines
    assert "nmap" in job_engines

async def test_orchestrator_idempotency_key_format(db_session, sample_scan):
    orch = ScanOrchestrator()
    jobs = await orch.dispatch(sample_scan)
    for job in jobs:
        assert job.idempotency_key.startswith(str(sample_scan.id))
        assert ":" in job.idempotency_key
```

**Step 2: Implement**

```python
# src/app/services/orchestrator.py
from uuid import uuid4
import time
from app.models.scans import Scan, ScanJob, ScanType
from app.services.redis import ScanQueue

# Scan type → tools mapping (mirrors tool_router.go in Go worker)
SCAN_TYPE_TOOLS = {
    ScanType.QUICK: ["subfinder", "httpx", "nuclei_fast", "sslyze"],
    ScanType.FULL_WEB: ["subfinder", "httpx", "nuclei", "zap", "wapiti", "nikto", "nmap", "sslyze"],
    ScanType.FULL_WEB_SOURCE: [
        "subfinder", "httpx", "nuclei", "zap", "wapiti", "nikto", "nmap", "sslyze",
        "semgrep", "gitleaks", "dependency_check"
    ],
    ScanType.API: ["nuclei_api", "zap_api", "corstest"],
    ScanType.MOBILE: ["mobsf"],
    ScanType.CONTAINER: ["trivy_image", "checkov"],
    ScanType.FULL_SPECTRUM: [
        "subfinder", "httpx", "nuclei", "zap", "wapiti", "nikto", "nmap", "sslyze",
        "semgrep", "gitleaks", "dependency_check", "trivy", "corstest", "checkov"
    ],
}

class ScanOrchestrator:
    def __init__(self):
        self.queue = ScanQueue()

    async def dispatch(self, scan: Scan) -> list[ScanJob]:
        tools = SCAN_TYPE_TOOLS[scan.scan_type]
        jobs = []
        now = int(time.time())

        for tool in tools:
            job = ScanJob(
                scan_id=scan.id,
                organization_id=scan.organization_id,
                engine=tool,
                target_url=scan.project.target_url if hasattr(scan, 'project') else "",
                status="queued",
                idempotency_key=f"{scan.id}:{tool}:{now}",
            )
            jobs.append(job)

        # Persist jobs
        for j in jobs:
            db_session.add(j)
        await db_session.commit()

        # Dispatch to Redis queue — build payload per Specification §7.1
        for job in jobs:
            payload = await self._build_job_payload(scan, job)
            await self.queue.dispatch(scan.priority, payload)

        return jobs

    async def _build_job_payload(self, scan: Scan, job: ScanJob) -> dict:
        payload = {
            "id": str(job.id),
            "scan_id": str(scan.id),
            "organization_id": str(scan.organization_id),
            "engine": job.engine,
            "idempotency_key": job.idempotency_key,
            "target": {
                "url": scan.project.target_url,
                "target_type": self._target_type(scan),
                "domain_verified": scan.project.domain_verified,
            },
            "config": scan.config,
            "callback_channel": f"shieldscan:progress:{scan.id}",
            "created_at": datetime.utcnow().isoformat() + "Z",
        }
        # Mobile-specific
        if scan.scan_type == ScanType.MOBILE and scan.mobile_upload_id:
            upload = await db_session.get(MobileUpload, scan.mobile_upload_id)
            payload["mobile_config"] = {
                "upload_ref": f"r2://{upload.r2_key}",
                "platform": upload.platform.value,
                "analysis_type": "both",
            }
        return payload
```

**Step 3-5: Tests + commit**

---

### Task 4.3: Scan creation endpoint

**Files:**
- Create: `src/app/routes/scans.py`
- Create: `src/app/schemas/scans.py`
- Test: `tests/routes/test_scans.py`

**Step 1: Failing tests**

```python
async def test_create_full_web_scan_dispatches_jobs(client, auth_headers, sample_project):
    r = await client.post(
        f"/v1/orgs/{sample_project.organization_id}/projects/{sample_project.id}/scans",
        json={"scan_type": "full_web", "recon_enabled": True, "subdomain_limit": 100},
        headers=auth_headers,
    )
    assert r.status_code == 201
    data = r.json()
    assert data["status"] == "queued"
    assert len(data["jobs"]) == 8  # 8 tools for full_web

async def test_create_mobile_scan_requires_upload_ref(client, auth_headers, sample_project):
    r = await client.post(
        f"/v1/orgs/{sample_project.organization_id}/projects/{sample_project.id}/scans",
        json={"scan_type": "mobile", "mobile_config": {}},  # missing upload_ref
        headers=auth_headers,
    )
    assert r.status_code == 400

async def test_create_mobile_scan_succeeds(client, auth_headers, sample_project, sample_mobile_upload):
    r = await client.post(
        f"/v1/orgs/{sample_project.organization_id}/projects/{sample_project.id}/scans",
        json={
            "scan_type": "mobile",
            "mobile_config": {
                "upload_ref": f"r2://{sample_mobile_upload.r2_key}",
                "platform": "android",
                "analysis_type": "both",
            },
        },
        headers=auth_headers,
    )
    assert r.status_code == 201

async def test_create_scan_requires_domain_verified(client, auth_headers, unverified_project):
    r = await client.post(
        f"/v1/orgs/{unverified_project.organization_id}/projects/{unverified_project.id}/scans",
        json={"scan_type": "full_web"},
        headers=auth_headers,
    )
    assert r.status_code == 403
    assert "domain not verified" in r.json()["error"]["message"].lower()
```

**Step 2-5: Implement + commit**

---

### Task 4.4: SSE scan progress endpoint

**Files:**
- Create: `src/app/routes/scan_progress.py`
- Test: `tests/routes/test_scan_progress.py`

**Step 1: Failing test**

```python
async def test_sse_stream_receives_progress_event(client, auth_headers, sample_running_scan):
    # Publish progress event
    await redis_client.publish(
        f"shieldscan:progress:{sample_running_scan.id}",
        json.dumps({"event_type": "job_progress", "progress": 50})
    )

    async with client.stream("GET",
        f"/v1/orgs/{sample_running_scan.organization_id}/scans/{sample_running_scan.id}/progress",
        headers=auth_headers
    ) as resp:
        assert resp.status_code == 200
        async for line in resp.aiter_lines():
            if line.startswith("data: "):
                data = json.loads(line[6:])
                assert data["progress"] == 50
                break
```

**Step 2: Implement with FastAPI StreamingResponse**

```python
# src/app/routes/scan_progress.py
from fastapi import APIRouter, Depends, Request
from sse_starlette.sse import EventSourceResponse
from app.services.redis import redis_client

router = APIRouter()

@router.get("/orgs/{org_id}/scans/{scan_id}/progress")
async def stream_progress(org_id: UUID, scan_id: UUID, request: Request,
                           current_user: User = Depends(get_current_user)):
    async def event_generator():
        pubsub = redis_client.pubsub()
        await pubsub.subscribe(f"shieldscan:progress:{scan_id}")
        try:
            async for message in pubsub.listen():
                if await request.is_disconnected():
                    break
                if message["type"] == "message":
                    data = json.loads(message["data"])
                    yield {"event": data.get("event_type", "message"), "data": json.dumps(data)}
        finally:
            await pubsub.unsubscribe()

    return EventSourceResponse(event_generator())
```

**Step 3-5: Tests + commit**
```bash
git commit -m "feat(scans): add SSE progress streaming endpoint"
```

---

### Task 4.5: Scan cancellation endpoint

**Files:**
- Modify: `src/app/routes/scans.py`

**Step 1: Failing test** — cancel sets status, publishes cancel event, returns 204.

**Step 2: Implement** — set `scans.status = 'canceled'`, publish cancel to Redis, return 204.

**Step 3: Commit**
```bash
git commit -m "feat(scans): add scan cancellation endpoint + Redis cancel event"
```

---

### Task 4.6: Scan comparison endpoint (v3 gap #15)

**Files:**
- Modify: `src/app/routes/scans.py`
- Create: `src/app/services/scan_compare.py`

**Step 1: Failing test**

```python
async def test_compare_two_scans_returns_diff(client, auth_headers, baseline_scan, current_scan):
    r = await client.post("/v1/orgs/.../scans/compare",
        json={"baseline_scan_id": baseline_scan.id, "current_scan_id": current_scan.id},
        headers=auth_headers)
    assert r.status_code == 200
    data = r.json()
    assert "new_vulnerabilities" in data
    assert "resolved_vulnerabilities" in data
    assert "persisting_vulnerabilities" in data
```

**Step 2: Implement** — compare by fingerprint; new/resolved/persisting categorization.

**Step 3: Commit**
```bash
git commit -m "feat(scans): add scan comparison endpoint for regression detection"
```

---

*[Continued in next section — Milestones 5-15]*

---

## Milestone 5: Go Worker Foundation (Week 3)

### Task 5.1: Scaffold shieldscan-engine repo

**Files:**
- Create: `shieldscan-engine/go.mod`
- Create: `shieldscan-engine/cmd/worker/main.go`
- Create: `shieldscan-engine/internal/config/config.go`
- Create: `shieldscan-engine/.env.example`

**Step 1: Initialize**
```bash
mkdir shieldscan-engine && cd shieldscan-engine
go mod init github.com/odyssey/shieldscan-engine
```

**Step 2: go.mod dependencies**
```go
module github.com/odyssey/shieldscan-engine

go 1.22

require (
    github.com/hibiken/asynq v0.24.1
    github.com/redis/go-redis/v9 v9.4.0
    github.com/docker/docker v25.0.0+incompatible
    github.com/lib/pq v1.10.9
    github.com/aws/aws-sdk-go-v2 v1.24.0
    github.com/stretchr/testify v1.8.4
    github.com/rs/zerolog v1.31.0
    github.com/spf13/cobra v1.8.0
)
```

**Step 3: cmd/worker/main.go**
```go
package main

import (
    "context"
    "os"
    "os/signal"
    "syscall"

    "github.com/rs/zerolog/log"
    "github.com/odyssey/shieldscan-engine/internal/config"
    "github.com/odyssey/shieldscan-engine/internal/worker"
)

func main() {
    cfg := config.Load()
    ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer cancel()

    w, err := worker.New(cfg)
    if err != nil { log.Fatal().Err(err).Msg("worker init failed") }

    if err := w.Startup(ctx); err != nil {
        log.Fatal().Err(err).Msg("worker startup failed")
    }

    log.Info().Msg("worker running")
    if err := w.Run(ctx); err != nil {
        log.Fatal().Err(err).Msg("worker run failed")
    }
}
```

**Step 4: Commit**
```bash
git init && git add -A
git commit -m "feat: scaffold shieldscan-engine Go worker"
```

---

### Task 5.2: ToolRunner interface + NativeRunner

**Files:**
- Create: `internal/tools/runner.go`
- Create: `internal/tools/native.go`
- Create: `internal/tools/fingerprint.go`
- Test: `internal/tools/native_test.go`

**Step 1: Failing test**

```go
func TestNativeRunner_Executes(t *testing.T) {
    runner := &NativeRunner{
        ToolName:     "echo",
        ToolCategory: "test",
        BinaryPath:   "/bin/echo",
        Timeout:      5 * time.Second,
        BuildArgs: func(target Target, cfg ScanConfig) []string {
            return []string{"hello"}
        },
        ParseOutput: func(out []byte) ([]RawFinding, error) {
            return []RawFinding{{Title: strings.TrimSpace(string(out))}}, nil
        },
    }
    findings, err := runner.Run(context.Background(), Target{URL: "http://test.com"}, ScanConfig{})
    require.NoError(t, err)
    assert.Equal(t, "hello", findings[0].Title)
    assert.NotEmpty(t, findings[0].Fingerprint)
}

func TestFingerprint_Deterministic(t *testing.T) {
    f1 := RawFinding{ToolName: "nuclei", FindingType: "xss", TargetURL: "https://x.com", Parameter: "q"}
    f2 := RawFinding{ToolName: "nuclei", FindingType: "xss", TargetURL: "https://x.com", Parameter: "q"}
    assert.Equal(t, computeFingerprint(f1), computeFingerprint(f2))
}
```

**Step 2: Implement** — full ToolRunner, Target, ScanConfig, RawFinding structs per TOOL-ARCHITECTURE.md §3–4.

**Step 3: Run tests + commit**
```bash
go test ./internal/tools/... -v
git commit -m "feat(engine): add ToolRunner interface + NativeRunner + fingerprinting"
```

---

### Task 5.3: DockerServiceRunner

**Files:**
- Create: `internal/tools/docker_service.go`
- Test: `internal/tools/docker_service_test.go`

**Step 1: Failing test** — mock HTTP server simulating tool API, verify runner calls endpoint and returns findings.

**Step 2: Implement** — per TOOL-ARCHITECTURE.md §3.3.

**Step 3: Commit**
```bash
git commit -m "feat(engine): add DockerServiceRunner for persistent Docker APIs"
```

---

### Task 5.4: Redis queue consumer + progress publisher

**Files:**
- Create: `internal/redis/queue.go`
- Create: `internal/redis/pubsub.go`
- Test: `internal/redis/queue_test.go`

**Step 1: Failing tests** — pop job from queue, publish progress to scan channel, subscribe to cancel channel.

**Step 2: Implement**

```go
// internal/redis/queue.go
type JobConsumer struct {
    client *redis.Client
}

func (c *JobConsumer) Pop(ctx context.Context, priorities []string, timeout time.Duration) (*ScanJob, error) {
    keys := make([]string, len(priorities))
    for i, p := range priorities {
        keys[i] = "shieldscan:queue:" + p
    }
    result, err := c.client.BRPop(ctx, timeout, keys...).Result()
    if err != nil { return nil, err }

    var job ScanJob
    if err := json.Unmarshal([]byte(result[1]), &job); err != nil {
        return nil, fmt.Errorf("invalid job payload: %w", err)
    }
    return &job, nil
}

// internal/redis/pubsub.go
type ProgressPublisher struct {
    client *redis.Client
    scanID string
}

func (p *ProgressPublisher) Publish(ctx context.Context, eventType string, payload map[string]interface{}) error {
    payload["event_type"] = eventType
    payload["scan_id"] = p.scanID
    payload["timestamp"] = time.Now().Format(time.RFC3339)
    data, _ := json.Marshal(payload)
    return p.client.Publish(ctx, "shieldscan:progress:"+p.scanID, data).Err()
}
```

**Step 3-5: Tests + commit**

---

### Task 5.5: Worker processing loop + job idempotency

**Files:**
- Create: `internal/worker/processor.go`
- Test: `internal/worker/processor_test.go`

**Step 1: Failing tests** — duplicate idempotency_key is silently dropped, canceled jobs don't retry, failed jobs retry up to 3x.

**Step 2: Implement**

```go
// internal/worker/processor.go
func (w *Worker) ProcessJob(ctx context.Context, job *ScanJob) error {
    // Idempotency check
    key := "shieldscan:idem:" + job.IdempotencyKey
    ok, _ := w.redis.SetNX(ctx, key, "1", 24*time.Hour).Result()
    if !ok {
        log.Info().Str("idempotency_key", job.IdempotencyKey).Msg("duplicate job dropped")
        return nil
    }

    publisher := w.newProgressPublisher(job.ScanID)
    publisher.Publish(ctx, "job_started", map[string]interface{}{
        "job_id": job.ID, "engine": job.Engine, "target": job.Target.URL,
    })

    // Create cancellable context
    scanCtx, cancel := context.WithCancel(ctx)
    defer cancel()

    // Listen for cancel
    go w.watchCancel(scanCtx, job.ScanID, cancel)

    // Get tool runner
    runner, err := w.registry.Get(job.Engine)
    if err != nil {
        publisher.Publish(ctx, "job_failed", map[string]interface{}{"error": err.Error()})
        return err
    }

    // Execute tool
    findings, err := runner.Run(scanCtx, job.Target, job.Config)
    if err != nil {
        if errors.Is(scanCtx.Err(), context.Canceled) {
            publisher.Publish(ctx, "job_canceled", map[string]interface{}{})
            return nil
        }
        publisher.Publish(ctx, "job_failed", map[string]interface{}{"error": err.Error()})
        return err
    }

    // Enrich with scan context
    for i := range findings {
        findings[i].ScanID = job.ScanID
        findings[i].OrgID = job.OrganizationID
    }

    // Store + publish completion
    if err := w.storage.StoreFindings(ctx, findings); err != nil {
        return err
    }
    publisher.Publish(ctx, "job_completed", map[string]interface{}{
        "engine": job.Engine, "finding_count": len(findings),
    })
    return nil
}
```

**Step 3-5: Tests + commit**

---

### Task 5.6: Worker startup + health check

**Files:**
- Create: `internal/worker/startup.go`
- Create: `internal/worker/health.go`
- Test: `internal/worker/startup_test.go`

**Step 1: Failing test** — startup verifies native binaries exist, Docker services respond to health endpoints, worker registers in Redis.

**Step 2: Implement** — full startup sequence per TOOL-ARCHITECTURE.md §11.

**Step 3: Commit**
```bash
git commit -m "feat(engine): worker startup with health checks for all 19 tools"
```

---

## Milestone 6: Native Tool Runners (Week 3-4)

### Task 6.1: Nuclei runner

**Files:**
- Create: `internal/tools/nuclei.go`
- Test: `internal/tools/nuclei_test.go`
- Test fixture: `testdata/nuclei_output.jsonl`

**Step 1: Failing test**

```go
func TestNucleiParser(t *testing.T) {
    raw, _ := os.ReadFile("testdata/nuclei_output.jsonl")
    findings, err := parseNucleiOutput(raw)
    require.NoError(t, err)
    assert.Greater(t, len(findings), 0)
    assert.Equal(t, "dast", findings[0].EngineCategory)
    assert.NotEmpty(t, findings[0].CWEID)
}
```

**Step 2: Implement** per TOOL-ARCHITECTURE.md §6.1.

**Step 3: Commit**
```bash
git commit -m "feat(engine): add Nuclei native runner"
```

---

### Task 6.2: Semgrep runner

**Files:**
- Create: `internal/tools/semgrep.go`
- Test: `internal/tools/semgrep_test.go`

**Step 1: Failing test + fixture**
**Step 2: Implement** — parse JSON output, map `check_id` → finding_type, extract `path`, `start.line`, `extra.metadata.cwe`.
**Step 3: Commit**

---

### Task 6.3: Subfinder + httpx + combined Recon runner

**Files:**
- Create: `internal/tools/recon.go`
- Test: `internal/tools/recon_test.go`

**Step 1: Failing tests** — parse Subfinder output (newline-separated), parse httpx JSON lines, combine into ReconResult.

**Step 2: Implement** per TOOL-ARCHITECTURE.md §6.3-6.4.

```go
type ReconResult struct {
    Subdomains []string
    LiveHosts  []LiveHost
}

type LiveHost struct {
    URL        string
    StatusCode int
    Tech       []string
}

func RunRecon(ctx context.Context, domain string, limit int, publisher *ProgressPublisher) (*ReconResult, error) {
    publisher.Publish(ctx, "recon_started", map[string]interface{}{"domain": domain})

    // Subfinder
    subs, err := runSubfinder(ctx, domain, 60*time.Second)
    if err != nil {
        log.Warn().Err(err).Msg("subfinder failed")
        return &ReconResult{}, nil
    }
    if len(subs) > limit { subs = subs[:limit] }

    publisher.Publish(ctx, "subdomains_discovered", map[string]interface{}{
        "count": len(subs), "subdomains": subs,
    })

    if len(subs) == 0 { return &ReconResult{Subdomains: subs}, nil }

    // httpx
    liveHosts, err := runHttpx(ctx, subs, 120*time.Second)
    if err != nil {
        log.Warn().Err(err).Msg("httpx failed")
        return &ReconResult{Subdomains: subs}, nil
    }

    return &ReconResult{Subdomains: subs, LiveHosts: liveHosts}, nil
}
```

**Step 3: Commit**
```bash
git commit -m "feat(engine): add Subfinder+httpx recon runner with auto-expansion"
```

---

### Task 6.4: SSLyze runner

**Files:**
- Create: `internal/tools/sslyze.go`
- Test: `internal/tools/sslyze_test.go`

**Step 1: Failing test + fixture**
**Step 2: Implement** — parse SSLyze JSON, detect SSL 2.0/3.0 support, weak ciphers, invalid certificate chain, missing HSTS.
**Step 3: Commit**

---

### Task 6.5: Gitleaks runner

**Files:**
- Create: `internal/tools/gitleaks.go`
- Test: `internal/tools/gitleaks_test.go`

**Step 1: Failing test** — parse JSON output, every finding is `critical` severity with `CWE-798`.
**Step 2: Implement** per TOOL-ARCHITECTURE.md §6.6.
**Step 3: Commit**

---

### Task 6.6: Nikto, Wapiti, CORStest runners

**Files:**
- Create: `internal/tools/nikto.go`
- Create: `internal/tools/wapiti.go`
- Create: `internal/tools/corstest.go`
- Tests for each

**Steps:** One task per tool. Each follows the same TDD pattern — failing test with fixture, implement parser, commit.

---

### Task 6.7: Dependency-Check + Checkov runners

**Files:**
- Create: `internal/tools/dependency_check.go`
- Create: `internal/tools/checkov.go`

**Steps:** TDD, commit.

---

### Task 6.8: Tool registry

**Files:**
- Create: `internal/tools/registry.go`
- Test: `internal/tools/registry_test.go`

**Step 1: Failing test**

```go
func TestRegistry_GetByName(t *testing.T) {
    reg := NewRegistry()
    reg.Register("nuclei", NewNucleiRunner())
    reg.Register("sslyze", NewSSLyzeRunner())

    r, err := reg.Get("nuclei")
    require.NoError(t, err)
    assert.Equal(t, "nuclei", r.Name())

    _, err = reg.Get("unknown")
    assert.Error(t, err)
}
```

**Step 2: Implement + register all 11 native tools**
**Step 3: Commit**

---

## Milestone 7: Persistent Docker Service Runners (Week 4-5)

### Task 7.1: MobSF runner

**Files:**
- Create: `internal/tools/mobsf.go`
- Test: `internal/tools/mobsf_test.go`

**Step 1: Failing tests** — mock MobSF API with upload/scan/status/report endpoints, verify full flow downloads file from R2, uploads to MobSF, polls, parses report into RawFindings with mobile fields populated.

**Step 2: Implement** per TOOL-ARCHITECTURE.md §7.1 + §9.2-9.3 — full MobSFRunner with REST integration and comprehensive report parser (permissions, secrets, exported components, weak crypto, ATS misconfigs).

**Step 3: Commit**
```bash
git commit -m "feat(engine): add MobSF runner for mobile APK/IPA/source analysis"
```

---

### Task 7.2: ZAP runner

**Files:**
- Create: `internal/tools/zap.go`
- Test: `internal/tools/zap_test.go`

**Step 1: Failing tests** — mock ZAP API, verify spider → active scan → alerts flow.
**Step 2: Implement** per TOOL-ARCHITECTURE.md §7.2.
**Step 3: Commit**

---

### Task 7.3: Trivy runner (SCA + container)

**Files:**
- Create: `internal/tools/trivy.go`
- Test: `internal/tools/trivy_test.go`

**Step 1: Failing tests** — two modes: `fs` (source deps) and `image` (container).
**Step 2: Implement** per TOOL-ARCHITECTURE.md §7.3.
**Step 3: Commit**

---

### Task 7.4: SQLMap runner (conditional)

**Files:**
- Create: `internal/tools/sqlmap.go`
- Test: `internal/tools/sqlmap_test.go`

**Step 1: Failing test** — only runs when triggered by Nuclei SQLi hint. Uses sqlmapapi daemon, confirm-only mode.

**Step 2: Implement** per TOOL-ARCHITECTURE.md §7.4.

**Step 3: Commit**
```bash
git commit -m "feat(engine): add SQLMap runner for deep SQLi confirmation"
```

---

### Task 7.5: Docker warm pool for Nmap

**Files:**
- Create: `internal/docker/warm_pool.go`
- Test: `internal/docker/warm_pool_test.go`

**Step 1: Failing test** — pool maintains 2 paused Nmap containers, Get() unpauses, replenish happens in background.

**Step 2: Implement** per Spec v3 §22.

```go
type WarmPool struct {
    docker *docker.Client
    pools  map[string]chan string
    sizes  map[string]int
}

var defaultPoolSizes = map[string]int{
    "instrumentisto/nmap:latest": 2,
}

func (wp *WarmPool) Get(ctx context.Context, image string) (string, error) {
    select {
    case id := <-wp.pools[image]:
        if err := wp.docker.ContainerUnpause(ctx, id); err != nil {
            return "", err
        }
        go wp.replenish(ctx, image)
        return id, nil
    default:
        return wp.docker.CreateAndStart(ctx, image)
    }
}
```

**Step 3: Commit**

---

### Task 7.6: Nmap runner using warm pool

**Files:**
- Create: `internal/tools/nmap.go`
- Test: `internal/tools/nmap_test.go`

**Step 1: Failing test** — runner acquires container from warm pool, executes scan, releases container.
**Step 2: Implement.**
**Step 3: Commit**

---

## Milestone 8: Recon-First Pipeline (Week 5)

> **🔒 MILESTONE 8 CLOSED (2026-06-16; landed at M9 entry as natural boundary marker):** All M8 sub-tasks resolved — 8.1α CLOSED (Drift #60 1/6) + 8.1β.1 CLOSED (Drift #60 4/6) + 8.1β.2 CLOSED (Drift #60 6/6 END-TO-END + ADR-028 operational) + 8.2 N/A retired + 8.3α CLOSED (AttackSurface consumer infrastructure) + 8.3β CLOSED (attack-surface GET endpoint, api 1b7b314). M8 → M9 transition complete. See the Task 8.1 closure-annotation blockquote below + ADR-028 (SPEC §13) for the recon-first architecture.

### Task 8.1: Scan executor with recon-first target expansion

> **🔒 M8.1β.2 LIFECYCLE CLOSED — Task 8.1 SUPERSEDED BY ADR-028 (2026-06-10):**
>
> The Task 8.1 plan-literal pseudo-code below (`ScanExecutor.ExecuteScan` + `registry.Get` + `reportAttackSurface` patterns) is SUPERSEDED by ADR-028 "Scan-Executor Recon-First Architecture" at SPEC §13. Original pseudo-code is INFORMATIONAL not BINDING; ADR-013 sole-writer + ADR-022 recon-as-pre-scan-helper + ADR-028 two-phase recon-first dispatch are the canonical authority.
>
> **M8.1β.2 lifecycle artifacts:**
> - Stage 1 design doc: plans/2026-06-09-scan-executor-recon-first-design.md (commit 3f07611; 252 LoC; Q1-Q10 architectural decisions + V-W refinements)
> - Stage 2 implementation plan: plans/2026-06-09-scan-executor-recon-first-implementation.md (commit fb61129; 429 LoC; 4 plan-level Y-decisions + Stage 3 sub-step breakdown)
> - Stage 3 4-commit cross-repo trio:
>   - C1 docs 9507acb (+96 LoC): ADR-028 canonical + ADR-022 addendum continuation + ADR-020 closure + TOOL-ARCH speculative resolution
>   - C2 engine 2cf6f5d (+328/-13 LoC): processor.go recon dispatch + ProcessorDeps concrete-publisher extension + DRIFT-LOG #60 6/6 + #62 resolution
>   - C3 api orchestrator 04a9b5c (+612/-92 LoC): SCAN_TYPE_TOOLS rename + phase-1 web-ScanType branching + dispatch_phase2 method + SCAN_DISPATCHED_PHASE2 audit + Drift #63 resolution
>   - C4 api completions_consumer dc39fd1 (+320/-27 LoC): phase-2 dispatch hook + fail-loud-audit + e2e integration tests
> - Stage 4 P5.A annotations: this commit + engine DRIFT-LOG #63 entry
>
> **Aggregate Stage 3:** +1356/-132 LoC (ADR-style architectural-decision implementations at ~1300-1400 LoC density level).
>
> **Drift #60 6/6 closure END-TO-END operationally verified at e2e test** (no subfinder/httpx anywhere; recon ScanJob dispatched via orchestrator-implicit logic; per-(target, tool) jobs created via dispatch_phase2).
>
> **Drift catalog at M8.1β.2 lifecycle:**
> - Drift #60 6/6 closure END-TO-END (name-mismatch 1/1 M8.1α + engine-variant 3/3 M8.1β.1 + recon-orphan 2/2 M8.1β.2)
> - Drift #61 (V-WD Scan.created_by_user_id absence; Q7.4 refined to Option α audit-log lookup)
> - Drift #62 (V-Z processor→RunRecon interface-vs-concrete type mismatch; resolved at C2 per Sub-Decisions (B) + (B.i))
> - Drift #63 (V-AAE test-scope-incompleteness; resolved at C3 with expanded scope per Option 1)
>
> **4 plan-level Y-decisions resolved at execution:**
> - Y-AUDIT-LOOKUP-QUERY-SHAPE (a) direct ORM query
> - Y-AUTHIDENTITY-RECONSTRUCTION-SHAPE refined at execution: audit() takes actor_id/organization_id directly (AuthIdentity reconstruction abandoned)
> - Y-RECON-SCANJOB-IDEMPOTENCY-KEY (a) {scan_id}:recon:{ts}
> - Y-PHASE2-DISPATCH-FAILURE-AUDIT-SHAPE (b) diagnostic-rich
>
> **4 micro-refinements absorbed without new catalog entries** (scan-bound publisher factory + bootstrap inside NewProcessorFromRedis at C2; _build_job_payload target_url_override + AuthIdentity reconstruction abandoned at C3).
>
> **Discipline-level forward-pin chain extended at M8.1β.2 close:**
> - "Audit-driven model+spec orphan check into pre-verification template" (Drift #60 rule-of-three)
> - "DEFERRED-EMPIRICAL marking for ALL concrete-empirical-references in plan pseudo-code — field names + type signatures + interface contracts + dependency injection patterns" (Drift #61 + #62 catch-class evolution)
> - "Examine recon-invocation architectural seam at pre-verification for future engine-side dispatch additions" (Drift #59 + #62 adjacent-layer meta-pattern)
> - "Pre-verification scope completeness across test-impact-surface — grep ALL test files for behavior-change-impact assertions when ScanType/SCAN_TYPE_TOOLS/audit event/dispatch shape changes" (Drift #63 extension)
>
> **M8 closure path:** 8.1α CLOSED + 8.1β.1 CLOSED + 8.1β.2 CLOSED (this lifecycle) + 8.2 retired + 8.3α CLOSED + 8.3β PENDING (trigger: "Begin Task 8.3β attack-surface endpoint task"). After Task 8.3β → M8 declarable CLOSED → M9 entry triggered.

**Files:**
- Create: `internal/orchestrator/scan_executor.go`
- Test: `internal/orchestrator/scan_executor_test.go`

**Step 1: Failing test** — executor runs recon first if scan_type is web, expands target list with discovered live subdomains, stores attack_surface records.

**Step 2: Implement** per TOOL-ARCHITECTURE.md §8.2.

```go
func (e *ScanExecutor) ExecuteScan(ctx context.Context, job *ScanJob) error {
    publisher := e.newPublisher(job.ScanID)

    targets := []Target{job.Target}
    if isWebScan(job.Engine) {
        targets = e.buildTargetList(ctx, job, publisher)
    }

    // Store attack surface in PostgreSQL via API (separate endpoint)
    e.reportAttackSurface(ctx, job.ScanID, targets)

    // Run the tool on each target
    tool, _ := e.registry.Get(job.Engine)
    for _, target := range targets {
        findings, err := tool.Run(ctx, target, job.Config)
        if err != nil { continue }
        for i := range findings {
            findings[i].ScanID = job.ScanID
            findings[i].OrgID = job.OrganizationID
        }
        e.storage.StoreFindings(ctx, findings)
        publisher.Publish(ctx, "target_completed", ...)
    }
    return nil
}
```

**Step 3: Commit**
```bash
git commit -m "feat(engine): add recon-first scan executor with target expansion"
```

---

### Task 8.2: Scan type → tool matrix (Go side) — **RETIRED (N/A)**

> **🚫 RETIREMENT NOTICE (2026-06-07; M5/M6/M7/M8 milestone audit):**
>
> Task 8.2 is **N/A — retired** per ADR-013 sole-writer pattern (SPEC §13). Empirically `internal/orchestrator/tool_router.go` does NOT exist at engine; arc-evolution determined that an engine-side scan-type-tool matrix would create dual sources of truth for `ScanType → tool` mapping and risk drift between api dispatch decisions and engine routing decisions.
>
> **Architectural rationale:** plan-literal called for an engine-side matrix mirroring api-side dispatch. ADR-013 (orchestrator-as-sole-writer; landed before M5 implementation began) supersedes this. The api orchestrator is the canonical authority for `ScanType → tool` dispatch; the engine consumes per-tool `ScanJob` dispatches as they arrive and executes whatever tool the wire payload's `engine` field names.
>
> **Canonical location:** `shieldscan-api` `src/app/services/orchestrator.py` — `SCAN_TYPE_TOOLS: Final[dict[ScanType, tuple[str, ...]]]` (per-ScanType tool tuples; module-import assertion ensures every non-`PUBLIC` `ScanType` has a mapping).
>
> **Audit reference:** M5/M6/M7/M8 milestone audit (shieldscan-docs `ac82d48` P5.A close + V-K-F surface report this session) classified Task 8.2 as N/A; M5 + M6 + M7 declared CLOSED; M8 remaining gaps are Task 8.1 (scan-executor; arc-evolution-pivot territory; brainstorming forward-pinned to fresh session) + Task 8.3 (attack-surface API endpoint; mechanical gap; compressed-lifecycle eligible).
>
> **Cross-references:** SPEC §13 ADR-013 (sole-writer canonical); shieldscan-api `services/orchestrator.py` `SCAN_TYPE_TOOLS` (api-side canonical matrix); shieldscan-api commit `8dbcbab` (source-ingestion fix Stage 3 C2; latest api state extending the matrix surface).

**Files (RETIRED; do not implement):**
- ~~Create: `internal/orchestrator/tool_router.go`~~
- ~~Test: `internal/orchestrator/tool_router_test.go`~~

**~~Step 1: Failing tests~~** ~~— matrix returns correct tool list per scan type, mobile scan returns only mobsf, full_spectrum returns all.~~

**~~Step 2: Implement~~** ~~per TOOL-ARCHITECTURE.md §10.1.~~

**~~Step 3: Commit~~**

---

### Task 8.3: Attack surface API endpoint

> **📌 STATUS (2026-06-10):** 8.3α CLOSED (Task 8.3α infrastructure at fc75a98 + 05023f4); 8.3β PENDING activation (trigger: "Begin Task 8.3β attack-surface endpoint task").

**Files:**
- Create: `shieldscan-api/src/app/routes/attack_surface.py`
- Test: `shieldscan-api/tests/routes/test_attack_surface.py`

**Step 1: Failing test** — GET endpoint returns root_domain, total_discovered, live/dead counts, per-subdomain status + tech_stack + vuln_count.

**Step 2: Implement** — join attack_surface + count vulnerabilities per subdomain URL.

**Step 3: Commit**
```bash
git commit -m "feat(api): add attack surface endpoint"
```

---

## Milestone 9: AI Analysis Pipeline (Week 5-6)

> **🔒 MILESTONE 9 — AI ANALYSIS PIPELINE — ENTIRELY CLOSED (2026-07-02).** All 5 sub-milestones closed: **M9.0** (foundation; ADR-029) + **M9.A** (embed/dedup; ADR-030) + **M9.B** (correlate/score; ADR-031) + **M9.C** (fix-gen/summary; ADR-032; FIRST real Anthropic Claude integration; cost-tracking operational per Gotcha 5) + **M9.D** (orchestrator; ADR-033; closure-by-composition). Pipeline operational end-to-end: completions-event → dispatch → embed → dedup → correlate → score → fix-gen → summary → terminal metadata (completed_at/error_message); 751 tests green. ⇒ **M10 activation trigger:** ***"Begin M10 — Report Architecture"***.

> **🔒 M9.0 — AI ANALYSIS PIPELINE FOUNDATION (ADR-029) — LIFECYCLE CLOSED (2026-06-16):**
>
> M9 is decomposed into sub-milestones per ADR-029 Q11 (strict linear): **M9.0** (this foundation) → **M9.A** (embed/dedup; Tasks 9.1+9.2) → **M9.B** (correlate/score; Tasks 9.3+9.4) → **M9.C** (fix-gen/summary; Tasks 9.5+9.6) → **M9.D** (orchestrator; Task 9.7). The Task 9.1-9.7 plan-literal pseudo-code below is INFORMATIONAL not BINDING; ADR-029 (SPEC §13) is the canonical authority, and the sub-milestones land the real pipeline stages.
>
> **M9.0 lifecycle artifacts (8-commit chain):**
> - Stage 1 design doc: plans/2026-06-12-m9-ai-pipeline-foundation-design.md (commit a46fedd; 240 LoC §1-§9; Q1-Q12 locks)
> - Stage 2 implementation plan: plans/2026-06-12-m9-ai-pipeline-foundation-implementation.md (commit 55dbe32; 293 LoC; 4 plan-level Y-decisions)
> - Stage 3 C0 docs ADR-029 at SPEC §13 (commit 45dcabe; +73 LoC; 9 phases + Drift #64/#65 architectural closure)
> - Stage 3 C1 api schema + modules (commit 51b26ea; +427 LoC; ai_api_calls migration + AIAPICall + Scan/Vulnerability/raw_finding columns + AI clients + cost-tracking + lifespan; Drift #64/#65 code resolution)
> - Stage 3 C2 api consumer + dispatch hook (commit 1c98330; +398/-40 LoC; AIPipelineConsumer + dispatch_ai_pipeline + completions_consumer interposition + DQ1-DQ5)
> - Stage 3 C3 api tests + smoke (commit 8410df4; +704 LoC; 25 tests covering clients/cost-tracking/consumer/e2e no-op flow)
> - Stage 4 P5.A docs annotations (this commit) + persistent DRIFT-LOG sync (api + engine commits)
>
> **What M9.0 lands (foundation only; no real AI calls — Q11 C.c-lite no-op smoke):**
> - ADR-029 two-phase architecture: completions_consumer (all jobs terminal + raw_findings) → orchestrator.dispatch_ai_pipeline (Scan ANALYZING + LPUSH shieldscan:ai_pipeline + SCAN_DISPATCHED_AI_PIPELINE) → AIPipelineConsumer (no-op pipeline) → consumer-driven COMPLETED/PARTIAL + SCAN_COMPLETED (DQ2/DQ5)
> - ai_api_calls cost-tracking table + per-scan budget + circuit-breaker scaffolding (Q2; closes CLAUDE.md Gotcha 5 gap)
> - AI provider client singletons (OpenAI + Anthropic + Qdrant) at app lifespan (Q7)
> - Schema foundation: Scan.executive_summary + Scan.ai_pipeline_degraded + Vulnerability.qdrant_point_id + Vulnerability.raw_finding_ids + raw_findings.promoted_at + raw_findings.vulnerability_id (Q3/Q5/Q6/Q8)
> - DQ3 empty-findings fast-path (no AI dispatch for findings-free scans); DQ4 ANALYZING idempotency guard
>
> **Drift catalog at M9.0 lifecycle:**
> - Drift #64 (ai_api_calls absence; Drift #60 catch-class 4th-instance) — architectural closure at ADR-029 + code resolution at C1
> - Drift #65 (Scan.executive_summary absence; Drift #61 catch-class 5th-instance) — architectural closure at ADR-029 + code resolution at C1
> - Drift #66-averted (V-JJC predicted Path β/γ test-impact at C2; DQ3 design choice averted manifestation — 0 tests broken; no catalogue increment per averted-prediction discipline established at V-IIB/C1)
>
> **Cumulative session-tail framing-drift count at M9.0 close: 65** (unchanged through M9.0 lifecycle).
>
> **M9.A activation trigger:** ***"Begin M9.A — Embedding + Deduplication"***.

> **🔒 M9.A — AI PIPELINE: EMBEDDING + DEDUPLICATION (ADR-030) — LIFECYCLE CLOSED (2026-06-23):**
>
> First real-AI sub-milestone (Tasks 9.1+9.2). Composes ADR-029 by activating the M9.0 no-op `run_no_op` scaffold into the real embed → dedup → Path C promotion pipeline. ADR-030 (SPEC §13) is the canonical authority.
>
> **M9.A lifecycle artifacts (8-commit chain):**
> - Stage 1 design doc: plans/2026-06-16-m9a-embedding-dedup-design.md (commit aaf7ea0; §1-§9; Y-PROMOTION-TIMING Path C + Q1-Q12 + 35+ sub-decisions + §9 V-MM/V-NNE pre-grounding)
> - Stage 2 implementation plan: plans/2026-06-17-m9a-embedding-dedup-implementation.md (commit a8ad52c; PY1-PY4 plan-level Y-decisions + Stage 3 3-commit sub-step breakdown)
> - Stage 3 C0 docs ADR-030 at SPEC §13 (commit d408b2c; +94 LoC; between ADR-029 + §14)
> - Stage 3 C1 api implementation (commit 91ec273; +401/-23 LoC; pipeline.py rewrite — run() + 4 stage helpers; Q5 B.c→query_points + Q9 B.a→AsyncQdrantClient(":memory:") deferred-resolutions locked at execution; Drift #66 catalogued + resolved)
> - Stage 3 C2 api tests + smoke (commit 251960a; +1097 LoC; 35 new tests + full suite 657 ZERO regressions; Drift #66 regression-guard operational; Y2 activation verified)
> - Stage 4 P5.A docs annotations (this commit) + persistent DRIFT-LOG sync (api + engine commits forthcoming)
>
> **What M9.A lands (first real AI calls; SPEC §8.1 stages [1]+[2]):**
> - Embedding via OpenAI text-embedding-3-small (1536-dim) — batch=100, hybrid 429 retry, rule-based fingerprint fallback → ai_pipeline_degraded (Q2/Q3 + SPEC §8.6)
> - Dedup via Qdrant cosine ≥0.92, per-scan filter, deterministic uuid5 point ids (Q5/Q6)
> - Path C: first finding of each cluster → Vulnerability (project_id derived from Scan per PY2/V-NNE); subsequent matches append raw_finding_id via _merge_evidence (Q7)
> - Per-batch ai_api_calls cost logging (Q8; activates CLAUDE.md Gotcha 5 first real-cost path)
>
> **Y2 Task 8.3β vulnerability_count forward-pin ACTIVATED:** the attack-surface endpoint join (Vulnerability.target_url == AttackSurface.full_url filtered by scan_id) returns actual counts post-pipeline; pre-M9.A returned 0. Verified at test_m9a_smoke_y2_vulnerability_count_activation (251960a).
>
> **Drift catalog at M9.A lifecycle:**
> - Drift #66 (Resolution γ FK-ordering — NOVEL "ORM-vs-DB layer assumption mismatch" catch-class; first instance in arc) — surfaced at C1 test execution (raw_findings.vulnerability_id FK requires Vulnerability persistence before the raw_finding UPDATE; SQLAlchemy UOW can't infer order without a relationship()); resolved via intermediate flush in _create_vulnerability_from_finding; regression-guarded at C2. Persistent api + engine DRIFT-LOG sync at P5.A Commits 2+3.
> - Disambiguation: distinct from the prior "#66-averted lineage" documentation shorthand (V-JJC/DQ3 at M9.0 C2; V-MM/V-NNE at M9.A Stage 1) — that used #66 as a would-be-next-number placeholder and never incremented (count stayed 65). Drift #66 is the first real increment (65→66).
>
> **Cumulative session-tail framing-drift count at M9.A close: 66** (Drift #58-#66; +1 since M9.0 close for the FK-ordering catch).
>
> **M9.B activation trigger:** ***"Begin M9.B — Correlation + Scoring"*** (Tasks 9.3+9.4; per Q11-M9.0 strict linear sequencing).

> **🔒 M9.B — AI PIPELINE: CORRELATION + SCORING (ADR-031) — LIFECYCLE CLOSED (2026-06-29):**
>
> Second real-AI sub-milestone (Tasks 9.3+9.4). Composes ADR-030 by extending `pipeline.run()` with cross-layer correlation + composite severity scoring. ADR-031 (SPEC §13) is the canonical authority. Y-CORRELATION-MERGE-VS-LINK **Path B** (link + corroborate; no row deletion) preserves the M9.A Path C lock literally.
>
> **M9.B lifecycle artifacts (9-commit chain):**
> - Stage 1 design doc: plans/2026-06-25-m9b-correlation-scoring-design.md (commit d10be51; Path B gate + Q1-Q18 + ~45+ sub-decisions + §9 V-UUE tool_name pre-grounding)
> - Stage 2 implementation plan: plans/2026-06-25-m9b-correlation-scoring-implementation.md (commit a172fd1; PY1-PY8 + 4-commit Stage 3 breakdown + D-deviation forecasts)
> - Stage 3 C0 docs ADR-031 at SPEC §13 (commit fb4b07e; +117 LoC; between ADR-030 + §14)
> - Stage 3 C1 api schema + modules (commit 5dee684; +446 LoC; alembic ecfed70e05e4 migration — correlation_cluster_id indexed + severity_score; cwe_hierarchy.py + correlation.py + scoring.py NEW; PY5 tool_name + PY6 lower-FK-risk validated; 0 drifts at the strongest-risk commit)
> - Stage 3 C2 api pipeline integration (commit 6c9e270; +310 LoC; `_correlate_vulnerabilities` + `_score_vulnerabilities` wired into run(); Q1 evidence-loading correction caught pre-commit by the test gate; within-lock, not catalogued)
> - Stage 3 C3 api tests + smoke (commit f82c38a; +584 LoC; 41 new tests; full suite 709 ZERO regressions; fixture-scoping correction caught pre-commit by the test gate; within-lock, not catalogued)
> - Stage 4 P5.A docs annotations (this commit) + persistent DRIFT-LOG sync (api + engine commits forthcoming)
>
> **What M9.B lands (deterministic; no AI calls per Q14; SPEC §8.2 + §8.3):**
> - Cross-layer correlation: rule-based weighted scoring (cwe_exact 0.40 + cwe_parent 0.25 + url_path 0.20 + finding_type 0.30 + parameter_name 0.15; threshold ≥0.75); cross-engine_category generalized; correlation over the representative raw_finding's evidence fields (Q1)
> - CWE hierarchy (Q5 hardcoded top-~50) + heuristic route_map (Q6) + multi-language parameter extraction (Q7)
> - Path B link: `Vulnerability.correlation_cluster_id` via union-find (transitive A↔B↔C); `corroborated_count` = engine-distinct within cluster (Q12); **no row deletion**
> - Composite scoring: `severity_score` = base_cvss × corroboration × exploitability, capped 10.0 (Q9); PoC-derivation from `RawFinding.tool_name ∈ {nuclei, sqlmap}` (Q10/PY5); CVSS-band → severity enum (Q11)
> - Schema: `correlation_cluster_id` (indexed) + `severity_score` columns (migration `ecfed70e05e4`, revises `b7e4a1f93c2d`)
>
> **Drift catalog at M9.B lifecycle:** **none** — cumulative count **preserved at 66** through all of M9.B. Drift #66 (M9.A C1) remains the latest catalogue entry. C1+C2+C3 landed at 0 catalogued drifts via PY5 (tool_name pre-grounding), PY6 (standalone-column lower-FK-risk), and test-gate discipline (the C2 Q1-evidence and C3 fixture-scoping corrections were within-lock under-implementations caught pre-commit — not Y-lock deviations, not increments).
>
> **Cumulative session-tail framing-drift count at M9.B close: 66** (unchanged since M9.A).
>
> **M9.C activation trigger:** ***"Begin M9.C — Fix Generation + Executive Summary"*** (Tasks 9.5+9.6; per Q11-M9.0 strict linear sequencing).

### Task 9.1: OpenAI embeddings service

> **🔒 CLOSED at M9.A Stage 3 C1 (api 91ec273) + tests C2 (251960a); ADR-030 Path C operational.** The plan-literal pseudo-code below is INFORMATIONAL (superseded by ADR-030). Real implementation: `_embed_findings` + `_construct_embedding_input` + `_embed_batch_with_retry_and_usage` in `src/app/services/ai/pipeline.py` (batch=100, labeled multi-field input incl. target_url, hybrid 429 retry + rule-based fingerprint fallback, per-batch ai_api_calls cost logging). Tests: `tests/services/ai/test_embed_findings.py`.

**Files:**
- Create: `shieldscan-api/src/app/services/ai/embeddings.py`
- Test: `shieldscan-api/tests/services/ai/test_embeddings.py`

**Step 1: Failing test** — batch embeds 100 findings in single API call, caches by fingerprint.

**Step 2: Implement**

```python
import openai
from app.config import settings

client = openai.AsyncOpenAI(api_key=settings.OPENAI_API_KEY)

async def embed_findings(findings: list[RawFinding]) -> list[list[float]]:
    texts = [f"{f.title} {f.description} {f.finding_type} {f.cwe_id}" for f in findings]
    # Batch up to 100 at a time
    embeddings = []
    for chunk in chunks(texts, 100):
        resp = await client.embeddings.create(model="text-embedding-3-small", input=chunk)
        embeddings.extend([d.embedding for d in resp.data])
    return embeddings
```

**Step 3: Commit**

---

### Task 9.2: Qdrant deduplication

> **🔒 CLOSED at M9.A Stage 3 C1 (api 91ec273) + tests C2 (251960a); ADR-030 Path C operational.** The plan-literal pseudo-code below is INFORMATIONAL (superseded by ADR-030). Real implementation: `_dedup_and_promote` + `_ensure_collection_exists` + `_search_similar` (cosine ≥0.92, per-scan filter, `query_points` per Q5 B.c V-QQC) + `_upsert_finding` + `_create_vulnerability_from_finding` + `_merge_evidence` in `src/app/services/ai/pipeline.py`. Path C: first cluster finding → Vulnerability (project_id from Scan per PY2/V-NNE); matches append raw_finding_id. Tests: `tests/services/ai/test_dedup_and_promote.py`.

**Files:**
- Create: `src/app/services/ai/deduplication.py`
- Test: `tests/services/ai/test_deduplication.py`

**Step 1: Failing test** — two findings with cosine similarity > 0.92 are marked as duplicates, below threshold kept separate.

**Step 2: Implement**

```python
from qdrant_client import AsyncQdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

qdrant = AsyncQdrantClient(url=settings.QDRANT_URL)

async def deduplicate(findings: list[RawFinding], embeddings: list[list[float]]) -> list[RawFinding]:
    collection = f"findings_{findings[0].organization_id}"
    await qdrant.create_collection(
        collection_name=collection,
        vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
    ) if not await qdrant.collection_exists(collection) else None

    deduped = []
    for finding, emb in zip(findings, embeddings):
        hits = await qdrant.search(collection_name=collection, query_vector=emb, limit=1, score_threshold=0.92)
        if hits:
            # Duplicate — skip this finding, merge evidence into existing vuln
            await merge_evidence(hits[0].id, finding)
            continue
        # New finding — store embedding + keep it
        await qdrant.upsert(collection_name=collection, points=[
            PointStruct(id=str(finding.fingerprint), vector=emb, payload={"id": str(finding.id)})
        ])
        deduped.append(finding)
    return deduped
```

**Step 3: Commit**

---

### Task 9.3: Cross-layer correlation (DAST↔SAST)

> **🔒 CLOSED at M9.B Stage 3 C1 (api 5dee684) + C2 (6c9e270); tests at C3 (f82c38a); ADR-031 Path B operational.** The plan-literal pseudo-code below is INFORMATIONAL (superseded by ADR-031). Real implementation: `correlation.py` (CORRELATION_WEIGHTS + correlation_score + build_route_map + extract_params_* + iter_cross_engine_pairs + union_find_clusters) + `cwe_hierarchy.py` (Q5) + `_correlate_vulnerabilities` in `pipeline.py`. Path B link via `Vulnerability.correlation_cluster_id` (union-find; no row deletion); `corroborated_count` engine-distinct. Tests: `tests/services/ai/test_correlation.py` + `test_cwe_hierarchy.py`.

**Files:**
- Create: `src/app/services/ai/correlation.py`
- Test: `tests/services/ai/test_correlation.py`

**Step 1: Failing tests**

```python
def test_correlation_exact_cwe_and_finding_type_matches():
    dast = RawFinding(engine_category="dast", cwe_id="CWE-89", finding_type="sqli",
                     target_url="https://x.com/api/users", parameter="id")
    sast = RawFinding(engine_category="sast", cwe_id="CWE-89", finding_type="sqli",
                     code_file="routes/users.py", code_snippet="SELECT * FROM users WHERE id=" + "id")
    route_map = {"/api/users": "routes/users.py"}
    score = correlation_score(dast, sast, route_map)
    assert score >= 0.75  # Should correlate
```

**Step 2: Implement** per Specification §8.2 + v3 §3.

**Step 3: Commit**
```bash
git commit -m "feat(ai): add DAST-SAST cross-layer correlation with weighted scoring"
```

---

### Task 9.4: Severity scoring

> **🔒 CLOSED at M9.B Stage 3 C1 (api 5dee684) + C2 (6c9e270); tests at C3 (f82c38a); ADR-031 §8.3 operational.** The plan-literal pseudo-code below is INFORMATIONAL (superseded by ADR-031). Real implementation: `scoring.py` (compute_severity_score + compute_corroboration_multiplier + compute_exploitability_multiplier + _map_score_to_severity_enum + POC_PROVEN_TOOL_NAMES) + `_score_vulnerabilities` in `pipeline.py`. `severity_score` = base_cvss × corroboration × exploitability (capped 10.0); PoC via `RawFinding.tool_name` (Q10/PY5). Tests: `tests/services/ai/test_scoring.py`.

**Files:**
- Create: `src/app/services/ai/scoring.py`
- Test: `tests/services/ai/test_scoring.py`

**Step 1: Failing tests** — corroborated multiplier 1.3, exploitability multiplier 1.5 for PoC-proven.

**Step 2: Implement** per Specification §8.3.

**Step 3: Commit**

---

> **🔒 M9.C — AI Pipeline: Fix Generation + Executive Summary** (Tasks 9.5+9.6) — **CLOSED at Stage 3 + P5.A lifecycle closure (2026-07-02).**
>
> Stage 1 design doc: `plans/2026-06-28-m9c-fix-gen-summary-design.md` (commit `261cf10`) — 18 architectural locks (2 gate-decisions + Q1-Q16) via Mode 1 brainstorming chain.
> Stage 2 implementation plan: `plans/2026-06-28-m9c-fix-gen-summary-implementation.md` (commit `f86a7b0`) — PY1-PY8 plan-level Y-decisions + 3-commit Stage 3 sub-step breakdown + V-BB+V-CC pre-verification cascade design.
> Stage 3 C0 ADR-032 SPEC §13 canonical: `e8fbd8c`.
> Stage 3 C1 api modules + integration: `a0522a1` (fix_generation.py + summary.py NEW + pipeline.run() sequential extension; 4 V-BB averted-prediction catches + 1 test-gate-within-lock narrowing).
> Stage 3 C2 api tests + smoke: `6eff189` (36 new tests; 747 total ZERO regressions; V-CCF averted-prediction catch).
> Stage 4 P5.A Commit 1 docs annotations: this commit. P5.A Commit 2 api DRIFT-LOG + Commit 3 engine DRIFT-LOG: forthcoming.
>
> ADR-032 architectural authority operational: Path A per-vuln Claude Sonnet calls; sequential pipeline.run() extension after M9.B scoring; per-vuln check_budget + 3-attempt retry + _fallback_fix_template + threshold-based ai_pipeline_degraded (≥3 OR ≥30%); deterministic fallback summary. M9.C is THIRD real-AI sub-milestone + FIRST making real Anthropic Claude API calls; cost-tracking transitions from architectural-readiness to operational-load per CLAUDE.md Gotcha 5 mandate.
>
> **Drift catalog at M9.C lifecycle: none** — cumulative count **preserved at 66**. 7-instance averted-prediction lineage operational (V-BB 4-catch at C1 + V-CCF at C2 extended 6→7); 3-instance test-gate-within-lock pattern operational (M9.B C2 + M9.B C3 + M9.C C1 test-assertion narrowing per Option A; none catalogue increments).
>
> **M9.D activation trigger:** ***"Begin M9.D — Pipeline Orchestrator"*** (Task 9.7; per Q11-M9.0 strict linear sequencing).

---

### Task 9.5: Claude fix generation with mobile context

> **🔒 CLOSED at M9.C Stage 3 C1 (api `a0522a1`); tests at C2 (`6eff189`); ADR-032 Path A per-vuln Claude Sonnet operational.** The plan-literal pseudo-code below is INFORMATIONAL (superseded by ADR-032). Real implementation: `fix_generation.py` NEW (FIX_GEN_SYSTEM_PROMPT + FIX_GEN_USER_PROMPT_TEMPLATE + `_build_target_context` mobile/sast/dast/generic dispatch per Q1/Q3 + `_format_evidence` + `_fallback_fix_template` per Q4 + `_call_claude_with_retry` 3-attempt exp backoff per Q8 + `generate_fix` per-vuln orchestrator with check_budget + log_ai_call per Q5 + `_generate_fixes` scan-level orchestrator per Q6/Q7/Q8 B.b) wired into `pipeline.run()` per Q13; no migration per V-AAH (ai_fix_text scaffolded at M9.0 C1). Tests: `tests/services/ai/test_fix_generation.py`.

**Files:**
- Create: `src/app/services/ai/fix_generation.py`
- Test: `tests/services/ai/test_fix_generation.py`

**Step 1: Failing tests** — fix includes code block, mobile finding includes platform-specific context in prompt, failure falls back gracefully.

**Step 2: Implement**

```python
import anthropic
from app.models.vulnerabilities import Vulnerability

client = anthropic.AsyncAnthropic(api_key=settings.ANTHROPIC_API_KEY)

FIX_PROMPT_TEMPLATE = """You are a security engineer generating a precise code fix.

FINDING: {title}
SEVERITY: {severity}
CWE: {cwe_id}
DESCRIPTION: {description}

TARGET CONTEXT:
{target_context}

EVIDENCE:
{evidence}

Generate:
1. A plain-language explanation of why this is a security issue
2. The exact corrected code
3. Additional hardening recommendations

Format: Markdown with code blocks."""

async def generate_fix(vuln: Vulnerability) -> str:
    target_context = _build_target_context(vuln)
    evidence = _format_evidence(vuln)

    prompt = FIX_PROMPT_TEMPLATE.format(
        title=vuln.title, severity=vuln.severity, cwe_id=vuln.cwe_id,
        description=vuln.description, target_context=target_context, evidence=evidence,
    )

    try:
        resp = await client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=2000,
            messages=[{"role": "user", "content": prompt}],
        )
        return resp.content[0].text
    except Exception as e:
        log.error(f"fix generation failed: {e}")
        return _fallback_fix_template(vuln)

def _build_target_context(vuln: Vulnerability) -> str:
    if vuln.engine_category == "mobile":
        return f"""Platform: {vuln.mobile_os.title()}
Language: {'Kotlin/Java' if vuln.mobile_os == 'android' else 'Swift/ObjC'}
Component: {vuln.mobile_component_name or 'N/A'}
Permission: {vuln.mobile_permission or 'N/A'}
Code Location: {vuln.code_file}:{vuln.code_line}

Follow {'Android Security Guidelines' if vuln.mobile_os == 'android' else 'Apple App Transport Security guidelines'}."""
    elif vuln.engine_category == "sast":
        return f"Source code location: {vuln.code_file}:{vuln.code_line}\nLanguage inferred from file extension."
    else:
        return f"Web endpoint: {vuln.target_url}\nParameter: {vuln.parameter}"
```

**Step 3: Commit**
```bash
git commit -m "feat(ai): add Claude fix generation with mobile-specific prompts"
```

---

### Task 9.6: Executive summary generation

> **🔒 CLOSED at M9.C Stage 3 C1 (api `a0522a1`); tests at C2 (`6eff189`); ADR-032 single-call Claude Sonnet summary operational.** The plan-literal pseudo-code below is INFORMATIONAL (superseded by ADR-032). Real implementation: `summary.py` NEW (SUMMARY_SYSTEM_PROMPT 4-section structure per Q9 + `_build_summary_user_prompt` severity-tiered input per Q10 + `_deterministic_fallback_summary` per Q11 C.c + `_generate_executive_summary` with Q12 A idempotency + Q13 C.b zero-vuln defensive + check_budget/log_ai_call per Q11) wired into `pipeline.run()` per Q13; no migration per V-AAI (executive_summary scaffolded at M9.0 C1). Tests: `tests/services/ai/test_summary.py` + `tests/integration/test_m9c_smoke.py`.

**Files:**
- Create: `src/app/services/ai/summary.py`
- Test: `tests/services/ai/test_summary.py`

**Step 1: Failing test** — summary is 3-5 paragraphs, business language, includes count breakdown.
**Step 2: Implement** using Claude Sonnet.
**Step 3: Commit**

---

> **🔒 M9.D — AI Pipeline: Orchestrator** (Task 9.7) — **CLOSED at Stage 3 + P5.A lifecycle closure (2026-07-02).**
>
> Stage 1 design doc (compressed; absorbs Stage 2 per Q5): `plans/2026-07-02-m9d-orchestrator-design.md` (commit `988326e`) — 8 locks (Gate-1 gap-closure + Gate-2 throttle-pin-honored + Q1-Q6) via Mode 2 compressed brainstorming + adaptive Path δ SMALL classification.
> Stage 3 C0 ADR-033 SPEC §13 canonical: `1583f27`.
> Stage 3 C1 ai_pipeline_consumer.py terminal-metadata wiring: `5c092df` (completed_at at COMPLETED/PARTIAL/FAILED + error_message FAILED str(exc) via `_fail_scan(*, error)` threading + PARTIAL "N of M" summary; V-EEC 8th averted-prediction catch; ADR-013 sole-writer compliant; Gate-2 honored — fix_generation.py untouched).
> Stage 3 C2 e2e orchestrator smoke: `2f155f0` (4 tests green first-run; V-EEC threading verified e2e; suite 751 ZERO regressions).
> Stage 4 P5.A Commit 1 docs annotations: this commit. P5.A Commit 2 api DRIFT-LOG + Commit 3 engine cross-ref: forthcoming.
>
> **Drift catalog at M9.D lifecycle: none** — cumulative count **preserved at 66**. 8-instance averted-prediction lineage (V-EEC exception-threading latest); 3-instance test-gate-within-lock pattern (not incremented at M9.D — C2 tests green first-run).
>
> Forward-pins: webhooks→Task 12.5; CANCELED completed_at→routes-touch task; fix-gen throttle→production-readiness (M9.C Gate-1 pin honored through M9.D); autouse-anthropic-stub→production-readiness; stream-key TTL→OPS.

---

### Task 9.7: Full AI pipeline orchestrator

> **🔒 CLOSED-BY-COMPOSITION at M9.D (ADR-033; per Q6).** The plan-literal `run_ai_pipeline` pseudo-code below is INFORMATIONAL (superseded by ADR-033 per the Milestone-9 INFORMATIONAL-not-BINDING header). The orchestrator landed incrementally: Task 4.2 `cf3b30a` (ScanOrchestrator dispatch + CompletionsConsumer) → M9.0 C2 (ai-pipeline dispatch seam + AIPipelineConsumer) → M9.A/B/C (`pipeline.run()` 6 stages: embed → dedup → correlate → score → fix-gen → summary) → M9.D `5c092df` (terminal-metadata wiring: completed_at + error_message) + `2f155f0` (e2e both-paths smoke). Real files: `src/app/services/ai_pipeline_consumer.py` + `tests/integration/test_m9d_orchestrator_smoke.py`.

**Files:**
- Create: `src/app/services/ai/pipeline.py`
- Test: `tests/services/ai/test_pipeline.py`

**Step 1: Failing test** — on AllScanJobsCompleted event, full pipeline runs: embed → dedup → correlate → score → fix → summary → status=completed.

**Step 2: Implement**

```python
async def run_ai_pipeline(scan_id: UUID, db: AsyncSession):
    scan = await db.get(Scan, scan_id)
    raw_findings = await db.execute(select(RawFinding).where(RawFinding.scan_id == scan_id))
    raw_findings = raw_findings.scalars().all()

    if not raw_findings:
        scan.status = ScanStatus.COMPLETED
        await db.commit()
        return

    scan.status = ScanStatus.ANALYZING
    await db.commit()

    try:
        # 1. Embed
        embeddings = await embed_findings(raw_findings)
        # 2. Dedup
        unique = await deduplicate(raw_findings, embeddings)
        # 3. Correlate
        correlated_groups = await correlate(unique)
        # 4. Score
        vulns = [await score(group) for group in correlated_groups]
        # 5. Generate fixes (parallel with throttle)
        async with asyncio.Semaphore(5):
            await asyncio.gather(*[generate_fix_and_store(v) for v in vulns])
        # 6. Executive summary
        summary = await generate_summary(scan, vulns)
        scan.executive_summary = summary

        scan.status = ScanStatus.COMPLETED
        scan.completed_at = datetime.utcnow()
        await db.commit()
        # Fire webhooks
        await fire_scan_completed_webhooks(scan)
    except Exception as e:
        log.exception("AI pipeline failed")
        scan.status = ScanStatus.PARTIAL
        scan.error_message = str(e)
        await db.commit()
```

**Step 3: Commit**
```bash
git commit -m "feat(ai): add full pipeline orchestrator with error recovery"
```

---

## Milestone 10: Vulnerability & Report APIs (Week 6)

> 🧭 **M10 — REPORT ARCHITECTURE — DECOMPOSED (2026-07-06; V-AA LARGE classification 2026-07-02 + DQ1-DQ5 mini-chain).** Sub-milestones + sequencing: **M10.A** Vulnerability Endpoints ✅ **CLOSED (2026-07-06;** chain `df68e69` + `5cc968a` + `0251ee7` + P5.A; all γ; no C0; first M10 sub-milestone**)** (Task 10.1; data layer ready: VulnerabilityStatus + VulnerabilityHistory operational — history table now WIRED at PATCH per Q5) → **M10.B** Report Generation + Delivery (Tasks 10.2+10.3+10.4; ADR-034 reserved; WeasyPrint 62.3 + Jinja2 rendering stack locked by convergent authority — pinned dependency + plan + SPEC agree) → **M10.C** Compliance Mapping (Task 10.5; 2-table migration + SOC2/ISO-27001 seed) → **M10.D** Tool Health (Task 10.6; compressed shape). No M10.0 foundation needed (R2 + WeasyPrint + models + RLS + test harness all operational; contrast M9).
>
> **Y-REPORT-PERSISTENCE (DQ3, pulled forward):** per-format split — PDF generated LAZILY at first download request, stored in R2 (365-day retention per SPEC data-retention canon), tracked via NEW reports table (M10.B migration); JSON + SARIF rendered on-demand from DB (no storage; trivially regenerable, no retention mandate). Eager-at-scan-completion PDF generation forward-pinned to production-readiness (would reopen the M9.D-closed consumer + waste renders at pre-launch volume; activate on first-download-latency complaints).
>
> **ADR allocation (DQ2):** this blockquote ratifies the decomposition (no dedicated decomposition ADR); ADR-034 "Report Architecture" allocated at M10.B entry; M10.A expected to need no new ADR (anchors: ADR-012 tenant-scoping + ADR-002 PostgreSQL); M10.C/M10.D ADR-need decided at their entries.
>
> **Triggers (DQ5), sequential:** ***"Begin M10.A — Vulnerability Endpoints"*** ✅ consumed → ***"Begin M10.B — Report Generation + Delivery"*** ⇐ **NEXT** → ***"Begin M10.C — Compliance Mapping"*** → ***"Begin M10.D — Tool Health"***.
>
> Cumulative drift count at decomposition: 66; 8-instance averted-prediction lineage operational.

### Task 10.1: Vulnerability endpoints

> **🔒 Task 10.1 — CLOSED at M10.A (2026-07-06).** Landed as shieldscan-api `routes/vulnerabilities.py` + `schemas/vulnerabilities.py` + `services/audit.py` NEW VulnerabilityAction enum (per-domain discipline; 9th averted-prediction catch) + `tests/routes/test_vulnerabilities.py` (24 tests; suite 777 ZERO regressions). Scope: **full SPEC §6 7-endpoint canon** per the M10.A Q1 scope-delta resolution (per-scan list + org-wide list + detail + PATCH + /evidence + /fix + /history) — the 4-endpoint sketch, `data` envelope key, and `engine=nuclei` filter below are INFORMATIONAL, superseded by the lock-record (`items`+`pagination` envelope per Q3; **engine_category** filter per Q2; ai_fix_text EXCLUSIVE to /fix per Q7). PATCH is the FIRST write-site for the scaffolded VulnerabilityHistory table (Q5) + emits `vulnerability.status_changed` audit rows per actual transition (Q8; same-status = 200 no-op per Q4). Chain (all γ; **no C0** per Q10 — first C0-less sub-milestone): Stage 1 lock-record `df68e69` + C1 `5cc968a` + C2 `0251ee7` + P5.A this commit. Full lock-record: `plans/2026-07-06-m10a-vulnerability-endpoints-design.md`.

**Files:**
- Create: `src/app/routes/vulnerabilities.py`
- Create: `src/app/schemas/vulnerabilities.py`
- Test: `tests/routes/test_vulnerabilities.py`

**Step 1: Failing tests**

```python
async def test_list_vulnerabilities_with_filters(client, auth_headers, sample_vulns):
    r = await client.get(
        "/v1/orgs/.../vulnerabilities?severity=critical,high&status=open&engine=nuclei",
        headers=auth_headers
    )
    assert r.status_code == 200
    assert all(v["severity"] in ["critical", "high"] for v in r.json()["data"])

async def test_update_vulnerability_status(client, auth_headers, sample_vuln):
    r = await client.patch(
        f"/v1/orgs/.../vulnerabilities/{sample_vuln.id}",
        json={"status": "resolved", "notes": "Fixed in PR #123"},
        headers=auth_headers
    )
    assert r.status_code == 200
    # History should be recorded
    history_r = await client.get(f"/v1/orgs/.../vulnerabilities/{sample_vuln.id}/history")
    assert len(history_r.json()["data"]) > 0
```

**Step 2-5: Implement list (filterable + paginated), detail, patch status, history endpoints. Commit.**

---

> **M10.B — Report Generation + Delivery** (Tasks 10.2+10.3+10.4) — IN PROGRESS at Stage 3 C0 ADR-034 landing this commit.
>
> Stage 1 design: `plans/2026-07-08-m10b-report-generation-design.md` (`92fe1c7`) — 5 gates / 17 locks; shared ReportContext assembler; lazy per-format persistence per DQ3.
>
> Stage 3 C0 ADR-034 SPEC §13: this commit. C1 migration+model+jsonschema, C2 assembler+JSON+SARIF, C3 PDF+R2, C4 endpoints: forthcoming. Stage 4 P5.A forthcoming (Task 10.2/10.3/10.4 CLOSED annotations at P5.A).
>
> Rendering stack WeasyPrint+Jinja2 locked; reports-table migration parent `ecfed70e05e4`; `jsonschema` dependency escalation per Rule 4 (VERSIONS.md at C1). Tasks 10.2-10.4 Status UNTOUCHED at C0.

### Task 10.2: PDF report generation

**Files:**
- Create: `src/app/services/reports/pdf.py`
- Create: `src/app/templates/report.html`
- Create: `src/app/templates/report.css`
- Test: `tests/services/reports/test_pdf.py`

**Step 1: Failing test** — generates PDF < 30s for 100 vulns, includes exec summary, vuln table, evidence, mobile section if applicable.

**Step 2: Implement** — WeasyPrint + Jinja2 template.

**Step 3: Commit**
```bash
git commit -m "feat(reports): add WeasyPrint PDF generation with mobile sections"
```

---

### Task 10.3: SARIF + JSON report generation

**Files:**
- Create: `src/app/services/reports/sarif.py`
- Create: `src/app/services/reports/json_report.py`

**Step 1: Failing tests** — SARIF validates against schema, JSON includes all fields.
**Step 2: Implement.**
**Step 3: Commit**

---

### Task 10.4: Report download endpoints

**Files:**
- Create: `src/app/routes/reports.py`

**Step 1: Failing tests** — /report/pdf, /report/sarif, /report/json, /report/executive all return correct content-types.
**Step 2: Implement.**
**Step 3: Commit**

---

### Task 10.5: Compliance mapping + endpoints

**Files:**
- Create: `src/app/services/compliance/mapper.py`
- Create: `src/app/services/compliance/frameworks.py` (SOC2 + ISO 27001 seed data)
- Create: `src/app/routes/compliance.py`

**Step 1: Failing tests** — CWE-89 (SQLi) maps to SOC2 CC6.1 + ISO A.14.2.5, compliance posture returns pass/fail per control.

**Step 2: Implement** — seed compliance_frameworks and cwe_control_mappings tables via Alembic data migration.

**Step 3: Commit**
```bash
git commit -m "feat(compliance): add SOC2 + ISO 27001 mapping and endpoints"
```

---

### Task 10.6: Tool health endpoint

**Files:**
- Create: `src/app/routes/tools.py`

**Step 1: Failing test** — returns list of workers with per-tool status (healthy/degraded/unhealthy).
**Step 2: Implement** — aggregate health data published by workers to Redis.
**Step 3: Commit**

---

## Milestone 11: React Dashboard (Week 6-8)

### Task 11.1: Frontend scaffold

**Files:**
- Create: `shieldscan-web/` (Vite + React + TypeScript + Tailwind)

**Steps:** `npm create vite@latest shieldscan-web -- --template react-ts`. Add Tailwind, TanStack Query, React Router, axios, sse.js, react-dropzone, react-syntax-highlighter, Recharts.

**Directory structure:**
```
shieldscan-web/src/
├── pages/
├── components/
├── hooks/
├── lib/
│   ├── api.ts
│   ├── auth.ts
│   └── queryKeys.ts
└── App.tsx
```

**Commit:** `feat(web): scaffold frontend with Vite + React 18 + TypeScript`

---

### Task 11.2: Auth context + API client

**Files:**
- Create: `src/lib/api.ts`
- Create: `src/lib/auth.tsx`
- Create: `src/hooks/useAuth.ts`

**Step 1: Failing tests (Vitest)** — API client auto-refreshes expired JWT, auth context persists across reloads.
**Step 2: Implement** — httpOnly cookie JWT with automatic refresh, React Context for auth state.
**Step 3: Commit**

---

### Task 11.3: Query key convention (v3 §25)

**Files:**
- Create: `src/lib/queryKeys.ts`

**Step 2: Implement** per Spec v3 §25.1.

**Step 3: Commit**

---

### Task 11.4: Login + Register + ForgotPassword pages

**Files:**
- Create: `src/pages/Login.tsx`, `Register.tsx`, `ForgotPassword.tsx`, `VerifyEmail.tsx`

**Step 1-5:** TDD with React Testing Library. Commit.

---

### Task 11.5: Dashboard home page

**Files:**
- Create: `src/pages/Dashboard.tsx`
- Create: `src/components/ScanSummaryCard.tsx`
- Create: `src/components/VulnerabilityChart.tsx`

**Step 2: Implement** — total projects, recent scans, vuln count donut chart (Recharts), recent scans table.
**Step 3: Commit**

---

### Task 11.6: Project management pages

**Files:**
- Create: `src/pages/Projects.tsx`, `ProjectDetail.tsx`, `ProjectCreate.tsx`, `ProjectVerify.tsx`

**Step 2: Implement** — list, create, detail with scan history + vuln timeline, domain verification flow with DNS/meta tag instructions.

**Step 3: Commit**

---

### Task 11.7: Scan execution page with SSE progress

**Files:**
- Create: `src/pages/ScanRun.tsx`
- Create: `src/components/ScanProgress.tsx`
- Create: `src/components/FindingFeed.tsx`
- Create: `src/hooks/useScanProgress.ts`

**Step 1: Failing tests** — SSE reconnects on disconnect, progress bar updates per engine.

**Step 2: Implement** — per-engine progress bars, live finding feed as vulns discovered, completion animation.

**Step 3: Commit**
```bash
git commit -m "feat(web): add scan execution page with SSE real-time progress"
```

---

### Task 11.8: Mobile scan page

**Files:**
- Create: `src/pages/MobileScan.tsx`
- Create: `src/components/MobileUpload.tsx`

**Step 1: Failing tests** — file extension validation, size validation, platform auto-detection.

**Step 2: Implement** — react-dropzone for drag-drop, file size validation, platform badge after selection, analysis type radio, submits upload + scan in sequence.

**Step 3: Commit**
```bash
git commit -m "feat(web): add mobile scan page with APK/IPA upload"
```

---

### Task 11.9: Attack Surface Map component

**Files:**
- Create: `src/components/AttackSurfaceMap.tsx`
- Create: `src/components/SubdomainCard.tsx`

**Step 1: Failing tests** — expandable tree, live/dead badges, "quick-scan this subdomain" action.

**Step 2: Implement** — fetches /attack-surface, renders tree with tech stack chips, vuln count badges.

**Step 3: Commit**
```bash
git commit -m "feat(web): add attack surface map showing discovered subdomains"
```

---

### Task 11.10: Vulnerability explorer

**Files:**
- Create: `src/pages/Vulnerabilities.tsx`
- Create: `src/pages/VulnerabilityDetail.tsx`
- Create: `src/components/VulnFilters.tsx`
- Create: `src/components/EvidenceViewer.tsx`
- Create: `src/components/CodeFix.tsx`

**Step 2: Implement** — filterable table (severity, status, engine, CWE, category), detail page with evidence (request/response with syntax highlighting), AI fix with copy button, status update dropdown, history timeline. Mobile findings show MobileOS, Permission, ComponentName fields.

**Step 3: Commit**

---

### Task 11.11: Scan report viewer

**Files:**
- Create: `src/pages/ScanReport.tsx`

**Step 2: Implement** — rendered HTML report, tabs for Executive Summary / Vulnerabilities / Attack Surface / Compliance, download buttons for PDF/SARIF/JSON.

**Step 3: Commit**

---

### Task 11.12: Settings pages

**Files:**
- Create: `src/pages/Settings.tsx`, `Billing.tsx`, `Integrations.tsx`, `ApiKeys.tsx`, `Members.tsx`

**Step 2: Implement** — org settings, member management with role selector, API key generation (show plain key once), Stripe billing portal redirect, integration setup.

**Step 3: Commit**

---

## Milestone 12: Billing & Integrations (Week 8-9)

### Task 12.1: Stripe subscription integration

**Files:**
- Create: `shieldscan-api/src/app/services/billing.py`
- Create: `shieldscan-api/src/app/routes/billing.py`

**Step 1: Failing tests** — checkout session created for plan, webhook updates subscription on payment, usage metered after each scan.

**Step 2: Implement**

```python
import stripe
stripe.api_key = settings.STRIPE_SECRET_KEY

async def create_checkout_session(org: Organization, plan: Plan) -> str:
    if not org.stripe_customer_id:
        customer = stripe.Customer.create(email=org.billing_email, name=org.name)
        org.stripe_customer_id = customer.id

    session = stripe.checkout.Session.create(
        customer=org.stripe_customer_id,
        line_items=[{"price": plan.stripe_price_id, "quantity": 1}],
        mode="subscription",
        success_url=f"{settings.FRONTEND_URL}/billing?success=true",
        cancel_url=f"{settings.FRONTEND_URL}/billing?canceled=true",
    )
    return session.url

async def handle_stripe_webhook(payload: bytes, sig: str):
    event = stripe.Webhook.construct_event(payload, sig, settings.STRIPE_WEBHOOK_SECRET)
    if event.type == "checkout.session.completed":
        await activate_subscription(event.data.object)
    elif event.type == "customer.subscription.updated":
        await update_subscription(event.data.object)
    elif event.type == "customer.subscription.deleted":
        await cancel_subscription(event.data.object)
    elif event.type == "invoice.payment_failed":
        await notify_payment_failure(event.data.object)
```

**Step 3: Commit**

---

### Task 12.2: Usage metering + rate limiting

**Files:**
- Create: `src/app/services/usage.py`
- Create: `src/app/middleware/rate_limit.py`

**Step 1: Failing tests** — rate limit per tier enforced, usage record created per scan, monthly counter resets.

**Step 2: Implement** per Spec v3 §24 (Redis sliding window).

**Step 3: Commit**

---

### Task 12.3: GitHub integration

**Files:**
- Create: `src/app/services/integrations/github.py`
- Create: `src/app/routes/integrations.py`

**Step 1: Failing tests** — OAuth flow, token storage (encrypted), SARIF upload to Code Scanning, webhook for post-push scan trigger.

**Step 2: Implement.**

**Step 3: Commit**

---

### Task 12.4: Slack integration

**Files:**
- Create: `src/app/services/integrations/slack.py`

**Step 2: Implement** — incoming webhook URL, Block Kit formatted messages on scan complete / critical finding / failure.

**Step 3: Commit**

---

### Task 12.5: Custom webhooks + retry logic

**Files:**
- Create: `src/app/services/webhooks.py`

**Step 2: Implement** per Spec v3 §8 — HMAC signing, retry with exponential backoff, dead letter queue.

**Step 3: Commit**

---

### Task 12.6: Email notifications

**Files:**
- Create: `src/app/services/notifications/email.py`
- Create: `src/app/templates/emails/` (welcome, scan_complete, critical_finding, weekly_digest, verification)

**Step 2: Implement** — SendGrid/Resend, transactional templates.

**Step 3: Commit**

---

## Milestone 13: CLI & CI/CD (Week 9)

### Task 13.1: Scaffold CLI

**Files:**
- Create: `shieldscan-engine/shieldscan-cli/`
- Create: `cmd/shieldscan/main.go`

**Step 2: Implement** with Cobra. Commands: `auth`, `scan`, `results`, `projects`, `mobile`.

**Step 3: Commit**

---

### Task 13.2: Auth commands

**Files:**
- Create: `internal/commands/auth.go`

**Steps:** `shieldscan auth login` (interactive API key prompt → stored in ~/.shieldscan/config), `logout`, `status`. Commit.

---

### Task 13.3: Scan commands

**Files:**
- Create: `internal/commands/scan.go`

**Steps:** 
```bash
shieldscan scan --target https://example.com --type full_web --fail-on critical
shieldscan scan --mobile ./app.apk --platform android
shieldscan scan --type full_spectrum --wait
```

Commit.

---

### Task 13.4: Results commands

**Files:**
- Create: `internal/commands/results.go`

**Steps:** `shieldscan results list`, `results detail <vuln_id>`, `results json`, `results sarif`. Commit.

---

### Task 13.5: Cross-platform build

**Files:**
- Create: `Makefile`
- Create: `.github/workflows/release.yml`

**Steps:** Goreleaser config, cross-compile for Linux/macOS/Windows/ARM64. Push binaries to GitHub releases on tag.

---

### Task 13.6: GitHub Action

**Files:**
- Create: `shieldscan-action/` (separate repo)
- Create: `action.yml`
- Create: `entrypoint.sh`

**Steps:** Action installs CLI, runs scan against `target`, uploads SARIF, comments on PR with summary, fails build if `fail-on` threshold exceeded.

```yaml
# action.yml
name: ShieldScan Security Scan
inputs:
  api-key:
    required: true
  target:
    required: true
  scan-type:
    default: full_web
  fail-on:
    default: high
runs:
  using: docker
  image: Dockerfile
```

Commit.

---

## Milestone 14: On-Prem Agent (Week 10)

### Task 14.1: Agent binary scaffold

**Files:**
- Create: `shieldscan-engine/cmd/agent/main.go`
- Create: `internal/agent/client.go`

**Step 2: Implement** — single Go binary, no Docker dependency. Connects to cloud API via HTTPS + API key. Registers with org, receives jobs, runs tools, reports results.

**Step 3: Commit**

---

### Task 14.2: Agent tool subset

Agent includes: Nuclei, Subfinder, httpx, SSLyze, Nikto, Wapiti, Nmap (native invocation, no Docker). MobSF not included in on-prem agent (too heavy). ZAP bundled as Java jar.

**Commit.**

---

### Task 14.3: Agent registration + heartbeat

**Files:**
- Create: `internal/agent/registration.go`

**Step 2:** Agent sends heartbeat every 30s with tool health. Cloud API dispatches jobs when agent is healthy.

**Commit.**

---

## Milestone 15: Testing, Hardening & Launch (Week 10)

### Task 15.1: Load testing

**Files:**
- Create: `tests/load/locustfile.py`

**Scenarios per Spec §11:**
1. 50 concurrent dashboard users
2. 10 concurrent full-spectrum scans
3. 1,000 req/min API keys (CI/CD sim)
4. 20 simultaneous SSE streams
5. 5 concurrent 100MB mobile uploads

**Run:** `locust -f tests/load/locustfile.py --users 50 --spawn-rate 5`

All performance targets from Spec §11 must pass.

**Commit.**

---

### Task 15.2: Security audit (self-scan)

**Steps:** Run ShieldScan against itself in CI. All critical and high findings must be resolved before launch. Internal pentest by external firm recommended.

**Commit.**

---

### Task 15.3: End-to-end scan flow tests

**Files:**
- Create: `tests/e2e/`

**Scenarios:**
1. Register → create project → verify domain → run full web scan → view report
2. Register → create project → upload APK → run mobile scan → verify mobile findings
3. Run scan → cancel mid-execution → verify cleanup
4. Run scan on non-existent domain → verify graceful failure
5. Trigger recon → verify subdomains discovered → verify scanned

**Commit.**

---

### Task 15.4: Documentation

**Files:**
- Create: `docs/user-guide.md`
- Create: `docs/api-reference.md` (auto-generated from OpenAPI)
- Create: `docs/cli-reference.md`
- Create: `docs/integrations.md`

**Commit.**

---

### Task 15.5: Pre-launch checklist

**Files:**
- Modify: `README.md`

- [ ] All 15 milestones complete
- [ ] All tests passing (unit + integration + E2E + load)
- [ ] Security self-scan: zero critical/high findings
- [ ] Stripe products + prices configured in production
- [ ] SendGrid/Resend account provisioned
- [ ] GitHub OAuth app registered
- [ ] Domain configured (shieldscan.io) with SSL
- [ ] Monitoring: Sentry + Prometheus + Grafana operational
- [ ] Uptime monitoring (Better Uptime or similar)
- [ ] Backup automation tested (database + R2)
- [ ] Disaster recovery runbook validated
- [ ] Terms of Service + Privacy Policy published
- [ ] Customer support system ready (Intercom/Crisp)

**Final commit + tag:**
```bash
git tag v1.0.0
git push origin v1.0.0
```

---

## Timeline Summary

| Week | Milestone | Deliverable |
|---|---|---|
| 1 | M1, M2 (start) | DB + models + auth foundation |
| 2 | M2 (finish), M3, M4 (start) | Auth complete + projects + mobile upload + scan orchestration |
| 3 | M4 (finish), M5, M6 (start) | Redis contracts + Go worker + native runners (Nuclei, Semgrep, Recon) |
| 4 | M6 (finish), M7 (start) | All native runners + persistent Docker service integration start |
| 5 | M7 (finish), M8, M9 (start) | MobSF + ZAP + Trivy runners + recon pipeline + AI pipeline start |
| 6 | M9 (finish), M10, M11 (start) | AI pipeline complete + vuln/report APIs + frontend scaffold |
| 7-8 | M11 (all) | Full React dashboard: mobile scan, attack surface, vuln explorer, reports |
| 8-9 | M12 | Stripe billing + GitHub + Slack + webhooks + emails |
| 9 | M13 | CLI tool + GitHub Action |
| 10 | M14, M15 | On-prem agent + testing + launch |

**Total: 10 weeks to launch-ready MVP.**

---

*Plan complete. All 15 milestones, 80+ tasks, TDD throughout, exact file paths and code provided. Ready for implementation via Claude Code.*

---

## Execution Handoff

Plan saved. Two execution options:

**1. Subagent-Driven (this session)** — Dispatch fresh subagent per task, review code between tasks, fast iteration.

**2. Parallel Session (Claude Code)** — Open Claude Code in each repo (`shieldscan-api` and `shieldscan-engine`), execute task by task with checkpoints.

For a project of this size, **Claude Code parallel sessions** is the recommended approach — one session per repo, both can run in parallel, each with its own context focused on its codebase.
