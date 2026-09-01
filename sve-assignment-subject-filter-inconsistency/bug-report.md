# Bug Report: SVE Assignment List Tidak Filter by Subject Teacher

## Metadata
- **Reported by:** Mahdy Arief (via Jam.dev)
- **Date:** 2026-09-01
- **Jam URL:** https://jam.dev/c/28f2a529-e459-4802-88ef-b0fa80185547
- **Severity:** Medium
- **Module:** SVE (Subject Verification/Evaluation) — Assignment List
- **Environment:** Teacher Portal (teacher.smartbag.binabangsaschool.com)

## Deskripsi Bug
Saat user (teacherId=20703) membuka halaman **SVE Assignment** (`/sves`), data assignment yang tampil seharusnya hanya untuk pelajaran Chinese Language (sesuai dengan subject yang diajar oleh teacher tersebut). Namun, assignment dari pelajaran **lain juga ikut muncul** — data tidak filter by subject teacher.

## Steps to Reproduce
1. Login sebagai teacher yang mengajar Chinese Language (dan bukan Head of Department)
2. Buka halaman /sves
3. Observasi list SVE Assignment cards yang muncul

## Expected Behavior
Hanya assignment SVE untuk subject yang diajar oleh teacher tersebut (Chinese Language) yang muncul, sesuai dengan data `SubjectYear` placement teacher.

## Actual Behavior
Semua SVE assignment dimana user terdaftar sebagai **setter, vetter, ATAU expert** muncul — tanpa filter by subject. Akibatnya, jika user pernah ditugaskan sebagai setter/vetter/expert untuk pelajaran lain (misal Mathematics, English), assignment pelajaran tersebut juga muncul.

## Root Cause Analysis

### Frontend
**File:** `bbs/client-teacher/src/views/sve/Sve.jsx` (lines 121-128)

`getSveExamTypesByUser` dipanggil **tanpa** parameter `bySubjectTeacher: true`:

```js
const sveExamTypeUserApi = useFromApi(
  fromApi.getSveExamTypesByUser({
    page,
    pageSize: normalizedPageSize,
    subjectName,
    academicYearId,
    masterSveExamTypeId
    // ❌ bySubjectTeacher: true tidak dipassing
  }),
```

### Backend
**File:** `api_nest/src/modules/sve-exam-type/sve-exam-type.service.ts` (lines 411-436)

Di method `findAllByUser`, terdapat `if/else` dua mode:

- **Jika `bySubjectTeacher: true`** (line 412-425): filter by subject IDs dari `SubjectYear` — hanya menampilkan SVE untuk subject yang diajar teacher
- **Jika `bySubjectTeacher: false` (default)** (line 426-435): filter by `(setterId OR vetterId OR expertId)` — TANPA filter subject sama sekali

Karena frontend tidak passing `bySubjectTeacher`, backend masuk ke `else` branch (line 426-435), menghasilkan query:
```sql
WHERE (set.setterId = 20703 OR set.vetterId = 20703 OR set.expertId = 20703)
```
Query ini mengembalikan **semua SVE** dimana user adalah setter/vetter/expert, tanpa peduli subject apa.

### Confirmation via Network Request
Jam network log menunjukkan frontend memanggil:
```
GET /api/v1/sveExamTypes/byUser?page=0&pageSize=12&academicYearId=27&activeStatus=ACTIVE
```
(tanpa parameter `bySubjectTeacher=true`)

### Existing Pattern
File lain di codebase SUDAH menggunakan `bySubjectTeacher: true` dengan benar:
- `bbs/client-teacher/src/views/sve/files/SveFiles.jsx` (line 53, 69) ✅
- `bbs/client-teacher/src/views/bmt-file-download/BmtFileDownload.jsx` (line 81) ✅
- `bbs/client-teacher/src/views/sve/Sve.jsx` (line 121-128) ❌ **ketinggalan**

## Saran Fix

### Option A (Recommended — Frontend only)
**File:** `bbs/client-teacher/src/views/sve/Sve.jsx` (line 122-128)

Tambahkan `bySubjectTeacher: true` ke parameter `getSveExamTypesByUser`:

```js
const sveExamTypeUserApi = useFromApi(
  fromApi.getSveExamTypesByUser({
    page,
    pageSize: normalizedPageSize,
    subjectName,
    academicYearId,
    masterSveExamTypeId,
    bySubjectTeacher: true   // ✅ tambahkan ini
  }),
```

**Efek:** Backend akan filter by subject dari `SubjectYear` — hanya SVE untuk subject yang diajar teacher yang muncul. Filter setter/vetter/expert tidak lagi dipakai (backend pakai if/else).

### Option B (Backend fix — Combine both filters)
**File:** `api_nest/src/modules/sve-exam-type/sve-exam-type.service.ts` (lines 411-436)

Ubah `if/else` menjadi **AND** — apply BOTH filter bySubjectTeacher AND setter/vetter/expert:

```typescript
// Apply subject filter (always for non-HOD users)
const subjectTeaching = await SubjectYear.find({
  where: {
    teachers: { id: this.req.user.id },
    classYear: { academicYear: { id: Number(options.academicYearId) } },
  },
  relations: { subject: true },
});
const subjectsIds = subjectTeaching.map((st) => st.subjectId);
if (subjectsIds.length) {
  sveExamTypeQuery.andWhere('subject.id IN (:...subjectsIds)', { subjectsIds });
}

// Apply setter/vetter/expert filter
sveExamTypeQuery.andWhere(
  '(set.setterId = :setterId OR set.vetterId = :vetterId OR set.expertId = :expertId)',
  { setterId: this.req.user.id, vetterId: this.req.user.id, expertId: this.req.user.id },
);
```

**Efek:** User hanya melihat SVE untuk subject yang diajar DAN mereka ditugaskan sebagai setter/vetter/expert.

## Files Terkait
| File | Path | Peran |
|---|---|---|
| Halaman SVE List | `bbs/client-teacher/src/views/sve/Sve.jsx` | Frontend — panggil API |
| API Call Definition | `bbs/client-teacher/src/actions/fromApi.js` | Frontend — definisi endpoint |
| Backend Controller | `api_nest/src/modules/sve-exam-type/sve-exam-type.controller.ts` | Backend — route handler |
| Backend Service | `api_nest/src/modules/sve-exam-type/sve-exam-type.service.ts` | Backend — query logic |
| Backend DTO | `api_nest/src/modules/sve-exam-type/dto/get-sve-exam-type.dto.ts` | Backend — parameter definition |