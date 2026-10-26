---
feature: GPA Report — Subject dari Tahun Lain Muncul di Kolom Tahun Berjalan
slug: gpa-report-stale-subject-from-other-year
status: draft
author: OpenClaude
date: 2026-10-26
target_release: TBD
---

# GPA Report — Subject dari Tahun Lain Muncul di Kolom Tahun Berjalan

## Overview

Report **GPA Transcript** (secondary, `secondary-student-gpa-report.helper.ts`) menampilkan
daftar subject per kolom tahun ajaran. Daftar subject tersebut **tidak** diambil dari
`student_grade`, melainkan dari relasi `student.bandings` yang difilter per
`banding.subjectYear.classYearId === classYear.id` (baris 111-114). Riwayat tahun yang
dirender diambil dari `student.classYears.slice(0, 4)` (baris 73-74) — hanya 4 tahun
terakhir, **tanpa `orderBy`** pada relasinya.

Akibatnya, untuk siswa dengan riwayat panjang (mis. Aaronne Malachai Dizon Miclat, student
id **21416**, 7 tahun ajaran 2020/2021 s/d 2025/2026), payload report memuat subject dari
tahun **2023/2024** (antara lain `Bahasa Indonesia (Level 4)`, subject id **724**) ke dalam
rentang kolom report 25/26. Subject tersebut **bukan** milik AY 2025/2026 dan muncul di
report GPA 25/26.

**Kesimpulan investigasi: data migrasi tidak salah.** Pada AY 2025/2026, Aaronne memakai
subject `Bahasa Indonesia` biasa (id **623**), dan banding-nya memang tercatat pada
class_year 100392 (JC2 Mendel). Yang salah adalah **logic report** yang merender lebih dari
satu tahun dan mencampur baris subject antar tahun.

## Problem / Motivation

Pengguna (siswa/orang tua/kurikulum) melihat di GPA Transcript 25/26 ada baris
`Bahasa Indonesia (Level 4)` yang seharusnya baru muncul di tahun Secondary 4 (2023/2024).
User mengira data migrasi rusak. Investigasi DB membuktikan sebaliknya: `student_grade`
dan `banding` untuk 25/26 sudah benar; kesalahannya di rendering report.

Dampak: laporan akademik resmi (transkrip GPA) menampilkan subject yang tidak diikuti siswa
pada tahun berjalan → laporan tidak akurat dan menimbulkan pertanyaan orang tua.

### Akar Masalah (dengan bukti kode)

1. **Riwayat tahun dipotong `slice(0, 4)` tanpa urutan terjamin.**
   `secondary-gpa-report.helper.ts:73-74`:
   ```ts
   const classYearHistory =
     studentReportWithRelations?.student?.classYears?.slice(0, 4);
   ```
   Relasi `student.classYears` (`student.entity.ts:155-158`) dideklarasikan sebagai
   `@ManyToMany(...)` **tanpa `order`/`orderBy`**. Urutan elemen array mengikuti urutan
   yang dikembalikan join table (`student_class_years`) — tidak dijamin kronologis. Jadi
   `.slice(0, 4)` tidak deterministik: bisa memotong tahun terbaru, atau memasukkan tahun
   lama ke dalam 4-tahun yang dirender.

2. **Daftar subject per kolom berasal dari `bandings`, bukan filter tahun eksplisit.**
   `secondary-gpa-report.helper.ts:111-114`:
   ```ts
   const currentSubjectYearBandings =
     studentReportWithRelations?.student?.bandings.filter(
       (banding) => banding?.subjectYear?.classYearId === classYear.id,
     );
   ```
   Per kolom ini benar (hanya banding dengan classYearId cocok), tetapi **kolom mana saja
   yang dirender** ditentukan `classYearHistory` di poin 1. Jika tahun 2023/2024 ikut
   terpilih sebagai salah satu dari 4 tahun, subject 724 akan muncul sebagai baris di
   kolom 2023/2024.

3. **Semua tahun digabung dalam grid 2 kolom tanpa pemisah tegas.**
   `secondary-gpa-report.helper.ts:347-387` menyusun `tables` dengan pola
   `currentTableIdx = classYearIdx % 2 == 0 ? 0 : 1` — tahun index genap ke kolom kiri,
   index ganjil ke kolom kanan, lalu `tables` dirender di template
   `secondary-student-gpa-report.hbs` sebagai satu `<td>` per array. Karena 4 tahun
   dirender bersambung dalam 2 kolom, baris tahun 2023/2024 tampak menyatu dengan blok
   2025/2026 dan mudah disalahartikan sebagai milik 25/26.

### Bukti Data (DB produksi `binabangsa_prod_mig_v01`)

Banding Bahasa Indonesia Aaronne (student 21416), semua tahun:

| Tahun | Classroom | class_year_id | subject_id | subject | banding_id |
|---|---|---|---|---|---|
| 2020/2021 | Secondary 1 Pascal | 101417 | 41 | Bahasa Indonesia | 103785 |
| 2021/2022 | Secondary 2 Pascal | 101496 | 112 | Bahasa Indonesia | 104967 |
| 2021/2022 | Secondary 2 Pascal | 101496 | 653 | Bahasa Indonesia (Level 2) | 109813 |
| 2022/2023 | Secondary 3 Pascal | 101551 | 625 | Bahasa Indonesia (Level 3) | 109703 |
| **2023/2024** | **Secondary 4 Pascal** | **1857** | **724** | **Bahasa Indonesia (Level 4)** | **2705** |
| 2024/2025 | Junior College 1 Mendel | 2009 | 621 | Bahasa Indonesia | 32667 |
| **2025/2026** | **Junior College 2 Mendel** | **100392** | **623** | **Bahasa Indonesia** | **109634** |

- Banding 25/26 memakai subject **623** (`Bahasa Indonesia`), **bukan** 724. Tidak ada
  satu pun banding 25/26 memakai subject 724.
- Subject 724 hanya punya satu banding, dan class_year-nya **benar** (2023/2024, Secondary
  4 Pascal, master_level 14).
- `student_grade` Aaronne AY 2025/2026 (12 baris, class_year 100392) juga memakai subject
  623 untuk Bahasa Indonesia. Tidak ada penambahan subject 724 di 25/26.

## Scope

### In Scope
- Dokumentasi root cause rendering report GPA (2–4 tahun terakhir, order tidak dijamin,
  grid 2 kolom).
- Spesifikasi perbaikan: filter tahun & urutan deterministik pada `classYearHistory`,
  dan tanda batas visual antar tahun di template.
- Acceptance criteria yang bisa diverifikasi dengan student 21416 (Aaronne) dan siswa
  riwayat panjang lain.

### Out of Scope
- Perubahan data migrasi (`student_grade`, `banding`, `subject_year`). Data sudah benar.
- Perubahan perhitungan GPA/PUM/FYA (sudah dibahas di `gpa-fye-pum-mismatch/`).
- Redesign tampilan transkrip secara keseluruhan.
- Perubahan kode di `api_nest/` atau `bbs/` — brief ini hanya dokumen spesifikasi.

## User Stories

### As a Siswa / Orang Tua
I want GPA Transcript hanya menampilkan subject yang benar-benar diikuti pada tiap tahun
di kolomnya masing-masing
So that transkrip tidak menyesatkan dan tidak menimbulkan pertanyaan soal subject asing

### As a Wakil Kurikulum / Admin
I want report GPA menandai batas tahun dengan jelas dan memuat hanya N tahun terakhir yang
deterministik
So that review transkrip konsisten lintas siswa dan bisa dipertanggungjawabkan

## Acceptance Criteria

- [ ] `classYearHistory` diurutkan **eksplisit** (mis. `ORDER BY academic_year.year DESC`
      lalu di-`slice(0, N)`), tidak bergantung urutan join table.
- [ ] Untuk student 21416 dengan parameter AY 2025/2026, report **tidak** memuat baris
      `Bahasa Indonesia (Level 4)` (subject 724) di kolom mana pun kecuali kolom tahun
      2023/2024 — dan bila 2023/2024 dirender, harus ada label tahun yang terlihat jelas
      di atas blok itu.
- [ ] Setiap kolom/tahun pada PDF memiliki header tahun + nama kelas yang jelas, sehingga
      baris subject tidak bisa disalahartikan milik tahun lain.
- [ ] Jumlah tahun yang dirender terdokumentasi (mis. tetap 4 tahun terakhir) dan
      konsisten untuk semua siswa, termasuk yang riwayatnya < 4 tahun.
- [ ] Regression: siswa tanpa riwayat `classYears` (baru) atau dengan banding kosong per
      tahun tidak menyebabkan 500 / report kosong.
- [ ] Verifikasi query: `banding` student 21416 AY 2025/2026 hanya berisi subject
      623/380/1347/383/... (bukan 724) — lampirkan hasil query di PR.

## UI / UX Changes

- Template `secondary-student-gpa-report.hbs`: header tahun per blok menjadi **tegas**
  (bold/border/shading), sehingga batas antar tahun terlihat walau dua kolom berdampingan.
- (Opsional) Jika report dimaksudkan khusus AY tertentu, tampilkan **hanya** kolom AY itu.

### Affected Portals
- [x] Admin (client/) — generate report via student report
- [x] Student (client-student/) — siswa melihat GPA Transcript
- [ ] Teacher (client-teacher/)

## API Changes

Tidak ada endpoint baru. Endpoint yang men-generate report (melalui
`createStudentReport` → `createSecondaryStudentGpaReport`) berubah output HTML-nya.

| Method | Path | Description |
|--------|------|-------------|
| POST/GET | (existing report-generation route) | Output HTML GPA Transcript: filter+urutan tahun & header kolom berubah |

## Database Changes

Tidak ada perubahan skema. Perbaikan berbasis query/logic + template.

### New Tables
- Tidak ada.

### Modified Tables
- Tidak ada.

### Migrations
- Tidak ada.

## Business Rules / Validation

1. **Sumber tahun yang dirender = `student.classYears` yang sudah diurutkan.** Urutkan
   berdasarkan `academicYear.year` (atau `academicYearId`) **DESC** sebelum `slice`, agar
   "N tahun terakhir" deterministik.
2. **Sumber subject per kolom tetap `bandings`** yang difilter
   `subjectYear.classYearId === classYear.id` (perilaku ini sudah benar; jangan diganti ke
   `student_grade` tanpa keputusan terpisah).
3. **Jangan campur tahun dalam satu blok visual.** Setiap tahun wajib punya label header.
4. **Data grade/banding bukan penyebab bug.** Jangan melakukan "fix" dengan menghapus
   subject 724 dari `banding` Aaronne — itu akan merusak record 2023/2024 yang sah.
5. **Nilai default N = 4** (sesuai perilaku existing) kecuali product memutuskan lain.

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| `classYears` kosong / null | 200 | Report dirender tanpa blok tahun (kosong), tidak 500 |
| `gradingScale` tidak ditemukan | 200 | Fallback ke grading scale global (`effectiveFromAcademicYearId IS NULL`) — perilaku existing |
| `banding.subjectYear` null (relasi hilang) | 200 | Skip banding tersebut, jangan crash filter |

## Dependencies

- `api_nest/src/helpers/reports/secondary-gpa-report.helper.ts` — `classYearHistory`
  (baris 73-74), filter banding (baris 111-114), grid `tables` (baris 347-387)
- `api_nest/src/helpers/reports/separating-report-relations.ts` — loader relasi report
  (tempat menambahkan `order` bila memilih opsi order di query)
- `api_nest/src/modules/student/entities/student.entity.ts` — relasi `classYears`
  (baris 155-158, saat ini tanpa `order`)
- `api_nest/src/templates/student-reports/secondary-student-gpa-report.hbs` — rendering
  `tables` (header tahun, penanda batas)
- Tabel DB: `student.classYears` (`student_class_years`), `class_year`, `academic_year`,
  `banding`, `banding_students_student`, `subject`, `subject_year`
- Referensi terkait: `features/gpa-fye-pum-mismatch/` (nilai FYA/PUM — beda isu),
  `features/gpa-report-jc2-alevel/`
