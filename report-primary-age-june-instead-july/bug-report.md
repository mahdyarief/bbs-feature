---
title: Report Primary — "Age on" terhitung per 1 June, seharusnya 1 July
status: open
severity: major
product: BBS LMS
portal: Admin | Teacher
author: OpenClaude
date: 2026-11-02
jam: -
---

# Report Primary — Age dihitung dari 1 June, seharusnya 1 July

## Summary

Pada report Primary (P1–P2), field **"Age on …"** menampilkan **1 June** dan umur siswa **1 bulan lebih muda** dari seharusnya. Bisnis mensyaratkan umur dihitung per **1 July** setiap academic year. Kesalahan ada pada cara sistem membentuk tanggal acuan umur di frontend — bug ini bergantung pada tanggal report di-generate, sehingga muncul/tidaknya tidak konsisten.

**Ekspektasi:** `Age on 1 July {year}` dengan umur sesuai 1 July (contoh: DOB 3 Feb 2025 → **1 years 4 months**, bukan `1 June 2026` → 1 years 3 months).

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Principal / Ms Lina (via email Alice Shajima Bright) |
| Email (Jam account) | - |
| Jam author ID | - |
| User ID (dari console log, mis. `selfUser`) | `ihsan` (employee 100188) untuk verifikasi API |
| Portal URL | Admin `client/`, Teacher `client-teacher/` — halaman Student Reports |
| Environment API | `https://api.binabangsaschool.dev` |
| Data konteks | Student `Kosaga Scrore` (ID `100215`), DOB `2025-02-03`, class `Primary 1 PIK TEST` (class_year `100024`), AY 2026/2027, studentReportId `15609` |
| Browser / OS | - |

---

## Steps to Reproduce

1. Login Admin/Teacher, buka Student Reports untuk P1–P2 (mis. siswa `Kosaga Scrore`, class `Primary 1 PIK TEST`, AY 2026/2027).
2. Generate / preview report card primary (semester).
3. Lihat header bagian **"Age on …"**.

**Actual Result:** `Age on 1 June 2026` — `1 years 3 months`.

**Expected Result:** `Age on 1 July 2026` — `1 years 4 months`.

> Catatan: hasil bisa berbeda tergantung **tanggal saat report di-generate**. Pada hari tertentu `set({month:6,...})` menghasilkan 29 June (lalu `startOf("month")` → 1 June); pada tanggal lain bisa kebetulan menghasilkan Juli. Lihat Root Cause & tabel bukti.

---

## Root Cause Analysis

### Bug #1 — `moment().set({ month: 6, ... })` tidak menghasilkan 1 July secara deterministik — `pdf-report-headers.js(x)` (line 25 & 54/59)

Kode header report menghitung tanggal acuan umur dengan:

```js
// path:25 — di semua 3 client (admin/student/teacher), 2 fungsi header masing-masing
const year = moment(term1StartDate).year();
const dateFrom = moment().set({ month: 6, year, day: 1 });   // niatnya 1 July
const age = studentAge(student?.dateOfBirth, dateFrom);
```

`studentAge` (`utilFunctions.js:513-527`) kemudian melakukan:

```js
const ageDate = moment(dateTarget).startOf("month").format("D MMMM YYYY");
```

Dua masalah berantai:

1. **`moment().set({ month: 6, year, day: 1 })` tidak selalu = 1 July.** `set()` menerapkan field pada objek "sekarang", dan dengan moment `2.31.0` hasilnya **29 June 2026** saat dijalankan pada 2 Nov 2026 (padahal `month: 6` = Juli, 0-indexed). Nilai tanggal dari `moment()` "sekarang" ikut terbawa/overflow sehingga tanggalnya bukan 1.
2. **`startOf("month")` menutupi error.** Karena `ageDate` di-`startOf("month")`, tanggal 29 June dipotong menjadi **1 June** — sehingga yang tampil "1 June", bukan error yang kelihatan.

Bukti runtime (node + moment 2.31.0, logika verbatim dari repo):

| "now"   | `.set({month:6,year:2026,day:1})` | setelah `.startOf("month")` | Umur (DOB 2025-02-03) |
|---------|-----------------------------------|-----------------------------|------------------------|
| 2026-11-02 | 2026-06-29 | **1 June 2026** (BUG) | 1 years 3 months |
| 2026-10-30 | 2026-07-27 | 1 July 2026 (kebetulan benar) | 1 years 4 months |

Alternatif konstruktor yang deterministik menghasilkan 1 July dengan benar:

| Ekspresi | Hasil |
|----------|-------|
| `moment([2026, 6, 1])` | 1 July 2026 |
| `moment({ year: 2026, month: 6, day: 1 })` | 1 July 2026 |

**Kesimpulan:** bug ini **aktif** (bukan artefak PDF lama). Kandidat bug saat ini adalah bagaimana `moment().set(...)` dibentuk — perlu diganti ke konstruktor yang eksplisit agar selalu 1 July.

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend (Admin) | `bbs/client/src/utils/pdf-generator/pdf-report-headers.jsx` (line 25, 54) | Age header salah bila di-generate pada tanggal tertentu |
| Frontend (Teacher) | `bbs/client-teacher/src/utils/pdf-generator/pdf-report-headers.js` (line 25, 59) | Sama |
| Frontend (Student) | `bbs/client-student/src/utils/pdf-generator/pdf-report-headers.jsx` (line 25, 54) | Sama |
| Frontend (shared util) | `bbs/client*/src/utils/utilFunctions.js` — `studentAge` (line 513-527) | `startOf("month")` menutupi tanggal yang salah |
| Backend | - | Tidak terlibat — header/age di-generate di frontend |

---

## Proposed Solution Options

### Option A: Pakai konstruktor moment eksplisit (Recommended)

Ganti semua 6 call site:

```js
const dateFrom = moment({ year, month: 6, day: 1 });   // 0-indexed → 1 July, deterministik
```

Tidak bergantung pada tanggal "sekarang" sama sekali.

### Option B: Pakai array constructor

```js
const dateFrom = moment([year, 6, 1]);                 // 1 July, deterministik
```

Hasil identik; pilih sesuai gaya kode tim.

---

## Notes

- **Bukan salah guru / data siswa.** Nilai, subject, dan remark tidak terpengaruh — hanya field "Age on".
- Setelah fix, report perlu **di-generate ulang** agar header terbarui.
- Definisi bisnis (dari email Alice Shajima Bright): *"For P1-P2: Age should be Age on 1 July and not 1 June."* — kode sudah berniat Juli, tetapi implementasinya tidak deterministik.
- Topik lain di email Alice (batas jumlah kata remarks P3–P6) **di luar scope** brief ini.
- Verifikasi API: `POST /api/v1/reports` untuk `studentReportId 15609` mengembalikan HTML body (tanpa header) — membuktikan header/age dibuat di frontend, konsisten dengan root cause di atas.
