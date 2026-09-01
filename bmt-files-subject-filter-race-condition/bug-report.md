# Bug Report: BMT Files Download — Subject Filter Race Condition (HOD)

## Metadata
- **Reported by:** Mahdy Arief (via Jam.dev)
- **Date:** 2026-09-01
- **Jam URL:** https://jam.dev/c/eb14ca3e-3b5a-4165-91ae-5469c1815713
- **Severity:** Medium
- **Module:** BMT (Biannual/Mid-Term) Files Download
- **Environment:** Teacher Portal (teacher.smartbag.binabangsaschool.com)

## Deskripsi Bug
Saat user HOD (teacherId=4994) membuka halaman **BMT Files Download** (`/bmt-files`), user hanya seharusnya melihat BMT files untuk subject Chinese Language (sesuai dengan department yang dipimpin). Namun, BMT files untuk **English Language juga ikut muncul** — data tidak konsisten karena race condition pada API call.

## Steps to Reproduce
1. Login sebagai HOD (Head of Department) — misal HOD Chinese Language
2. Buka halaman `/bmt-files?pageSize=0&page=0&masterSveExamTypeId=3&academicYearId=27`
3. Observasi list BMT file cards yang muncul

## Expected Behavior
Hanya BMT files untuk subject yang sesuai dengan department HOD (Chinese Language) yang muncul.

## Actual Behavior
BMT files untuk Chinese Language DAN English Language muncul bersamaan. User juga melihat "There's No BMT Files" pada beberapa section.

## Root Cause Analysis

### Timeline API Calls (dari Jam network log)
| Waktu | Endpoint | Status | Response | Keterangan |
|---|---|---|---|---|
| 7204ms | `GET /api/v1/masterSubjects?academicDepartmentIds=` | 200 | 175 bytes | Masih kosong, headDepartments belum resolve |
| 7203ms | `GET /api/v1/headDepartments?teacherId=4994&academicYearId=27` | 200 | 2703 bytes | User adalah HOD, departmentId=5 |
| **7303ms** | **`GET /api/v1/sveExamTypes/bmtOnly?masterSubjectsIds=&bySubjectTeacher=false&...`** | **200** | **19192 bytes** | **masterSubjectsIds masih kosong → NO FILTER → semua subject** |
| 7302ms | `GET /api/v1/masterSubjects?academicDepartmentIds=5` | 200 | 20223 bytes | Baru setelah ini masterSubjects resolve |
| **7452ms** | **`GET /api/v1/sveExamTypes/bmtOnly?masterSubjectsIds=16&bySubjectTeacher=false&...`** | **200** | **11103 bytes** | **masterSubjectsIds=16 → filter Chinese Language** |

### Frontend
**File:** `bbs/client-teacher/src/views/bmt-file-download/BmtFileDownload.jsx` (lines 78-92)

```js
const bmtSveExamTypeUserApi = useFromApi(
  fromApi.getBMTSveExamTypesByUser({
    academicYearId,
    bySubjectTeacher: isHOD ? false : true,
    masterSveExamTypeId,
    substractBy: 7,
    ...(isHOD ? { masterSubjectsIds } : {})
  }),
  [academicYearId, masterSveExamTypeId],  // ❌ masterSubjectsIds tidak di dependency
  () =>
    Boolean(academicYearId) &&
    Boolean(masterSveExamTypeId) &&
    !headDepartmentsApi?.loading &&       // ⚠️ headDepartments selesai duluan
    !masterSubjectsApi?.loading           // ⚠️ masterSubjects (kosong) selesai duluan
);
```

**Race condition flow:**
1. `masterSubjects?academicDepartmentIds=` (dept kosong) resolve duluan → `masterSubjectsApi.loading = false` → bmtOnly API fire dengan `masterSubjectsIds=[]` (empty)
2. Backend `findBmtOnly` (line 569-575): `masterSubjectsIds` empty → `subjectsIds` tetap `[]` → **TIDAK ADA subject filter** → return ALL BMT files (19192 bytes)
3. Setelah headDepartments resolve dengan departmentId=5 → `masterSubjects?academicDepartmentIds=5` baru resolve → `masterSubjectsIds=[16]` → bmtOnly fire lagi, kali ini dengan filter
4. API call pertama menghasilkan data **semua subject** (termasuk English Language)

### Backend
**File:** `api_nest/src/modules/sve-exam-type/sve-exam-type.service.ts` (lines 569-590)

```typescript
// Line 569-575
if (options.masterSubjectsIds) {
  const subjects = await Subject.find({
    where: { masterSubject: { id: In(options.masterSubjectsIds) } },
  });
  subjectsIds = subjects?.map((s) => s.id);
}
// Line 577-590
const sveExamTypes = await SveExamType.find({
  where: {
    ...(subjectsIds.length > 0 && { sve: { subjectId: In(subjectsIds) } }),
    // ^^ ketika masterSubjectsIds empty, subjectsIds.length = 0 → NO FILTER
  },
});
```

### Redux Merge Problem
**File:** `bbs/client-teacher/src/actions/fromApi.js` (line 858-861)
```js
getBMTSveExamTypesByUser(query) {
  return makeApiRequestThunk(
    HTTP_METHODS.GET,
    buildQueryStr(`/sveExamTypes/bmtOnly`, query),
    null,
    ACTION_TYPES.MERGE  // ← MERGE, bukan SET!
  );
}
```

Kedua API call menggunakan `ACTION_TYPES.MERGE` ke redux store `sveExamTypeUsers`. Data dari call pertama (19192 bytes, semua subject) tetap tersimpan di store dan tidak pernah dibersihkan. Call kedua menambahkan data Chinese Language. Hasil akhir: **data dari kedua call tercampur** → English Language + Chinese Language muncul bersamaan.

## Saran Fix

### Option A (Recommended — Tambah masterSubjectsIds ke dependency array)
**File:** `bbs/client-teacher/src/views/bmt-file-download/BmtFileDownload.jsx` (line 86)

Tambah `masterSubjectsIds` ke dependency array `useFromApi` agar API call menunggu sampai masterSubjectsIds benar-benar siap:

```js
const bmtSveExamTypeUserApi = useFromApi(
  fromApi.getBMTSveExamTypesByUser({
    academicYearId,
    bySubjectTeacher: isHOD ? false : true,
    masterSveExamTypeId,
    substractBy: 7,
    ...(isHOD ? { masterSubjectsIds } : {})
  }),
  [academicYearId, masterSveExamTypeId, JSON.stringify(masterSubjectsIds)],  // ✅ tambah masterSubjectsIds
  () =>
    Boolean(academicYearId) &&
    Boolean(masterSveExamTypeId) &&
    !headDepartmentsApi?.loading &&
    !masterSubjectsApi?.loading &&
    (isHOD ? masterSubjectsIds.length > 0 : true)  // ✅ tunggu masterSubjectsIds terisi
);
```

### Option B (Backend fix — Guard empty filter)
**File:** `api_nest/src/modules/sve-exam-type/sve-exam-type.service.ts` (line 577-580)

Tambah guard agar ketika `masterSubjectsIds` dikirim tapi kosong, backend tidak mengembalikan semua data:

```typescript
// Jika isHOD mengirim masterSubjectsIds kosong, return empty array
if (options.masterSubjectsIds !== undefined && options.masterSubjectsIds.length === 0) {
  return { sveExamTypes: [] };
}
```

## Files Terkait
| File | Path | Peran |
|---|---|---|
| Halaman BMT Files | `bbs/client-teacher/src/views/bmt-file-download/BmtFileDownload.jsx` | Frontend — panggil API |
| API Call Definition | `bbs/client-teacher/src/actions/fromApi.js` | Frontend — definisi endpoint |
| Backend Service | `api_nest/src/modules/sve-exam-type/sve-exam-type.service.ts` | Backend — query logic `findBmtOnly` |
| Backend DTO | `api_nest/src/modules/sve-exam-type/dto/get-sve-exam-type.dto.ts` | Backend — parameter definition |
| Reducer | `bbs/client-teacher/src/reducers/root.js` | Frontend — redux store |