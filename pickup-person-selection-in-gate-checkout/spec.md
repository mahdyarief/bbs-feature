---
status: draft
owner: product
feature-id: FEAT-PICKUP-PERSON-IN-GATE-CHECKOUT
---

# Pickup Person Display in Gate Check-Out Flow

## Overview
Display the list of *pre-registered authorized pickup persons* inside the `PickUpModal` during gate check-out — as a read-only reference for gate staff to verify identity, without changing or slowing down the existing manual name entry flow.

## Problem Statement
Currently, gate staff must manually type the pickup person’s name during check-out (e.g., in `GateChecking.jsx`). They have no way to quickly verify:
- ❌ Whether the person they’re typing is actually authorized
- ❌ How many people are authorized (e.g., father, aunt, driver)
- ❌ Spelling consistency (e.g., "Budi Santoso" vs "Budy Santoso")
This increases risk of human error and reduces confidence in release verification.

## Goals
- ↑ 100% of gate staff report improved confidence in verifying pickup person identity within 1 week of release.
- ✅ Every `PickUpModal` shows accurate, real-time list of registered pickup persons for the student.
- ✅ Zero impact on check-out time (< 0.5s additional load time, measured).

## Scope

### ✅ In Scope
- Frontend: Extend `PickUpModal.jsx` to fetch and display `studentPickUpPersons` for the selected student as a read-only list (not selectable, not editable).
- UI: Render as clean, scannable list below the existing name input field — e.g., "Authorized for [Student Name]:"
  - • Budi Santoso (Father)
  - • Rani Wijaya (Aunt)
  - • PT ABC Transport (Driver)
- Backend: No changes required — only frontend display using existing `GET /studentPickUpPersons?studentId=:id` endpoint.
- Performance: List must load asynchronously without blocking modal open or form submission.

### ❌ Out of Scope
- Any change to `createStudentPickUpRequest()` payload or backend logic.
- Adding dropdown, autocomplete, or auto-fill functionality.
- Modifying audit log schema or content — no new fields added.
- Caching strategy beyond default React + Redux — use existing patterns.

## User Stories

| As a | I want to | So that |
|------|-----------|---------|
| Gate Staff | see who is authorized to pick up this student — right inside the pickup modal | I can quickly verify identity before releasing, reducing errors and increasing safety |
| School Admin | know gate staff have real-time access to authorized pickup data | I improve operational oversight without adding process friction |
| Parent (via portal) | know gate staff can see exactly who I registered | I feel more confident in the school’s release control |

## Acceptance Criteria

| # | Requirement | Testable? |
|---|-------------|-----------|
| AC-1 | When opening `PickUpModal` for a student, a section titled "Authorized Pickup Persons" appears below the name input field | ✅ Yes (UI) |
| AC-2 | Section displays all active `studentPickUpPersons` for that student, formatted as `• [Full Name] ([Relationship])` | ✅ Yes (UI + network trace) |
| AC-3 | If zero persons exist, section shows: "No authorized pickup persons registered. Please contact admin or update via student portal." | ✅ Yes (UI) |
| AC-4 | List loads asynchronously — modal opens instantly; list appears after API response | ✅ Yes (DevTools timing) |
| AC-5 | Loading state shown (e.g., skeleton or spinner) while fetching | ✅ Yes (UI) |
| AC-6 | No change to `createStudentPickUpRequest()` behavior — name field remains manual, unmodified | ✅ Yes (network trace + no payload change) |
| AC-7 | List updates automatically if `studentId` prop changes (e.g., switching students in modal) | ✅ Yes (UI + network trace) |

## Dependencies

| Dependency | Owner | Status |
|------------|-------|--------|
| `PickUpModal.jsx` exists and is used in `GateChecking.jsx` | Frontend | ✅ Ready |
| `getStudentPickUpPersons(studentId)` action exists | Frontend | ✅ Ready (`bbs/client/src/actions/api/studentPickUpPersons.js`) |
| `studentPickUpPersons` data includes `full_name` and `relationship` fields | Backend | ✅ Confirmed in `CreateStudentPickUpPersonDto` |

## Risks

| Risk | Mitigation |
|------|------------|
| **Slow API response delays modal usability** | Implement timeout (5s) + fallback to cached last-known list (if available) + clear error message. |
| **List renders but data is stale** | Use `useEffect` with `studentId` dependency + skip if `studentId` is falsy. Do not cache across students. |
| **Layout breaks on small screens** | Use responsive grid/list; max-height + scroll if > 5 items. |

## Release Notes

- **Version**: `v2.12.1` (frontend only)  
- **Environment**: Staging → UAT → Production  
- **Rollout**: All gate stations (no feature flag needed)  
- **Go-live Date**: 2026-09-22  
- **Comms**: Briefing for gate staff + 1-line SOP update: "Authorized pickup persons now visible in pickup modal."