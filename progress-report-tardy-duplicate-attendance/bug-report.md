---
title: Progress Report — Angka "Tardy" Tidak Konsisten akibat Baris `attendance` Ganda dan Flag `is_first_inserted` yang Ambigu
status: open
severity: major
product: BBS LMS
portal: Teacher
author: System Analyst
date: 2026-09-23
jam: (tidak tersedia — laporan berbasis investigasi DB + code review)
---

# Progress Report — "Tardy" Menampilkan 2 padahal Data Late Hanya 1

## Summary

Pada progress report (REPORT_TERM) term 1 Academic Year 2026/2027 untuk student
**25398**, bagian attendance menampilkan **`Tardy: 2`**. Namun saat dicek di halaman attendance
(class-in-year `100912`, daily attendance `daId=32099`), jumlah late-nya hanya **1** — dan
`student_attendance_report` (tabel kanonik) juga mencatat **`late_count = 1`**.

Investigasi menemukan akar masalahnya di **data**: tabel `attendance` menyimpan **baris ganda**
untuk `(student_id, class_year_id, hari)` yang sama. Khusus student 25398, pada **2026-09-09**
terdapat **5 baris berstatus LATE** yang dibuat dalam rentang **~39 milidetik** (id 582706,
582709, 582713, 582717, 582719) — semuanya untuk `daily_attendance_id = 32099`.

Nilai `Tardy` pada report dihitung oleh `AttendanceService.findByStudent()` sebagai
`COUNT(attendance WHERE attendance_status = LATE AND is_first_inserted = true)`. Flag
`is_first_inserted` dimaksudkan menandai satu baris kanonik per hari, tetapi saat ada baris ganda
flag ini bisa ambigu/tidak deterministik — sehingga angka `Tardy` bisa bergeser ±1 dari nilai
sebenarnya.

**Ekspektasi:** angka `Tardy` di progress report harus sama dengan jumlah hari siswa tercatat LATE
(mengikuti data kanonik `student_attendance_report`), yaitu **1** pada kasus ini.

> Catatan: dari 4.776 report `REPORT_TERM` term 1 yang diperiksa, **1.904 cocok persis** dengan
> hitungan kanonik, 391 stale (nilai 0 padahal ada data), 146 under-count (selisih selalu −1), dan
> **hanya 1 over-count** — yaitu student 25398 pada laporan ini.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | — (laporan dari user) |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | — |
| Portal URL | https://admin.smartbag.binabangsaschool.com/students/25398 (progress report term 1 AY 2026/2027) |
| Environment API | api.binabangsaschool.com (production) |
| Data konteks (class ID / daId / tanggal) | student_id=25398; class_year=100912; academic_year=27 (2026/2027); term=1; daily_attendance daId=32099 (2026-09-09); student_report id=5236; report id=56861 (REPORT_TERM term 1) |
| Database diperiksa | `binabangsa_prod_mig_v01` via `D:\Work\BBS\requirement\binabangsa-db-tools` |
| Browser / OS | — |

---

## Steps to Reproduce

1. Buka progress report student **25398** term 1 AY 2026/2027 (REPORT_TERM).
2. Perhatikan bagian attendance: tertulis **`Tardy: 2`** (bersama `Present: 47`, `Absent: 2`,
   `Days of School: 49`).
3. Buka halaman attendance class-in-year `100912` (daily attendance `daId=32099`) — late untuk
   student tersebut hanya **1**. Begitu juga `student_attendance_report.late_count = 1`.
4. Verifikasi di DB bahwa student 25398 punya **5 baris** `attendance` LATE pada 2026-09-09
   (query di bagian "Query Verifikasi").

**Actual Result:**
- Progress report menampilkan `Tardy: 2`, padahal data kanonik late hanya `1`.
- Tabel `attendance` menyimpan 5 baris identik (status LATE, gate 19, daily attendance 32099)
  untuk satu student pada satu hari, dibuat dalam ~39 ms.

**Expected Result:**
- `Tardy` di progress report = jumlah hari siswa tercatat LATE secara kanonik (= 1).
- Tidak ada baris `attendance` ganda untuk `(student, class_year, hari)` yang sama.

---

## Root Cause Analysis

### Bug #1 (akar masalah utama) — baris `attendance` ganda tanpa unique constraint

Tabel `attendance` **tidak punya unique constraint** pada `(student_id, class_year_id, date)`
maupun `(student_id, class_year_id, daily_attendance_id)`. Index yang ada semuanya non-unique
(pkey `id`, index biasa `student_id`, `class_year_id`, `daily_attendance_id`, `is_first_inserted`,
beberapa index komposit — tidak ada yang `UNIQUE`).

Baris ganda muncul dari race condition saat insert (check-in gate dan/atau penandaan guru
menyasar hari yang sama). Bukti paling jelas pada student 25398:

| id | status | gate | daily_attendance | is_first_inserted | created_at |
|---|---|---|---|---|---|
| 582706 | 2 (LATE) | 19 | 32099 | **true** | 2026-09-09 01:08:29.692671 |
| 582709 | 2 (LATE) | 19 | 32099 | false | 2026-09-09 01:08:29.709969 |
| 582713 | 2 (LATE) | 19 | 32099 | false | 2026-09-09 01:08:29.729368 |
| 582717 | 2 (LATE) | 19 | 32099 | false | 2026-09-09 01:08:29.731367 |
| 582719 | 2 (LATE) | 19 | 32099 | false | 2026-09-09 01:08:29.731428 |

Lima INSERT dalam ~39 ms hanya mungkin terjadi pada request konkuren. Skala global: **30.364 grup
duplikat** `(student_id, class_year_id, day)` dengan total **~34.399 baris ekstra**.

### Bug #2 (penguat) — filter `is_first_inserted` yang ambigu

Nilai `Tardy` di report berasal dari `AttendanceService.findByStudent()`
(`api_nest/src/modules/attendance/attendance.service.ts`), yang menghitung:

```
late = COUNT(attendance WHERE attendance_status = '2' (LATE)
             AND is_first_inserted = true
             AND date BETWEEN termStart AND termEnd)
```

`progress-report.helper.ts:455` memanggil method ini dan memakai `studentAttendance.late` untuk
nilai tardiness (baris ~496).

Flag `is_first_inserted` di-set `true` bila **belum ada** baris attendance untuk hari itu
(`attendance.isFirstInserted = !todayAttendance`, `attendance.service.ts:198` dan `:358`). Ada
logika dedup di `attendance.service.ts:898-911` yang, bila menemukan lebih dari satu baris
ber-flag `true` untuk `(dailyAttendanceId, studentId)`, menyisakan hanya yang `updatedAt` terbaru
— **tetapi logika ini hanya berjalan pada jalur update tertentu**, sehingga jalur insert lain bisa
meninggalkan lebih dari satu flag `true`.

Akibatnya jumlah baris LATE ber-flag `true` untuk satu hari bisa > 1 (atau 0), dan nilai `Tardy`
bisa bergeser. Untuk student 25398, snapshot report menangkap `2`, sedangkan kondisi kanonik
sekarang `1`.

**Bukti rumus & isolasi kasus (perbandingan lintas siswa):** dari 21 siswa di class_year 100912,
**20 cocok persis** antara `Tardy` di HTML report dan `late_first`. Dari 4.776 report
`REPORT_TERM` term 1 global: **1.904 cocok**, **391 stale** (report 0, data ada),
**146 under** (selisih selalu −1), dan **1 over** — hanya student 25398.

### Bug #3 (sekunder) — konfigurasi `report_setting` AY 2026/2027 belum ada untuk Primary

Tabel `report_setting` tidak memiliki baris untuk `academic_year_id = 27` pada level Primary 1–6
(hanya ada satu baris untuk level 12). Akibatnya report memakai fallback setting default
**AY 2025/2026** (id=25) yang memetakan label `Present` / `Absent` / `Tardy`. Ini menjelaskan label
"Tardy" yang muncul, tetapi bukan penyebab selisih angkanya.

**Catatan:** angka `Tardy` **bukan** berasal dari tabel `discipline` — student 25398 memiliki
**0 baris** discipline, jadi bukan dari halaman discipline-overview.

---

## Bukti dari Database

Diambil dari `binabangsa_prod_mig_v01`:

| Metrik | Nilai |
|--------|-------|
| Report diperiksa | id 56861 (REPORT_TERM / type 1, term 1), student_report 5236 |
| Angka di HTML report | `Present: 47`, `Absent: 2`, `Tardy: 2`, `Days of School: 49` |
| `student_attendance_report` id 12120 | present=44, **late=1**, absent=2, unmarked=2 |
| Baris `attendance` student 25398 (aktif) | 58 baris → status '1'=47 (44 first), **status '2'=9 (1 first)**, '3'=2 |
| Tanggal LATE | 07-23, 07-30, 08-04, **09-09 (5 baris)**, 09-16 → 5 hari berbeda |
| Baris LATE 2026-09-09 | 5 baris (582706 `first=true`, 4 lainnya `false`) dibuat dalam ~39 ms |
| `daily_attendance` 32099 (09-09) | present_count=19, late_count=2, absent_count=0 |
| Unique constraint `attendance (student_id, class_year_id, date)` | **tidak ada** (semua index non-unique) |
| Grup duplikat `attendance` global | **30.364 grup**, ~34.399 baris ekstra |
| Grup dengan `is_first_inserted=true` > 1 | 23.824 (kunci `date::date`); **374 baris** (kunci `daily_attendance_id`) |
| Baris identik-murni (aman di-dedupe) | **~4.976 baris** |
| Baris `discipline` student 25398 | **0** |
| `report_setting` untuk AY 27 Primary 1–6 | **tidak ada** (fallback ke AY 26 default) |

**Bukti bug masih aktif (2026-09-23):** setelah cleanup `--dedupe` pertama (14:14 WIB) menghapus
4.976 baris dan menyisakan **0** duplikat, dalam ~1,5 jam berikutnya aplikasi produksi
**kembali membuat 34 baris duplikat baru** (semua `created_at` 2026-09-23 07:11–08:47 UTC,
status LATE, hari ini) — diperbaiki dengan dedupe kedua. Ini membuktikan duplikat terus
terbentuk selama akar masalah (tidak ada unique constraint + insert non-atomik) belum diperbaiki.
Nilai `late_first` student 25398 tetap **1** setelah cleanup (tidak berubah; dedupe count-neutral).

**Hasil scan 4.776 report `REPORT_TERM` term 1:**

| Kategori | Jumlah | Catatan |
|---|---|---|
| Cocok (Tardy = `late_first`) | 1.904 | rumus terbukti |
| Stale (report 0, data ada) | 391 | report lama tak pernah di-regenerate |
| Under (report < kanonik) | 146 | selisih **selalu −1** |
| **Over (report > kanonik)** | **1** | **student 25398** (2 vs 1) |

---

## Dampak

- Angka kehadiran (khususnya `Tardy`/late) pada progress report bisa **tidak akurat** dan
  tidak konsisten dengan halaman attendance serta `student_attendance_report`.
- Dokumen resmi (progress report) yang dibagikan ke orang tua berpotensi memuat angka yang salah.
- Data ganda di produksi (30.364 grup, ~34.399 baris ekstra) dan bertambah seiring setiap
  check-in/preview konkuren; tidak ada proteksi DB.

---

## Rekomendasi Perbaikan

1. **Database (paling penting):** tambahkan unique constraint/index pada baris attendance kanonik,
   mis. `UNIQUE (student_id, class_year_id, daily_attendance_id) WHERE is_first_inserted = true`
   (partial unique index), atau pada `(student_id, class_year_id, date)` sesuai definisi hari.
2. **Backend:** jadikan insert attendance **idempotent/atomic** (upsert atau `ON CONFLICT`) agar
   concurrency tidak menghasilkan baris ganda, dan pastikan logika dedup `is_first_inserted`
   (attendance.service.ts:898-911) berlaku pada **semua** jalur insert.
3. **Definisikan ulang perhitungan `late`:** pertimbangkan menghitung per **hari** yang andal
   (mis. distinct `daily_attendance_id`), bukan per baris dengan flag yang rawan ambigu.
4. **Konfigurasi:** lengkapi `report_setting` untuk AY 2026/2027 level Primary 1–6 agar tidak
   bergantung pada fallback AY lama.
5. **Bersihkan data existing:** jalankan script `binabangsa-db-tools/cleanup_duplicate_attendance.py`
   (`--dedupe` untuk baris identik-murni; `--normalize-first` untuk menegakkan satu flag kanonik
   per hari — opsional, setelah review).
6. **Regenerate report** student 25398 setelah data bersih agar HTML-nya konsisten (`Tardy: 1`).

---

## Query Verifikasi

```sql
-- baris late student 25398 (termasuk flag kanonik)
SELECT id, date::date, attendance_status::text, is_first_inserted,
       campus_gate_id, daily_attendance_id, created_at
FROM attendance
WHERE student_id = 25398 AND attendance_status::text = '2'
ORDER BY date, id;

-- hitung kanonik vs mentah
SELECT
  COUNT(*) FILTER (WHERE attendance_status::text='2') AS late_all,
  COUNT(*) FILTER (WHERE attendance_status::text='2' AND is_first_inserted) AS late_first
FROM attendance
WHERE student_id = 25398 AND class_year_id = 100912 AND deleted_at IS NULL;

-- baris ganda per (student, class_year, hari)
SELECT student_id, class_year_id, daily_attendance_id, COUNT(*) AS n
FROM attendance
WHERE deleted_at IS NULL
GROUP BY 1,2,3 HAVING COUNT(*) > 1
ORDER BY n DESC;

-- grup dengan lebih dari satu flag is_first_inserted dalam satu hari
SELECT student_id, class_year_id, daily_attendance_id,
       COUNT(*) FILTER (WHERE is_first_inserted) AS n_first
FROM attendance
WHERE deleted_at IS NULL
GROUP BY 1,2,3 HAVING COUNT(*) FILTER (WHERE is_first_inserted) > 1
ORDER BY n_first DESC;
```

Script cleanup tersedia di:
`D:\Work\BBS\requirement\binabangsa-db-tools\cleanup_duplicate_attendance.py`

```bash
cd D:\Work\BBS\requirement\binabangsa-db-tools
python cleanup_duplicate_attendance.py --student-id 25398            # dry-run
python cleanup_duplicate_attendance.py --dedupe --yes               # soft-delete duplikat murni
python cleanup_duplicate_attendance.py --dedupe --normalize-first --yes  # + normalisasi flag
```

---

## Notes

- Enum `attendance_status`: **1=PRESENT, 2=LATE, 3=ABSENT, 4=ABSENT_WITHOUT_EXCUSE**.
- Kolom `attendance.date` bertipe **timestamptz**, bukan `date`; mayoritas baris tersimpan di
  `17:00:00 UTC` (= tengah malam WIB hari berikutnya). Karena itu kunci hari yang andal adalah
  `daily_attendance_id`, bukan `date::date` (UTC).
- Generator report: `progress-report.helper.ts:455` (panggil `findByStudent`) dan baris ~496
  (`tardiness = studentAttendance.late`); `presence` dibangun dari `reportSetting.attendances`.
- Tiga dari empat angka di report (Present 47 = 49−2, Absent 2, Days of School 49) **cocok**
  dengan perhitungan server; hanya `Tardy` yang menyimpang.
- Tidak ada FK yang mereferensikan `attendance`, sehingga soft-delete aman sepenuhnya.
