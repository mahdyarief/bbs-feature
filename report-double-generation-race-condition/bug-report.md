---
title: Class Report (Combined) — Report Tergenerate Ganda akibat Race Condition pada ReportService.create()
status: open
severity: critical
product: BBS LMS
portal: Teacher
author: System Analyst
date: 2026-09-22
jam: (tidak tersedia — laporan berbasis investigasi DB + code review)
---

# Class Report (Combined) — Report Tergenerate Ganda (Double Report) di Teacher Portal

## Summary

Pada halaman **Class Report combined** di Teacher Portal (`/classrooms/:classYearId/:term/combined`),
sebagian student menampilkan **laporan yang sama dua kali** (halaman Progress Report tercetak dobel di PDF
combined). Kasus yang dilaporkan: student **Hino Tantono**, class **Primary 4 Self-Control** (Pantai Indah
Kapuk Primary), Academic Year **2026/2027**, `classYearId = 100901`, `term = 1`.

Investigasi menemukan akar masalahnya di **data**, bukan di template PDF: tabel `report` menyimpan
**baris ganda** untuk kombinasi logis `(student_report_id, report_type, term)`. Secara global ada
**177 grup duplikat (180 baris redundan)**; **54 baris di antaranya masih aktif** (belum soft-deleted).
Duplikat dibuat oleh **race condition** pada `ReportService.create()` yang memakai pola
*check-then-insert* non-atomik tanpa unique constraint di level DB.

**Ekspektasi:** satu student report untuk satu `(reportType, term)` hanya boleh memiliki **satu** baris
`report`. Halaman combined harus menampilkan laporan tersebut **tepat satu kali**.

> Catatan: kondisi class 100901 (kasus Hino) **saat ini bersih** di DB — tidak ditemukan baris duplikat
> aktif. Kemungkinan duplikat untuk kelas ini sudah dibersihkan/di-regenerate sejak laporan, atau muncul
> kembali saat proses preview/regenerate memanggil `createReport`. Duplikat aktif yang terverifikasi
> berada di class_year **100919** dan **100922** (lihat bagian "Kelas Terdampak") — dua kelas ini bisa
> dipakai sebagai reproduksi konkret.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | — (laporan dari user) |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | — |
| Portal URL | https://teacher.smartbag.binabangsaschool.com/classrooms/100901/1/combined |
| Environment API | api.binabangsaschool.com (production) |
| Data konteks (class ID / daId / tanggal) | classYearId=100901 (classroom 1021 "Primary 4 Self-Control", academic_year 27 = 2026/2027, master_level 4 = Primary 4); student: Hino Tantono (student_id=24205, NIS 222040146); student_report class 100901 id=4625 |
| Database diperiksa | `binabangsa_prod_mig_v01` via `D:\Work\BBS\requirement\binabangsa-db-tools` |
| Browser / OS | — |

---

## Steps to Reproduce

Reproduksi paling andal menggunakan kelas yang **saat ini punya duplikat aktif** (bukan 100901):

1. Buka Teacher Portal, halaman combined classroom **100919** (Primary 3, AY 2026/2027) term 1:
   `/classrooms/100919/1/combined`.
2. Amati laporan student **Panggih Dewi Chandrika** (student_id 103409) — halaman Progress Report
   (`REPORT_TERM`, term 1) muncul **dua kali** di PDF combined.
3. Alternatif: kelas **100922** (Primary 6, AY 2026/2027) term 1, student **MICHAEL JEREMIAH XAVIER TJIA**
   (student_id 24916) — gejala sama.
4. Verifikasi di DB bahwa student tersebut punya 2 baris `report` untuk key yang sama (lihat query di
   bagian "Query Verifikasi").

**Actual Result:**
- PDF combined menampilkan halaman laporan student yang sama **dua kali** untuk student yang baris
  `report`-nya terduplikasi.
- Tabel `report` menyimpan 2 (atau lebih) baris dengan `student_report_id`, `report_type`, dan `term`
  identik — dua baris pada contoh term 1 dibuat hanya berselisih **1,8 ms** dan **6,9 ms**.

**Expected Result:**
- Satu baris `report` per `(student_report_id, report_type, term)`.
- Halaman combined menampilkan laporan tepat sekali.

---

## Root Cause Analysis

### Bug #1 (akar masalah utama) — `ReportService.create()` non-atomik + tidak ada unique constraint

File: `api_nest/src/modules/report/report.service.ts` (method `create()`, sekitar baris 26-155).

```ts
const isReportExist = await Report.findOneBy({
  studentReport: { id: studentReportId },
  reportType: ReportTypeEnum[options.reportType],
  term: options.term,
});
if (isReportExist) {
  return await this.update(isReportExist.id, callerUserId, { html, weightedAverage });
}
const report = new Report();
report.studentReport = studentReport;
report.term = options.term;
report.reportType = ReportTypeEnum[options.reportType];
...
await report.save();
```

Ini pola **check-then-insert** (TOCTOU). Bila dua request konkuren menyasar key yang sama, keduanya
sama-sama membaca "belum ada baris" lalu keduanya `INSERT` → baris ganda.

Tabel `report` **tidak punya unique constraint** pada `(student_report_id, report_type, term)`.
Index yang ada hanya: primary key `id`, index biasa `student_report_id`, `created_by_teacher_id`,
`updated_by_teacher_id`, plus foreign key. Jadi tidak ada proteksi di level DB.

Kedua endpoint memakai method ini:
- `POST /v1/reports` (dipakai tombol "Combined Report" / preview di client)
- `POST /v1/reports/update` (dipakai tool bulk `update_report.py`)

**Bukti race condition:** pada satu grup duplikat (`student_report_id = 15526`, `REPORT_TERM`, term 1),
dua baris dibuat pada `created_at` yang identik — `2026-09-17 10:37:37.775742`. Pada grup 100919 dan
100922, selisih `created_at` hanya 1,8 ms dan 6,9 ms. Dua INSERT dalam rentang milidetik hanya terjadi
pada request konkuren.

**Pemicu konkurensi yang paling mungkin:** tool bulk
`D:\Work\BBS\requirement\binabangsa-db-tools\update_report\update_report.py`
(config: `dry_run: false`, `concurrency: 3`, `academic_year_id: 27`, `term: 1`, `campus_ids: 3`,
`master_levels: PRIMARY1-6`) yang memanggil `POST /reports/update` secara paralel — persis skenario
PIK Primary AY 2026/2027 term 1. Report Hino (id 52845, REPORT_TERM term 1) tercatat terakhir
di-`update` pada **2026-09-22 05:22:18** (hari yang sama dengan laporan ini).

### Bug #2 (penguat) — duplikat DB bocor ke tampilan sebagai halaman ganda

File: `api_nest/src/helpers/transform-response.helpers.ts` (`flattenData`, sekitar baris 149).

`flattenData` mengubah relasi array menjadi field `xxxIds` **tanpa dedupe**:

```ts
datumAny[key + (ifArray ? 'Ids' : 'Id')] = ifArray
  ? datumAny[key].map((d: any) => d?.id?.toString())
  : datumAny[key]?.id?.toString();
```

Sehingga `studentReport.reports` = `[reportA, reportA]` menjadi `reportsIds = ["<id>", "<id>"]`.

Di frontend, `bbs/client-teacher/src/views/classrooms/viewer/ClassReportViewer.jsx`:

```js
const rawReports = resourceMapper("reports", studentReport?.reportsIds || []);
```

`resourceMapper` (`utils/resourceMapper.js`) dan `useResourceMapper` (`hooks/useResourceMapper.js`)
**tidak melakukan dedupe** — keduanya memakai `ids.map((id) => reducer[id])`. Akibatnya laporan yang
sama dimuat dua kali ke `sortedReports` → `holisticReports` → `renderHolisticBlob` menggabungkan
`flatMapDeep(segmentReports, "html")` sehingga HTML laporan yang sama di-*join* dua kali →
**halaman laporan tampil dobel di PDF combined**.

Loop kompilasi (`ClassReportViewer.jsx` baris ~470) juga membuat **tepat satu blob PDF per entri**
`groupingStudentWithStudentReport`, dan tidak ada dedupe pada `students`/`reportsIds`. Jadi duplikat
upstream langsung terlihat sebagai halaman ganda.

### Bug #3 (sekunder) — duplikat tabel `student_report`

Tabel `student_report` juga **tidak punya unique constraint** pada `(student_id, class_year_id)`.
Ditemukan **170 grup duplikat (189 baris redundan)**. Contoh nyata: student **21440** punya **dua**
`student_report` untuk class_year 100068 (id 2562 dan 3755). Semua 189 baris ini saat ini sudah
soft-deleted, sehingga tidak muncul di UI, tetapi bugnya sejenis dan bisa terulang.

---

## Bukti dari Database

Diambil dari `binabangsa_prod_mig_v01`:

| Metrik | Nilai |
|--------|-------|
| Total baris `report` | 48.986 |
| Baris `report` soft-deleted | 634 |
| Duplikat `report` (semua baris) | **177 grup / 180 baris redundan** |
| Duplikat `report` (**aktif**) | **54 grup / 54 baris** |
| Distribusi duplikat aktif `report` | type 7 (LEARNING_OUTCOME) term 4 = 27, term 2 = 11, term 3 = 7, term 1 = 7; type 1 (REPORT_TERM) term 1 = 2 |
| Duplikat `student_report` (semua baris) | 170 grup / 189 baris |
| Duplikat `student_report` (aktif) | 0 grup |
| Unique constraint `report (student_report_id, report_type, term)` | **tidak ada** |
| Unique constraint `student_report (student_id, class_year_id)` | **tidak ada** |

**Contoh grup duplikat aktif:**

| student_report_id | student | class_year | report_type | term | report ids | selisih created_at |
|---|---|---|---|---|---|---|
| 15526 (setara) | Panggih Dewi Chandrika | 100919 (Primary 3, AY 27) | REPORT_TERM (1) | 1 | 57286, 57287 | 1,8 ms |
| — | MICHAEL JEREMIAH XAVIER TJIA | 100922 (Primary 6, AY 27) | REPORT_TERM (1) | 1 | 57262, 57263 | 6,9 ms |

---

## Kelas Terdampak (duplikat aktif)

**Yang berdampak ke tampilan combined (REPORT_TERM, term 1)** — AY 2026/2027:
- `class_year 100919` — Primary 3 — student 103409 Panggih Dewi Chandrika
- `class_year 100922` — Primary 6 — student 24916 MICHAEL JEREMIAH XAVIER TJIA

**LEARNING_OUTCOME (tidak dirender di viewer combined, tetapi tetap data ganda)**:
- AY 2025/2026: class_year 100051, 100055, 100056, 100057, 100060, 100061, 100062, 100079, 100085, 100086, 100096, 100097, 100102, 100103
- AY 2026/2027: class_year 100914, 100918, 100927, 100938

**Catatan penting:** `class_year 100901` (kasus Hino) **tidak** termasuk dalam daftar duplikat aktif saat
ini.

---

## Dampak

- Dokumen resmi (report card / Progress Report) yang dibagikan ke orang tua dapat tampil **dobel** —
  menurunkan kepercayaan dan membingungkan penerima.
- Data ganda di produksi (180 baris) dan bertambah seiring setiap regenerate/preview konkuren.
- Tidak ada proteksi DB, sehingga bug bisa terulang pada periode/kelas berikutnya.

---

## Rekomendasi Perbaikan

1. **Database (paling penting):** tambahkan unique constraint/index pada:
   - `report (student_report_id, report_type, term)`
   - `student_report (student_id, class_year_id)`
2. **Backend:** ubah `ReportService.create()` menjadi **atomic upsert** (`INSERT ... ON CONFLICT
   (student_report_id, report_type, term) DO UPDATE`) agar bebas race. Setelah constraint ada, pola
   `findOneBy` → insert tetap rawan, jadi wajib upsert.
3. **Serializer (pertahanan berlapis):** tambahkan dedupe pada `flattenData` saat membentuk `xxxIds`.
4. **Frontend (pertahanan berlapis):** dedupe `reportsIds` di `ClassReportViewer` sebelum render
   (mis. `[...new Set(reportsIds)]`), dan dedupe `students` list.
5. **Bersihkan data existing:** jalankan script `binabangsa-db-tools/cleanup_duplicate_reports.py`
   (soft-delete 54 baris `report` duplikat aktif; backup CSV otomatis ke `result/`).
6. **Tool bulk `update_report.py`:** pertimbangkan idempotensi di sisi API (atan cara upsert) agar
   concurrency tidak lagi memicu duplikat.

---

## Query Verifikasi

```sql
-- duplikat global (semua baris termasuk soft-deleted)
SELECT student_report_id, report_type, term, COUNT(*) AS n
FROM report
GROUP BY 1,2,3 HAVING COUNT(*) > 1 ORDER BY n DESC;

-- duplikat aktif
SELECT student_report_id, report_type, term, COUNT(*) AS n
FROM report
WHERE deleted_at IS NULL
GROUP BY 1,2,3 HAVING COUNT(*) > 1 ORDER BY n DESC;

-- duplikat khusus class 100901
SELECT r.student_report_id, r.report_type, r.term, COUNT(*) AS n
FROM report r JOIN student_report sr ON sr.id = r.student_report_id
WHERE sr.class_year_id = 100901
GROUP BY 1,2,3 HAVING COUNT(*) > 1;

-- duplikat student_report
SELECT student_id, class_year_id, COUNT(*) AS n
FROM student_report
GROUP BY 1,2 HAVING COUNT(*) > 1;
```

Script cleanup tersedia di:
`D:\Work\BBS\requirement\binabangsa-db-tools\cleanup_duplicate_reports.py`

```bash
cd D:\Work\BBS\requirement\binabangsa-db-tools
python cleanup_duplicate_reports.py            # dry-run (read-only)
python cleanup_duplicate_reports.py --execute  # soft-delete duplikat aktif (dengan backup CSV)
```

---

## Notes

- Enum `report_type`: 1=REPORT_TERM, 2=REPORT_SEMESTER, 3=LEAPS, 4=SED, 5=SPR, 6=FTP,
  7=LEARNING_OUTCOME, 8=FTP_EVALUATION, 9-11=CAMBRIDGE_*, 12=GPA.
- Di viewer combined, `LEARNING_OUTCOME` difilter keluar (`ClassReportViewer.jsx:347`), sehingga
  duplikat tipe 7 tidak muncul di combined. Yang berdampak ke combined adalah duplikat pada
  REPORT_TERM / REPORT_SEMESTER / FTP_EVALUATION / LEAPS.
- Untuk kasus Hino (class 100901) yang dilaporkan: perlu verifikasi ulang live. Bila masih tampil
  dobel, cek apakah baris `report` untuk student_report 4625 kembali terduplikasi setelah regenerate.
