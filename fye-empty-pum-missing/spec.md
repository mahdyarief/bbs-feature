---
feature: FYE Kosong Saat PUM Belum Di-Migrasi — CS JC2 Mendel 2025/2026
slug: fye-empty-pum-missing
status: draft
author: OpenClaude
date: 2026-10-19
target_release: TBD
---

# FYE Kosong Saat PUM Belum Di-Migrasi — CS JC2 Mendel 2025/2026
**Context:** JC2 Mendel (class_year 100392), Computer Science (subject 383), Cambridge A Level (cambridge_programme=3), AY 2025/2026.

## Overview

Master data: `class_year_id=100392` (JC2 Mendel, 2025/2026), `subject_year_id=109372` = Computer Science (9618), `cambridge_programme=3` (ALEVEL). Gradeboard term 4 menampilkan kolom FYE (bobot `fye=35` di `secondary_grade_weight`, master_level 17). Nilai FYE **ada di tabel `grade`** (11 row, exam_type=4, term=4, score terisi). Namun **tidak tampil di frontend** karena frontend untuk A Level mengambil FYE dari `predicted_pum` di `student_grade`, yang **semua NULL**.

## Problem / Motivation

Guru input nilai FYE di form gradebook sekunder. Nilai itu tersimpan sebagai `Grade` (`exam_type=4`, `term=4`) dengan `score` terisi. Tapi di GradebookTeacher, sel FYE tetap kosong. Root cause: A Level FYE di-source dari `predicted_pum` (teacher composite prediction via PUM migration), bukan dari nilai `Grade` mentah. Saat ini `predicted_pum` masih kosong untuk semua siswa, dan belum di-migrasi dari nilai PUM AIS `enrollprelims` (`aspum`/`apum`). Jadi sel FYE kosong.

### Masalah yang Terjadi
1. Semua 11 siswa JC2 Mendel AY26 (CS 109372) punya `student_grade.predicted_pum = NULL`.
2. `grade` table punya 11 row FYE (term 4) dengan score terisi: 95, 92, 49, 78, 95, 11, 75, 95, 57. Dua siswa (21219, 22642) punya `score = -1` (ignored, artinya tidak dihitung) — selebihnya nilai valid.
3. Frontend `SecondaryGradebookFlexibleTable.jsx` untuk A Level: `fyeFromPum = true` → `fyeScore = resolveFyePum(ssg) = ssg.predictedPum ?? null` → `fyeScore == null` → `fyeActive = false` → sel FYE tidak di-render (return null).
4. Nilai FYE manual di `grade` table tidak dipakai sebagai fallback.

## Scope

### In Scope
- Dokumentasi kondisi di mana FYE tidak tampil meskipun nilai ada di `grade` table.
- Identifikasi bahwa tipe data `grade_exam_type_enum` adalah integer (1-10, sesuai `GradeExamTypeEnum`: TT=1, BMT=2, MYE=3, FYE=4, ...).
- Data referensi proporsi nilai FYE yang ada vs yang kosong.

### Out of Scope
- Perubahan kode frontend (`SecondaryGradebookFlexibleTable.jsx`) — untuk diputuskan oleh product (lihat task ini).
- Perubahan API/Nest (`api_nest/src/**`).
- Perubahan skema database.
- Perubahan logika `recalculate()`.

## User Stories

### As a Guru
I want FYE dari nilai yang saya input untuk tampil di GradebookTeacher
So that tidak ada perbedaan antara nilai di form dan di table.

### As a Wakil Kurikulum
I want nilai FYE tersedia di GradebookTeacher tanpa menunggu migrasi PUM
So that laporan/gradeboard mengandalkan data yang sudah ada, bukan menunggu PUM selesai di-migrasi.

## Acceptance Criteria

- [ ] GradebookTeacher term 4 menampilkan nilai FYE untuk CS JC2 Mendel AY26 (9 siswa dengan nilai valid), sesuai `grade` table `score` — bukan sel kosong.
- [ ] Saat `predicted_pum` masih NULL, FYE fallback menampilkan nilai dari `grade` (exam_type=4, term=4), sesuai keputusan (B) EC-01.
- [ ] Saat `predicted_pum` terisi (setelah migrasi PUM), FYE otomatis bergeser menampilkan nilai PUM tanpa perubahan kode lagi, sesuai keputusan EC-04.
- [ ] Catatan untuk engineer: `exam_type` di `grade` table adalah enum integer (FYE=4), bukan string — query harus pakai literal angka atau cast.

## UI / UX Changes

### Affected Portals
- [ ] Teacher (client-teacher / `src/components/SecondaryGradebookFlexibleTable.jsx`)
- [ ] Admin (client) — hanya jika masih ada useGradebookTeacher
- [ ] Student — bukan path, gradeboard terpisah

### Change Description
Ganti logika `fyeFromPum` dari A Level: selain ambil `predictedPum`, tambahkan fallback: jika `predictedPum == null` dan ada `Grade` dengan `exam_type=4, term=4` dan `score != null`, tampilkan `score` tersebut. Jangan lupa catatan pada `notes.md`/`page.md` bahwa FYE sebenarnya di-input manual, bukan dari PUM — sehingga saat PUM selesai di-migrasi, sel FYE otomatis bergeser ke nilai PUM (konsisten dengan kebutuhan migrasi).

### Affected Data
- `grade` table (exam_type=4, term=4)
- `student_grade` table (`predicted_pum` belum diisi)

## API Changes
Tidak ada endpoint baru. Endpoint yang dipakai tetap `client-teacher/src/components/SecondaryGradebookFlexibleTable.jsx`.

## Database Changes
Tidak ada skema baru. Kolom yang relevan:
- `grade.exam_type` = integer (enum `grade_exam_type_enum` → 1..10)
- `student_grade.predicted_pum` = integer (belum diisi)
- `grade.score` = double precision (VALUE 0 valid, bukan NULL)
- `grade.student_grade_id` = FK ke `student_grade.id`

## Business Rules / Validation

1. **FYE adalah nilai manual** di gradebook sekunder, bukan PUM. `grade` table adalah sumber keandalan FYE.
2. **PUM (predicted_pum) adalah sumber nilai FYE untuk Cambridge A Level** sesuai konfigurasi gradeboard (bobot `fye=35`).
3. **Nilai 0 di `grade.score` adalah valid** — tidak boleh dirubah jadi NULL, dan tidak boleh diabaikan sebagai "tidak ada nilai". Hanya `score IS NULL` yang tidak tampil.
4. Saat PUM selesai di-migrasi dari AIS (`enrollprelims.aspum` / `apum`), `predicted_pum` akan diisi, dan sel FYE otomatis menampilkan nilai PUM (logika gradeboard tetap pertama prioritas: PUM > nilai manual).

## Error Handling
Tidak ada error baru. Pengecekan yang sudah ada: `fyeScore == null || isNaN(fyeScore)` → `fyeActive = false` → sel FYE disembunyikan.

## Dependencies
- `api_nest/src/modules/student-secondary-grade/student-secondary-grade.service.ts` — `recalculate()` (FYE/FY A perhitungan optional)
- `api_nest/src/types/enums/grade-exam-type.ts` — `GradeExamTypeEnum` (FYE = 4)
- `api_nest/src/types/enums/cambridge-programme-type.ts` — `CambridgeProgrammeTypeEnum` (ALEVEL = 3)
- `api_nest/src/modules/student-secondary-grade/entities/student-grade.entity.ts` — `student_grade` table (`predicted_pum`)
- `api_nest/src/modules/grade-weight/entities/secondary-grade-weight.entity.ts` — bobot `fye`=35 untuk master_level 17

## Notes
- **Type enum**: `grade_exam_type_enum` ternyata adalah `integer` (bukan string). Saat query `grade` table, `exam_type` harus dibandingkan dengan angka literal, atau cast `exam_type::text='FYE'`.
- **Nilai 0**: `score=0` tetap terhitung sebagai nilai valid. Tiga row `score=0` milik 3 siswa yang sama (21219, 21233, 22642).
- **Syarat FYE aktif**: `gradeWeight.fye > 0` (35), `fyeScore != null`. Saat `predictedPum` terisi, `fyeActive` berubah ke PUM.
- **Cara verifikasi di DB**: `SELECT g.scored FROM grade g JOIN student_grade sg ON sg.id=g.student_grade_id WHERE sg.subject_year_id=109372 AND g.exam_type=4 AND g.term=4 ORDER BY sg.student_id;` — return 11 row.
