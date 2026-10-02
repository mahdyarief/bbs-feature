---
feature: Subject Folder Auto-Load (Filter as Visibility Only)
slug: subject-folder-auto-load
status: draft # draft | in-review | approved | implemented
author: System Analyst
date: 2026-10-02
target_release: TBD
---

# Subject Folder Auto-Load (Filter as Visibility Only)

## Overview

Ubah halaman **Subject Folder** di teacher portal (`/file-sharing/subject`, komponen `SubjectFiles.jsx`) agar data subject folders/files langsung dimuat saat halaman dibuka — mencakup semua subject dan level yang diajar guru tersebut di academic year berjalan — tanpa mengharuskan user memilih filter (Cohort Level / Subject) terlebih dahulu. Filter (level dropdown + subject pill buttons) tetap ada tetapi perannya hanya **show/hide** data yang sudah dimuat, bukan sebagai gerbang fetch API.

## Problem / Motivation

Saat ini halaman Subject Folder kosong saat pertama dibuka: user wajib memilih Cohort Level dulu, lalu memilih salah satu subject pill, baru tabel files muncul. Ini menambah friksi untuk guru yang hanya ingin langsung melihat/mengelola files-nya, dan tidak konsisten dengan ekspektasi "buka halaman → data langsung terlihat". Data konteks: elena.kjp (`employee.id 20854`) di AY 2026/2027 mengajar **English Language** di **Primary 4 Faith** dan **Primary 4 Hope** — guru seperti ini harus langsung melihat kedua folder tersebut tanpa klik filter apa pun.

## Scope

### In Scope
- `bbs/client-teacher/src/views/fileSharing/SubjectFiles.jsx` — ubah strategi fetch + render:
  - Fetch daftar subjects guru (`fromApi.getSubjects` dengan `teacherId: selfUser.id`, `breakthrough: true`, `pageSize: 0`) **tanpa** conditional `masterLevelId`.
  - Fetch subject files (`fromApi.getSubjectFiles` dengan `pageSize: 0`) untuk **semua** subject guru di AY berjalan **tanpa** conditional `activeSubject` — kirim daftar subjectIds sekaligus atau tanpa `subjectId` (semua filter di `GetSubjectFileDto` opsional, jadi fetch tanpa filter valid).
  - Level dropdown (`masterLevelId`) dan subject pills (`activeSubject`) menjadi **client-side visibility filter** atas data yang sudah ada (atau cukup menjadi query param tambahan yang me-refetch, tetapi fetch awal tanpa filter tetap jalan).
- Empty state yang informatif bila guru tidak memiliki subject/files di AY berjalan.

### Out of Scope
- Perubahan backend (`api_nest` `subject-file` / `subject` modules) — tidak diperlukan karena semua parameter filter DTO sudah `@IsOptional()`.
- Perubahan skema database / migrasi.
- Halaman `AcademicDepartmentFiles.jsx` (fitur terpisah).

## User Stories

### As a Teacher
I want to membuka halaman Subject Folder dan langsung melihat semua subject folders/files yang saya ampu di academic year berjalan
So that saya tidak perlu memilih filter level/subject terlebih dahulu

### As a Teacher
I want to memakai filter level/subject hanya untuk menyembunyikan/menampilkan data
So that saya bisa fokus ke subject tertentu tanpa kehilangan konteks daftar lengkap

## Acceptance Criteria

- [ ] Buka halaman Subject Folder tanpa query param filter → daftar subjects guru (AY berjalan) langsung tampil sebagai pills, dan tabel files langsung terisi (tidak kosong, tidak perlu klik apa pun).
- [ ] Memilih Cohort Level di dropdown hanya memfilter pills + tabel yang terlihat (data tidak di-fetch ulang dari nol / tidak blank saat dropdown belum dipilih).
- [ ] Mengklik subject pill hanya menyorot/memfilter tabel ke subject tersebut; mengklik ulang (toggle off) mengembalikan tampilan semua subjects.
- [ ] Guru tanpa subject di AY berjalan melihat empty state yang jelas (bukan tabel kosong tanpa penjelasan).
- [ ] Tidak ada regresi: pagination, sort, dan search `title` tetap berfungsi.

## UI / UX Changes

Halaman sama, urutan interaksi berubah: pills subject + tabel terisi sejak awal; dropdown level dan pills menjadi filter visibilitas. Portal: Teacher.

### Affected Portals
- [ ] Admin (client/)
- [ ] Student (client-student/)
- [x] Teacher (client-teacher/)

## Root Cause (ringkasan — detail di notes.md)

Fetch digate oleh argumen `conditional` dari `useFromApi`:

```jsx
// SubjectFiles.jsx lines 64-72 — files HANYA di-fetch jika activeSubject truthy
const sfApi = useFromApi(
  fromApi.getSubjectFiles({ ...query, pageSize: 0, subjectId: activeSubject?.id }),
  [...Object.values(query), activeSubject?.id],
  () => activeSubject            // ← gate: halaman dibuka → undefined → TIDAK fetch
);

// SubjectFiles.jsx lines 81-90 — subjects HANYA di-fetch jika masterLevelId truthy
const subjectApi = useFromApi(
  fromApi.getSubjects({ ...query, pageSize: 0, breakthrough: true, teacherId: selfUser?.id }),
  [...Object.values(query)],
  () => masterLevelId           // ← gate: dropdown belum dipilih → TIDAK fetch
);
```

Mekanisme gate (`bbs/client-teacher/src/hooks/useFromApi.js` lines 60-66):

```js
useEffect(() => {
  if (conditional()) {
    fetch();
  } else {
    setLoading(false);          // ← fetch dilewati sepenuhnya
  }
}, [dispatch, conditional(), refreshState, ...dependencyList]);
```

Render pun digate: `{masterLevelId && subjects.map(...)}` (line 147) — pills tidak dirender sebelum level dipilih.

Backend **tidak menghalangi**: `GET /api/v1/subjectFile` → `GetSubjectFileDto` semua field `@IsOptional()` (`subjectId`, `level`, `masterLevelId`, `createdByTeacherId`); `GET /api/v1/subjects` mendukung `teacherId` + `breakthrough: true` (melewati headDepartments/coordinator check — komentar di `subject.service.ts` line 237 menyebut eksplisit "used on the subject-folder feature in teacher-portal"). Jadi fetch tanpa filter valid di sisi API.

## API Changes

Tidak ada perubahan API. Referensi kontrak eksisting:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/subjects?teacherId={id}&breakthrough=true&pageSize=0` | Daftar subjects yang diajar guru (tanpa gate level) |
| GET | `/api/v1/subjectFile?pageSize=0` (+ opsional `subjectId`/`masterLevelId`) | Daftar subject files; semua filter opsional |

## Database Changes

Tidak ada.

### New Tables
- —

### Modified Tables
- —

### Migrations
- —

## Business Rules / Validation

1. Cakupan data awal = subjects/files milik guru login (`selfUser.id` → `teacherId`) pada academic year berjalan. Cara penentuan "AY berjalan" mengikuti pola eksisting filter AY di teacher portal (lihat EC-03).
2. Filter tidak pernah boleh menyebabkan tabel kosong karena "belum memilih" — state kosong hanya valid bila memang tidak ada data, atau bila kombinasi filter tidak cocok dengan data yang dimuat.
3. `pageSize: 0` (ambil semua) dipertahankan untuk fetch awal agar filter client-side bekerja di atas dataset lengkap; pagination tabel tetap seperti sekarang.

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| Guru tanpa subject di AY berjalan | 200 (empty) | Empty state "No subjects assigned for this academic year" |
| Subject tanpa files | 200 (empty) | Empty state tabel per-subject yang sudah ada |

## Dependencies

- `bbs/client-teacher/src/hooks/useFromApi.js` — hook fetch dengan `conditional` gate.
- `bbs/client-teacher/src/actions/fromApi.js` — `getSubjects` (±line 116-123), `getSubjectFiles` (±line 2342-2349).
- `api_nest/src/modules/subject-file/` — controller + `GetSubjectFileDto` (tanpa perubahan).
- `api_nest/src/modules/subject/subject.service.ts` — `findAll` dengan `teacherId` + `breakthrough` (tanpa perubahan).
