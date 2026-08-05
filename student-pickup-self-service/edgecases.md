---
status: draft
owner: product
reviewed-by: engineering, security
date: 2026-08-05
---

# Edge Cases & Decisions: Student Pickup Self-Service

> This document captures known edge cases, their impact, and the agreed-upon decision. All decisions are binding for implementation.

| # | Edge Case | Impact if Unhandled | Decision | Owner / Notes |
|---|-----------|----------------------|----------|---------------|
| EC-1 | Student submits duplicate `full_name` + `relationship` (e.g., "Budi Santoso" + "Father" already exists) | Duplicate entries in DB → confusion during pickup verification | ✅ **Allow duplicates**. No uniqueness constraint on name+relationship. Business rule: same person may be listed for different students (e.g., shared nanny), or same name used for different people (e.g., two "Uncles"). Backend does NOT deduplicate. | Backend — no code change needed |
| EC-2 | Student sets `is_primary: true` for multiple persons | Ambiguity during pickup handover — no single point of contact | ✅ **Allow multiple `is_primary: true`**. Frontend will show warning: "You have multiple primary contacts. School will contact all of them." No backend enforcement. | Frontend — add inline warning in form/UI |
| EC-3 | Student tries to create pickup person with invalid `phone_number` (e.g., letters, too short) | Bad UX; failed request; no feedback | ✅ **Frontend validates E.164 format client-side** (using existing `lib/utils/validate-phone.js`). If invalid → show error below field: "Enter valid phone number (e.g., +6281234567890)". Backend does *not* re-validate — assumes frontend did it. | Frontend — reuse existing validator |
| EC-4 | Student deletes their *only* pickup person | No one authorized to pick them up — safety risk | ✅ **Block deletion if it’s the last active pickup person**. Show toast: "You must have at least one authorized pickup person. Add another before removing this one." Backend returns 400 if `DELETE` would leave zero records. | Backend — add check in `remove()` method; Frontend — disable delete button if count === 1 |
| EC-5 | Student edits `studentId` in devtools/network tab and sends payload with another student’s ID | Critical security breach — data leakage/modification | ✅ **Backend ownership check is non-negotiable**. `remove()`, `update()`, and `findOne()` *must* verify `record.student_id === req.user.studentId`. If not → 403. Frontend must *never send `studentId`*. | Backend — confirmed present; Frontend — remove any `studentId` from payload construction |
| EC-6 | Student submits `relationship` longer than 50 chars (e.g., "Mother's second cousin twice removed") | Truncation or 500 error → poor UX | ✅ **Frontend enforces max 50 chars client-side** (with counter). Backend truncates to 50 chars on save (no error). | Backend — use `@Length(1, 50)` in DTO; Frontend — input maxlength + counter |
| EC-7 | Concurrent edits: Student A opens edit modal, Student B deletes same person, then Student A saves | Student A overwrites deletion → ghost record reappears | ✅ **Use optimistic concurrency control**: include `updatedAt` timestamp in GET response, send it in PUT payload. Backend rejects update if `updatedAt` mismatch (HTTP 409 Conflict). | Backend — add `@Version()` decorator to entity; Frontend — read & send `updatedAt` |
| EC-8 | Student uses special characters in `full_name` (e.g., emojis, RTL scripts) | Display issues, potential XSS if rendered unescaped | ✅ **Allow all Unicode characters in `full_name`**. Backend stores as-is. Frontend renders using `React.createElement` (not `dangerouslySetInnerHTML`). No sanitization — names are trusted input. | Backend — no change; Frontend — ensure safe rendering in table/modal |
| EC-9 | Student portal loads pickup list before auth is fully resolved (e.g., race condition) | Empty list shown → user thinks no pickup persons exist | ✅ **Frontend shows skeleton loader until `state.auth.user.studentId` is available**. Do not call `GET /studentPickUpPersons` until auth state is `authenticated`. | Frontend — add auth guard hook before data fetch |
| EC-10 | Audit log fails silently (e.g., DB connection down) | Loss of compliance evidence | ✅ **Audit logging is best-effort, not transactional**. If audit log fails, proceed with main action (create/update/delete) — but log error to Sentry. Do NOT roll back main action. | Backend — wrap audit call in try/catch + Sentry capture |

## Open Questions (To Be Resolved Before Implementation)

| # | Question | Owner | Status |
|---|----------|-------|--------|
| OQ-1 | Should `phone_number` be encrypted at rest? Current `student_pick_up_persons.phone_number` is plaintext. | Security | ⏳ Pending review |
| OQ-2 | Is there a business requirement to limit total number of pickup persons per student? (e.g., max 5) | Product | ⏳ Pending confirmation |

> 🔔 **Note**: All decisions above are final unless updated via formal change request (`notes.md`). Engineering must implement *exactly as decided* — no reinterpretation.
