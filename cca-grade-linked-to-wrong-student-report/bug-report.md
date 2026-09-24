---
title: Progress Report — Nilai CCA Tidak Muncul karena `student_cca_grade` Menunjuk ke `student_report` yang Salah
status: open
severity: major
product: BBS LMS
portal: Teacher | Admin
author: System Analyst
date: 2026-09-24
jam: (tidak tersedia — laporan berbasis investigasi DB + code review)
---

# Progress Report — Nilai CCA (Basketball) Sudah Diinput tapi Tidak Muncul di Report

## Summary

Pada student **100683 (Nathanael Jose Soelistyo)**, `class_year` **100894**
(AY 2026/2027, **Primary 6 Compassion**), nilai CCA **Basketball** sudah diinput lewat
`/ccaYear/100300/27` (term 1, `attendance` 10, `active_participation` 10,
`final_score` 20, **grade A**), **namun tidak muncul di report**.

Akar masalahnya bukan pada input nilainya (baris `student_cca_grade` id 12943 memang ada
dan valid), melainkan pada **`student_report_id` yang salah**: baris nilai itu menunjuk ke
`student_report` **15606** — sebuah student_report **"dummy"** dengan `class_year_id = 0`
(AY24, "Classroom 001") — bukan ke student_report AY27 yang benar (**4406**).

Report builder (`helpers/reports/progress-report.helper.ts` dan `semester-report.helper.ts`)
mengambil nilai CCA dengan filter **dua kondisi sekaligus**:

```typescript
// helpers/reports/progress-report.helper.ts — pengambilan CCA untuk report
const studentCcaGrades = await StudentCcaGrade.find({
  where: {
    ccaYearId: In(studentCcaYear.map((cy) => cy.id)),   // pivot cca_year milik student
    studentReport: { id: studentReport.id },            // HARUS sama persis dengan report ini
  },
});
```

Karena `student_report_id` baris nilai menunjuk ke student_report dummy (bukan 4406), nilai
CCA tidak pernah ketemu saat report AY27 di-generate. **Regenerate report saja tidak
mengubah apa pun** selama `student_report_id`-nya masih salah.

**Ekspektasi:** report student 100683 term 1 menampilkan CCA **Basketball** dengan grade A.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | — (laporan dari user) |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | — |
| Portal URL | `students/100683` + `/ccaYear/100300/27` (Teacher/Admin portal, AY 2026/2027 term 1) |
| Environment API | api.binabangsaschool.dev (production) |
| Data konteks (class ID / daId / tanggal) | student_id=100683; class_year=100894; academic_year=27 (2026/2027); term=1; cca_year=100300 (Basketball, campus 3); student_report AY27=4406; student_report dummy=15606; student_cca_grade id=12943; report REPORT_TERM=52626 |
| Database diperiksa | `binabangsa_prod_mig_v01` via `D:\Work\BBS\requirement\binabangsa-db-tools` |
| Browser / OS | — |

---

## Steps to Reproduce

1. Input nilai CCA **Basketball** untuk student **100683** di `/ccaYear/100300/27` (term 1).
2. Buka report (progress report/REPORT_TERM) student 100683 term 1 AY 2026/2027.
3. Perhatikan: section CCA **tidak muncul** (bahkan tidak ada kata "CCA"/"Basketball" di HTML).
4. Cek DB: baris `student_cca_grade` id 12943 ada, tapi `student_report_id = 15606`
   (student_report dummy `class_year_id = 0`), bukan 4406 (AY27).

**Actual Result:**
- Nilai CCA tersimpan, tetapi `student_report_id` menunjuk ke student_report dummy → nilai
  tidak muncul di report AY27.
- Report 52626 (sr=4406, term 1) tidak memuat "Basketball"/"CCA" sama sekali.

**Expected Result:**
- Report AY27 term 1 menampilkan CCA Basketball dengan grade A.
- Nilai CCA harus tertaut ke student_report **tahun ajaran yang sebenarnya** (AY27 → sr 4406).

---

## Root Cause Analysis

### Bug #1 (pemicu) — frontend mengirim `classYearId = "0"`

`client-teacher/src/views/cca/CCADetail.jsx` (sekitar baris 304) membentuk payload submit
dengan:

```javascript
{
  ccaYearId: ccaYear?.id?.toString(),
  classYearId: data?.targetClassYearId ?? "0",   // <-- default "0" bila targetClassYearId kosong
  ...
}
```

`targetClassYearId` hanya di-set saat field **baru** ditambahkan (dari `student.currentClassYearId`).
Untuk baris yang **dimuat dari nilai yang sudah ada** (useEffect yang membaca
`studentCcaGrades`), `targetClassYearId` **tidak pernah di-set** — sehingga default `"0"`
terkirim ke API. `Number("0") === 0`.

### Bug #2 (kerusakan data) — backend memercayai `classYearId` mentah & membuat student_report dummy

`student-cca-grade.service.ts` method `create()`:

```typescript
const classYear = await ClassYear.findOne({ where: { id: Number(options.classYearId) } });
if (!classYear) NoClassYearFoundError();

const availabilityStudentReport = await StudentReport.findOne({
  where: { student: { id: options.studentId }, classYear: { id: classYear.id } },
  relations: { student: true, classYear: true },
});

let studentReport = availabilityStudentReport;
if (!studentReport) {
  // BUAT student_report baru dengan classYear tsb
  const createNewReport = new StudentReport();
  createNewReport.student = student;
  createNewReport.classYear = classYear;   // class_year id=0 (placeholder!)
  await createNewReport.save();
  studentReport = createNewReport;
}
```

Kunci masalahnya: **`class_year` id `0` ternyata ADA di database** — baris placeholder
(AY24, `classroom` "Classroom 001", level 3). Jadi `ClassYear.findOne({ id: 0 })` tidak
gagal; service lalu membuat `student_report` baru dengan `class_year_id = 0` dan menempelkan
nilai CCA ke sana.

### Bug #3 (kenapa tidak terlihat) — report builder butuh kecocokan `studentReport.id` persis

`progress-report.helper.ts` dan `semester-report.helper.ts` menyaring nilai CCA dengan
`studentReport: { id: studentReport.id }`. Karena baris nilai tertaut ke student_report dummy
(bukan report AY yang sedang di-generate), nilai CCA tidak pernah disertakan. Ini juga
menjelaskan kenapa "regenerate report" berulang kali tidak menolong.

**Bukti:** report 52626 (sr=4406, REPORT_TERM term 1) sebelum perbaikan **tidak memuat**
kata "CCA" maupun "Basketball"; setelah `student_report_id` di-repoint ke 4406 dan report
di-regenerate, keduanya muncul (`True`) dan panjang HTML naik 7.604 → 7.882 char.

---

## Bukti dari Database

Diambil dari `binabangsa_prod_mig_v01`:

| Metrik | Nilai |
|--------|-------|
| Student | 100683 — Nathanael Jose Soelistyo; `current_class_year_id` = 100894 (AY27) |
| `cca_year` 100300 | Basketball, AY27, campus 3, aktif |
| Pivot `cca_year_students_student` | student 100683 **terdaftar** di cca_year 100300 |
| `student_cca_grade` id 12943 | cca_year 100300, term 1, grade A, final_score 20, **`student_report_id` = 15606** |
| `student_report` 15606 (salah) | student 100683, **`class_year_id` = 0**, AY24, "Classroom 001", dibuat 2026-09-22 |
| `student_report` 4406 (benar) | student 100683, class_year 100894, AY27, "Primary 6 Compassion" |
| `class_year` id 0 | ADA (placeholder: AY24, classroom 0 "Classroom 001", level 3) — inilah kenapa `findOne(id=0)` lolos |
| `student_report` dengan `class_year_id = 0` | **3 baris** (semuanya dummy, satu per student terdampak) |
| Skala mismatch (`student_report` AY ≠ `cca_year` AY) | **3 baris** |
| Report terdampak (kasus 100683) | 52626 (REPORT_TERM, type 1, term 1) |
| Panjang HTML report | **7.604 → 7.882** char setelah CCA muncul |

**Rincian 3 baris mismatch:**

| `student_cca_grade` id | Student | `cca_year` (AY) | term | `student_report` sekarang (dummy) | `student_report` benar |
|---|---|---|---|---|---|
| 12943 | 100683 Nathanael Jose Soelistyo | 100300 (27) | 1 | 15606 (class_year 0) | **4406** (Primary 6 Compassion) |
| 11089 | 103152 Donghwi Kang | 100190 (26) | 4 | 3824 (class_year 0) | **3813** (Primary 4 Love) |
| 11090 | 103153 Donghyeon Kang | 100190 (26) | 4 | 3825 (class_year 0) | **3814** (Primary 4 Love) |

**Hasil eksekusi perbaikan (student 100683, AY27):**
- Repoint `student_cca_grade` id 12943: `student_report_id` 15606 → **4406** (backup:
  `result/cca_grade_link_all_s100683_backup_20260924_171732.csv`), sisa mismatch = 0.
- Regenerate report 52626 (REPORT_TERM term 1) → HTTP OK.
- Verifikasi: `has_basketball = True`, `has_cca = True`; panjang HTML 7.604 → 7.882 char.

> **Catatan skala:** 2 kasus lain (student 103152 & 103153, AY 2025/2026) juga rusak dengan
> pola identik, namun belum diperbaiki (menunggu keputusan — data historis).

---

## Dampak

- **Nilai CCA yang sudah diinput hilang dari report** (dokumen resmi ke orang tua) — guru
  merasa nilainya "tidak tersimpan", padahal ada; rawan input ulang yang menambah polusi data.
- Report tidak lengkap: section CCA kosong / hilang meski student aktif di CCA tersebut.
- **Sistemik**: pemicunya dari default `classYearId = "0"`, sehingga **setiap penginputan CCA
  untuk baris yang dimuat dari data lama berpotensi** membuat student_report dummy baru.
- **Regenerate report tidak menolong** — akar masalah ada di `student_report_id`, bukan HTML.
- Menambah **`student_report` dummy** (`class_year_id = 0`) yang mengotori data dan bisa
  memengaruhi perhitungan `ccaGrading` antar-term (fungsi `ccaGrading` mencari nilai term
  sebelumnya lewat `studentReport: { id }` yang sama).

---

## Rekomendasi Perbaikan

### Option A: Perbaiki data + perbaiki alur CCA (Recommended)

1. **Perbaikan data (sudah dilakukan untuk AY27):** jalankan
   `scripts/fix_cca_grade_report_link.py` untuk me-repoint `student_report_id` ke
   student_report dengan tahun ajaran yang cocok, lalu regenerate report terdampak.
2. **Perbaikan frontend:** di `CCADetail.jsx`, jangan gunakan default `"0"`. Gunakan
   `classYearId` dari student_report yang benar (mis. `studentCcaGrade.studentReportId`
   → ambil `classYearId`-nya), atau ambil dari `currentClassYearId` student pada AY yang
   relevan. Hindari mengirim `"0"`/kosong.
3. **Perbaikan backend:** di `student-cca-grade.service.ts` `create()`, **tolak** `classYearId`
   yang tidak valid (`0`/placeholder) dan **jangan** membuat `student_report` baru implisit —
   resolusi `student_report` harus berdasarkan (student + academic year dari `cca_year`),
   atau gagalkan dengan error yang jelas alih-alih membuat baris dummy.

### Option B: Pertahanan di report builder

`progress-report.helper.ts` / `semester-report.helper.ts` sebaiknya mencari nilai CCA
berdasarkan **(student, cca_year, term)** — bukan hanya kecocokan `studentReport.id` —
sehingga nilai tetap muncul meski `student_report_id` sempat salah. Bisa dikombinasikan
dengan Option A.

### Option C: Proteksi DB

- Tambahkan constraint/trigger agar `student_report.class_year_id` tidak boleh `0`/placeholder.
- Tambahkan unique index untuk mencegah `student_report` duplikat per (student, class_year).
- Pertimbangkan menghapus baris `class_year` id `0` (placeholder) agar `findOne(id=0)`
  tidak lagi lolos.

---

## Query Verifikasi

```sql
-- nilai CCA student 100683 + student_report yang ditunjuk
SELECT scg.id, scg.cca_year_id, scg.term, scg.grade, scg.final_score,
       scg.student_report_id, sr.class_year_id,
       cy.academic_year_id AS ay_report,
       ccy.academic_year_id AS ay_cca
FROM student_cca_grade scg
JOIN student_report sr ON sr.id = scg.student_report_id
LEFT JOIN class_year cy ON cy.id = sr.class_year_id
JOIN cca_year ccy ON ccy.id = scg.cca_year_id
WHERE sr.student_id = 100683 AND scg.deleted_at IS NULL
ORDER BY scg.id;

-- deteksi global: nilai CCA yang student_report-nya beda academic year dgn cca_year
SELECT scg.id, sr.student_id, sr.class_year_id,
       cy.academic_year_id AS ay_report,
       scg.cca_year_id, ccy.academic_year_id AS ay_cca, scg.term, scg.grade
FROM student_cca_grade scg
JOIN student_report sr ON sr.id = scg.student_report_id
LEFT JOIN class_year cy ON cy.id = sr.class_year_id
JOIN cca_year ccy ON ccy.id = scg.cca_year_id
WHERE scg.deleted_at IS NULL
  AND (cy.academic_year_id IS NULL OR cy.academic_year_id <> ccy.academic_year_id)
ORDER BY sr.student_id;

-- apakah HTML report memuat CCA/Basketball?
SELECT id, report_type::text, term, length(html) AS html_len,
       (html ILIKE '%cca%')     AS has_cca,
       (html ILIKE '%basket%')  AS has_basket
FROM report
WHERE student_report_id = 4406 AND deleted_at IS NULL
ORDER BY term, report_type;
```

Script tersedia di: `D:\Work\BBS\requirement\binabangsa-db-tools\scripts\`

```bash
cd D:\Work\BBS\requirement\binabangsa-db-tools\scripts

python fix_cca_grade_report_link.py                          # dry-run (semua AY)
python fix_cca_grade_report_link.py --student-id 100683      # dry-run (satu student)
python fix_cca_grade_report_link.py --student-id 100683 --execute --yes   # repoint + regenerate
```

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend (teacher) | `client-teacher/src/views/cca/CCADetail.jsx` (~baris 304) | mengirim `classYearId: data?.targetClassYearId ?? "0"` → memicu class_year dummy |
| Backend (CCA grading) | `api_nest/src/modules/student-cca-grade/student-cca-grade.service.ts` | `create()` memercayai `classYearId` & membuat `student_report` dummy saat tidak ada |
| Backend (report builder) | `api_nest/src/helpers/reports/progress-report.helper.ts`, `semester-report.helper.ts` | menyaring CCA dengan `studentReport.id` persis → nilai pada sr salah tidak muncul |
| Backend (grading helper) | `api_nest/src/helpers/grading/cca-grading.ts` | `prevCcaGrades` juga terfilter `studentReport.id` → perhitungan antar-term ikut terdampak |
| Database | `student_cca_grade`, `student_report` (`class_year_id = 0`), `class_year` id 0 | tidak ada constraint yang mencegah student_report dummy |

---

## Notes

- Istilah: `ccaYear` = penawaran CCA dalam satu academic year (UI `/ccaYear/{id}/{ay}`).
  `student_report` = identitas report seorang student pada satu `class_year`
  (yang menjadi acuan semua nilai: LIG, CCA, dll.).
- `class_year` id `0` adalah **placeholder** ber-"Classroom 001" (AY24) yang seharusnya tidak
  dipakai; keberadaannya membuat `ClassYear.findOne({ id: 0 })` lolos dan memungkinkan
  pembuatan `student_report` dummy.
- Report `REPORT_TERM` (type 1) memuat CCA bila level-nya **bukan** grading guide
  `LEARNING_OUTCOME` (lihat `helpers/create-report.helper.ts`). Level student 100683
  (`master_level_id` 6) tidak punya baris `grading_guide`, jadi report-nya memakai
  `createStudentTermReport` (progress-report.helper.ts) yang memang menyertakan CCA.
- Report `REPORT_SEMESTER` (type 2) memuat CCA lewat `semester-report.helper.ts` — relevan
  untuk dua kasus historis (term 4).
- Pola akar masalah ini (data turunan/produk tertaut ke entitas placeholder, tanpa validasi
  & tanpa proteksi DB) sejalan dengan bug `unenroll-orphan-learning-indicator-grade`
  (LIG orphan) dan `progress-report-tardy-duplicate-attendance` (baris attendance ganda).
