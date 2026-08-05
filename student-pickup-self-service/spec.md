---
status: draft
owner: product
date: 2026-08-05
feature-id: FEAT-STUDENT-PICKUP-SELF-SERVICE
---

# Student Pickup Self-Service

## Overview
Enable students (and their parents/guardians) to manage their own list of authorized pickup persons directly via the student portal — without requiring admin intervention.

## Problem Statement
Currently, only school administrators can add or update pickup persons via `/admin/student-pick-up`. This creates operational delays, poor user experience, and increases support load. Parents must contact admin for every change — even urgent ones (e.g., last-minute caregiver switch).

## Goals
- Reduce admin support tickets related to pickup person updates by ≥70% within 30 days of release.
- Achieve ≥85% adoption among active students within Q3.
- Maintain full data integrity and child safety compliance (ownership enforcement, audit logging).

## Scope

### ✅ In Scope
- Frontend: Add "Add", "Edit", and "Delete" actions to `/student/pickup` page (`bbs/client/src/views/attendance/studentPickUp/StudentPickUp.jsx`).
- Backend: Enable existing `POST /studentPickUpPersons`, `PUT /studentPickUpPersons/:id`, and `DELETE /studentPickUpPersons/:id` endpoints for role `student`, with strict ownership validation (`studentId === req.user.studentId`).
- Security: Leverage existing ACL system (`ModulesTypeEnum.STUDENT_PICK_UP_PERSON`) — no new permissions model.
- Audit: Log all self-service actions in `audit_logs` table (existing middleware).

### ❌ Out of Scope
- Multi-student household management (e.g., shared pickup person across siblings).
- Biometric/photo verification or ID upload.
- SMS/email notifications on update (to be handled in comms epic).
- Export/print pickup list (UI-only view).

## User Stories

| As a | I want to | So that |
|------|-----------|---------|
| Student / Parent (logged in) | add, edit, or remove authorized pickup persons from my student portal | I can keep pickup information accurate and up-to-date without waiting for admin approval |
| School Admin | see clear audit logs of all self-service pickup changes | I retain visibility and accountability without manual tracking |
| Security Officer | ensure no student can modify another student’s pickup list | Data integrity and child safety policies remain enforced |

## Acceptance Criteria

| # | Requirement | Testable? |
|---|-------------|-----------|
| AC-1 | Authenticated student sees "+ Add Pickup Person" button on `/student/pickup` | ✅ Yes (UI) |
| AC-2 | Form includes fields: `full_name` (required), `relationship` (required), `phone_number` (optional), `is_primary` (checkbox, default false) | ✅ Yes (UI + schema) |
| AC-3 | Submitting form calls `POST /studentPickUpPersons` with `studentId` auto-injected from auth context | ✅ Yes (network trace) |
| AC-4 | Backend rejects `POST` if `studentId` in payload ≠ `req.user.studentId` (HTTP 403) | ✅ Yes (unit test) |
| AC-5 | List shows only pickup persons belonging to the authenticated student | ✅ Yes (DB query validation) |
| AC-6 | Each item has "Edit" and "Delete" buttons; edit opens pre-filled form; delete shows confirmation dialog | ✅ Yes (UI) |
| AC-7 | Successful create/update/delete triggers toast notification: "Pickup person updated successfully" | ✅ Yes (UX) |
| AC-8 | Audit log records `studentId`, `action`, `pickupPersonId`, `timestamp`, `actorRole=student` | ✅ Yes (log DB check) |

## Dependencies

| Dependency | Owner | Status |
|------------|-------|--------|
| ACL permission seed for `student` role (`STUDENT_PICK_UP_PERSON:CREATE`, `UPDATE`, `READ_OWN`) | Backend | ⏳ Not implemented |
| Ownership validation in `student-pick-up-person.service.create()` | Backend | ✅ Confirmed present (code scan) |
| `studentPickUpPersons.js` action library (already exists) | Frontend | ✅ Ready |
| `StudentPickUp.jsx` component (base page exists) | Frontend | ✅ Ready |

## Risks

| Risk | Mitigation |
|------|------------|
| **Ownership bypass in service logic** | Verify `create()` method enforces `dto.studentId === req.user.studentId`; add unit test if missing. |
| **Frontend sends wrong `studentId`** | Auto-inject `studentId` from Redux store (`state.auth.user.studentId`) — never accept from form input. |
| **Admins lose visibility into changes** | Ensure audit log is enabled and searchable in Kibana/LogRocket. |

## Release Notes

- **Version**: `v2.12.0` (frontend), `v3.8.0` (backend)  
- **Environment**: Staging → UAT → Production  
- **Rollout**: Gradual (10% → 50% → 100% of students)  
- **Go-live Date**: 2026-09-15  
- **Comms**: Email to parents + banner in student portal 7 days prior.