---
status: draft
owner: product
reviewed-by: engineering, security
date: 2026-08-05
---

# Edge Cases & Decisions: Student Sync Pull API

> This document captures known edge cases, their impact, and the agreed-upon decision. All decisions are binding for implementation.

| # | Edge Case | Impact if Unhandled | Decision | Owner / Notes |
|---|-----------|----------------------|----------|---------------|
| EC-1 | `academicYearId` points to academic year that is not yet active (e.g., future year) | External app syncs data before it’s valid → confusion, incorrect reporting | ✅ Return HTTP 400 with `code: "ACADEMIC_YEAR_NOT_ACTIVE"`. Do NOT allow sync until `status = 'active'`. | Backend — validate in `AcademicYearService` |
| EC-2 | `mode=incremental` but `since` param missing or invalid format | Sync returns full list instead of delta → bandwidth waste, no incremental benefit | ✅ Return HTTP 400 with `code: "MISSING_SINCE_PARAM"` or `"INVALID_SINCE_FORMAT"`. | Backend — strict validation before query |
| EC-3 | `limit=0` or `limit > 5000` | Server overload or silent truncation → data loss | ✅ Enforce `limit` min=1, max=5000. Return HTTP 400 if violated. | Backend — pipe through DTO validation |
| EC-4 | No students found for `academicYearId` (empty year) | Response is `[]` — external app can’t distinguish empty vs error | ✅ Still return 200 OK with `totalCount: 0`, `students: []`, `syncedAt`, and `X-Data-Checksum` of empty array. | Backend — never 404 for empty result |
| EC-5 | `X-Data-Checksum` calculation fails (e.g., OOM on large payload) | Sync becomes unverifiable → breaks sigma data guarantee | ✅ Fail fast with HTTP 500 + log error. Do NOT return response without checksum. | Backend — wrap checksum in try/catch; re-throw |
| EC-6 | `nextSyncToken` decoded but `lastModified` timestamp is invalid | Pagination breaks → external app stuck or duplicates | ✅ Return HTTP 400 with `code: "INVALID_SYNC_TOKEN"`. Log token for debugging. | Backend — validate timestamp before use |
| EC-7 | External app calls endpoint without `Authorization` header | Unauthorized access attempt → security risk | ✅ Return HTTP 401 with generic message (`"Unauthorized"`). Do NOT leak auth method details. | Backend — handled by `ApiKeyGuard` |
| EC-8 | `external_app_keys` record has `permissions: []` or missing `STUDENT_READ` | App thinks it has access but gets 403 | ✅ Return HTTP 403 with `code: "INSUFFICIENT_PERMISSIONS"`. Audit log includes `permissions_requested`. | Backend — ACL guard checks array inclusion |
| EC-9 | DB query times out (> 10s) during bulk fetch | Sync hangs → external app timeout, retry storm | ✅ Return HTTP 504 Gateway Timeout. Log slow query. Add DB index on `class_years.academic_year_id` + `class_years.student_id`. | Backend — add timeout + index migration |
| EC-10 | `studentId` contains special characters (emoji, RTL) in `fullName` | JSON encoding breaks or renders poorly in external app | ✅ Allow all Unicode. JSON.stringify() handles it. No sanitization needed. | Frontend/External — safe by default |

## Open Questions (To Be Resolved Before Implementation)

| # | Question | Owner | Status |
|---|----------|-------|--------|
| OQ-1 | Should we support `Accept: application/json; version=1.0` header for future schema evolution? | Product | ⏳ Pending confirmation |
| OQ-2 | Is there a business requirement to include parent/guardian contact info in sync payload? | Product | ⏳ Pending confirmation |

> 🔔 **Note**: All decisions above are final unless updated via formal change request (`notes.md`). Engineering must implement *exactly as decided* — no reinterpretation.
