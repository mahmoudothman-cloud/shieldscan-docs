# M10.B Report Generation + Delivery — Stage 1 Design (Mode-1 full)

**Status:** Stage 1 ratified; Stage 3 **5-commit** implementation forthcoming — **NEXT SESSION per budget checkpoint**. ADR-034 "Report Architecture" forthcoming at Stage 3 C0. M10.B re-enters full-ADR territory after M10.A's C0-less shape (new data model + new dependency + new migration + lazy-generation mechanics = genuinely architectural).

**Date:** 2026-07-08.

**Authority:** M10 decomposition `41ad3e6` (M10.B = Tasks 10.2+10.3+10.4; ADR-034 reserved; DQ3 per-format persistence RATIFIED); M10.B V-AA surface report (greenfield report surface; WeasyPrint 62.3 + Jinja2 3.1.6 live; R2 `put_object` + `generate_presigned_get_url` operational; alembic head `ecfed70e05e4`; SARIF 2.1.0; no jsonschema dep yet); M10.B Mode-1 brainstorming CLOSED (5 gates, ~17 locks); M10.A CLOSED (`VulnerabilityDetail`/`EvidenceItem`/`FixResponse` reusable as report data sources). 66 cumulative drift discipline; 9-instance averted-prediction lineage; 2-instance ADR-mislabel lesson; 3-instance date-stamp lesson.

---

## 1. Gate-Lock Table (5 gates / ~17 locks)

### Gate 1 — Data Model
- **G1-L1 SHARED ASSEMBLER:** one `ReportContext` assembler (`services/reports/context.py`) is the load-bearing seam — gathers Scan + Vulnerabilities (severity-ordered) + metadata ONCE and feeds all 3 format generators (PDF/SARIF/JSON). No per-format re-querying.
- **G1-L2 REPORTS TABLE:** NEW `Report(Base, TimestampMixin, TenantMixin)` — `id` + `scan_id` FK + `format` enum + `r2_key` + `size_bytes` + `generated_at`. `format` enum kept for forward-compat even though only PDF is persisted at MVP (JSON/SARIF regenerated on-demand per DQ3).
- **G1-L3 EXECUTIVE_SUMMARY STAYS ON SCAN:** SPEC-2616's *possible* `Scan.executive_summary → Report.summary_text` migration resolved as **NO-MOVE** — the summary is eagerly pipeline-produced (M9.C) while reports are lazy; moving it would couple an eager artifact to a lazy table. No column migration for the summary.
- **G1-L4 SOLE-WRITER:** api is the sole writer of `reports` rows per ADR-013; engine never touches them.

### Gate 2 — Generation + Delivery
- **G2-L5 GENERATE-OR-REUSE:** a report-service method owns generate-or-reuse; routes stay thin. Query `reports` first → if a row + R2 object exist, reuse; else generate → `put_object` → write row.
- **G2-L6 IDEMPOTENCY + CONCURRENT-RACE:** query-reports-first idempotency; a benign concurrent-render race (two first-downloads) is ACCEPTED at pre-launch — a unique-constraint on `(scan_id, format)` is forward-pinned, not added now.
- **G2-L7 PDF DELIVERY:** PDF delivered as a **presigned-GET URL returned in a JSON body** (per the MobSF R2 pre-signed lineage — ADR-013 addendum), NOT streamed through the API.
- **G2-L8 CUSTOMER EXPIRY:** presigned expiry **3600s** at the report call-site (vs the 600s `generate_presigned_get_url` default tuned for the MobSF engine-fetch window); per-download-deadline tuning forward-pinned.

### Gate 3 — Formats + Dependencies
- **G3-L9 SARIF 2.1.0:** SARIF **2.1.0** per OASIS + the Task-12 GitHub Code Scanning requirement.
- **G3-L10 JSONSCHEMA DEP:** `jsonschema` dependency ADDED + a bundled SARIF 2.1.0 schema for provable conformance — **VERSIONS.md escalation per Rule 4, approved** in brainstorming.
- **G3-L11 JSON ENVELOPE:** versioned full-fidelity JSON envelope reusing M10.A schemas; `ai_fix_text` inline; NO pagination (a report is the whole scan).

### Gate 4 — Endpoints + Tier
- **G4-L12 BASE /report = MANIFEST:** base `GET …/report` is a **manifest** reusing the `reports`-table query (available formats + generation state) — resolves the SPEC-§6-5 vs plan-literal-4 scope-delta in canon's favor.
- **G4-L13 UNGATED + FORWARD-PIN:** all 5 endpoints ship ungated; tier-enforcement forward-pinned to the billing milestone (M10.A /fix precedent). White-label deferred to an Enterprise feature.
- **G4-L14 409 NON-TERMINAL:** reports only for terminal-with-results scans (COMPLETED/PARTIAL); non-terminal → 409.

### Gate 5 — Template + Stage 3
- **G5-L15 PDF SECTIONS:** cover/branding → executive_summary → findings-overview → per-vuln detail (incl. `ai_fix_text`) → evidence → conditional mobile (see §6).
- **G5-L16 AI_FIX IN PDF:** the PDF is the paid deliverable — access is tier-gated later, fixes are NOT stripped from the content.
- **G5-L17 5-COMMIT STAGE 3:** C0–C4 (see §5).

---

## 2. Architecture

Lazy per-format persistence (DQ3): **PDF** generated at first `/report/pdf`, stored in R2 (365-day retention per SPEC §data-retention), tracked via the `reports` table; **JSON + SARIF** rendered on-demand from the DB (trivially regenerable, no storage, no retention mandate); **executive** served from `Scan.executive_summary`.

The **`ReportContext` assembler** is the single seam: `build_context(db, scan) -> ReportContext` gathers the Scan + severity-ordered Vulnerabilities + counts once; every generator (`pdf.py`, `sarif.py`, `json_report.py`) is a pure `ReportContext -> bytes|dict` function. R2 reuse via `put_object(key, content, content_type)` + `generate_presigned_get_url(r2_key, expiry=3600)`. api is sole writer of `reports` rows (ADR-013).

**Tenancy grounding caveat (mislabel-avoidance):** the `reports` table is a **standard `TenantMixin` + RLS table per Gotcha 3** — NOT ADR-012's app-layer-scoping exception. ADR-012 is cited only for the route-layer cross-tenant-404 collapse (as M10.A used it). C0 cites them distinctly.

---

## 3. Reports-Table Shape

```
class Report(Base, TimestampMixin, TenantMixin):
    __tablename__ = "reports"
    id: UUID pk
    scan_id: UUID FK -> scans.id          # report is per-scan
    format: ReportFormat enum             # pdf (persisted) | sarif | json (forward-compat)
    r2_key: str                           # reports/{scan_id}/{report_id}.pdf
    size_bytes: int
    generated_at: datetime
    # organization_id via TenantMixin; created_at/updated_at via TimestampMixin
```

- **RLS policy in the migration + a cross-tenant test are MANDATORY** per Gotcha 3 (silent RLS miss = critical hole).
- Migration parent: **`ecfed70e05e4`** (current alembic head; M9.B columns).

---

## 4. Implementation Surface

- `src/app/services/reports/context.py` — `ReportContext` + `build_context` assembler (the seam).
- `src/app/services/reports/json_report.py` — DB→dict versioned envelope.
- `src/app/services/reports/sarif.py` — DB→SARIF-2.1.0 dict + jsonschema validation.
- `src/app/services/reports/pdf.py` — WeasyPrint `HTML(string=...).write_pdf()` + Jinja2 render; lazy-generate→R2→reports-row.
- `src/app/templates/report.html` + `src/app/templates/report.css` — Jinja2 PDF template (greenfield Jinja2 `Environment` + `FileSystemLoader`).
- `src/app/routes/reports.py` — 5 endpoints (manifest + pdf + sarif + json + executive) + registration in `main.py`.
- `src/app/models/reports.py` — `Report` model + `ReportFormat` enum.
- **NEW dependency:** `jsonschema` (pyproject + **NEW VERSIONS.md entry per Rule 4**) + bundled SARIF 2.1.0 schema asset.
- Tests: `tests/services/reports/test_*.py` (PY7 cross-dir precedent = `tests/services/ai/`) + `tests/routes/test_reports.py`.

---

## 5. Stage 3 — 5-Commit Decomposition (NEXT SESSION)

- **C0** — ADR-034 "Report Architecture" (SPEC §13) + IMPLEMENTATION-PLAN annotations; DQ3 lazy-persistence canonicalized; ADR-002/012/013 anchors cited distinctly.
- **C1** — reports-table migration (parent `ecfed70e05e4`) + `Report` model + `ReportFormat` enum + RLS policy + cross-tenant test + `jsonschema`/VERSIONS.md + bundled SARIF-2.1.0 schema.
- **C2** — `ReportContext` assembler + JSON generator + SARIF generator + jsonschema validation + tests. **(Task 10.3.)**
- **C3** — PDF service (WeasyPrint + `report.html`/`report.css`) + lazy-generate→R2→reports-row + idempotent reuse + tests. **(Task 10.2.)**
- **C4** — `routes/reports.py` 5 endpoints + manifest + presigned-URL PDF + inline JSON/SARIF/executive + 409 non-terminal + registration + route tests. **(Task 10.4.)**

Flexible: C2/C3 may merge if the assembler is lean; flag at C-entry.

---

## 6. PDF Section Composition (G5-L15)

1. **Cover / branding** — scan target, org, date, tier-neutral branding (white-label deferred).
2. **Executive summary** — `Scan.executive_summary` verbatim (M9.C business-language).
3. **Findings overview** — severity-count table (critical/high/medium/low/info).
4. **Per-vulnerability detail** — severity-ordered; title, severity_score, CWE/CVSS, engine_category, corroborated_count, **`ai_fix_text`** (fixes not stripped — G5-L16).
5. **Evidence** — condensed RawFinding projections (truncated, per M10.A EvidenceItem shape).
6. **Mobile section** — conditional, only when mobile_* fields present.

---

## 7. Forward-Pins

- Eager-PDF-at-completion → production-readiness (would reopen the M9.D-closed consumer + waste renders at pre-launch volume).
- Unique-constraint on `(scan_id, format)` for the concurrent-render race → production-readiness.
- Per-download-deadline expiry tuning → production-readiness.
- White-label branding → Enterprise feature (SPEC §9: white-label Enterprise-only).
- Report-endpoint tier-gating → billing milestone (M10.A /fix precedent).
- Tool-level engine filter (RawFinding join) → carried from M10.A.
- Arabic-language reports → Phase 2.
- SARIF → GitHub Code Scanning upload → Task 12 (integrations).

---

## 8. V-JJ Pre-C Cascade — DEFERRED-EMPIRICAL Grounding List

- **V-JJA:** reports-migration alembic revision-chain — exact parent + `down_revision` convention from a recent migration file.
- **V-JJB:** RLS-policy migration DDL pattern from an existing TenantMixin-table migration (Gotcha 3 exact SQL — `ENABLE ROW LEVEL SECURITY` + `CREATE POLICY` using `app.current_org_id`).
- **V-JJC:** Jinja2 `Environment` + `FileSystemLoader` config pattern (greenfield — no repo precedent; ground on WeasyPrint `HTML(string=...).write_pdf()` API).
- **V-JJD:** R2 `put_object` content-type + key-convention `reports/{scan_id}/{report_id}.pdf`.
- **V-JJE:** `ScanStatus` terminal-set for the 409 gate (COMPLETED/PARTIAL allow; reuse M9.D `_fail_scan`/terminal awareness).
- **V-JJF:** SARIF 2.1.0 required-fields (`runs[].tool.driver.name`/`rules` + `results[].ruleId`/`level`/`message`/`locations`) for the generator + jsonschema assertion.
- **V-JJG:** `jsonschema` dependency add mechanics — poetry + VERSIONS.md entry format from an existing entry.
- **V-JJH (highest-value):** report-generation audit-action enum — does a `ReportAction` exist, or is one needed per the M10.A `VulnerabilityAction` precedent (per-domain anti-catch-all discipline)?
