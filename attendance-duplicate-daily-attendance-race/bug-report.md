---
title: Attendance — Duplicate `daily_attendance` per class/day, AY 2026/2027 Term 2
status: open
severity: critical
product: BBS LMS
portal: Admin | Teacher
author: System Analyst (AI-assisted, verified on production read-only)
date: 2026-10-22
jam: https://jam.dev/c/9c11983c-624e-4198-a5ad-99e74a01f540
---

# Attendance — Duplicate `daily_attendance` per class/day (AY 2026/2027 Term 2)

## Summary

Satu kelas bisa punya **lebih dari satu `daily_attendance` untuk tanggal yang sama**. Akibatnya
baris `attendance` siswa terpecah ke beberapa batch, dan tiap batch punya `is_first_inserted = true`
sendiri-sendiri. Individual summary (`student_attendance_report`) menghitung
`COUNT(DISTINCT daily_attendance_id)` per status, jadi satu hari yang dobel bisa terhitung
**2x** — misal 1 PRESENT + 1 LATE di tanggal yang sama.

Scope audit ini: **academic year 2026/2027 (`academic_year_id = 27`), term 2**
(`2026-10-05` s/d `2026-12-10`). Di rentang ini ada **73 grup** `(class_year_id, date)` dengan
> 1 batch, melibatkan **162 baris `daily_attendance`** di **32 kelas** (dari 203 class year aktif).

Kasus yang dilaporkan (Gayathri, class `100913`, `2026-10-21`) adalah satu instance.

**Ekspektasi:** satu kelas hanya punya **satu** `daily_attendance` per tanggal, dan satu siswa hanya
punya **satu** baris `attendance` first-inserted per tanggal. Satu `daily_attendance_id` juga hanya
boleh punya satu baris `is_first_inserted` per siswa. Summary tidak boleh menghitung hari yang sama
dua kali.

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
| Data konteks (class ID / daId / tanggal) | `class_year_id=100913`, `daId=34724` lalu `daId=34723`, tanggal `2026-10-21` |
| Browser / OS | Chrome 155 / Windows 11 |

---

## Steps to Reproduce

1. Buka `https://teacher.smartbag.binabangsaschool.com/classrooms/100913?term=2&pageSize=8&page=0&tab=attendance&daId=34724`
   — batch pertama yang terbuka.
2. Baris Gayathri (student `26557`): status **LATE**, Check In `13:26:19`, Check Out kosong.
   Header summary: Total 21, Present 6 (29%), Lateness 5 (24%), Unmarked 15 (71%).
3. Ganti ke tab **Attendance Summary**: Total 21, Present **21 (100%)**, Late 0 — beda dari list.
4. Ganti `daId` ke `34723`: Gayathri jadi **PRESENT**, Check In `07:24:03`, Check Out `12:32:54`,
   dan seluruh siswa lain juga PRESENT dengan check-out lengkap.

**Actual Result:** tiga `daily_attendance` untuk `(100913, 2026-10-21)` — `34723`, `34724`, `34726`.
Gayathri punya dua baris `attendance` yang dua-duanya `is_first_inserted = true`: `628320`
(PRESENT, batch `34723`) dan `628638` (LATE, batch `34724`). UI menampilkan data berbeda tergantung
`daId` mana yang dibuka, tanpa penanda bahwa batch itu duplikat.

**Expected Result:** hanya ada satu `daily_attendance` untuk `(100913, 2026-10-21)`, satu baris
`attendance` per siswa, dan tab list serta tab summary menampilkan angka yang konsisten.

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
    await dailyAttendance.save();   // <-- INSERT tanpa unique constraint & tanpa lock
  }

  return dailyAttendance;
}
```

Pola check-then-insert tanpa transaksi/advisory lock. Kalau ada N request konkuren di
`(classYearId, date)` yang sama, semuanya bisa lolos `findOneBy` (null) lalu semuanya `save()`,
menghasilkan N batch.

**Bukti:** `34723`, `34724`, `34726` tercipta dalam **7 milidetik** — `00:50:16.856`, `.863`, `.863`.
Tiga request konkuren.

### Bug #2 — Tidak ada unique constraint yang mencegahnya — tabel `daily_attendance`

| Index | Kolom | Unique |
|---|---|---|
| `PK_4ffa42597fe155c60389fe26528` | `id` | ya |
| `IDX_9c540b3c5b5dd1b48c7fd734b7` | `(class_year_id, date)` | **tidak** |
| `IDX_816319b8cabb8093ac072c680e` | `class_year_id` | tidak |
| `IDX_17e16881fa40246cd8faaebe88` | `date` | tidak |

Tidak ada guard di level database sama sekali.

### Bug #3 — Dedup guard hanya per-batch — `attendance.service.ts` (line 856-876)

```typescript
// path:api_nest/src/modules/attendance/attendance.service.ts:856-862
const duplicateFirstInserted = await Attendance.find({
  where: {
    dailyAttendanceId: attendance.dailyAttendanceId,  // <-- hanya dalam batch yang sama
    studentId: student.id,
    isFirstInserted: true,
  },
});
```

Guard ini hanya menormalkan `isFirstInserted` **di dalam satu `dailyAttendanceId`**. Kalau dua batch
sudah terbentuk (Bug #1), siswa mendapat `is_first_inserted = true` di masing-masing batch, dan guard
ini tidak akan pernah melihat yang lain.

Celah serupa ada di `saveIntegrationAttendanceExclusive()` (line 1978-2024): advisory lock memakai
key `(studentId, dailyAttendanceId)` — dua batch berbeda berarti dua lock key berbeda, jadi tidak
saling serialize.

### Faktor pendukung — `daily_attendance_status` (enum 1/2)

Status batch: `1` = INCOMPLETE, `2` = COMPLETE. Dalam 73 grup duplikat term 2:

- `1` complete + sisa incomplete: **58 grup** (normal — satu versi final + residu)
- `0` batch complete: **12 grup** — tidak punya versi final sama sekali
- `2` batch complete: **3 grup** — dua-duanya dianggap final

Snapshot `student_attendance_report` **tidak punya kolom `deleted_at`**, jadi recalc akan membuat
baris baru, bukan menimpa.

---

## Bukti dari Jam (https://jam.dev/c/9c11983c-624e-4198-a5ad-99e74a01f540)

| Sumber | Temuan |
|--------|--------|
| **Video (t=4s)** | `daId=34724`, Gayathri **LATE** 13:26:19, check-out kosong; Go Collin juga late 13:22:31. Header summary: 21 siswa, Present 6 (29%), Lateness 5 (24%), Unmarked 15 (71%). |
| **Video (t=11s)** | Tab **Attendance Summary**: 21 siswa, Present **21 (100%)**, Late 0. Kontradiktif dengan list di t=4s. |
| **Video (t=16s)** | `daId=34723`, Gayathri **PRESENT** 07:24:03 → check-out 12:32:54; semua siswa present dengan check-out lengkap. |
| **Network (t=3.0s)** | `GET /api/v1/attendances?dailyAttendanceId=34724&isFirstInserted=true&activeStatus=ACTIVE` → **200**, response **12.710 byte**. |
| **Network (t=11.9s)** | `GET /api/v1/attendances?dailyAttendanceId=34723&isFirstInserted=true&activeStatus=ACTIVE` → **200**, response **43.736 byte**. |
| **Console** | Kosong di rekaman. |
| **User events** | 1 interaksi, tidak ada indikasi error di UI. |

Kesimpulan dari Jam: **server tidak menolak apa pun** (dua-duanya 200). Perbedaan tampilan murni
karena UI fetch satu batch per `daId`.

---

## Data Impact — AY 2026/2027 Term 2 (production, read-only 2026-10-22)

Scope: `class_year.academic_year_id = 27 AND deleted_at IS NULL`, `date` antara `2026-10-05` dan
`2026-12-10`. Total class year aktif AY ini: **203**.

### Skala duplikat `daily_attendance`

| Metrik | Nilai |
|---|---|
| Grup `(class_year_id, date)` dengan > 1 batch | **73** |
| Total baris `daily_attendance` di grup duplikat | **162** |
| Kelas terdampak | **32** dari 203 |
| Rentang | 2026-10-05 → 2026-10-22 |
| Grup dengan 2 batch | 58 |
| Grup dengan 3 batch | 14 |
| Grup dengan 4 batch | 1 |
| Batch terbanyak dalam satu grup | 4 |

### Duplikat di level `attendance`

| Metrik | Nilai |
|---|---|
| Pasangan `(daily_attendance_id, student_id)` dengan > 1 baris `is_first_inserted` | **46** |
| **Baris berlebih di dalam batch** | **47** |
| Batch terdampak | **25** |
| Pasangan `(student_id, date)` dengan baris di **> 1 batch** | **39** |
| Siswa unik terdampak | **38** |
| Baris berlebih (multi-batch) | **43** |

Kombinasi status pada pasangan `(student_id, date)` multi-batch:

| Kombinasi status | Pasangan | Baris lebih |
|---|---|---|
| `1,1` (present + present) | 18 | 18 |
| **`1,2` (present + late)** | **18** | **18** |
| `1,1,1` | 2 | 4 |
| `1,1,1,1` | 1 | 3 |

**18 pasangan `1,2` adalah pola kasus Gayathri**: satu hari terhitung PRESENT di satu batch dan LATE
di batch lain. Sisanya `1,1` juga salah karena `present_count` menggelembung 2x.

### Kualitas data: batch COMPLETE vs INCOMPLETE

Di grup yang punya tepat 1 batch COMPLETE:

| Status batch | Jumlah | Rata-rata baris | Rata-rata siswa unik | Rata-rata present | Rata-rata punya check-out | Total baris |
|---|---|---|---|---|---|---|
| `1` (INCOMPLETE) | 70 | 2,53 | 2,04 | 2,06 | 1,10 | 177 |
| `2` (COMPLETE) | 58 | 26,21 | 20,36 | 20,72 | 12,79 | 1.520 |

Batch COMPLETE berisi data kelas yang lengkap; batch INCOMPLETE nyaris kosong. Ini dasar aman untuk
memilih batch "pemenang" saat cleanup.

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

Detail class `100913`:

| Tanggal | batch_id | status | present | late | baris | siswa unik |
|---|---|---|---|---|---|---|
| 2026-10-16 | 34432 | 2 (COMPLETE) | 21 | 0 | 23 | 21 |
| 2026-10-16 | 34434 | 1 (INCOMPLETE) | 8 | 0 | 8 | 8 |
| 2026-10-21 | 34723 | 2 (COMPLETE) | 21 | 0 | 22 | 21 |
| 2026-10-21 | 34724 | 1 (INCOMPLETE) | 1 | 5 | 28 | 6 |
| 2026-10-21 | 34726 | 1 (INCOMPLETE) | 1 | 0 | 1 | 1 |

### Snapshot `student_attendance_report`

Snapshot Gayathri term 2 (`id 14043`, tanggal `2026-10-05`) masih `present 13, late 2, unmarked 35`.
**Recalc belum jalan** — kalau dijalankan sekarang, `late` naik ke **3** karena baris `628638`
(batch `34724`) masih ada.

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/attendance/attendance.service.ts:2026-2047` | `createOrUseExistingDailyAttendance()` race → batch duplikat |
| Backend Service | `api_nest/src/modules/attendance/attendance.service.ts:1978-2024` | Advisory lock key per-`dailyAttendanceId`, tidak mengunci per class+tanggal |
| Backend Service | `api_nest/src/modules/attendance/attendance.service.ts:856-876` | Dedup `isFirstInserted` hanya dalam satu batch |
| Backend Service | `api_nest/src/modules/attendance/attendance.service.ts:2309-2361` | `createOrUpdateStudentRerportAttendance()` pakai `COUNT(DISTINCT daily_attendance_id)` |
| Backend Entity | `DailyAttendance` entity | Tidak ada unique index `(class_year_id, date)` |
| Backend Entity | `Attendance` entity | Tidak ada unique constraint `(student_id, daily_attendance_id, is_first_inserted)` |
| Database | tabel `daily_attendance` | 73 grup duplikat, 162 baris (term 2 AY 26/27) |
| Database | tabel `attendance` | 46 pasangan duplikat intra-batch + 39 student-day multi-batch |
| Frontend (Teacher) | `/classrooms/:id?tab=attendance` | Tidak ada indikator batch duplikat; UI hanya fetch satu `daId` |

---

## Proposed Solution Options

### Option A: Unique constraint + lock per class+tanggal (Recommended)

1. **Migration:** dedup data dulu (lihat cleanup di bawah), lalu
   `CREATE UNIQUE INDEX CONCURRENTLY uq_daily_attendance_class_date ON daily_attendance (class_year_id, date);`
2. **Service:** ubah `createOrUseExistingDailyAttendance()` jadi transaction + `pg_advisory_xact_lock`
   dengan key `(classYearId, date)`, lalu `INSERT ... ON CONFLICT DO NOTHING` + re-select.
3. **Service:** perlebar guard `duplicateFirstInserted` di `update()` supaya mencari per
   `(studentId, classYearId, date)`.
4. **Service:** `saveIntegrationAttendanceExclusive()` — tambah advisory lock per
   `(classYearId, date)` di upstream sebelum batch dipilih.

### Option B: Application-level guard saja (tanpa migration)

Tambah advisory lock per class+tanggal di semua caller `createOrUseExistingDailyAttendance()`.
Lebih cepat rilis, tapi tidak menjamin konsistensi kalau ada jalur tulis lain (script, backfill,
instance berbeda).

### Cleanup data (terpisah, butuh approval)

1. Untuk tiap grup dengan > 1 batch: pilih **batch COMPLETE dengan baris terbanyak** sebagai
   pemenang.
2. Baris `attendance` dari batch kalah → soft-delete (`deleted_at`), **jangan hard-delete**.
3. Grup dengan **0 batch COMPLETE (12 grup)** butuh review manual — tidak ada versi final.
4. Baris duplikat di dalam batch yang sama (47 baris berlebih) → pertahankan baris terbaru,
   soft-delete sisanya.
5. **Baru setelah semua di atas:** jalankan recalc `student_attendance_report`.

Urutan ini wajib. Kalau recalc jalan sebelum cleanup, angka `late`/`present` yang salah akan
ter-anchor ke snapshot baru.

---

## Notes

- Semua angka diambil **read-only** dari production pada 2026-10-22. Tidak ada perubahan data.
- Bug ini menjelaskan kenapa "attendance log hadir semua" — log gate scan mencatat check-in yang
  benar (07:24 PRESENT); baris LATE 13:26 berasal dari request kedua di batch duplikat, bukan dari
  log gate yang sama.
- Rekaman Jam hanya menunjukkan **2 dari 3** batch aktif di tanggal itu. Batch `34726` tidak pernah
  di-fetch UI, jadi tidak terlihat di video — bukti bahwa jumlah batch sebenarnya lebih banyak dari
  yang tampak di layar.
- `student_attendance_report` **tidak punya `deleted_at`**, jadi cleanup snapshot lama harus pakai
  strategi lain (`UPDATE` in-place atau hapus baris lama per `(student_id, class_year_id, term)`).
- Scope audit sengaja dibatasi **AY 2026/2027 term 2** sesuai permintaan. Duplikat serupa juga ada
  di AY sebelumnya (lihat riwayat query 2026-10-22), tapi tidak dimasukkan ke brief ini.
- Folder ini pakai format `bug-report.md` (mengikuti `features/_templates/bug-report-template.md`).
