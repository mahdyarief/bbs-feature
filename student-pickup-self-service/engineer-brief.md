---
status: draft
owner: engineering
reviewed-by: security, product
date: 2026-08-05
---

# Engineering Brief: Student Pickup Self-Service

> ⚠️ This is *not* a technical design. It is a *contract* between Product and Engineering: what must be delivered, what must not change, and what is out of scope for this release.

## ✅ What Must Be Delivered

### 1. Backend Behavior (API)
- Endpoint `POST /studentPickUpPersons` **must accept requests from users with role `student`**, provided the request passes ACL check for `STUDENT_PICK_UP_PERSON:CREATE`.
- Endpoint `GET /studentPickUpPersons` **must return only records where `student_id = req.user.studentId`** — no exceptions.
- Endpoint `PUT /studentPickUpPersons/:id` and `DELETE /studentPickUpPersons/:id` **must enforce ownership**: the record’s `student_id` must match `req.user.studentId`. If not, return HTTP 403.
- All successful create/update/delete actions **must generate an audit log entry** in the `audit_logs` table with:
  - `actor_role = 'student'`
  - `actor_id = req.user.id`
  - `target_type = 'student_pick_up_person'`
  - `target_id = <created/updated/deleted id>`
  - `action = 'create' | 'update' | 'delete'`

### 2. Frontend Behavior (Student Portal)
- On page `/student/pickup`, authenticated students **must see a functional "+ Add Pickup Person" button**.
- Clicking it opens a form with exactly these fields:
  - `full_name` (text, required, max 100 chars)
  - `relationship` (text, required, max 50 chars, e.g., "Father", "Aunt")
  - `phone_number` (text, optional, E.164 format validated client-side)
  - `is_primary` (checkbox, default `false`)
- Submitting the form **must call `POST /studentPickUpPersons`** with payload:
  ```json
  {
    "full_name": "Budi Santoso",
    "relationship": "Father",
    "phone_number": "+6281234567890",
    "is_primary": true
  }
  ```
  → **`studentId` MUST NOT be sent in payload.** It is auto-injected by frontend auth context (e.g., from Redux store `state.auth.user.studentId`).
- After success, **show toast**: "Pickup person added successfully".
- Each existing pickup person in the list **must have "Edit" and "Delete" buttons**, both functional.
- Edit opens pre-filled form; delete shows confirmation dialog with text: "Are you sure you want to remove [Name] as a pickup person? This cannot be undone."

## ❌ What Must NOT Be Changed or Added

| Area | Constraint |
|------|------------|
| **Backend ownership logic** | Do not modify `student-pick-up-person.service.ts` logic that enforces `studentId === req.user.studentId`. If missing, *add it* — but do not replace or bypass existing checks. |
| **ACL permission model** | Do not introduce new permissions (e.g., `student-pick-up-person:manage-all`). Use only existing `STUDENT_PICK_UP_PERSON` module and `CREATE`/`UPDATE`/`READ_OWN` actions. |
| **Frontend data source** | Do not fetch pickup persons from any endpoint other than `GET /studentPickUpPersons`. Do not cache or merge data from admin APIs. |
| **Student identity injection** | Never accept `studentId` from form input, URL param, or localStorage. It must come *only* from authenticated session context. |
| **Audit log schema** | Do not change `audit_logs` table structure or column names. Reuse existing columns and conventions. |

## 🧩 What Is Assumed Available (Do Not Rebuild)

| Artifact | Location / Note |
|----------|----------------|
| ACL system | `src/decorators/permission.decorator.ts` + `src/modules/acl/` — already supports granular permissions. |
| Ownership validation guard | Already implemented in `student-pick-up-person.service.ts` (confirmed via code scan). |
| Audit logging middleware | `src/middlewares/audit-logger.middleware.ts` — automatically logs on success/failure for decorated endpoints. |
| Student portal auth context | `state.auth.user.studentId` is available in Redux store (`bbs/client`). |
| API action library | `bbs/client/src/actions/api/studentPickUpPersons.js` exports `createStudentPickUpPerson()`, `updateStudentPickUpPerson()`, etc. |
| Base UI components | `CButton`, `CTooltip`, `BBSDataTable`, `Helmet` — all available in `bbs/client`. |

## 🚨 Non-Negotiable Constraints

1. **Security-first**: If any path allows a student to read/write another student’s pickup person — the PR is rejected. No exceptions.  
2. **No breaking changes**: Existing admin flows (`/admin/student-pick-up`) must work *exactly as before*. This feature is additive only.  
3. **No new dependencies**: Do not add new npm packages (frontend) or npm modules (backend) without explicit security review.  
4. **All ACs in `spec.md` must pass** — this brief does not replace them; it *operationalizes* them.  

## 📋 Verification Checklist (For PR Review)

- [ ] Backend unit tests cover ownership enforcement for `create()`, `update()`, `delete()` — with mocked `req.user.studentId ≠ dto.studentId` → 403.  
- [ ] Frontend e2e test verifies: logged-in student can add/edit/delete *only their own* pickup persons.  
- [ ] Audit log entries are present and correct for all 3 actions (check DB or logs).  
- [ ] No `studentId` appears in network request payloads (verified in DevTools).  
- [ ] Admin flows (`/admin/student-pick-up`) remain fully functional and unchanged.  
