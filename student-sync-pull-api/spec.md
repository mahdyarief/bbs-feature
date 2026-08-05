---
status: draft
owner: product
feature-id: FEAT-STUDENT-SYNC-PULL-API
---

# Student Sync Pull API: Bulk Sigma Data Endpoint

## Overview
Provide a secure, versioned, and verifiable `GET /api/v1/sync/students` endpoint for external applications (SIS, LMS, HR) to pull complete, consistent, and auditable student data per academic year — enabling reliable yearly sync and compliance reporting.

## Problem Statement
External applications need to import BBS student data annually, but currently:
- ❌ No official API exists — teams resort to manual CSV exports or database dumps (unauditable, insecure).
- ❌ No consistency guarantee — concurrent updates during export cause partial/inconsistent snapshots.
- ❌ No integrity verification — no way to confirm downloaded data matches source.
- ❌ No audit trail — impossible to prove *who* synced *what*, *when*, and *for which academic year*.

## Goals
- ✅ 100% of external apps use `/api/v1/sync/students` for yearly sync within 3 months of release.
- ✅ Every sync response includes `X-Data-Checksum`, `X-Total-Count`, and `syncedAt` for full sigma data verification.
- ✅ Zero failed syncs due to timeout or inconsistency — all responses are atomic snapshots.

## Scope

### ✅ In Scope
- Backend: New endpoint `GET /api/v1/sync/students` with query params `academicYearId` (required), `mode` (default `full`), `limit` (default 1000), `includeArchived` (default `false`).
- Response: JSON array of minimal student data (`studentId`, `fullName`, `gradeLevel`, `status`, `lastModified`) + root fields `syncedAt`, `totalCount`, `nextSyncToken`.
- Security: Auth via `Authorization: Bearer <API_KEY>`; ACL `SYNC:STUDENT_READ`; audit log with `actor_role = 'external_app'`.
- Integrity: Response headers `X-Data-Checksum: sha256:...` and `X-Total-Count: N`.

### ❌ Out of Scope
- Real-time streaming or webhook push.
- Support for `POST`/`PUT` — this is read-only pull only.
- Syncing non-student data (e.g., teachers, classes) — future scope.
- Client-side SDK or library — external apps consume raw HTTP.

## User Stories

| As a | I want to | So that |
|------|-----------|---------|
| External App Developer (SIS/LMS) | call `GET /api/v1/sync/students?academicYearId=123` and receive a complete, consistent, and checksum-verified JSON array | I can reliably import BBS student data without manual intervention or risk of corruption |
| School Compliance Officer | see audit logs showing every sync request with `academicYearId`, `app_name`, and `syncedAt` | I can demonstrate data provenance and meet regulatory requirements |
| BBS Admin | generate an API key for an external app with permission scoped to one academic year | I maintain strict data governance without over-provisioning |

## Acceptance Criteria

| # | Requirement | Testable? |
|---|-------------|-----------|
| AC-1 | Endpoint returns HTTP 200 with JSON array when valid `academicYearId` and API key provided | ✅ Yes (curl + status code) |
| AC-2 | Response includes root field `syncedAt` (ISO string) — timestamp of snapshot start | ✅ Yes (JSON assertion) |
| AC-3 | Response includes `X-Total-Count` header equal to total students in academic year (active + archived if requested) | ✅ Yes (header check + DB count) |
| AC-4 | Response includes `X-Data-Checksum: sha256:...` header matching SHA-256 of JSON body (excluding whitespace) | ✅ Yes (checksum verify) |
| AC-5 | For `mode=incremental`, response only includes students with `lastModified >= since` param | ✅ Yes (DB query + network trace) |
| AC-6 | Audit log records `actor_role = 'external_app'`, `actor_id = app_name`, `target_type = 'academic_year'`, `action = 'sync_students'` | ✅ Yes (log DB query) |
| AC-7 | If `academicYearId` does not exist or is inactive, returns HTTP 404 with `code: "ACADEMIC_YEAR_NOT_FOUND"` | ✅ Yes (invalid ID test) |

## Dependencies

| Dependency | Owner | Status |
|------------|-------|--------|
| `ClassYearService.findStudentsByAcademicYearId()` exists | Backend | ✅ Confirmed in codebase scan |
| `StudentService.findByIds()` supports batch loading | Backend | ✅ Standard TypeORM pattern — will implement if missing |
| `external_app_keys` table exists for API key storage | Backend | ⏳ To create — migration required |
| Audit logging middleware supports `actor_role = 'external_app'` | Backend | ✅ Confirmed in `audit-logger.middleware.ts` |

## Risks

| Risk | Mitigation |
|------|------------|
| **Large academic year (>10k students) causes timeout** | Implement cursor-based pagination (`nextSyncToken`) + max `limit=5000`. Fail fast with 413 Payload Too Large if unpaginated full export exceeds 5MB. |
| **Checksum calculation too slow** | Compute SHA-256 on serialized JSON *before* sending — cache if same payload reused (rare). |
| **`lastModified` not indexed in DB** | Add DB index on `students.last_modified` and `class_years.student_id` during migration. |

## Release Notes

- **Version**: `v3.9.0` (backend only)  
- **Environment**: Staging → UAT → Production  
- **Rollout**: Gradual — enable for 3 pilot apps first  
- **Go-live Date**: 2026-09-30  
- **Comms**: Email to external app partners + updated integration docs at `docs.bbs.school/api/sync`