---
title: Progress Report — Subject yang Sudah Di-unenroll Masih Muncul akibat Baris `learning_indicator_grade` Orphan
status: open
severity: major
product: BBS LMS
portal: Teacher | Student | Admin
author: System Analyst
date: 2026-09-24
jam: (tidak tersedia — laporan berbasis investigasi DB + code review)
---

# Progress Report — Subject Sisa Unenroll (Faith Builder) Masih Tampil Bersama Subject Baru (Character First)

## Summary

Pada student **103372 (Holi Xia Mo)**, class 100884 (AY 2026/2027, level 6), terjadi
**salah enroll**: student seharusnya di subject **Character First** (`subject_year` 102201),
tetapi sebelumnya sempat terdaftar di **Faith Builder** (`subject_year` 102205). Student sudah
di-**unenroll** dari Faith Builder (baris pivot `banding_students_student` sudah dihapus, kini
student hanya punya 12 banding tanpa Faith Builder), **namun di report kedua subject tetap muncul**.

Akar masalahnya ada di **data turunan yang tidak ikut terhapus**: proses unenroll hanya menghapus
pivot enrollment, sementara baris **`learning_indicator_grade` (LIG)** milik Faith Builder
(id 102106) **tertinggal aktif**. Karena grading guide level ini = `LEARNING_OUTCOME`, report
`REPORT_TERM` dirender oleh `createLoReport` yang menyusun daftar subject dari **LIG yang punya
mark** — bukan dari pivot enrollment. Baris LIG orphan itu membuat Faith Builder "resurrection"
di report. **Regenerate report saja tidak akan pernah menghilangkannya** selama baris LIG ada.

> Cakupan: **3 baris** orphan LIG di AY 2026/2027 (semuanya Faith Builder term 1: student 103372,
> 102653, 23772) dan **398 baris** orphan LIG historis global (AY 2024/2025 = 2, AY 2025/2026 = 196,
> AY 2026/2027 = 197 sebelum cleanup). AY 2026/2027 sudah dibersihkan; historis belum.

**Ekspektasi:** report student 103372 hanya menampilkan **Character First**, bukan Faith Builder.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | — (laporan dari user) |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | — |
| Portal URL | Report student 103372 (Teacher/Admin portal, AY 2026/2027 term 1) |
| Environment API | api.binabangsaschool.dev (production) |
| Data konteks (class ID / daId / tanggal) | student_id=103372; class_year=100884; academic_year=27 (2026/2027); term=1; student_report=5085; subject_year Faith Builder=102205 (subject 482), Character First=102201 (subject 452); report REPORT_TERM=57887, LEARNING_OUTCOME=54433 |
| Database diperiksa | `binabangsa_prod_mig_v01` via `D:\Work\BBS\requirement\binabangsa-db-tools` |
| Browser / OS | — |

---

## Steps to Reproduce

1. Buka report student **103372 (Holi Xia Mo)** term 1 AY 2026/2027.
2. Perhatikan daftar subject: muncul **FAITH BUILDER** dan **CHARACTER FIRST** sekaligus.
3. Cek enrollment di DB: pivot `banding_students_student` untuk student 103372 **tidak memuat**
   Faith Builder (`subject_year` 102205) — hanya Character First (102201) dan 11 subject lainnya.
4. Cek `learning_indicator_grade`: masih ada baris aktif ke Faith Builder (id 102106, term 1).

**Actual Result:**
- Report menampilkan Faith Builder meskipun student sudah di-unenroll dari subject tersebut.
- Baris `learning_indicator_grade` untuk Faith Builder masih aktif (tidak terhapus saat unenroll).

**Expected Result:**
- Report hanya menampilkan subject yang masih ter-enroll (Character First, dst), bukan Faith Builder.
- Unenroll subject harus ikut membersihkan data turunan (LIG) agar tidak ada baris orphan.

---

## Root Cause Analysis

### Bug #1 (akar masalah utama) — unenroll tidak membersihkan `learning_indicator_grade`

Enrollment per-student ke subject direpresentasikan oleh pivot **`banding_students_student`**
(`banding` → `subject_year` → `subject`). Saat unenroll, aplikasi menghapus baris pivot tersebut,
tetapi **tidak** menghapus baris turunan di `learning_indicator_grade` (nilai LO per student per
subject). Baris LIG menjadi **orphan**: `(student_id, subject_year_id)` sudah tidak ada di pivot,
tetapi barisnya masih aktif.

Bukti pada student 103372: dari **8 tabel** yang menyimpan relasi (student_id + subject_year_id/
subject_id), **hanya `learning_indicator_grade`** yang masih menyimpan data Faith Builder —
sisanya 0. Pivot banding sudah bersih.

### Bug #2 (kenapa report tetap salah) — daftar subject report disusun dari LIG, bukan dari enrollment

`helpers/create-report.helper.ts` memaksa `REPORT_TERM` (type 1) dirender oleh `createLoReport`
bila grading guide level = `LEARNING_OUTCOME`:

```typescript
// helpers/create-report.helper.ts — dispatch report type
if ([ReportTypeEnum.REPORT_TERM, ReportTypeEnum.REPORT_SEMESTER].includes(
      ReportTypeEnum[reportType])) {
  if (gradingGuide?.gradingGuideType === GradingGuideTypeEnum.LEARNING_OUTCOME) {
    reportType = ReportTypeEnum[ReportTypeEnum.LEARNING_OUTCOME]; // dipaksa ke LO
  }
}
```

`helpers/reports/lo-report.helper.ts` lalu menyusun daftar subject dari learning outcome kurikulum
yang **punya baris LIG**:

```typescript
// helpers/reports/lo-report.helper.ts — subject list dari LIG
const liMarks = learningIndicators
  .map((li) => {
    const liGrade = liGrades.find(
      (liGrade) => liGrade.learningIndicatorId === li.id && liGrade.term === li.term,
    );
    return { ..., mark: getLoMark(liGrade, loGradeSetting), ... };
  })
  .filter((li) => showUngradedSubject || li.mark !== null)   // hanya yang punya mark
  ...
```

`show_ungraded_subject` bernilai **`false`** di seluruh baris `report_setting`, jadi filter
`li.mark !== null` aktif: subject hanya muncul jika ada baris LIG-nya. Baris LIG Faith Builder yang
orphan tetap lolos filter → Faith Builder muncul kembali di report. Karena REPORT_TERM dan
LEARNING_OUTCOME memakai builder yang sama, kedua report panjang HTML-nya identik (14.570 char).

**Bukti silang (decisive):**

| Subject | Ada di pivot banding? | Punya baris LIG? | Muncul di report? |
|---|---|---|---|
| Computer Science (102195) | Ya | **Tidak** | **Tidak** |
| Faith Builder (102205) | **Tidak** | Ya | **Ya** |

Artinya report mengikuti **LIG**, bukan pivot enrollment.

### Bug #3 (pendukung) — `learning_indicator_grade` tidak punya pembersihan otomatis

Tidak ada mekanisme (trigger/FK cascade/service) yang menghapus LIG saat baris banding/pivot
dihapus. Jadi setiap unenroll berpotensi meninggalkan orphan — hasil deteksi: **398 baris** orphan
LIG aktif secara global (AY24=2, AY25=196, AY26=197, AY27=3 sebelum cleanup).

---

## Bukti dari Database

Diambil dari `binabangsa_prod_mig_v01`:

| Metrik | Nilai |
|--------|-------|
| Student | 103372 — Holi Xia Mo, class_year 100884, AY 27, level 6, master_level_id 2 |
| `subject_year` Faith Builder | id 102205 (subject 482), masih `deleted_at = NULL` (offering class, normal) |
| `subject_year` Character First | id 102201 (subject 452) |
| Pivot `banding_students_student` student 103372 | **12 baris**, Faith Builder **tidak ada** (sudah di-unenroll) |
| `learning_indicator_grade` student 103372 (term 1) | **23 baris aktif**, termasuk **FB indikator 271 (id 102106)** dan **CF indikator 254 (id 109111)** |
| Tabel lain yang masih menyimpan FB | **hanya `learning_indicator_grade`** (7 tabel lain = 0) |
| `show_ungraded_subject` di `report_setting` | **`false`** di semua baris |
| Orphan LIG AY 2026/2027 | **3 baris** — semuanya Faith Builder term 1 (student 103372, 102653, 23772) |
| Orphan LIG global | **398 baris** (AY24=2, AY25=196, AY26=197, AY27=3) |
| Report terdampak | 57887 (REPORT_TERM, type 1) & 54433 (LEARNING_OUTCOME, type 7) |
| Panjang HTML report | **14.570 → 13.889** char setelah Faith Builder dihapus |

**Hasil eksekusi cleanup AY 2026/2027:**
- Soft-delete **3 baris** orphan LIG (backup: `result/orphan_lig_ay27_backup_20260924_115716.csv`), sisa 0.
- Regenerate report 5085 (REPORT_TERM + LEARNING_OUTCOME term 1) → HTTP 201.
- Verifikasi: `FAITH BUILDER` **hilang** (`False`), `CHARACTER FIRST` **tetap ada** (`True`), sisa LIG FB = 0.

---

## Dampak

- Report (dokumen resmi ke orang tua) menampilkan **subject yang sudah tidak diikuti** student —
  menyesatkan, khususnya saat subject di-switch (Faith Builder → Character First).
- Nilai/learning outcome subject lama bisa ikut tercetak seolah masih berlaku.
- Masalah **sistemik**: 398 baris orphan LIG global; setiap unenroll subject berpotensi menambah orphan.
- Regenerate report tidak memperbaiki apa pun (akar masalah di data LIG), sehingga signal bug ini
  mudah hilang saat tim hanya mencoba "regenerate ulang".

---

## Rekomendasi Perbaikan

### Option A: Bersihkan data + perbaiki proses unenroll (Recommended)

1. **Perbaikan proses (utama):** saat unenroll student dari subject (hapus pivot
   `banding_students_student`), ikut soft-delete baris `learning_indicator_grade` (dan data turunan
   lain) untuk `(student_id, subject_year_id)` tersebut — idealnya dalam **satu transaksi**.
2. **Bersihkan data existing:** jalankan `scripts/cleanup_orphan_lig.py` (`--academic-year-id 27
   --execute` untuk AY sekarang; historis setelah review).
3. **Regenerate report** student terdampak via `scripts/regenerate_report.py` (REPORT_TERM +
   LEARNING_OUTCOME) agar HTML konsisten.

### Option B: Pertahanan di sisi report builder

`lo-report.helper.ts` sebaiknya **meng-intersect** daftar subject/learning outcome dengan enrollment
aktif student (pivot banding) — sehingga baris LIG orphan tidak lagi "membocorkan" subject ke report,
terlepas dari kebersihan data. Bisa dikombinasikan dengan Option A.

### Option C: Proteksi DB

Tambahkan constraint/trigger yang menjaga integritas LIG terhadap enrollment (mis. FK/trigger yang
menghapus LIG saat pivot banding dihapus), agar orphan tidak bisa terbentuk lagi.

---

## Query Verifikasi

```sql
-- baris LIG student 103372 beserta nama subject
SELECT lig.id, lig.subject_year_id, s.name, lig.learning_indicator_id, lig.term,
       lig.student_report_id, lig.deleted_at
FROM learning_indicator_grade lig
LEFT JOIN subject s ON s.id = lig.subject_id
WHERE lig.student_id = 103372
ORDER BY lig.subject_year_id, lig.id;

-- pivot enrollment student 103372 (apakah Faith Builder ada?)
SELECT b.id AS banding_id, b.subject_year_id, sy.subject_id, s.name
FROM banding_students_student bss
JOIN banding b ON b.id = bss.banding_id
JOIN subject_year sy ON sy.id = b.subject_year_id
LEFT JOIN subject s ON s.id = sy.subject_id
WHERE bss.student_id = 103372
ORDER BY b.subject_year_id;

-- deteksi orphan LIG (subject sudah tak ada di pivot banding)
SELECT cy.academic_year_id, COUNT(*) AS n
FROM learning_indicator_grade lig
JOIN student_report sr ON sr.id = lig.student_report_id
JOIN class_year cy ON cy.id = sr.class_year_id
WHERE lig.deleted_at IS NULL
  AND NOT EXISTS (
    SELECT 1 FROM banding_students_student bss
    JOIN banding b ON b.id = bss.banding_id
    JOIN subject_year sy ON sy.id = b.subject_year_id
    WHERE bss.student_id = lig.student_id
      AND sy.id = lig.subject_year_id
      AND b.deleted_at IS NULL
  )
GROUP BY cy.academic_year_id
ORDER BY cy.academic_year_id;
```

Script tersedia di: `D:\Work\BBS\requirement\binabangsa-db-tools\scripts\`

```bash
cd D:\Work\BBS\requirement\binabangsa-db-tools\scripts

# satu pintu: check + handle (dry-run default)
python handle_orphan_enrollment.py --student-id 103372             # check
python handle_orphan_enrollment.py --student-id 103372 --execute   # handle + regenerate

# atau terpisah
python cleanup_orphan_lig.py --academic-year-id 27                 # dry-run
python cleanup_orphan_lig.py --academic-year-id 27 --execute --yes # soft-delete
python regenerate_report.py --student-report-id 5085 --report-type REPORT_TERM --term 1
python regenerate_report.py --student-report-id 5085 --report-type LEARNING_OUTCOME --term 1
```

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend (unenroll flow) | service yang menghapus pivot `banding_students_student` | tidak menghapus `learning_indicator_grade` → orphan |
| Backend (report dispatch) | `api_nest/src/helpers/create-report.helper.ts` | memaksa REPORT_TERM → LO saat grading guide = LEARNING_OUTCOME |
| Backend (report builder) | `api_nest/src/helpers/reports/lo-report.helper.ts` | menyusun subject dari LIG (bukan enrollment) → subject orphan muncul |
| Database | `learning_indicator_grade`, `banding_students_student`, `banding`, `subject_year` | tidak ada constraint/trigger integritas enrollment |

---

## Notes

- Istilah: "unenroll" = menghapus baris pivot `banding_students_student` (student ↔ banding).
  `banding` = pemetaan subject (via `subject_year`) ke teacher; `subject_year` = subject dalam
  sebuah class_year (yang di UI disebut "subject-in-year").
- `show_ungraded_subject = false` di semua `report_setting` (AY25, AY26, AY27) — ini kunci kenapa
  baris LIG menentukan subject yang tampil.
- Report REPORT_TERM (type 1) dan LEARNING_OUTCOME (type 7) untuk student bergrading-guide LO
  memakai **builder yang sama**, sehingga HTML-nya identik — keduanya harus di-regenerate
  setelah cleanup.
- Histori AY 2024/2025–2025/2026 punya 395 orphan LIG. Belum dibersihkan; script
  `handle_orphan_enrollment.py` (tanpa filter AY) siap dipakai bila diperlukan.
- Pola akar masalah ini (data turunan tidak dibersihkan saat operasi induk, tanpa proteksi DB)
  sama dengan bug `progress-report-tardy-duplicate-attendance` (baris `attendance` ganda tanpa
  unique constraint).
