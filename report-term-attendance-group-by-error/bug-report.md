---
title: Report Term (Primary LO) — Generate gagal 400 (Postgres 42803 GROUP BY attendance)
status: open
severity: critical
product: BBS LMS
portal: Admin
author: AI operate (operate-smartbag)
date: 2026-09-30
jam: # tidak ada recording — direproduksi via curl API (lihat Steps to Reproduce)
---

# Report Term gagal di-generate (HTTP 400, Postgres 42803) untuk kelas Primary mode Learning Outcome

## Summary

Generate **Report Term** dari Admin Portal (`/student-report`, class 100024 P1 AY 2026/2027)
untuk semua siswa menghasilkan **error 400** pada `POST /api/v1/reports/update`
(`reportType: "REPORT_TERM"`). Report type lain pada sesi yang sama — `FTP` — **sukses 201**
untuk 3 studentReport yang sama, jadi masalahnya spesifik di `REPORT_TERM`.

**Actual Result (evidence dari network log + reproduksi manual):**
```
POST /api/v1/reports/update  body {reportType:"REPORT_TERM",term:1,studentReportId:"15611"}
→ 400  {"statusCode":400,"code":"42803",
        "message":"column \"att.created_at\" must appear in the GROUP BY clause or be used in an aggregate function"}
POST /api/v1/reports/update  body {reportType:"FTP",term:1,studentReportId:"15611"}   → 201 ✅
```
Bukti tambahan — endpoint attendance-nya sendiri juga rusak, terlepas dari report:
```
GET /api/v1/attendances/student?studentId=100334&classYearId=100024&term=1
→ 400, error 42803 identik
```

**Ekspektasi:** `REPORT_TERM` ter-generate 201 seperti `FTP`; `GET /attendances/student` → 200.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | AI operate (dari laporan user via admin portal) |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | admin token id 35008 (ACCESS_TOKEN) |
| Portal URL | https://admin.smartbag.binabangsaschool.com/student-report?academicYearId=27&classYearId=100024&reports=FTP,REPORT_TERM&programmeId=58&term=1 |
| Environment API | api.binabangsaschool.dev (production) |
| Data konteks (class ID / daId / tanggal) | classYearId=100024 (P1, AY 2026/2027); studentReportId 15609/15610/15611 (student 100215/100312/100334); 2026-09-29 23:06 UTC |
| Database diperiksa | via `D:\Work\BBS\requirement\binabangsa-db-tools` |
| Browser / OS | Brave / Windows 11 |

---

## Steps to Reproduce

1. Login Admin Portal → Student Report → class 100024, AY 27, term 1, pilih semua siswa,
   centang report `FTP` + `REPORT_TERM` → Generate.
   (Atau langsung via API dengan token admin:)
2. `POST https://api.binabangsaschool.dev/api/v1/reports/update`
   ```json
   {"data":{"attributes":{"reportType":"REPORT_TERM","term":1,"studentReportId":"15611"}}}
   ```
3. Uji kontrol: body sama dengan `"reportType":"FTP"`.

**Actual Result:**
- `REPORT_TERM` → **400**, body 142 byte: `{"statusCode":400,"code":"42803","message":"column \"att.created_at\" must appear in the GROUP BY clause or be used in an aggregate function"}` — konsisten untuk ketiga studentReport (15609, 15610, 15611).
- `FTP` → **201**, report ter-generate normal.
- `GET /api/v1/attendances/student?studentId=100334&classYearId=100024&term=1` → **400 42803** (error identik).

**Expected Result:**
- `REPORT_TERM` → 201, report ter-generate.
- `GET /attendances/student` → 200.

---

## Root Cause Analysis

### Bug #1 (akar masalah) — query attendance di build produksi salah GROUP BY — `attendance.service.ts`

**Jalur error:**

1. `POST /reports/update` → `ReportService.create()` (`api_nest/src/modules/report/report.service.ts:47-156`).
   Bukan validasi DTO — enum `ReportTypeEnum.REPORT_TERM = 1` valid.
2. `createStudentReport()` (`api_nest/src/helpers/create-report.helper.ts:104-109`): untuk
   `REPORT_TERM`/`REPORT_SEMESTER` dia baca `grading_guide` level class. Level Primary
   (`master_level_id = 1`) → `grading_guide_type = '2'` = **LEARNING_OUTCOME**, sehingga
   `reportType` **di-rewrite** menjadi `LEARNING_OUTCOME` → memanggil `createLoReport()`.
   `FTP` tidak melewati cabang ini — makanya sukses.
3. `createLoReport()` (`api_nest/src/helpers/reports/lo-report.helper.ts:347-352`) memanggil
   `attendanceService.findByStudent(...)` untuk data tardiness/absence.
4. Query di `findByStudent` (`api_nest/src/modules/attendance/attendance.service.ts`)
   **gagal di Postgres dengan error 42803**: kolom `att.created_at` ter-SELECT pada query
   yang punya `GROUP BY` tetapi tidak ikut masuk GROUP BY-nya / aggregate.

**Bukti versi kode:**
- Fix **sudah ada di `develop`**: PR **#1961** `fix(report): fix tardiness on every report`
  (commit `10064127`, merge `9f95c14c`, 28 Sep 2026). Commit itu mengganti isi
  `findByStudent` (versi lama `findAndCount`) menjadi `countDistinctFirstInsertedDays()` +
  `findUniqueFirstInsertedDetails()` yang memakai `.distinctOn([...])` — **tanpa GROUP BY**.
- **Build produksi belum memuatnya.** Kandidat branch rilis `origin/26-09-28-reports-01`
  masih berisi versi lama (tidak ada `firstInsertedInRangeQuery`/`distinctOn`), sehingga
  `findByStudent` masih memicu 42803.
- `diff 10064127..HEAD` untuk `attendance.service.ts` = 0 baris → fix di HEAD sudah final;
  tinggal deploy.

---

## Bukti dari Reproduksi API (tanpa Jam)

| Sumber | Temuan |
|--------|--------|
| Network (browser user) | 3× POST `reports/update` FTP → 201 (79 byte), 3× REPORT_TERM → 400 (87 byte req / 142 byte resp) |
| Reproduksi curl | `REPORT_TERM` 15611 → 400 `code:"42803"`; `FTP` 15611 → 201 |
| Reproduksi curl | `GET /attendances/student?studentId=100334&classYearId=100024&term=1` → 400 `code:"42803"` sama persis |
| DB (db-tools) | `grading_guide master_level_id=1 → grading_guide_type='2'` (LEARNING_OUTCOME); `learning_outcome_year` AY27 L1 t1 = id 2731 ✅; `lo_grade_setting` AY27 L1 = id 9 ✅ — prereq LO lengkap, jadi bukan error data |
| DB (db-tools) | student_report 15609→100215, 15610→100312, 15611→100334, semua class 100024 |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/attendance/attendance.service.ts` (`findByStudent`) | query 42803 — sumber error utama |
| Backend Helper | `api_nest/src/helpers/reports/lo-report.helper.ts` (`createLoReport` L347-352) | pemanggil; semua Report Term/SEMESTER kelas LO terblokir |
| Backend Helper | `api_nest/src/helpers/create-report.helper.ts` (L104-109) | rewrite REPORT_TERM → LEARNING_OUTCOME (perilaku benar, bukan bug) |
| Frontend | Admin `StudentReport` page | menampilkan error generate; bukan akar masalah |

---

## Proposed Solution Options

### Option A: Deploy ulang backend dari `develop` (Recommended)

`develop` sudah memuat PR #1961 yang mengganti query bermasalah dengan `distinctOn`
tanpa GROUP BY. Cek SHA build produksi → merge/cherry-pick `10064127` ke branch rilis
yang dipakai (kandidat `26-09-28-reports-01`) → redeploy.

### Option B: Hotfix query attendance di branch rilis

Kalau tidak bisa redeploy penuh, cherry-pick `10064127` saja ke branch rilis lalu
hotfix backend. Praktis sama dengan Option A, hanya cakupan deploy yang lebih sempit.

---

## Yang diminta / Verifikasi setelah fix

1. `GET /api/v1/attendances/student?studentId=100334&classYearId=100024&term=1` → **200**
2. `POST /api/v1/reports/update` `reportType:"REPORT_TERM"` (15611) → **201**
3. Generate Report Term untuk 3 siswa class 100024 (15609/15610/15611) via UI → sukses

---

## Dampak

- **Seluruh fitur Report Term (Primary / mode LEARNING_OUTCOME) tidak bisa di-generate sama sekali** sampai build produksi diperbarui.
- Kelas terdampak: class 100024 saat ini, tapi berlaku umum untuk semua kelas mode LO.
- Tidak ada data yang perlu diperbaiki — nilai (117 baris `learning_indicator_grade`), mapping LO, dan student_report sudah benar. Ini murni bug kode/deploy.

---

## Notes

- Diagnosa dilakukan dari `D:\Work\BBS\requirement\operate-smartbag` (action 06 + investigasi report).
- FTP tetap dipakai sebagai kontrol di reproduksi karena tidak melewati jalur LO.
- Bug terkait (belum dibuat terpisah): 5 subject P1 (112393, 112394, 112397, 112401, 112403)
  tidak punya learning indicator di AY27 karena outcome tidak ter-link ke LO year
  (link ke AY24; LO year AY27 id 2731/2737 campus NULL) — kemungkinan perlu fix data/master.
