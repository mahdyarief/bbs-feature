# Feature Specifications

This directory contains all feature requirement specs for the BBS application.

## Structure

```
features/
├── README.md                  # This file (index)
├── {feature-slug}/
│   ├── spec.md                # Feature specification (required)
│   ├── edgecases.md           # Edge cases & decisions (required)
│   ├── api-contract.md        # API endpoint details (optional)
│   ├── schema.md              # DB schema changes (optional)
│   └── notes.md               # Open questions, decisions (optional)
```

Templates are in `.pi/templates/`:
- `spec-template.md` — Main specification template
- `edgecases-template.md` — Edge cases template

## Naming Convention

- Folder name: lowercase kebab-case slug (e.g., `gpa-threshold-config`, `student-attendance-export`)
- Spec file: always `spec.md`
- Edge cases file: always `edgecases.md`

## Required Files

Every feature spec **must** have:
1. `spec.md` — Main specification (what, why, acceptance criteria)
2. `edgecases.md` — Edge cases with options and decisions

Optional files (create if needed):
- `api-contract.md` — API endpoint details
- `schema.md` — Database schema changes
- `notes.md` — Open questions, decisions, discussion notes

## Existing Specs

| Feature | Status | Spec Path |
|---------|--------|-----------|
| GPA Threshold Config Fix | Draft | `gpa-threshold-config/spec.md` |
| External Attendance Integration | Draft | `external-attendance-integration/spec.md` |

## Bug Reports (AI Operate Test, 2026-09-30)

Temuan dari eksekusi test case read-only di `../operate-smartbag/test-cases/`
(TC-001 s/d TC-019, semua fitur Smartbag). Laporan lengkap di tiap folder.

| Bug / Brief | Folder | Sumber Test | Status |
|-------------|--------|-------------|--------|
| Student Billing endpoint rusak (500, service kosong + tanpa version URI) | `student-billing-endpoint-broken/` | TC-015 | open |
| sortBy invalid → 500 di semua list endpoint (`findOptionsHelper`) | `list-sortby-invalid-500/` | TC-034 | open |
| Impersonate portalType mismatch → 500 (seharusnya 400) | `impersonate-portal-mismatch-500/` | TC-038 | open |
| Audit `throw new Error` lintas api_nest: 37 lokasi, 24 user-error → seharusnya 4xx | `generic-error-throw-audit/audit.md` | audit TC-038 | open |
| Student list tanpa filter campus → 502 konsisten | `student-list-no-filter-timeout/` | TC-006 | open |
| Topics list eager-load berat → 502 konsisten | `topics-list-eager-loading-timeout/` | TC-009 | open |
| Banding enrollment delta class 100024 (42 vs 48) — **DECIDED: keep (unenroll via UI, LIG ikut bersih)** | `banding-enrollment-reconciliation/` (spec + edgecases) | TC-010 | decided |
| Wiki basi: OTP toggle hilang, teacher-leave ada, pageSize CCA (spec + edgecases) | `wiki-catalog-stale-flags/` | TC-019 | draft |
| ccaYearCoordinators 500 — method findAll dikosongkan (scaffold rusak) | `cca-year-coordinators-empty-500/` | TC-039 | open |
| transfer-audit: 500 tanpa `?page=1&limit=10` (default param tidak diterapkan) + statistics body rusak | `transfer-audit-default-param-500/` | TC-039/040 | open |
| securityDeposits/:studentId → 500 untuk siswa tanpa deposit (harusnya 404) | `security-deposit-no-deposit-500/` | TC-039 | open |

Catatan sweep TC-039/TC-040 (2026-09-30): coverage path API **100%** (216/216
@Controller path terverifikasi di produksi). Temuan kecil tambahan tanpa file
sendiri (terdokumentasi di `TC-039-full-controller-path-sweep/result.md`):
`sveStudentGrades` 400 tanpa `page` (quirk DTO), `billingReports` 502 (keluarga
list-endpoint), 2 dead scaffold tanpa route (`cca-grade`, `ftp-evaluation-setting`),
dual-version sebenarnya 3 modul (parents/students/payments). |

---

## How to Use

### For PM / Product Owner
1. Create folder: `features/{feature-slug}/`
2. Copy `_template.md` → `spec.md` and fill in
3. Copy `_edgecases-template.md` → `edgecases.md` and list potential edge cases
4. Mark as `Status: Draft` in frontmatter

### For Engineer
1. Read `spec.md` for the feature
2. Read `edgecases.md` for edge case decisions
3. Reference `api-contract.md` and `schema.md` if they exist
4. Check `notes.md` for open questions / decisions

### For AI Agent (Pi)
- Load `.pi/context/codebase-overview.md` for codebase structure
- Load `features/{feature}/spec.md` as primary context
- Load `features/{feature}/edgecases.md` for edge case handling
- Supplement with `api-contract.md` / `schema.md` as needed

## Workflow

1. **Draft** — PM writes spec + edge cases
2. **Review** — Team reviews, discuss edge cases
3. **Decide** — Fill in edge case decisions
4. **Implement** — Engineer builds based on spec + edge case decisions
5. **Done** — Update status to `implemented`
