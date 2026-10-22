---
title: Attendance — Data Remediation for Overwritten Records (AY 2026/2027 Term 2)
status: proposed
severity: critical
product: BBS LMS
portal: Admin | Teacher
author: System Analyst (AI-assisted, verified on production read-only)
date: 2026-10-22
parent: bug-report.md
scope: academic_year_id = 27, date 2026-10-05 → 2026-12-10
---

# Attendance — Data Remediation for Overwritten Records

Pelengkap [`bug-report.md`](bug-report.md). Bug report menjelaskan **kenapa** duplikat terjadi; dokumen
ini hanya menjawab **data yang sudah terlanjur rusak diapakan**.

Tujuan: satu `daily_attendance` per `(class_year_id, date)`, satu baris `attendance` first-inserted per
siswa per tanggal, dan snapshot `student_attendance_report` tidak menghitung hari yang sama dua kali.

> Semua SQL di dokumen ini **belum dieksekusi**. Angka diambil read-only dari production 2026-10-22.
> Baseline data (76 grup / 168 baris / 33 kelas) ada di §1 di bawah — sengaja diulang karena angka di
> bug report (73/162/32) sudah kedaluwarsa.

---

## 1. Empat Temuan yang Mengubah Strategi Cleanup

Ringkasan 5 langkah cleanup di `bug-report.md` §"Cleanup data" **tidak cukup** dan sebagian akan
menghapus data. Empat koreksi:

| # | Temuan aktual | Kenapa langkah di bug report gagal |
|---|---|---|
| 1 | **11 siswa hanya ada di batch kalah** (5 grup; terbesar `100892 / 2026-10-06` = 7 siswa) | "Soft-delete baris batch kalah" akan menghapus siswa ini. Harus **merge** (`UPDATE daily_attendance_id`), bukan delete |
| 2 | **35 grup ambigu**, bukan 12: 15 grup tanpa batch COMPLETE + 20 grup dengan skor seri (mis. `100887 / 2026-10-14` punya 2 batch COMPLETE berisi 18 siswa identik) | Klasifikasi "0 batch COMPLETE" di bug report tidak menangkap kasus dua batch COMPLETE. Keduanya butuh penanganan berbeda |
| 3 | **23 grup `(student_id, class_year_id, term)` punya baris snapshot duplikat** | Recalc memakai `findOne` tanpa `ORDER BY` (`attendance.service.ts:2265-2271`), jadi baris yang di-update tidak deterministik. Snapshot harus dibersihkan dulu |
| 4 | **Unique index harus dipasang setelah code fix deployed** | Bug report menaruh index di langkah pertama. Kalau index live sebelum fix, race berubah dari "membuat duplikat" jadi "error 500 pada user" |

Baseline ekseskusi: **76 grup, 168 baris `daily_attendance`, 33 kelas** (naik dari 73/162/32 karena
data baru masuk setelah audit pertama).

---

## 2. Prinsip

1. Batch kalah **di-soft-delete** (`deleted_at = now()`), tidak pernah hard-delete.
2. Batch COMPLETE menang atas INCOMPLETE (COMPLETE ≈ 26 baris/20 siswa; INCOMPLETE ≈ 2,5 baris/2 siswa).
3. Data unik di batch kalah **dipindahkan** ke batch pemenang, tidak dihapus.
4. Status yang dipertahankan adalah check-in paling awal — LATE yang lahir dari batch duplikat dibuang.
5. Recalc snapshot **paling akhir**, setelah `attendance` bersih.

---

## 3. Alur (urutan wajib)

```
[0] Backup → [1] Klasifikasi → [2] Merge siswa unik → [3] Dedup intra-batch
→ [4] Soft-delete batch kalah → [5] Review manual 35 grup → [6] Recalc snapshot
→ [7] Verifikasi → [8] Unique index (setelah code fix deploy)
```

---

## 4. Langkah Detail

### [0] Backup

```sql
CREATE TABLE _bak_daily_attendance_20261022 AS
SELECT * FROM daily_attendance WHERE date BETWEEN '2026-10-05' AND '2026-12-10';

CREATE TABLE _bak_attendance_20261022 AS
SELECT * FROM attendance WHERE date BETWEEN '2026-10-05' AND '2026-12-10';

CREATE TABLE _bak_sar_20261022 AS
SELECT * FROM student_attendance_report WHERE academic_year_id = 27 AND term = 2;
```

Hapus tabel `_bak_*` di sesi terpisah setelah observasi, jangan di transaksi yang sama.

### [1] Klasifikasi grup

```sql
CREATE TABLE _remediasi_plan AS
WITH dup AS (
  SELECT class_year_id, date::date AS d
  FROM daily_attendance
  WHERE deleted_at IS NULL AND date BETWEEN '2026-10-05' AND '2026-12-10'
  GROUP BY 1,2 HAVING COUNT(*) > 1
),
b AS (
  SELECT da.id AS batch_id, da.class_year_id, da.date::date AS d,
         da.daily_attendance_status AS st,
         COUNT(a.id) FILTER (WHERE a.deleted_at IS NULL) AS rows_n,
         COUNT(DISTINCT a.student_id) FILTER (WHERE a.deleted_at IS NULL) AS students_n
  FROM daily_attendance da
  JOIN dup ON dup.class_year_id = da.class_year_id AND dup.d = da.date::date
  LEFT JOIN attendance a ON a.daily_attendance_id = da.id
  WHERE da.deleted_at IS NULL
  GROUP BY da.id, da.class_year_id, da.date, da.daily_attendance_status
),
ranked AS (
  SELECT *,
         ROW_NUMBER() OVER (
           PARTITION BY class_year_id, d
           ORDER BY st DESC, students_n DESC, rows_n DESC, batch_id
         ) AS rk,
         COUNT(*) OVER (PARTITION BY class_year_id, d, st, students_n, rows_n) AS ties
  FROM b
),
grp AS (
  SELECT class_year_id, d,
         COUNT(*) FILTER (WHERE st = '2') AS n_complete,
         MAX(ties) FILTER (WHERE rk = 1) AS winner_ties
  FROM ranked GROUP BY 1,2
)
SELECT r.class_year_id, r.d, r.batch_id, r.st, r.rows_n, r.students_n, r.rk,
       CASE
         WHEN g.n_complete = 0 THEN 'MANUAL_NO_COMPLETE'
         WHEN g.winner_ties > 1 AND r.st = '2' THEN 'MANUAL_TIE'
         ELSE CASE WHEN r.rk = 1 THEN 'KEEP' ELSE 'DROP' END
       END AS decision
FROM ranked r JOIN grp g USING (class_year_id, d);
```

Verifikasi klasifikasi:

```sql
SELECT decision, COUNT(*) AS batches FROM _remediasi_plan GROUP BY 1 ORDER BY 1;

-- Harus 0 baris: setiap grup punya KEEP atau MANUAL
SELECT class_year_id, d FROM _remediasi_plan
GROUP BY 1,2
HAVING COUNT(*) FILTER (WHERE decision = 'KEEP') = 0
   AND COUNT(*) FILTER (WHERE decision LIKE 'MANUAL%') = 0;
```

### [2] Merge siswa yang hanya ada di batch kalah

```sql
CREATE TABLE _remediasi_lost_students AS
WITH w AS (
  SELECT class_year_id, d, batch_id AS win_batch
  FROM _remediasi_plan WHERE decision = 'KEEP'
),
dup_students AS (
  SELECT DISTINCT a.class_year_id, a.date::date AS d, a.student_id
  FROM attendance a
  JOIN _remediasi_plan p ON p.class_year_id = a.class_year_id AND p.d = a.date::date
  WHERE a.deleted_at IS NULL
)
SELECT ds.*, w.win_batch
FROM dup_students ds
JOIN w USING (class_year_id, d)
WHERE NOT EXISTS (
  SELECT 1 FROM attendance x
  WHERE x.daily_attendance_id = w.win_batch
    AND x.student_id = ds.student_id
    AND x.deleted_at IS NULL
);
```

Cek dulu — harus **~11 baris**. Kalau jauh lebih banyak, klasifikasi di [1] salah:

```sql
SELECT class_year_id, d, COUNT(*) FROM _remediasi_lost_students GROUP BY 1,2 ORDER BY 3 DESC;
```

```sql
UPDATE attendance a
SET daily_attendance_id = ls.win_batch, updated_at = now()
FROM _remediasi_lost_students ls
WHERE a.class_year_id = ls.class_year_id
  AND a.date::date = ls.d
  AND a.student_id = ls.student_id
  AND a.deleted_at IS NULL
  AND a.daily_attendance_id <> ls.win_batch;
```

Ulangi query `_remediasi_lost_students` — harus 0 baris.

### [3] Dedup baris first-inserted dalam satu batch

Pertahankan check-in paling awal, soft-delete sisanya (46 pasangan di bug report):

```sql
CREATE TABLE _remediasi_intra_dups AS
SELECT a.id AS attendance_id
FROM attendance a
WHERE a.deleted_at IS NULL
  AND a.is_first_inserted = true
  AND a.date BETWEEN '2026-10-05' AND '2026-12-10'
  AND a.id NOT IN (
    SELECT DISTINCT ON (daily_attendance_id, student_id) id
    FROM attendance
    WHERE deleted_at IS NULL AND is_first_inserted = true
      AND date BETWEEN '2026-10-05' AND '2026-12-10'
    ORDER BY daily_attendance_id, student_id, check_in_date ASC NULLS LAST, id
  );

UPDATE attendance a
SET deleted_at = now(), updated_at = now()
FROM _remediasi_intra_dups d
WHERE a.id = d.attendance_id AND a.deleted_at IS NULL;
```

### [4] Soft-delete batch kalah

```sql
UPDATE daily_attendance da
SET deleted_at = now(), updated_at = now()
FROM _remediasi_plan p
WHERE da.id = p.batch_id AND p.decision = 'DROP' AND da.deleted_at IS NULL;

UPDATE attendance a
SET deleted_at = now(), updated_at = now()
FROM _remediasi_plan p
WHERE a.daily_attendance_id = p.batch_id
  AND p.decision = 'DROP'
  AND a.deleted_at IS NULL;
```

Setelah ini batch `DROP` tidak terlihat oleh aplikasi (semua query memakai `deleted_at IS NULL`).

### [5] Review manual 35 grup

**Jangan jalankan cleanup otomatis pada grup ini.**

```sql
SELECT p.class_year_id, p.d, p.decision, p.batch_id, p.st, p.rows_n, p.students_n,
       (SELECT string_agg(DISTINCT a.student_name, ', ')
        FROM attendance a WHERE a.daily_attendance_id = p.batch_id AND a.deleted_at IS NULL) AS siswa
FROM _remediasi_plan p
WHERE p.decision LIKE 'MANUAL%'
ORDER BY p.class_year_id, p.d, p.batch_id;
```

- **`MANUAL_NO_COMPLETE` (15 grup)** — tidak ada batch yang di-finalisasi guru. Bandingkan dengan
  `attendance_log` tanggal itu, pilih batch terisi paling lengkap, set `daily_attendance_status = '2'`,
  lalu soft-delete sisanya seperti [4].
- **`MANUAL_TIE` (20 grup)** — dua batch COMPLETE sama lengkap. Pilih yang `updated_at` paling akhir.
  **Jangan merge** dua batch COMPLETE karena isinya duplikat 100%.

Catat keputusan:

```sql
CREATE TABLE _remediasi_manual_log (
  class_year_id int, d date, kept_batch int, dropped_batches int[],
  decided_by text, decided_at timestamptz DEFAULT now(), note text
);
```

### [6] Recalc `student_attendance_report`

Baru jalan setelah [2]–[5] hijau. Recalc memakai `COUNT(DISTINCT daily_attendance_id)` per status
(`attendance.service.ts:2334-2337`); setelah cleanup satu siswa per tanggal hanya ada di satu batch.

Lewat endpoint aplikasi supaya filter `student_status = CURRENT` dan logika `unmarked` ikut jalan:

```
POST /api/v1/attendances/recalculate-student-attendance-report
{ "studentId": <id>, "classYearId": <id>, "term": 2 }
```

**Sebelum recalc, bersihkan duplikat snapshot** (temuan #3 di §1):

```sql
DELETE FROM student_attendance_report s
USING (
  SELECT id, ROW_NUMBER() OVER (
    PARTITION BY student_id, class_year_id, term ORDER BY date DESC, id DESC
  ) rk
  FROM student_attendance_report
  WHERE academic_year_id = 27 AND term = 2
) d
WHERE s.id = d.id AND d.rk > 1;
```

Target recalc:

```sql
SELECT DISTINCT a.student_id, a.class_year_id
FROM attendance a
WHERE a.class_year_id IN (SELECT DISTINCT class_year_id FROM _remediasi_plan)
  AND a.deleted_at IS NULL
  AND a.date BETWEEN '2026-10-05' AND '2026-12-10';
```

### [7] Verifikasi

Semua harus 0 baris kecuali #4:

```sql
-- 1. Tidak ada grup duplikat tersisa
SELECT COUNT(*) FROM (
  SELECT class_year_id, date FROM daily_attendance
  WHERE deleted_at IS NULL AND date BETWEEN '2026-10-05' AND '2026-12-10'
  GROUP BY 1,2 HAVING COUNT(*) > 1
) x;

-- 2. Tidak ada siswa dobel per tanggal
SELECT COUNT(*) FROM (
  SELECT student_id, class_year_id, date::date FROM attendance
  WHERE deleted_at IS NULL AND is_first_inserted = true
    AND date BETWEEN '2026-10-05' AND '2026-12-10'
  GROUP BY 1,2,3 HAVING COUNT(*) > 1
) x;

-- 3. Tidak ada siswa tersebar di >1 batch per tanggal
SELECT COUNT(*) FROM (
  SELECT student_id, date::date FROM attendance
  WHERE deleted_at IS NULL AND is_first_inserted = true
    AND date BETWEEN '2026-10-05' AND '2026-12-10'
  GROUP BY 1,2 HAVING COUNT(DISTINCT daily_attendance_id) > 1
) x;

-- 4. Kasus Gayathri: harus tepat 1 baris PRESENT, check-out 12:32:54
SELECT a.id, a.attendance_status::text, a.check_in_date AT TIME ZONE 'Asia/Jakarta',
       a.check_out_date AT TIME ZONE 'Asia/Jakarta', a.daily_attendance_id
FROM attendance a
WHERE a.student_id = 26557 AND a.date::date = '2026-10-21' AND a.deleted_at IS NULL;
```

### [8] Unique index

**Setelah** code fix dari bug report deployed. Kalau index lebih dulu, race jadi error 500.

```sql
CREATE UNIQUE INDEX CONCURRENTLY uq_daily_attendance_class_date
ON daily_attendance (class_year_id, date)
WHERE deleted_at IS NULL;
```

`CONCURRENTLY` tidak boleh di dalam transaksi.

---

## 5. Dampak

| Area | Sebelum | Sesudah |
|---|---|---|
| Grup duplikat | 76 | 0 |
| Baris `daily_attendance` di grup duplikat | 168 | 76 |
| Siswa tercatat LATE dari batch duplikat | 18 pasangan `1,2` | 0 |
| Siswa hilang saat cleanup | — | 0 (11 di-merge) |

`attendance_log` dan baris `attendance` asli tetap ada, hanya di-soft-delete atau dipindah batch.

---

## 6. Checklist & Risiko

- [ ] Backup `_bak_*` dibuat
- [ ] `_remediasi_plan` dibuat, tidak ada grup tanpa KEEP/MANUAL
- [ ] `_remediasi_lost_students` ~11 baris → update → 0 baris
- [ ] `_remediasi_intra_dups` di-soft-delete
- [ ] Batch `DROP` + attendance-nya di-soft-delete
- [ ] 15 `MANUAL_NO_COMPLETE` + 20 `MANUAL_TIE` diputuskan & dicatat
- [ ] Duplikat snapshot dihapus, recalc dijalankan
- [ ] Verifikasi §7 #1–#4 hijau
- [ ] Code fix deployed → `uq_daily_attendance_class_date` dibuat
- [ ] `_bak_*` dihapus setelah observasi

| Risiko | Mitigasi |
|---|---|
| Salah pilih batch pemenang | Reversibel via `deleted_at` + backup |
| Recalc sebelum cleanup | Urutan §3; verifikasi #1–#3 wajib lulus dulu |
| Rapor sudah dicetak dari angka salah | Komunikasikan ke akademik sebelum eksekusi |
| Data AY lain juga bermasalah | Di luar scope (AY 25/26 & 24/25 punya pola serupa), audit terpisah |
