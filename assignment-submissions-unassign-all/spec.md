---
feature: Unassign All Submissions (Bulk Remove StudentAssignments by Assignment)
slug: assignment-submissions-unassign-all
status: draft
author: System Analyst
date: 2026-09-04
target_release: TBD
---

# Unassign All Submissions — Bulk Remove StudentAssignments for One Assignment

## Overview

Add a single-action "Unassign All" capability on the Assignment Submissions screen (Teacher portal, Lesson Builder) so a teacher can remove **every** student submission tied to one assignment in one click, instead of un-assigning each student row one by one. The action performs a **soft-delete (bulk)** of all `StudentAssignment` rows linked to the selected assignment, then the UI refreshes to empty. The goal is to let a teacher recover quickly when an assignment was wrongly distributed to a whole class (e.g., wrong submission date/time, wrong class) without tedious per-row deletion.

## Problem / Motivation

The Submissions tab at `/resources/:levelId/:lessonId/assignments/:assignmentId?...&tab=submissions` currently offers only a **per-student** unassign (row `...` → unassign → `deleteStudentAssignment`). When a teacher mis-assigns an assignment to an entire class — the exact scenario reported (wrong submission scheduling for the whole class) — they must click unassign + confirm for every single student. For large classes this is slow, error-prone, and wastes teacher time.

A bulk "Unassign All" removes all student assignment links for that assignment atomically, after a single confirmation, and lets the teacher re-distribute correctly.

## Scope

### In Scope
- New backend endpoint to bulk soft-remove all `StudentAssignment` rows for a given assignment.
- New frontend "Unassign All" button on the Submissions tab, visible only to the assignment/chapter owner.
- Confirmation dialog (`bbsConfirm`) summarizing the destructive nature.
- UI refresh of the submissions list after success.
- Reuse of the existing `deleteAllByAssignment` service method (already present in `StudentAssignmentService`).

### Out of Scope
- Bulk delete scoped to a filtered subset (e.g., only one classroom) — that remains per-row for now.
- Hard delete / physical purge — keep soft-delete (existing behaviour).
- Changes to grading, answer data, or the assignment itself — only the student linkage (submissions) is removed.
- Undo / restore of bulk removal (not in this iteration).
- Admin or Student portal usage of this action.

## User Stories

### As a Teacher (chapter owner)
I want to remove every student submission for an assignment in one action, so that I can quickly correct a mistake where I wrongly assigned/submitted an assignment to a whole class.

### As a Teacher (chapter owner)
I want a clear confirmation before the bulk removal, so that I do not accidentally wipe submissions.

## Acceptance Criteria

- [ ] On the Submissions tab, an "Unassign All" button is shown only when `isOwnChapter` is true **and** there is at least one submission in the current (filtered) list.
- [ ] Clicking "Unassign All" opens a `bbsConfirm` dialog warning that **ALL** students will be unassigned from the assignment.
- [ ] On confirm, the frontend calls the bulk-delete endpoint for the current `assignmentId`.
- [ ] On success, the submissions list refreshes and shows empty (no stale rows).
- [ ] The removal is a soft-delete (preserves audit / restore-ability) consistent with existing `deleteStudentAssignment`.
- [ ] Non-owners (and students) cannot trigger the bulk endpoint (UI hidden; route-level guard recommended — see Business Rules #5).
- [ ] Empty case handled gracefully (deleting when none exist returns `count: 0`, successful 2xx, no error).
- [ ] Re-clicking "Unassign All" after a successful run shows the button disabled (list is empty).

## UI / UX Changes

Add a danger-styled "Unassign All" `BBSButton` in the submissions header row, aligned next to the existing classroom filter `BBSResourceSelect`. Disabled when the current filtered list is empty. Gated behind `isOwnChapter` (teacher is the chapter creator), consistent with the existing per-row unassign visibility.

Wire a `handleUnassignAll()` that wraps `fromApi.deleteAllStudentAssignmentsByAssignment(assignmentId)` in a `bbsConfirm`; on confirmed success, call `studentAssignmentsApi?.refresh()` to clear stale rows.

### Affected Portals
- [ ] Admin (client/)
- [ ] Student (client-student/)
- [x] Teacher (client-teacher/)

Reference file (not edited in this brief):
`smartbag/bbs/client-teacher/src/views/lessonBuilder/assignments/AssignmentDetailsSubmissions.jsx`

## API Changes

| Method | Path | Description |
|--------|------|-------------|
| DELETE | `/api/v1/studentAssignments/assignment/:assignmentId` | Bulk soft-remove all `StudentAssignment` rows for the given assignment. Returns `{ data: [], count }` where `count` is the number of removed rows. |

Full contract: see `api-contract.md`.

## Database Changes

No schema migration required. The backend `deleteAllByAssignment` in `StudentAssignmentService` already exists (`smartbag/api_nest/src/modules/student-assignment/student-assignment.service.ts`) and performs `StudentAssignment.softRemove(assignedAssignments)` — a soft delete on the existing `student_assignment` table. The **only gap** is the missing controller route exposing it.

### New Tables
- None

### Modified Tables
- None (soft-delete column on `student_assignment` already present via `BaseEntityWithDates`)

### Migrations
- None

## Business Rules / Validation

1. Require a valid `assignmentId` (numeric, `ParseIntPipe`). If the assignment does not exist → `NoAssignmentFoundError` (already thrown by `deleteAllByAssignment`).
2. Only remove `StudentAssignment` rows belonging to that assignment (filter `where: { assignment: { id } }`).
3. Use soft-remove, not hard delete — preserve existing behaviour and audit trail.
4. Return count of removed rows; if zero rows matched, return `count: 0` and a successful (2xx) response, not an error.
5. Authorization: in the teacher portal, only the chapter owner sees the button. The api_nest `StudentAssignmentController` is `Scope.REQUEST` and relies on `req.user`; the existing single-delete route has **no** explicit ownership guard, so bulk should align with that existing convention — but a route-level teacher-role guard is **recommended** to prevent cross-teacher bulk removal. Flagged as open decision in `edgecases.md` (EC-03).
6. Frontend must refresh the list after the bulk call (see API Changes caveat) — an empty `data: []` MERGE will **not** drop stale rows from the redux store.

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| Assignment not found | 404 | `NoAssignmentFoundError` (existing) |
| Unauthorized / forbidden (if guard added) | 403 | existing access error |
| DB / unexpected | 500 | generic |

The frontend should show the standard `bbsToaster.danger` on failure (handled by `makeApiRequestThunk` `showError`) and, on success, refresh the list rather than trusting the empty merge.

## Dependencies

- Existing `StudentAssignmentService.deleteAllByAssignment(assignmentId)` — already implemented, needs a controller route.
- Existing `AssignmentDetailsSubmissions.jsx` list + `isOwnChapter` gate + `studentAssignmentsApi?.refresh()`.
- `bbsConfirm` and `BBSButton` from `bbs-client-common`.
- Existing `fromApi.deleteStudentAssignment` pattern for the action-layer shape.
- Redux pattern in `makeApiRequest.js` (`ACTION_TYPES.MERGE` / `DELETE`) — see `api-contract.md` caveat.

## Implementation Checklist (suggested)

Backend (`api_nest`):
1. In `student-assignment.controller.ts`, add before `@Delete(':id')`:
   ```ts
   @Delete('/assignment/:assignmentId')
   async deleteAllByAssignment(
     @Param('assignmentId', ParseIntPipe) assignmentId: number,
   ) {
     const count = await this.studentAssignmentService.deleteAllByAssignment(assignmentId);
     return { data: [], count };
   }
   ```
   (Register the more specific `/assignment/:assignmentId` before the generic `:id` route to avoid shadowing.)

Frontend (`client-teacher`):
2. In `fromApi.js`, add `deleteAllStudentAssignmentsByAssignment(assignmentId)` mirroring `deleteStudentAssignment` but targeting `/studentAssignments/assignment/${assignmentId}`.
3. In `AssignmentDetailsSubmissions.jsx`, add `handleUnassignAll()` + a gated "Unassign All" `BBSButton`; on confirm, dispatch the action then call `studentAssignmentsApi?.refresh()`.
