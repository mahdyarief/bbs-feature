---
feature: GPA Report FYE/PUM Mismatch — FYE Kosong Saat Nilai Telah Diinput
slug: gpa-fye-pum-mismatch
status: draft
author: OpenClaude
date: 2026-10-16
target_release: TBD
---

# GPA Report FYE/PUM Mismatch — FYE Kosong Saat Nilai Telah Diinput

## Overview

GPA report secondary menampilkan nilai `0` / kosong untuk mata pelajaran akademik meski guru sudah menginput FYE di gradebook. Root cause: GPA report **tidak membaca FYE secara langsung** — report membaca `FYA` (nilai jadi full-year, `GradeExamTypeEnum=12`) untuk subjek non-Cambridge, dan `predictedPum`/`asPredictedPum` untuk subjek Cambridge. FYE (`GradeExamTypeEnum=4`) hanya berperan sebagai bahan baku perhitungan FYA di `recalculate()`. Jika syarat perhitungan FYA tidak terpenuhi, row FYA dihapus dan GPA report tidak menemukan nilai apa-apa.

## Problem / Motivation

Guru menginput nilai FYE di form gradebook sekunder, tetapi di GPA report nilai tersebut muncul sebagai `0` / kosong. Hal ini menyebabkan laporan GPA tidak akurat dan memaksa admin/kurikulum mengecek secara manual apakah FYA telah tergenerate.

### Masalah yang Terjadi

1. **GPA report baca FYA, bukan FYE** (`secondary-gpa-report.helper.ts:193-198`). FYE mentah yang diinput guru tidak pernah langsung ditampilkan di GPA.
2. **FYA hanya terhitung jika semua syarat terpenuhi** (`student-secondary-grade.service.ts:318-343`):
   - `ca1 != null && ca2 != null`
   - `myeReady` dan `fyeReady` (MYE/FYE yang bobotnya >0 sudah ada atau di-ignore -1)
   - `activeWeight > 0`
3. **Jika FYA null, row FYA dihapus** (`upsertCalculatedGrade` di baris 424-430: `score == null` → `grade.remove()`). Row FYA hilang total, GPA report tidak menemukan apa-apa.
4. **Subjek Cambridge menggunakan PUM, bukan FYE/FYA** (`secondary-gpa-report.helper.ts:174-191`). Report baca `predictedPum` (term != 2) atau `asPredictedPum` (term == 2) dari `student_grade`, bukan dari tabel `grade` dengan `examType=FYE`.
5. **Bug `null !== undefined` di `resolvePumSource`** (`student-secondary-grade.service.ts:308-315`). Ketika `predictedPum` di DB = NULL, `pumSource.fye` bernilai `NULL` (bukan `undefined`). Karena `null !== undefined` adalah `true`, blok `upsertCalculatedGrade(FYE, ..., pumSource.fye)` tetap dieksekusi dengan `score = null`, yang pada baris 463-465 akan **menghapus** row FYE yang sudah ada. Ini berulang setiap kali `recalculate()` dijalankan untuk subjek Cambridge.

## Scope

### In Scope
- Dokumentasi alur data FYE → FYA → GPA report
- Identifikasi kondisi yang menyebabkan FYE tidak masuk hitungan FYA
- Reproduksi kasus Aarone (student id 21416, JC2 Mendel AY 2025/2026) di DB produksi
- Acceptance criteria untuk verifikasi oleh engineer

### Out of Scope
- Perubahan kode di `api_nest/` atau `bbs/` (brief ini hanya dokumen spesifikasi)
- Perubahan logika `recalculate()` atau GPA report itu sendiri (hanya dokumentasi root cause)
- Migrasi data atau seeder bobot weight

## User Stories

### As a Guru / Wakil Kurikulum
I want GPA report menampilkan nilai yang benar setelah saya input FYE
So that laporan GPA mencerminkan capaian siswa secara akurat

### As a Admin
I want bisa memverifikasi apakah FYA sudah tergenerate setelah input FYE
So that dapat membedakan apakah masalahnya di input nilai atau di perhitungan FYA

## Acceptance Criteria

- [ ] GPA report menampilkan `finalMark` dari `FYA` (non-Cambridge) atau `predictedPum`/`asPredictedPum` (Cambridge), bukan dari `FYE` mentah
- [ ] Ketika `FYE = -1` (ignored) dan bobot `fye > 0`, FYE tidak aktif dalam perhitungan FYA (`fyeActive = false`)
- [ ] Ketika `ca1` atau `ca2` belum diisi, `FYA = null` dan row FYA dihapus → GPA menampilkan `0`
- [ ] Verifikasi di DB: `grade` tabel dengan `exam_type = 4` (FYE) untuk student id 21416, AY 2025/2026, class_year 100392 — semua nilai = -1.0 (ignored)
- [ ] Verifikasi di DB: `student_grade.final_grade` (FYA) untuk Chemistry = 54.70, Computer Science = 83.57, English GP = 88.47, Mathematics = 59.25, Physics = 63.18 — hanya subjek non-Cambridge yang punya final_grade

## UI / UX Changes

Tidak ada perubahan UI (bug fixing dokumentasi saja).

### Affected Portals
- [ ] Admin (client/)
- [ ] Student (client-student/)
- [ ] Teacher (client-teacher/)

## API Changes

Tidak ada perubahan endpoint.

## Database Changes

Tidak ada perubahan skema.

## Business Rules / Validation

1. **FYE bukan nilai yang ditampilkan di GPA.** GPA menampilkan FYA (non-Cambridge) atau PUM (Cambridge).
2. **FYA dihitung hanya jika:** ca1 dan ca2 terisi, dan MYE/FYE (bobot >0) sudah siap atau di-ignore (-1).
3. **FYE = -1 berarti "diabaikan"** — tidak ikut serta dalam perhitungan weighted average FYA, tapi tetap dianggap "ready" untuk syarat FYA.
4. **Bobot `fye` harus > 0** agar FYE dianggap diperlukan dalam perhitungan FYA. Jika bobot fye = 0, FYE tidak perlu diisi dan tidak mempengaruhi FYA.
5. **Jika FYA = null, row FYA dihapus dari database** — GPA report tidak akan menemukan nilai dan menampilkan `0`.

## Error Handling

Tidak ada error baru yang diperkenalkan. Perilaku existing: GPA report menampilkan `finalMark='0'` dan tidak masuk `gpaNumerator/gpaDenominator` ketika `subjectAssessmentGrade` tidak ditemukan atau `score < 0`.

## Dependencies

- `api_nest/src/helpers/reports/secondary-gpa-report.helper.ts` — GPA report logic (baris 171-271)
- `api_nest/src/modules/student-secondary-grade/student-secondary-grade.service.ts` — `recalculate()` dan `upsertCalculatedGrade()` (baris 280-343, 424-430)
- `api_nest/src/types/enums/grade-exam-type.ts` — `GradeExamTypeEnum` (FYE=4, FYA=12)
- `api_nest/src/types/enums/cambridge-programme-type.ts` — `CambridgeProgrammeTypeEnum` (IGCSE=1, ASLEVEL=2, ALEVEL=3, NONE=4)
- `api_nest/src/modules/grade-weight/entities/grade-weight.entity.ts` — bobot `fye` per master_level/subject
- Tabel DB: `grade`, `student_grade`, `secondary_grade_weight`, `subject`, `class_year`, `academic_year`
