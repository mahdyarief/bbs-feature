---
status: draft
owner: engineering
reviewed-by: security, product
date: 2026-08-05
---

# Engineering Brief: Student Sync Pull API — Bulk Sigma Data

> ⚠️ This is a strict implementation contract. Do not implement 1-by-1 fetching, manual string concatenation, or unverified checksums.

## ✅ What Must Be Delivered

### 1. Backend (`api_nest`)
- New controller: `StudentSyncController` in `src/modules/sync/student-sync.controller.ts`.
- Endpoint: `GET /api/v1/sync/students` with required query param `academicYearId: number`.
- Query params:
  - `mode: 'full' | 'incremental'` (default `'full'`)
  - `since?: string` (ISO datetime, required for `incremental`)
  - `limit: number` (default `1000`, max `5000`)
  - `includeArchived: boolean` (default `false`)
- Response body must be:
  ```json
  {
    "syncedAt": "2026-07-15T08:30:00Z",
    "academicYearId": 123,
    "mode": "full",
    "totalCount": 2147,
    "students": [
      {
        "studentId": "S12345",
        "fullName": "Budi Santoso",
        "gradeLevel": "Grade 7",
        "status": "active",
        "lastModified": "2026-07-10T14:22:15Z"
      }
    ],
    "nextSyncToken": "eyJzaW5jZSI6IjIwMjYtMDctMTVUMDg6MzA6MDBaIiwibGltaXQiOjEwMDB9"
  }
  ```
- Headers:
  - `X-Total-Count: 2147`
  - `X-Data-Checksum: sha256:abc123...` (SHA-256 of minified JSON body)
  - `X-Next-Sync-Token: ...` (for pagination)
- Auth: `Authorization: Bearer <API_KEY>` → validated via `ApiKeyGuard`.
- ACL: `@CheckPermissions({ module: ModulesTypeEnum.SYNC, action: ACLTypeEnum.STUDENT_READ })`.
- Audit log: `actor_role = 'external_app'`, `actor_id = app_name from external_app_keys`, `target_type = 'academic_year'`, `action = 'sync_students'`.

### 2. Database & Migration
- Create table `external_app_keys`:
  - `id`, `app_name`, `api_key_hash` (bcrypt), `academic_year_id`, `permissions: string[]`, `created_at`, `updated_at`
- Add index on `students.last_modified` and `class_years.student_id`.

## ❌ What Must NOT Be Changed

| Area | Constraint |
|------|------------|
| **Data source** | Do not fetch students from `student_admission` or `student_report` tables directly. Use only `ClassYearService.findStudentsByAcademicYearId()` + `StudentService.findByIds()`. |
| **Response format** | Do not omit `syncedAt`, `totalCount`, or `X-Data-Checksum`. Do not add extra fields (e.g., `createdAt`, `updatedAt`). |
| **Pagination** | Do not use offset/limit for `incremental` mode. Must use cursor-based `nextSyncToken` derived from `lastModified`. |
| **Auth method** | Do not accept `X-API-Key` header. Use only `Authorization: Bearer <token>` — reuse existing JWT guard pattern. |

## 🧩 What Is Assumed Available

| Artifact | Location / Note |
|----------|----------------|
| `ClassYearService.findStudentsByAcademicYearId()` | Confirmed in `transfer-student-validation.service.ts` — will reuse or wrap. |
| `StudentService.findByIds()` | Standard TypeORM service — implement if missing; must support batch loading. |
| `ApiKeyGuard` | Exists in `src/guards/api-key.guard.ts` — already used for other integrations. |
| Audit logging middleware | `src/middlewares/audit-logger.middleware.ts` — supports custom `actor_role`. |
| SHA-256 utility | `src/utils/crypto.util.ts` — already exports `sha256(string)`. |

## 🚨 Non-Negotiable Constraints

1. **Bulk loading only**: Zero `for-await` loops or `findById()` per student. Must use `.find({ where: { id: In([...]) } })`.  
2. **Atomic snapshot**: `syncedAt` must be generated *before* any DB query — all data reflects state at that exact time.  
3. **Checksum is mandatory**: If `X-Data-Checksum` header is missing or invalid, PR is rejected.  
4. **No unscoped access**: `academic_year_id` in `external_app_keys` must be enforced — never allow sync across years.  

## 📋 Verification Checklist (For PR Review)

- [ ] `GET /api/v1/sync/students?academicYearId=123` returns 200 + `syncedAt` + `totalCount` + `X-Data-Checksum`.  
- [ ] Checksum matches SHA-256 of minified response body.  
- [ ] `X-Next-Sync-Token` is present and decodable (base64 + JSON).  
- [ ] `mode=incremental&since=...` returns only students modified after `since`.  
- [ ] Invalid `academicYearId` returns HTTP 404 with structured error.  
- [ ] Audit log contains correct `actor_role`, `actor_id`, and `target_type`.  
- [ ] DB migration creates `external_app_keys` table with indexes.  
