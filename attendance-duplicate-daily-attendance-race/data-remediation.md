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

Dokumen ini melengkapi [`bug-report.md`](bug-report.md). Bug report menjelaskan **kenapa** data bisa
dobel (race di `createOrUseExistingDailyAttendance()`). Dokumen ini menjawab pertanyaan turunannya:
**data yang sudah terlanjur rusak harus diapakan?**

Tujuan akhirnya satu: **kembalikan data ke kondisi sebenarnya** — satu kelas satu `daily_attendance`
per tanggal, satu siswa satu baris `attendance` per tanggal dengan status yang benar, dan snapshot
`student_attendance_report` yang tidak menghitung hari yang sama dua kali.

> Semua angka di dokumen ini diambil **read-only** dari production pada 2026-10-22. Tidak ada
> perubahan data yang dilakukan saat penyusunan brief ini.

---

## 1. Ringkasan Kondisi Data (production, read-only)

Angka di sini dikonfirmasi ulang dari DB dan **sedikit berbeda** dari bug report (bug report: 73 grup /
162 baris / 32 kelas), karena data baru masuk setelah audit pertama.

| Metrik | Nilai |
|---|---|
| Grup `(class_year_id, date)` dengan > 1 batch | **76** |
| Total baris `daily_attendance` di grup duplikat | **168** |
| Kelas terdampak | **33** |
| Grup tanpa satu pun batch COMPLETE | **15** |
| Grup dengan skor tie-break seri (butuh keputusan) | **20** |
| Grup yang punya siswa unik hanya di batch kalah | **5** |
| Total siswa yang hilang jika hanya memakai batch pemenang | **11** |

Kombinasi status pada pasangan `(student_id, date)` yang tersebar di lebih dari satu batch tetap
seperti di bug report: 18 pasangan `1,1`, **18 pasangan `1,2` (PRESENT + LATE)**, 2 pasangan `1,1,1`,
1 pasangan `1,1,1,1`.

**Pasangan `1,2` adalah pola kasus yang dilaporkan** (Gayathri): check-in pagi 07:24 tercatat PRESENT
di batch yang benar, lalu ada baris kedua 13:26 yang tercatat LATE di batch duplikat. Guru melihat
LATE, padahal secara bisnis siswa itu hadir tepat waktu.

---

## 2. Prinsip Remediasi

Lima aturan yang menentukan semua keputusan di bawah:

1. **Satu `daily_attendance` per `(class_year_id, date)`.** Batch yang kalah **di-soft-delete**
   (`deleted_at = now()`), tidak pernah hard-delete. Semua operasi harus reversibel.
2. **Batch COMPLETE menang atas INCOMPLETE.** Batch `2` berisi data kelas lengkap (rata-rata 26 baris,
   20 siswa); batch `1` nyaris kosong (rata-rata 2,5 baris, 2 siswa). Lihat §1 bug report.
3. **Jangan menghapus data unik.** Jika ada siswa yang hanya muncul di batch kalah, barisnya
   **dipindahkan** (`daily_attendance_id` di-update) ke batch pemenang, bukan dihapus.
4. **Status yang dipertahankan adalah status check-in paling awal dan paling wajar** — PRESENT jika
   siswa terbukti hadir di jam normal. Baris LATE yang lahir dari batch duplikat di-soft-delete.
5. **Recalc snapshot paling akhir.** `student_attendance_report` dihitung ulang **setelah** tabel
   `attendance` bersih. Kalau recalc jalan lebih dulu, angka salah akan ter-anchor permanen.

---

## 3. Alur Eksekusi (urutan wajib)

```
[0] Backup tabel terdampak
      ↓
[1] Snapshot daftar grup + klasifikasi (auto vs manual)
      ↓
[2] Auto-cleanup grup yang jelas (batch COMPLETE tunggal, tanpa siswa unik)
      ↓
[3] Merge siswa unik dari batch kalah ke batch pemenang
      ↓
[4] Dedup baris first_inserted di dalam batch yang sama
      ↓
[5] Soft-delete batch kalah + batch kosong
      ↓
[6] Review manual 15 grup tanpa COMPLETE + 20 grup seri
      ↓
[7] Recalc student_attendance_report
      ↓
[8] Verify + kunci dengan unique index
```

Langkah `[8]` (unique index) sengaja paling akhir: index baru bisa dibuat setelah data bersih.

---

## 4. Langkah Detail

### [0] Backup

Wajib sebelum apa pun. Tanpa backup, langkah ini tidak boleh dijalankan.

```sql
CREATE TABLE _bak_daily_attendance_20261022 AS
SELECT * FROM daily_attendance
WHERE date BETWEEN '2026-10-05' AND '2026-12-10';

CREATE TABLE _bak_attendance_20261022 AS
SELECT * FROM attendance
WHERE date BETWEEN '2026-10-05' AND '2026-12-10';

CREATE TABLE _bak_sar_20261022 AS
SELECT * FROM student_attendance_report
WHERE academic_year_id = 27 AND term = 2;
```

Setelah semua langkah hijau dan diverifikasi, hapus tabel `_bak_*` di sesi terpisah. Jangan hapus
dalam transaksi yang sama dengan cleanup.

### [1] Klasifikasi grup

Buat tabel kerja berisi keputusan per grup. Ini yang dibaca oleh langkah-langkah berikutnya, supaya
cleanup tidak perlu menghitung ulang aturan ranking di setiap query.

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
FROM ranked r
JOIN grp g USING (class_year_id, d);
```

Cek hasil klasifikasi:

```sql
SELECT decision, COUNT(*) AS batches
FROM _remediasi_plan
GROUP BY 1 ORDER BY 1;
```

Ekspektasi: `KEEP` ≈ 61 batch, `DROP` ≈ 92 batch, `MANUAL_NO_COMPLETE` 33 baris (15 grup),
`MANUAL_TIE` 40 baris (20 grup). Angka pas bisa bergeser sedikit, yang penting adalah
**tidak ada grup yang tanpa KEEP dan tanpa MANUAL**.

```sql
-- Harus 0 baris. Kalau ada, grup itu belum punya pemenang.
SELECT class_year_id, d FROM _remediasi_plan
GROUP BY 1,2
HAVING COUNT(*) FILTER (WHERE decision = 'KEEP') = 0
   AND COUNT(*) FILTER (WHERE decision LIKE 'MANUAL%') = 0;
```

### [2] Grup yang punya siswa unik di batch kalah

Ada **5 grup** yang jika hanya memakai batch pemenang akan kehilangan siswa (total 11 siswa). Ini
khususnya `100892 / 2026-10-06` (7 siswa hanya ada di batch yang kalah). Grup-grup ini **tidak boleh**
membersihkan batch kalah sebelum baris siswanya dipindahkan.

```sql
CREATE TABLE _remediasi_lost_students AS
WITH w AS (
  SELECT class_year_id, d, batch_id AS win_batch
  FROM _remediasi_plan WHERE decision = 'KEEP'
),
dup_students AS (
  SELECT DISTINCT a.class_year_id, a.date::date AS d, a.student_id
  FROM attendance a
  JOIN _remediasi_plan p
    ON p.class_year_id = a.class_year_id AND p.d = a.date::date
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

Lihat isinya dulu — kalau jumlahnya jauh lebih besar dari 11, ada yang salah di klasifikasi:

```sql
SELECT class_year_id, d, COUNT(*) FROM _remediasi_lost_students GROUP BY 1,2 ORDER BY 3 DESC;
```

Pindahkan barisnya ke batch pemenang. Baris **tidak** dihapus, hanya dipindah ke batch yang benar:

```sql
UPDATE attendance a
SET daily_attendance_id = ls.win_batch,
    updated_at = now()
FROM _remediasi_lost_students ls
WHERE a.class_year_id = ls.class_year_id
  AND a.date::date = ls.d
  AND a.student_id = ls.student_id
  AND a.deleted_at IS NULL
  AND a.daily_attendance_id <> ls.win_batch;
```

Setelah update, ulangi query `_remediasi_lost_students` — harus **0 baris**.

### [3] Dedup baris first-inserted dalam satu batch

Bug report mencatat 46 pasangan `(daily_attendance_id, student_id)` dengan lebih dari satu baris
`is_first_inserted = true`. Baris yang dipertahankan adalah **yang paling awal check-in** — itu yang
paling dekat dengan kejadian sebenarnya di gerbang. Baris lainnya di-soft-delete.

```sql
CREATE TABLE _remediasi_intra_dups AS
SELECT a.id AS attendance_id, a.daily_attendance_id, a.student_id
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
```

```sql
UPDATE attendance a
SET deleted_at = now(), updated_at = now()
FROM _remediasi_intra_dups d
WHERE a.id = d.attendance_id AND a.deleted_at IS NULL;
```

### [4] Soft-delete batch kalah dan batch kosong

```sql
-- batch yang kalah per klasifikasi
UPDATE daily_attendance da
SET deleted_at = now(), updated_at = now()
FROM _remediasi_plan p
WHERE da.id = p.batch_id AND p.decision = 'DROP' AND da.deleted_at IS NULL;

-- semua baris attendance milik batch kalah ikut di-soft-delete
UPDATE attendance a
SET deleted_at = now(), updated_at = now()
FROM _remediasi_plan p
WHERE a.daily_attendance_id = p.batch_id
  AND p.decision = 'DROP'
  AND a.deleted_at IS NULL;
```

Setelah langkah ini, batch `DROP` tidak akan terlihat oleh query aplikasi mana pun (semua query
memakai `deleted_at IS NULL`).

### [5] Review manual

**Jangan jalankan cleanup otomatis pada grup ini.** Daftar grupnya:

```sql
SELECT p.class_year_id, p.d, p.decision,
       p.batch_id, p.st, p.rows_n, p.students_n,
       (SELECT string_agg(DISTINCT a.student_name, ', ')
        FROM attendance a WHERE a.daily_attendance_id = p.batch_id AND a.deleted_at IS NULL) AS siswa
FROM _remediasi_plan p
WHERE p.decision LIKE 'MANUAL%'
ORDER BY p.class_year_id, p.d, p.batch_id;
```

Dua kategori dan cara menanganinya:

**`MANUAL_NO_COMPLETE` (15 grup)** — tidak ada batch yang pernah di-finalisasi. Biasanya karena guru
belum menekan tombol selesai hari itu. Penanganan:
1. Bandingkan dengan `attendance_log` untuk tanggal itu (`GET /attendances?dailyAttendanceId=...`)
   dan dengan log gate scan.
2. Pilih batch yang isinya paling lengkap, ubah `daily_attendance_status = '2'` (COMPLETE) supaya
   konsisten dengan hari lain.
3. Soft-delete sisanya dengan cara yang sama seperti [4].

**`MANUAL_TIE` (20 grup)** — dua batch sama-sama COMPLETE dan sama-sama lengkap. Kasus nyata:
`100887 / 2026-10-14` dan `100887 / 2026-10-21`, masing-masing 18 siswa di dua batch COMPLETE.
Penanganan: pilih batch yang `updated_at`-nya paling akhir (yang terakhir di-finalisasi guru),
lalu soft-delete yang lain. **Jangan** merge dua batch COMPLETE karena isinya duplikat 100%.

Catat setiap keputusan manual di tabel `_remediasi_manual_log` supaya bisa diaudit:

```sql
CREATE TABLE _remediasi_manual_log (
  class_year_id int, d date, kept_batch int, dropped_batchs int[],
  decided_by text, decided_at timestamptz DEFAULT now(), note text
);
```

### [6] Recalc `student_attendance_report`

**Baru boleh dijalankan setelah langkah [2]–[5] selesai dan diverifikasi.** Recalc memakai
`COUNT(DISTINCT daily_attendance_id)` per status (`attendance.service.ts:2334-2337`). Setelah cleanup,
satu siswa per tanggal hanya muncul di satu `daily_attendance_id`, sehingga hitungan menjadi benar.

Cara amannya lewat endpoint aplikasi, bukan SQL langsung, supaya logika `unmarked` dan filter
`student_status = CURRENT` ikut jalan:

```
POST /api/v1/attendances/recalculate-student-attendance-report
{ "studentId": <id>, "classYearId": <id>, "term": 2 }
```

Untuk batch, iterasi semua siswa di 33 kelas terdampak (script Python di `sb_sql/` bisa dipakai ulang):

```sql
-- daftar target recalc
SELECT DISTINCT a.student_id, a.class_year_id
FROM attendance a
WHERE a.class_year_id IN (
  SELECT DISTINCT class_year_id FROM _remediasi_plan
)
AND a.deleted_at IS NULL
AND a.date BETWEEN '2026-10-05' AND '2026-12-10';
```

Sebelum recalc, catat kondisi awal snapshot untuk perbandingan:

```sql
SELECT * FROM student_attendance_report
WHERE academic_year_id = 27 AND term = 2
  AND class_year_id IN (SELECT DISTINCT class_year_id FROM _remediasi_plan)
ORDER BY class_year_id, student_id;
```

Perhatikan: `student_attendance_report` **tidak punya `deleted_at`**, dan hasil cek menunjukkan ada
**23 grup `(student_id, class_year_id, term)` dengan lebih dari satu baris snapshot**. Recalc akan
mengambil `findOne` tanpa urutan (`attendance.service.ts:2265-2271`), jadi baris mana yang di-update
tidak deterministik. Bersihkan duplikat snapshot dulu, simpan yang `date` paling akhir:

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

### [7] Verifikasi

Query-query ini harus lolos semua sebelum index dibuat:

```sql
-- 1. Tidak ada grup duplikat tersisa (harus 0)
SELECT COUNT(*) FROM (
  SELECT class_year_id, date FROM daily_attendance
  WHERE deleted_at IS NULL AND date BETWEEN '2026-10-05' AND '2026-12-10'
  GROUP BY 1,2 HAVING COUNT(*) > 1
) x;

-- 2. Tidak ada siswa dobel per tanggal (harus 0)
SELECT COUNT(*) FROM (
  SELECT student_id, class_year_id, date::date
  FROM attendance
  WHERE deleted_at IS NULL AND is_first_inserted = true
    AND date BETWEEN '2026-10-05' AND '2026-12-10'
  GROUP BY 1,2,3 HAVING COUNT(*) > 1
) x;

-- 3. Tidak ada siswa tersebar di >1 batch per tanggal (harus 0)
SELECT COUNT(*) FROM (
  SELECT student_id, date::date
  FROM attendance
  WHERE deleted_at IS NULL AND is_first_inserted = true
    AND date BETWEEN '2026-10-05' AND '2026-12-10'
  GROUP BY 1,2 HAVING COUNT(DISTINCT daily_attendance_id) > 1
) x;

-- 4. Kasus Gayathri: 2026-10-21 harus PRESENT dengan check-out terisi
SELECT a.id, a.attendance_status::text, a.check_in_date AT TIME ZONE 'Asia/Jakarta',
       a.check_out_date AT TIME ZONE 'Asia/Jakarta', a.daily_attendance_id
FROM attendance a
WHERE a.student_id = 26557 AND a.date::date = '2026-10-21' AND a.deleted_at IS NULL;

-- 5. Snapshot Gayathri term 2 tidak boleh late 3
SELECT present_count, late_count, absent_count, unmarked
FROM student_attendance_report
WHERE student_id = 26557 AND academic_year_id = 27 AND term = 2;
```

Cek #4 harus menghasilkan **tepat satu baris PRESENT dengan check-out `12:32:54`**. Cek #5 harus
menunjukkan `late_count` kembali ke angka sebelum insiden (recalc tidak boleh menaikkannya).

### [8] Kunci dengan unique index

Setelah verifikasi hijau, dan **setelah** fix code dari bug report (`createOrUseExistingDailyAttendance`
jadikan atomik) sudah siap deploy — urutan deploy code dulu, baru index. Kalau index dipasang sebelum
code fix, race yang sama akan berubah dari "membuat duplikat" menjadi "error 500 pada user".

```sql
CREATE UNIQUE INDEX CONCURRENTLY uq_daily_attendance_class_date
ON daily_attendance (class_year_id, date)
WHERE deleted_at IS NULL;
```

Catatan: `CONCURRENTLY` tidak bisa dijalankan di dalam transaksi. Jalankan statement ini terpisah.

---

## 5. Dampak yang Diharapkan

Setelah remediasi pada AY 2026/2027 term 2:

| Area | Sebelum | Sesudah |
|---|---|---|
| Grup duplikat | 76 | 0 |
| Baris `daily_attendance` di grup duplikat | 168 | 76 (satu per grup) |
| Siswa tercatat LATE dari batch duplikat | 18 pasangan `1,2` | 0 |
| Siswa hilang saat cleanup | — | 0 (11 siswa di-merge, bukan dihapus) |
| Present count siswa per hari | bisa 2x | 1x |
| Snapshot `student_attendance_report` | hari dobel terhitung | akurat |

Yang **tidak** berubah: data `attendance_log` (audit trail) dan `attendance` baris-baris asli —
semuanya tetap ada, hanya di-soft-delete atau dipindah batch. Tidak ada data implisit yang hilang.

---

## 6. Risiko & Mitigasi

| Risiko | Mitigasi |
|---|---|
| Salah memilih batch pemenang | Semua langkah reversibel via `deleted_at`; backup tabel `_bak_*` |
| Recalc jalan sebelum cleanup | Urutan langkah dikunci di §3; cek #1–#3 di §7 wajib lulus dulu |
| Snapshot duplikat membuat recalc tidak deterministik | Hapus duplikat snapshot di §4 [6] sebelum recalc |
| Unique index dipasang sebelum code fix deploy | Deploy code fix dulu, index terakhir (§4 [8]) |
| Rapor guru sudah terlanjur dicetak dari angka salah | Recalc mengubah data historis; komunikasikan ke akademik sebelum eksekusi |
| Data AY lain juga bermasalah | Di luar scope; audit terpisah (lihat §7) |

---

## 7. Di Luar Scope

- **AY 2025/2026 dan 2024/2025** juga memiliki duplikat (snapshot `student_attendance_report`
  untuk AY 26 dan 25 menunjukkan pola serupa). Tidak dibersihkan di sini karena rapor historis
  sudah dipakai; butuh keputusan akademik terpisah.
- **Fix kode** (unique constraint + advisory lock + dedup guard) ada di `bug-report.md`
  §Proposed Solution Options. Dokumen ini hanya data.

---

## 8. Checklist Eksekusi

- [ ] Backup `_bak_*` dibuat dan diverifikasi barisnya
- [ ] `_remediasi_plan` dibuat; `KEEP` tidak ada yang kosong per grup
- [ ] `_remediasi_lost_students` dicek (harus ~11 baris), lalu di-update, lalu dicek 0 baris
- [ ] `_remediasi_intra_dups` di-soft-delete
- [ ] Batch `DROP` + attendance-nya di-soft-delete
- [ ] 15 grup `MANUAL_NO_COMPLETE` diputuskan & dicatat
- [ ] 20 grup `MANUAL_TIE` diputuskan & dicatat
- [ ] Duplikat `student_attendance_report` dibersihkan
- [ ] Recalc dijalankan untuk semua siswa kelas terdampak
- [ ] Cek verifikasi §7 #1–#5 hijau semua
- [ ] Code fix deployed
- [ ] `uq_daily_attendance_class_date` dibuat
- [ ] Tabel `_bak_*` dihapus setelah masa observasi

---

## Notes

- Semua query di dokumen ini **belum dieksekusi**. Ini rancangan, bukan log eksekusi.
- Angka grup (76) dan baris (168) lebih tinggi dari bug report (73/162) karena data baru masuk dalam
  rentang audit setelah bug report disusun. Gunakan angka di dokumen ini sebagai baseline eksekusi.
- Definisi "batch pemenang": `daily_attendance_status` tertinggi, lalu jumlah siswa unik terbanyak,
  lalu jumlah baris terbanyak, lalu `id` terkecil. Ranking ini konsisten dengan §4 [1].
- Kategori `MANUAL_TIE` muncul karena peringkat siswa seri (mis. dua batch COMPLETE dengan 18 siswa
  identik), bukan karena data rusak — keduanya sah secara individual, yang salah adalah keberadaan dua
  batch untuk satu hari.
