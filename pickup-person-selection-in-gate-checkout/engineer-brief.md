---
status: draft
owner: engineering
reviewed-by: security, product
date: 2026-08-05
---

# Engineering Brief: Pickup Person Display in Gate Check-Out Flow

> ⚠️ This is a strict implementation contract. Do not add selection, autocomplete, or payload changes.

## ✅ What Must Be Delivered

### 1. Frontend (`bbs/client`)
- In `PickUpModal.jsx` (`bbs/client/src/views/attendance/checking/gateChecking/PickUpModal.jsx`):
  - ✅ Add `useEffect` to call `getStudentPickUpPersons({ studentId })` when `studentId` prop changes.
  - ✅ Render a read-only section titled "Authorized Pickup Persons" below the existing name input field.
  - ✅ Format each person as: `• [full_name] ([relationship])` (e.g., `• Budi Santoso (Father)`).
  - ✅ If zero persons exist: show static message: "No authorized pickup persons registered. Please contact admin or update via student portal."
  - ✅ Show loading state (skeleton or spinner) while fetching.
  - ✅ List must not block modal open or form submission — fully asynchronous.
- No changes to `createStudentPickUpRequest()` usage or payload.

## ❌ What Must NOT Be Changed

| Area | Constraint |
|------|------------|
| **Frontend data source** | Do not fetch `studentPickUpPersons` from any endpoint other than `GET /studentPickUpPersons?studentId=:id`. |
| **UI interactivity** | Do not make list clickable, selectable, or auto-fill the name field. This is display-only. |
| **Backend logic** | Do not modify `CreateStudentPickUpRequestDto`, `student-pick-up-request.service.ts`, or audit log behavior. Zero backend changes. |
| **Modal flow** | Do not change submit button behavior, validation, or success handling. Existing flow remains untouched. |

## 🧩 What Is Assumed Available

| Artifact | Location / Note |
|----------|----------------|
| `getStudentPickUpPersons(studentId)` action | `bbs/client/src/actions/api/studentPickUpPersons.js` — already exports `getStudentPickUpPersons(query)` — pass `{ studentId }` in query. |
| `studentPickUpPersons` data shape | Confirmed: includes `full_name` and `relationship` (from `CreateStudentPickUpPersonDto`). |
| `PickUpModal.jsx` structure | Existing layout supports adding new section below name input — no refactor needed. |
| Loading UX patterns | `BBSDataTable` and other components use skeleton loaders — reuse same pattern. |

## 🚨 Non-Negotiable Constraints

1. **Zero impact on submit flow**: Name field remains manual; no auto-fill, no dropdown, no required selection.  
2. **No backend dependency**: This is frontend-only. If API fails, show error message — do not crash or block modal.  
3. **No caching across students**: Fetch fresh list every time `studentId` changes. Do not reuse previous student’s list.  
4. **Responsive design**: List must wrap or scroll gracefully on small screens (e.g., tablet kiosk).  

## 📋 Verification Checklist (For PR Review)

- [ ] `PickUpModal.jsx` displays "Authorized Pickup Persons" section below name input (UI).  
- [ ] Section shows correct formatting: `• Full Name (Relationship)`.  
- [ ] Loading state appears while fetching (UI).  
- [ ] Empty state message shown when zero persons exist (UI).  
- [ ] Modal opens instantly — list appears after API response (DevTools timing).  
- [ ] `createStudentPickUpRequest()` payload unchanged — no `student_pick_up_person_id` added (network trace).  
- [ ] Switching between students in modal triggers fresh fetch (network trace).  
