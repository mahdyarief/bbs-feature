---
title: Attendance — Kasus Gayathri (class 100913, 21 Okt 2026) sebagai contoh: duplicate `daily_attendance` per class/day bikin hari dobel di individual summary — AY 2026/2027 Term 2
status: open
severity: critical
product: BBS LMS
portal: Admin | Teacher
author: System Analyst
date: 2026-10-22
jam: https://jam.dev/c/9c11983c-624e-4198-a5ad-99e74a01f540
---

# Attendance — Kasus Gayathri: duplicate `daily_attendance` per class/day bikin hari dobel di individual summary (AY 2026/2027 Term 2)

> Nama "Gayathri" di judul hanya penanda rekaman/kasus yang dilaporkan. Bug yang dijelaskan di
> sini **bukan bug satu siswa** — scope-nya seluruh **academic year 2026/2027 term 2**:
> 73 grup kelas+tanggal, 32 kelas, 38 siswa terdampak di individual summary.

## Summary

Pada halaman attendance homeroom class (Teacher portal), tombol **"Check Attendance"** di atas dan
alur tabel di bawah menampilkan data berbeda untuk hari yang sama. Kasus yang dilaporkan: Gayathri
(`student_id=26557`, `class_year_id=100913`) tanggal **2026-10-21** tampil **PRESENT** lewat tombol
atas (07:24:03 → 12:32:54) tapi **LATE** lewat tombol bawah (13:26:19, check-out kosong).

Akar masalahnya bukan di UI: satu kelas bisa punya **lebih dari satu `daily_attendance` untuk
tanggal yang sama**, karena pembuatannya pakai pola check-then-insert tanpa transaksi dan tanpa
unique constraint. Akibatnya baris `attendance` siswa terpecah ke beberapa batch, masing-masing
dengan `is_first_inserted = true`. Individual summary (`student_attendance_report`) menghitung
`COUNT(DISTINCT daily_attendance_id)` per status, jadi satu hari yang dobel bisa terhitung **2x** —
misal 1 PRESENT + 1 LATE di tanggal yang sama.

**Scope audit: AY 2026/2027 (`academic_year_id = 27`) term 2 (`2026-10-05` s/d `2026-12-10`).**
Di rentang ini ada **73 grup** `(class_year_id, date)` dengan > 1 batch, **162 baris
`daily_attendance`**, di **32 dari 203 kelas aktif**. Di level `attendance`: 46 pasangan duplikat
intra-batch (47 baris berlebih) dan 39 siswa-hari yang tersebar di > 1 batch, melibatkan
**38 siswa**, dengan **18** di antaranya berpola present+late persis seperti Gayathri.

**Ekspektasi:** satu kelas hanya punya **satu** `daily_attendance` per tanggal, satu siswa hanya
punya **satu** baris `attendance` first-inserted per tanggal, dan summary tidak menghitung hari yang
sama dua kali. Kasus Gayathri (tanggal 21) harus terhitung sekali sebagai PRESENT.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Ihsan Mursyid |
| Email (Jam account) | ihsan@kotakode.com |
| Jam author ID | — (tidak tersedia di metadata rekaman) |
| User ID (dari console log, mis. `selfUser`) | — (console log kosong di rekaman) |
| Portal URL | `https://teacher.smartbag.binabangsaschool.com/classrooms/100913?term=2&pageSize=8&page=0&tab=attendance` |
| Environment API | `https://api.binabangsaschool.dev` |
| Data konteks (class ID / daId / tanggal) | `class_year_id=100913`, `student_id=26557`, `daId=34724` lalu `daId=34723`, tanggal `2026-10-21` |
| Browser / OS | Chrome 155 / Windows 11 |

Data DB untuk kasus Gayathri tanggal 21:

| Batch | date | status | present | late | baris | siswa unik | dibuat (UTC) |
|---|---|---|---|---|---|---|---|
| `34723` | 2026-10-21 | 2 (COMPLETE) | 21 | 0 | 22 | 21 | 00:50:16.856 |
| `34724` | 2026-10-21 | 1 (INCOMPLETE) | 1 | 5 | 28 | **6** | 00:50:16.863 |
| `34726` | 2026-10-21 | 1 (INCOMPLETE) | 1 | 0 | 1 | 1 | 00:50:16.863 |

Baris `attendance` Gayathri:

- `628320` — batch `34723`, status `1` (PRESENT), check-in `07:24:03`, check-out `12:32:54`, `is_first_inserted=true`, dibuat `01:36:33 UTC`
- `628638` — batch `34724`, status `2` (LATE), check-in `13:26:19`, check-out kosong, `is_first_inserted=true`, dibuat `07:39:14 UTC`

---

## Steps to Reproduce

### A. Kasus yang dilaporkan (Gayathri, 100913, 2026-10-21)

1. Buka `https://teacher.smartbag.binabangsaschool.com/classrooms/100913?term=2&pageSize=8&page=0&tab=attendance&daId=34724`.
2. Baris Gayathri: **LATE**, Check In `13:26:19`, Check Out kosong. Header: 21 siswa, Present 6
   (29%), Lateness 5 (24%), Unmarked 15 (71%).
3. Tab **Attendance Summary**: 21 siswa, Present **21 (100%)**, Late 0 — beda dari list.
4. Ganti `daId` ke `34723`: Gayathri **PRESENT** `07:24:03 → 12:32:54`, semua siswa lain present
   lengkap dengan check-out.
5. Individual summary Gayathri: tanggal 21 muncul sebagai hari **LATE**.

### B. Reproduce bug umum (berlaku untuk semua kelas di term 2)

1. Buka class mana pun yang terdampak (mis. `100915`, `100905`, `100916`) → tab Attendance, term 2.
2. Bandingkan kartu "Today"/tombol atas dengan baris tanggal di tabel: bila `daId`-nya berbeda dan
   salah satunya `daily_attendance_status = 1 (INCOMPLETE)`, data akan berbeda.
3. Verifikasi di DB: `SELECT class_year_id, date, count(*) FROM daily_attendance
   WHERE class_year_id = <cy> GROUP BY 1,2 HAVING count(*) > 1;`

**Actual Result:** satu kelas/tanggal bisa punya 2–4 `daily_attendance` (maksimum 4 di term 2, satu
grup punya 21 batch di AY sebelumnya). Siswa bisa punya > 1 baris `is_first_inserted = true` untuk
hari yang sama. Summary menghitung hari itu dua kali (present+present, atau present+late). Snapshot
Gayathri term 2 (`id 14043`, `2026-10-05`) masih `present 13, late 2, unmarked 35` — belum recalc,
jadi begitu recalc jalan `late` jadi **3**.

**Expected Result:** satu `daily_attendance` per `(class_year_id, date)`, satu baris `attendance`
first-inserted per siswa per hari, dan angka list/summary konsisten.

---

## Root Cause Analysis

### Bug #1 — `createOrUseExistingDailyAttendance()` tidak atomik — `attendance.service.ts` (line 2026-2047)

```typescript
// path:api_nest/src/modules/attendance/attendance.service.ts:2026-2047
private async createOrUseExistingDailyAttendance(classYearId: number, date: Date) {
  let dailyAttendance: DailyAttendance = new DailyAttendance();
  const existingDailyAttendance = await DailyAttendance.findOneBy({ classYearId, date });

  if (existingDailyAttendance) {
    dailyAttendance = existingDailyAttendance;
  } else {
    dailyAttendance.classYearId = classYearId;
    dailyAttendance.date = date;
    await dailyAttendance.save();   // INSERT tanpa lock & tanpa unique constraint
  }
  return dailyAttendance;
}
```

Check-then-insert tanpa transaksi/advisory lock. N request konkuren di `(classYearId, date)` yang
sama semuanya lolos `findOneBy` (null) lalu semuanya `save()` → N batch.

**Bukti pada kasus Gayathri:** `34723`, `34724`, `34726` tercipta dalam **7 milidetik**
(`00:50:16.856`, `.863`, `.863`).

### Bug #2 — Tidak ada unique constraint di `daily_attendance`

| Index | Kolom | Unique |
|---|---|---|
| `PK_4ffa42597fe155c60389fe26528` | `id` | ya |
| `IDX_9c540b3c5b5dd1b48c7fd734b7` | `(class_year_id, date)` | **tidak** |
| `IDX_816319b8cabb8093ac072c680e` | `class_year_id` | tidak |
| `IDX_17e16881fa40246cd8faaebe88` | `date` | tidak |

Tidak ada guard di level database sama sekali.

### Bug #3 — Dedup `isFirstInserted` hanya per-batch — `attendance.service.ts` (line 856-876)

```typescript
// api_nest/src/modules/attendance/attendance.service.ts:856-862
const duplicateFirstInserted = await Attendance.find({
  where: {
    dailyAttendanceId: attendance.dailyAttendanceId,  // hanya dalam batch yang sama
    studentId: student.id,
    isFirstInserted: true,
  },
});
```

Karena batch-nya sudah terlanjur berbeda, tiap batch dapat `is_first_inserted = true` sendiri.
Celah sama ada di `saveIntegrationAttendanceExclusive()` (line 1978-2024) yang lock key-nya
`(studentId, dailyAttendanceId)` — dua batch berbeda = dua lock key berbeda.

### Bug #4 — Summary menghitung per-batch, bukan per-hari — `attendance.service.ts` (line 2309-2361)

`createOrUpdateStudentRerportAttendance()` memakai `COUNT(DISTINCT daily_attendance_id)` per status.
ID berbeda untuk tanggal yang sama → hari itu dihitung dua kali.

Replikasi query untuk Gayathri (class 100913, sejak `2026-07-01`):

```
present | late | absent | absent_wo_excuse
   62   |  3   |   0    |   0
```

Tiga `daily_attendance_id` penyumbang late: `32596`, `34259`, dan **`34724`** (tanggal 21).

### Faktor pendukung — `daily_attendance_status` (1=INCOMPLETE, 2=COMPLETE)

Dalam 73 grup duplikat term 2:

| Komposisi grup | Jumlah grup |
|---|---|
| 1 COMPLETE + sisa INCOMPLETE | 58 |
| **0 COMPLETE** (tidak ada versi final) | 12 |
| ≥ 2 COMPLETE | 3 |

Snapshot `student_attendance_report` tidak punya `deleted_at`, jadi recalc membuat baris baru.

---

## Bukti dari Jam (https://jam.dev/c/9c11983c-624e-4198-a5ad-99e74a01f540)

| Sumber | Temuan |
|--------|--------|
| **Video (t=4s)** | `daId=34724`, Gayathri **LATE** 13:26:19, check-out kosong; Go Collin late 13:22:31. Header: 21 siswa, Present 6 (29%), Lateness 5 (24%), Unmarked 15 (71%). |
| **Video (t=11s)** | Tab Attendance Summary: 21 siswa, Present **21 (100%)**, Late 0. |
| **Video (t=16s)** | `daId=34723`, Gayathri **PRESENT** 07:24:03 → check-out 12:32:54; semua siswa present dengan check-out lengkap. |
| **Network (t=3.0s)** | `GET /api/v1/attendances?dailyAttendanceId=34724&isFirstInserted=true&activeStatus=ACTIVE` → 200, 12.710 byte. |
| **Network (t=11.9s)** | `GET /api/v1/attendances?dailyAttendanceId=34723&isFirstInserted=true&activeStatus=ACTIVE` → 200, 43.736 byte. |
| **Console** | Kosong di rekaman. |
| **User events** | 1 interaksi, tidak ada error di UI. |

Server tidak menolak apa pun (dua-duanya 200); perbedaan tampilan murni karena UI fetch satu batch
per `daId`. Batch `34726` tidak pernah di-fetch, jadi tidak terlihat di video.

Bukti `attendance_log` tanggal 21:

| Log | Waktu | `statusGate` | Hasil |
|---|---|---|---|
| `25632` | 07:24:03 | 1 (check-in) | `attendance_id 628320`, **PRESENT** |
| `25748` | 13:26:19 | 1 | `attendance_id 628638`, **LATE** |
| `25726`, `25766` | — | 2 (check-out) | gagal, "Already has check-out record" |

---

## Data Impact — AY 2026/2027 Term 2 (production, read-only 2026-10-22)

Scope: `class_year.academic_year_id = 27 AND deleted_at IS NULL`, `date` antara `2026-10-05` dan
`2026-12-10`. Class year aktif AY ini: **203**.

### `daily_attendance`

| Metrik | Nilai |
|---|---|
| Grup `(class_year_id, date)` dengan > 1 batch | **73** |
| Total baris `daily_attendance` di grup duplikat | **162** |
| Kelas terdampak | **32** dari 203 |
| Rentang | 2026-10-05 → 2026-10-22 |
| Grup dengan 2 / 3 / 4 batch | 58 / 14 / 1 |
| Batch terbanyak dalam satu grup | 4 |

### `attendance`

| Metrik | Nilai |
|---|---|
| Pasangan `(daily_attendance_id, student_id)` dengan > 1 baris `is_first_inserted` | **46** |
| Baris berlebih di dalam batch | **47** |
| Batch terdampak | **25** |
| Pasangan `(student_id, date)` dengan baris di > 1 batch | **39** |
| Siswa unik terdampak | **38** |
| Baris berlebih (multi-batch) | **43** |

Kombinasi status pada pasangan `(student_id, date)` multi-batch:

| Kombinasi | Pasangan | Baris lebih |
|---|---|---|
| `1,1` (present + present) | 18 | 18 |
| **`1,2` (present + late)** | **18** | **18** |
| `1,1,1` | 2 | 4 |
| `1,1,1,1` | 1 | 3 |

18 pasangan `1,2` adalah pola Gayathri. 18 pasangan `1,1` juga salah karena `present_count`
menggelembung 2x.

### Kualitas data: COMPLETE vs INCOMPLETE

Di grup yang punya tepat 1 batch COMPLETE:

| Status batch | Jumlah batch | Rata-rata baris | Rata-rata siswa unik | Rata-rata present | Rata-rata punya check-out | Total baris |
|---|---|---|---|---|---|---|
| `1` (INCOMPLETE) | 70 | 2,53 | 2,04 | 2,06 | 1,10 | 177 |
| `2` (COMPLETE) | 58 | 26,21 | 20,36 | 20,72 | 12,79 | 1.520 |

Batch COMPLETE berisi data kelas lengkap; INCOMPLETE nyaris kosong. Dasar aman untuk memilih batch
"pemenang" saat cleanup.

### Kelas terdampak (semua 32)

| class_year_id | Hari duplikat | Total batch | Max batch/hari | Rentang |
|---|---|---|---|---|
| 100915 | 8 | 19 | 3 | 2026-10-05 → 2026-10-22 |
| 100905 | 6 | 15 | 3 | 2026-10-05 → 2026-10-22 |
| 100916 | 6 | 15 | 3 | 2026-10-05 → 2026-10-21 |
| 100909 | 5 | 10 | 2 | 2026-10-05 → 2026-10-21 |
| 100955 | 5 | 10 | 2 | 2026-10-07 → 2026-10-15 |
| 100898 | 4 | 8 | 2 | 2026-10-09 → 2026-10-17 |
| 100887 | 3 | 6 | 2 | 2026-10-14 → 2026-10-21 |
| 100892 | 3 | 6 | 2 | 2026-10-06 → 2026-10-21 |
| 100906 | 3 | 6 | 2 | 2026-10-16 → 2026-10-21 |
| 100895 | 2 | 4 | 2 | 2026-10-07 → 2026-10-13 |
| 100896 | 2 | 6 | 4 | 2026-10-06 → 2026-10-17 |
| 100908 | 2 | 6 | 3 | 2026-10-09 → 2026-10-12 |
| **100913** (Gayathri) | 2 | 5 | 3 | 2026-10-16 → 2026-10-21 |
| 100914 | 2 | 4 | 2 | 2026-10-08 → 2026-10-22 |
| 100941 | 2 | 4 | 2 | 2026-10-07 → 2026-10-15 |
| 100959 | 2 | 4 | 2 | 2026-10-05 → 2026-10-06 |
| 100884 | 1 | 2 | 2 | 2026-10-07 |
| 100893 | 1 | 2 | 2 | 2026-10-06 |
| 100897 | 1 | 3 | 3 | 2026-10-17 |
| 100900 | 1 | 2 | 2 | 2026-10-09 |
| 100901 | 1 | 2 | 2 | 2026-10-16 |
| 100902 | 1 | 2 | 2 | 2026-10-17 |
| 100904 | 1 | 2 | 2 | 2026-10-21 |
| 100910 | 1 | 2 | 2 | 2026-10-20 |
| 100911 | 1 | 2 | 2 | 2026-10-05 |
| 100912 | 1 | 2 | 2 | 2026-10-09 |
| 100944 | 1 | 2 | 2 | 2026-10-13 |
| 100946 | 1 | 2 | 2 | 2026-10-07 |
| 100949 | 1 | 3 | 3 | 2026-10-12 |
| 100954 | 1 | 2 | 2 | 2026-10-05 |
| 100958 | 1 | 2 | 2 | 2026-10-05 |
| 101745 | 1 | 2 | 2 | 2026-10-21 |

Detail class `100913` (dua hari terdampak):

| Tanggal | batch_id | status | present | late | baris | siswa unik |
|---|---|---|---|---|---|---|
| 2026-10-16 | 34432 | 2 (COMPLETE) | 21 | 0 | 23 | 21 |
| 2026-10-16 | 34434 | 1 (INCOMPLETE) | 8 | 0 | 8 | 8 |
| 2026-10-21 | 34723 | 2 (COMPLETE) | 21 | 0 | 22 | 21 |
| 2026-10-21 | 34724 | 1 (INCOMPLETE) | 1 | 5 | 28 | 6 |
| 2026-10-21 | 34726 | 1 (INCOMPLETE) | 1 | 0 | 1 | 1 |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/attendance/attendance.service.ts:2026-2047` | `createOrUseExistingDailyAttendance()` race → batch duplikat |
| Backend Service | `api_nest/src/modules/attendance/attendance.service.ts:1978-2024` | Advisory lock key per-`dailyAttendanceId` |
| Backend Service | `api_nest/src/modules/attendance/attendance.service.ts:856-876` | Dedup `isFirstInserted` hanya dalam satu batch |
| Backend Service | `api_nest/src/modules/attendance/attendance.service.ts:2309-2361` | Summary hitung per-batch → hari dobel dihitung 2x |
| Backend Entity | `DailyAttendance` entity | Tidak ada unique index `(class_year_id, date)` |
| Backend Entity | `Attendance` entity | Tidak ada unique constraint `(student_id, daily_attendance_id, is_first_inserted)` |
| Database | `daily_attendance` | 73 grup duplikat, 162 baris (term 2 AY 26/27) |
| Database | `attendance` | 46 pasangan duplikat intra-batch + 39 student-day multi-batch |
| Frontend (Teacher) | `/classrooms/:id?tab=attendance` | Tidak ada indikator batch duplikat; UI hanya fetch satu `daId` |

---

## Proposed Solution Options

### Option A: Unique constraint + lock per class+tanggal (Recommended)

1. **Migration:** dedup data dulu (lihat cleanup), lalu
   `CREATE UNIQUE INDEX CONCURRENTLY uq_daily_attendance_class_date ON daily_attendance (class_year_id, date);`
2. **Service:** ubah `createOrUseExistingDailyAttendance()` jadi transaction + `pg_advisory_xact_lock`
   dengan key `(classYearId, date)`, lalu `INSERT ... ON CONFLICT DO NOTHING` + re-select.
3. **Service:** perlebar guard `duplicateFirstInserted` di `update()` ke `(studentId, classYearId, date)`.
4. **Service:** `saveIntegrationAttendanceExclusive()` — tambah advisory lock per `(classYearId, date)`
   di upstream sebelum batch dipilih.

### Option B: Application-level guard saja (tanpa migration)

Advisory lock per class+tanggal di semua caller `createOrUseExistingDailyAttendance()`. Lebih cepat
rilis, tapi tidak menutup jalur tulis lain (script, backfill, instance berbeda).

### Cleanup data (terpisah, butuh approval)

1. Per grup dengan > 1 batch: pilih **batch COMPLETE dengan baris terbanyak** sebagai pemenang.
2. Baris `attendance` dari batch kalah → soft-delete (`deleted_at`), **jangan hard-delete**.
3. Grup dengan **0 batch COMPLETE (12 grup)** → review manual, tidak ada versi final.
4. Baris duplikat di dalam batch yang sama (47 baris berlebih) → pertahankan baris terbaru,
   soft-delete sisanya.
5. **Baru setelah semua di atas:** recalc `student_attendance_report`.

Urutan wajib. Kalau recalc jalan sebelum cleanup, `late`/`present` yang salah akan ter-anchor.

---

## Notes

- Semua angka **read-only** dari production pada 2026-10-22. Tidak ada data yang diubah.
- Bug ini menjelaskan kenapa "attendance log hadir semua" — log gate mencatat check-in yang benar
  (07:24 PRESENT); baris LATE 13:26 berasal dari request kedua di batch duplikat.
- Rekaman Jam hanya menunjukkan 2 dari 3 batch aktif; `34726` tidak pernah di-fetch UI.
- `student_attendance_report` tidak punya `deleted_at`, jadi cleanup snapshot lama harus pakai
  `UPDATE` in-place atau hapus baris lama per `(student_id, class_year_id, term)`.
- Scope audit sengaja dibatasi AY 2026/2027 term 2. Duplikat serupa juga ada di AY sebelumnya.
- Format mengikuti `features/_templates/bug-report-template.md`.
