---
title: Lesson Plan — HBL/HOB Resources tidak menampilkan file dari Lesson Plan Library (modal "Browse File Library" selalu kosong)
status: open
severity: major
product: BBS LMS
portal: Teacher
author: System Analyst
date: 2026-09-30
jam: ~
---

# HBL/HOB Resources pada Lesson Plan tidak menampilkan file Lesson Plan Library

## Summary

Pada **Teacher Portal**, saat membuat/mengedit lesson plan dan memilih **HBL/HOB Resources** (checkbox Material Resources → tombol **Browse**), modal **"Browse File Library"** tidak pernah menampilkan file yang sudah ada di halaman **Lesson Plan Library** — selalu tampil `No library files found`. Dampaknya teacher tidak bisa memilih resource HBL/HOB dari library saat menyusun lesson plan, padahal file-nya sudah di-upload dan tampil normal di halaman Lesson Plan Library.

Root cause: modal HBL memanggil **endpoint yang salah** — `GET /subjectFile` (modul File Sharing / tabel `subjectFile`) alih-alih `GET /lesson-plans/file-library` (Lesson Plan Library / tabel `lesson_plan_material_file`). Karena sumber datanya berbeda, file Lesson Plan Library tidak pernah muncul di modal.

**Ekspektasi:** Modal "Browse File Library" dari checkbox HBL/HOB Resources menampilkan file yang sama dengan yang tampil di halaman Lesson Plan Library (minimal untuk subject/level/AY yang relevan dengan lesson plan yang sedang dibuat), dan pencarian bekerja terhadap file tersebut.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | System Analyst |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | — |
| Portal URL | https://teacher.smartbag.binabangsaschool.com/lesson-plan/new (form create lesson plan) |
| Environment API | teacher.smartbag.binabangsaschool.com |
| Data konteks (file library yang sudah ada isi) | — |
| Browser / OS | — |

---

## Steps to Reproduce

1. Pastikan ada file di **Lesson Plan Library** (upload via menu Lesson Plan → File Library) — buka halaman tersebut dan pastikan file tampil.
2. Login ke **Teacher Portal** → **Lesson Plan** → buat lesson plan baru (atau edit).
3. Di section **Material Resources**, centang **"HBL Resources"** → klik tombol **Browse**.
4. Pada modal **"Browse File Library"**, amati list file (coba juga ketik search → Enter).

**Actual Result:** Modal selalu kosong — `No library files found` — padahal file yang sama tampil di halaman Lesson Plan Library. Network tab: request yang terkirim adalah `GET /subjectFile?pageSize=50[&fileName=...]` (bukan `/lesson-plans/file-library`).

**Expected Result:** Modal menampilkan file dari Lesson Plan Library (endpoint `GET /lesson-plans/file-library`) dan search berjalan terhadap file tersebut.

---

## Root Cause Analysis

### Bug #1 — Modal HBL memakai endpoint File Sharing, bukan Lesson Plan File Library — `LessonPlanForm.jsx` (line 215-225)

Picker HBL/HOB memanggil `getSubjectFiles` → `GET /subjectFile` (tabel `subjectFile`, diisi halaman **File Sharing**), bukan `getLessonPlanFileLibrary` → `GET /lesson-plans/file-library` (tabel `lesson_plan_material_file`, yang ditampilkan halaman **Lesson Plan Library**). Jika file hanya di-upload via Lesson Plan Library, `GET /subjectFile?pageSize=50` selalu return kosong → modal selalu `No library files found`.

```jsx
// client-teacher/src/views/lessonPlan/components/LessonPlanForm.jsx:215-225 — HOB/HBL picker
const hblApi = useFromApi(
  fromApi.getSubjectFiles({          // → GET /subjectFile ( Salah: sumber data File Sharing )
    pageSize: 50,
    fileName: hblQuery || undefined
  }),
  [hblQuery, hblModal],
  () => hblModal
);
const hblFiles = useResourceMapper("subjectFiles", hblApi.sortOrder);
```

```js
// client-teacher/src/actions/fromApi.js:2342-2348 — dipakai picker (salah)
getSubjectFiles(query) {
  return makeApiRequestThunk(HTTP_METHODS.GET, buildQueryStr(`/subjectFile`, query), ...);
}
// client-teacher/src/actions/fromApi.js:3295-3302 — sudah ada, TIDAK dipakai picker
getLessonPlanFileLibrary(query) {
  return makeApiRequestThunk(HTTP_METHODS.GET, buildQueryStr(`/lesson-plans/file-library`, query), ...);
}
```

Faktor pendukung:

- **Pembanding FE**: halaman Lesson Plan Library (`LessonPlanFileLibrary.jsx:60-130`) memakai `getLessonPlanFileLibrary` + filter `fileTypes, subjectId, masterLevelId, masterProgrammeId, q, page/pageSize` — inilah yang user lihat berisi file.
- **Parameter search mismatch**: modal kirim `fileName=...` (param `GET /subjectFile` — `get-subject-file.dto.ts:7-40`), sedangkan library memakai `q` (`get-lesson-plan-file-library.dto.ts`). Tidak ada filter `subjectId/masterLevelId/ay/fileTypes` yang dikirim modal, sehingga konteks lesson plan (subject/level/AY) juga tidak dipakai.
- **Permission berbeda**: `GET /subjectFile` butuh `CheckPermissions(READ, SUBJECT_FILE)` (`subject-file.controller.ts:31-38`) — scoping File Sharing, bukan scope library yang dipakai teacher.
- **Bukan mismatch kategori HOB**: `LessonPlanMaterialCategoryEnum` (`lesson-plan-material-file.entity.ts:13-17`) hanya `PPT/PDF/VIDEO` dan tidak ada string `HOB` di `api_nest/src` — jadi tidak ada kategori HOB untuk difilter; "HBL/HOB Resources" hanya checkbox FE (`LessonPlanForm.jsx:697-719`) yang mereferensikan file library bebas kategori.

---

## Bukti dari Jam (~)

| Sumber | Temuan |
|--------|--------|
| **Video (t=...)** | — (belum ada recording) |
| **Network** | `GET /subjectFile?pageSize=50` saat klik Browse — bukan `GET /lesson-plans/file-library` |
| **Console** | — |
| **User events** | Centang "HBL Resources" → klik Browse → modal kosong `No library files found` |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/subject-file/subject-file.service.ts` (dipakai) vs `api_nest/src/modules/lesson-plan/lesson-plan-file-library.service.ts:70-112` (seharusnya) | Endpoint yang benar sudah ada & berfungsi; tidak ada bug query di BE untuk kasus ini |
| Backend DTO | `api_nest/src/modules/subject-file/dto/get-subject-file.dto.ts:7-40` (`fileName`) vs `api_nest/src/modules/lesson-plan/dto/get-lesson-plan-file-library.dto.ts` (`q`, `fileTypes`, `subjectId`, `masterLevelId`, `ay`) | Param modal tidak cocok dengan API library |
| Backend Entity | `api_nest/src/modules/lesson-plan/entities/lesson-plan-material-file.entity.ts:13-17` | Referensi kategori `PPT/PDF/VIDEO` — tidak ada kategori HOB |
| Frontend | `bbs/client-teacher/src/views/lessonPlan/components/LessonPlanForm.jsx:215-225, 1006-1073` | Picker memanggil endpoint salah → modal selalu kosong |
| Frontend | `bbs/client-teacher/src/actions/fromApi.js:2342-2348` vs `:3295-3302` | `getSubjectFiles` dipakai, `getLessonPlanFileLibrary` tidak dipakai |

---

## Proposed Solution Options

### Option A: Ganti picker HBL ke `getLessonPlanFileLibrary` (Recommended)

Di `LessonPlanForm.jsx:215-225`, ganti `fromApi.getSubjectFiles({ pageSize, fileName })` → `fromApi.getLessonPlanFileLibrary({ pageSize: 50, q: hblQuery || undefined, subjectId, masterLevelId, ay })` dengan nilai dari form yang sedang dibuat, dan mapper resource ke result file-library. Search memakai param `q` (bukan `fileName`).

```jsx
// LessonPlanForm.jsx:217-225 — arah perubahan
const hblApi = useFromApi(
  fromApi.getLessonPlanFileLibrary({
    pageSize: 50,
    q: hblQuery || undefined,
    subjectId: selectedSubjectId,      // dari form
    masterLevelId: selectedMasterLevelId, // dari form
    ay: academicYearId                 // dari form
  }),
  [hblQuery, hblModal],
  () => hblModal
);
const hblFiles = useResourceMapper("lessonPlanFileLibrary", hblApi.sortOrder);
```

- Data yang tampil konsisten dengan halaman Lesson Plan Library.
- Filter subject/level/AY membuat list relevan dengan lesson plan yang dibuat (opsional: longgarkan filter jika requirement-nya "semua file library").

### Option B: Samakan definisi "library" (integrasi ulang File Sharing)

Ubah backend agar `GET /subjectFile` juga mengembalikan file Lesson Plan Library (gabung dua sumber data / view), atau pindahkan upload Lesson Plan Library ke tabel `subjectFile`. Perubahan BE lebih luas dan berisiko memengaruhi halaman File Sharing yang sudah ada.

---

## Notes

- Endpoint yang benar (`GET /lesson-plans/file-library`) sudah tersedia di backend (`lesson-plan.controller.ts:76`) dan helper FE (`fromApi.js:3295`) sudah ada — fix cukup di frontend (Option A), tanpa perubahan skema DB.
- Verifikasi setelah fix: modal menampilkan file yang sama dengan halaman Lesson Plan Library; search dari modal mengembalikan hasil; checkbox hasil pilih tersimpan ke `material.hblRefs` saat save lesson plan.
- Label UI konsisten: checkbox bertuliskan "HBL Resources", judul modal "Browse File Library", istilah laporan "HOB" — perlu disamakan (HBL vs HOB) agar tidak membingungkan QA.
- Belum ada Jam recording untuk ticket ini (jam: `~`) — disarankan rekam ulang repro untuk bukti Network/Video.
