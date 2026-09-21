---
title: HOB Resources picker pada Lesson Plan Form tidak menampilkan file dari Lesson Plan File Library (salah sumber data)
status: open
severity: major
product: BBS LMS
portal: Teacher
author: System Analyst
date: 2026-09-21
jam: ~
---

# HOB Resources tidak menampilkan file dari Lesson Plan File Library

> Catatan penamaan: fitur ini disebut **"HOB Resources"** (Homework-Based Learning) pada requirement; di kode internal state-nya bernama `hbl` (`hblRefs`, `hbl:` refs).

## Summary

Pada **Teacher Portal**, ketika teacher membuat/mengedit lesson plan dan pada section upload file mencentang **HOB Resources** lalu menekan tombol **Browse**, modal "Browse File Library" **tidak menampilkan file-file yang tersedia di Lesson Plan File Library** (`/lesson-plan/file-library`). Seharusnya semua file yang terdapat pada lesson plan file library muncul di picker HOB Resources.

Penyebab: modal Browse HOB **query endpoint yang berbeda** dari halaman File Library — modal memanggil `GET /subjectFile` (store `subjectFiles`), sedangkan halaman File Library memanggil `GET /lesson-plans/file-library` (store `lessonPlanMaterialFiles`). Dua sumber data ini berbeda, sehingga file yang di-upload via File Library tidak pernah muncul di HOB picker.

**Ekspektasi:** Modal Browse HOB Resources menampilkan daftar file yang sama dengan Lesson Plan File Library (fileName, kategori, subject, level, programme), dengan search/pagination yang berfungsi.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | System Analyst |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | — |
| Portal URL | https://teacher.smartbag.binabangsaschool.com/lesson-plan/create (form lesson plan) |
| Environment API | teacher.smartbag.binabangsaschool.com |
| Data konteks (lesson plan / file library) | — |
| Browser / OS | — |

---

## Steps to Reproduce

1. Login ke **Teacher Portal**, upload/pastikan ada beberapa file di menu **Lesson Plan → File Library** (`/lesson-plan/file-library`).
2. Buka form **Lesson Plan** (create atau edit).
3. Scroll ke section **Materials**, centang **HOB Resources**, klik tombol **Browse**.
4. Amati isi modal "Browse File Library".

**Actual Result:** File-file yang ada di Lesson Plan File Library tidak muncul di modal; daftar yang tampil berasal dari sumber data lain (subject files) sehingga terasa kosong/tidak relevan.

**Expected Result:** Modal menampilkan semua file dari Lesson Plan File Library, dan file yang dipilih tersimpan sebagai `hbl:` refs pada material resources lesson plan.

---

## Affected Files / Root Cause Analysis

- `bbs/client-teacher/src/views/lessonPlan/components/LessonPlanForm.jsx`
  - **Line ~215-225 (root cause):** data source modal HOB salah store:
    ```js
    const hblApi = useFromApi(
      fromApi.getSubjectFiles({ pageSize: 50, fileName: hblQuery || undefined }),
      [hblQuery, hblModal],
      () => hblModal
    );
    const hblFiles = useResourceMapper("subjectFiles", hblApi.sortOrder);
    ```
    → memanggil `GET /subjectFile` (`fromApi.js` line ~2333-2339), store `subjectFiles`.
  - Line ~697-719: UI HOB — `LpCheck id="mat-hbl" label="HOB Resources"` + tombol `Browse` → `setHblModal(true)`.
  - Line ~1006-1052: modal `CModal` "Browse File Library" — search input (`hblSearch`/`hblQuery`) dan render `hblFiles` (checkbox → `material.hblRefs`).
- Bandingkan dengan **Lesson Plan File Library** (sumber yang diharapkan):
  - `bbs/client-teacher/src/views/lessonPlan/LessonPlanFileLibrary.jsx` (line ~75-93): `fromApi.getLessonPlanFileLibrary({ fileTypes, masterProgrammeId, masterLevelId, subjectId, page, pageSize })` → `GET /lesson-plans/file-library` (`fromApi.js` line ~3286-3293), store `lessonPlanMaterialFiles`.
  - `bbs/client-teacher/src/views/lessonPlan/LessonPlanFileLibraryForm.jsx`: file library diisi via `uploadLessonPlanFiles(lessonPlanId, formData)` — jadi file library memang berasal dari lesson plan material files, bukan subject files.
- Backend terkait:
  - `api_nest/src/modules/lesson-plan/lesson-plan.controller.ts` (line ~76-98): `GET /lesson-plans/file-library`, `GET /lesson-plans/file-library/:id`, `DELETE /lesson-plans/file-library/:id` → `LessonPlanFileLibraryService`.
  - `api_nest/src/modules/subject-file/subject-file.controller.ts` (line ~18): `GET /subjectFile` → `SubjectFileService` — entitas `SubjectFile` (relasi ke Subject/MasterLevel/Employee), berbeda domain dari file library.

Root cause: **mismatch endpoint/store** — HOB picker di-wire ke `subjectFiles` sedangkan "file library" yang diharapkan user adalah `lesson-plans/file-library` (`lessonPlanMaterialFiles`).

---

## Impact

- Teacher tidak bisa memilih file HOB dari file library → fitur HOB Resources praktis tidak berfungsi sesuai harapan.
- Data `materialResources` lesson plan bisa terisi refs yang bukan dari file library (subject file), menimbulkan kebingungan saat review.
- Inkonsistensi antara halaman File Library dan picker di form.

---

## Saran Solusi

1. **Ganti sumber data modal HOB** di `LessonPlanForm.jsx` (line ~217-225): gunakan `fromApi.getLessonPlanFileLibrary({ ... })` + `useResourceMapper("lessonPlanMaterialFiles", ...)`, mengikuti pola `LessonPlanFileLibrary.jsx`. Tambahkan filter yang relevan (subject/level/programme sesuai lesson plan) dan search `fileName` (cek dukungan query param di `GetLessonPlanFileLibraryDto`).
2. Tambahkan **pagination** di modal (saat ini hardcoded `pageSize: 50` tanpa paging).
3. Pastikan label checkbox memakai field yang konsisten (`fileName`) dan `hblRefs` yang tersimpan dapat dipetakan kembali ke file library (pertimbangkan simpan file id, bukan hanya nama file, agar referensi tidak ambigu).
4. Regression check: form edit yang sudah tersimpan dengan `hbl: <fileName>` lama tetap ter-parse dengan benar (`parseMaterialResources` di `lessonPlanUtils.js`).

---

## Acceptance Criteria

- [ ] Modal Browse HOB Resources menampilkan file-file yang sama dengan halaman Lesson Plan File Library.
- [ ] Search di modal HOB bekerja terhadap file library.
- [ ] File yang dipilih tersimpan sebagai HOB refs pada lesson plan dan tampil benar saat form dibuka ulang (create & edit).
- [ ] Modal tidak lagi memanggil `GET /subjectFile`.
- [ ] Tidak ada regresi pada upload PPT/PDF/Video dan daftar "Uploaded files" di form lesson plan.
