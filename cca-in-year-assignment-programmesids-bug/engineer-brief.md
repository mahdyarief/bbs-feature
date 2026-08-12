---
status: draft
owner: engineering
reviewed-by: backend, frontend
date: 2026-08-12
source-jam: https://jam.dev/c/5217a681-c314-4c90-8ab9-4c97eb4af474
severity: major
---

# Engineering Brief: Fix `invalid input syntax for type integer` on CCA In Year — Assign Student Page

> ⚠️ This is a strict implementation contract. Fix ONLY the two code paths described below. Do not refactor the page, do not touch the student assignment payload, do not change DTO semantics.

## 🐛 Bug Summary

On `admin.smartbag.binabangsaschool.com/cca-in-year-assignment/:ccaInYearId/assign` (CCA In Year — Assign Student), the **"Students in [programme]"** picker fails to load. The new-students API call returns **HTTP 400** with error banner:

```
invalid input syntax for type integer: ""
```

This is a PostgreSQL error. The request that triggers it:

```
GET /api/v2/students?page=0&activeStatus=ACTIVE&pageSize=5&sortBy=fullName
    &academicYearIdInClassYear=27&programmesIds=%2C&...
→ 400 Bad Request
```

`programmesIds=%2C` decodes to `","` — a comma with **empty strings on both sides** (`["", ""]` after split). Postgres tries to cast `""` to `integer` inside `WHERE programme.id IN ('', '')` and fails.

**Evidence (Jam recording `5217a681`):**
- User typed "publ" into the CCA Name filter on the list page, clicked **Assign Student** → assign page opens → student picker fires the failing request.
- 12 console errors, all `invalid input syntax for type integer`.
- No description / transcript in the Jam; network + events are the source of truth.

## 🔍 Root Cause

### Layer 1 — Frontend sends garbage `programmesIds`

`bbs/client/src/views/ccaInYearAssignment/CCAInYearAssignmentFormUpdate.jsx:119-121`

```js
programmesIds: ccaYearCoordinators
  ?.map((ccaCoordinator) => ccaCoordinator?.programmeId)
  ?.toString(),
```

- `ccaYearCoordinators` comes from `useResourceMapper("ccaYearCoordinators", ccaInYear?.ccaYearCoordinatorsIds || [])` (line 45-48). When the CCA-Year has coordinator IDs but the coordinator entities are **not yet loaded in the Redux store** (race on first render), the mapper returns an array of `undefined` entries.
- `[undefined, undefined].map(c => c?.programmeId)` → `[undefined, undefined]`, and `[undefined, undefined].toString()` → `","` — exactly the `%2C` seen in the network log.
- Even in the "no coordinators" case it would send `""` (empty string), which the backend also cannot cast.

### Layer 2 — Backend does not sanitize the array before `In()`

`api_nest/src/modules/student/v2/student-lms.service.ts:149-157`

```ts
if (options.programmesIds || options.academicYearIdInClassYear)
  whereFilters.classYears = {
    ...(options.academicYearIdInClassYear
      ? { academicYear: { id: options.academicYearIdInClassYear } }
      : {}),
    ...(options.programmesIds
      ? { programme: { id: In(options?.programmesIds?.split(',')) } }
      : {}),
  };
```

- `options.programmesIds.split(',')` on `","` → `["", ""]` → `In(["", ""])` → TypeORM binds `""` to an integer column → Postgres `invalid input syntax for type integer: ""` → **400**.
- **This is the crash point.** The DTO (`get-students.dto.ts:74-77`) declares `programmesIds?: string` with only `@IsString()` — any string passes validation, so the garbage value reaches the service untouched.

**Note:** the same file already contains the correct defensive pattern for `notStudentsIds` (lines 159-167): `.split(',').filter(id => id.trim() !== '').map(parseInt)` and only applies the filter when the resulting array is non-empty. The `programmesIds` path simply never adopted that pattern.

## ✅ What Must Be Delivered

### 1. Frontend — `bbs/client/src/views/ccaInYearAssignment/CCAInYearAssignmentFormUpdate.jsx` (lines 119-121)

Replace the `programmesIds` computation with one that:
- drops falsy/`undefined`/non-numeric entries before joining;
- omits the query param entirely when there are no valid programme ids (pass `undefined`, not `""`).

Suggested shape:

```js
const coordinatorProgrammeIds = ccaYearCoordinators
  ?.map((ccaCoordinator) => ccaCoordinator?.programmeId)
  ?.filter((id) => typeof id === "number" && Number.isInteger(id));

// in the getStudentsV2 call:
programmesIds: coordinatorProgrammeIds?.length ? coordinatorProgrammeIds.join(",") : undefined,
```

Constraints:
- The page must still load the picker when `programmesIds` is absent (fall back to filtering by `academicYearIdInClassYear` only — backend behavior, see below).
- Do NOT change `fullName`, `notStudentsIds`, `academicYearIdInClassYear`, `selectsObj`, or `relationsObj` params.
- Do NOT change `handleAddStudent` / `handleRemoveStudent` / `handleSubmitStudentEdit` — assignment payload stays `{ studentIds }`.

### 2. Backend — `api_nest/src/modules/student/v2/student-lms.service.ts` (lines 149-157)

Make the `programmesIds` handling defensive, mirroring the existing `notStudentsIds` pattern (lines 159-167):

```ts
const programmesIds = options.programmesIds
  ?.split(',')
  .map((id) => Number(id))
  .filter((id) => Number.isInteger(id));

if (options.academicYearIdInClassYear || (programmesIds && programmesIds.length > 0))
  whereFilters.classYears = {
    ...(options.academicYearIdInClassYear
      ? { academicYear: { id: options.academicYearIdInClassYear } }
      : {}),
    ...(programmesIds && programmesIds.length > 0
      ? { programme: { id: In(programmesIds) } }
      : {}),
  };
```

Constraints:
- When `programmesIds` is missing/empty/invalid but `academicYearIdInClassYear` is present, filter **only** by academic year (no `programme` condition).
- When both are absent, `whereFilters.classYears` must not be created at all (same as today).
- Do NOT change the `programmesIds?: string` DTO field type (`get-students.dto.ts:74-77`) — the service-level guard is sufficient; keep the change minimal.

## ❌ What Must NOT Be Changed

| Area | Constraint |
|------|------------|
| **DTO** | `GetStudentsDto.programmesIds` stays `@IsString() @IsOptional()` — no new transform, no type change. |
| **API contract** | `GET /api/v2/students` query params and response shape unchanged. |
| **Assign payload** | `updateCcaYear(ccaInYearId, { studentIds })` unchanged. |
| **Other consumers** | `programmesIds` is also sent by `bbs/client-teacher/.../leaps/*` and consumed by `class-year.v2.service.ts` / `leaps-event.service.ts` — do NOT touch those; this fix is scoped to the CCA assign page + students v2 service. |
| **CCA list page** | `CCAInYearAsignment.jsx` (filter by CCA Name) works correctly — the 200 OK on `/api/v1/ccaYears?ccaName=publ` confirms it; leave it alone. |

## 🧩 What Is Assumed Available

| Artifact | Location / Note |
|----------|----------------|
| `getStudentsV2(query)` action | `bbs/client/src/actions/api/student.js:33` — forwards query params as-is; no change needed. |
| Students v2 handler | `api_nest/src/modules/student/v2/student-lms.service.ts` — builds `whereFilters` consumed by TypeORM find with relations. |
| Defensive pattern to copy | `notStudentsIds` handling at `student-lms.service.ts:159-167` — same file, proven pattern. |
| Resource mapper behavior | `useResourceMapper` may return `undefined` entries while entities are loading — this is why `filter` is mandatory on the frontend. |

## 🚨 Non-Negotiable Constraints

1. **Frontend must never send `programmesIds` as `""` or `","`** — filter to valid integers, else omit the param.
2. **Backend must never crash on malformed `programmesIds`** — sanitize before `In()`; invalid input degrades gracefully to "no programme filter".
3. **Behavior parity**: a CCA with 1 coordinator (`programmesIds=5`) must return the exact same students as today — regression in legitimate filtering is a release blocker.
4. **No UI copy changes** — the error banner disappears because the request stops failing; do not add custom error handling text.

## 📋 Verification Checklist (For PR Review)

- [ ] **Repro (pre-fix)**: Open `/cca-in-year-assignment/:id/assign` for a CCA-Year whose coordinators load slowly / are unset → network tab shows `GET /api/v2/students?...&programmesIds=%2C...` → 400 + red banner `invalid input syntax for type integer: ""`.
- [ ] **Frontend**: with the fix, the failing request no longer contains `programmesIds` at all (or contains only valid ints like `5,7`).
- [ ] **Backend unit/manual**: `GET /api/v2/students?academicYearIdInClassYear=27&programmesIds=%2C` → **200** (was 400). `programmesIds=` (empty) → **200**. `programmesIds=5` → **200** and returns students of programme 5 only.
- [ ] **Regression**: `programmesIds=5,7` returns students in programmes 5 OR 7 (same as before).
- [ ] **Assign flow**: adding a student via `+` still calls `updateCcaYear` with `{ studentIds }` and refreshes both tables (existing behavior).
- [ ] **No DTO change**: `get-students.dto.ts` diff is empty.
- [ ] `GET /api/v1/ccaYears?ccaName=publ` on the list page still returns 200 (unchanged).

## Related Context

- Jam: https://jam.dev/c/5217a681-c314-4c90-8ab9-4c97eb4af474
- Backend crash point: `student-lms.service.ts:149-157` (spread of `In(options.programmesIds.split(','))`)
- Frontend culprit: `CCAInYearAssignmentFormUpdate.jsx:119-121` (`?.map(...)?.toString()` on not-yet-loaded coordinators)
