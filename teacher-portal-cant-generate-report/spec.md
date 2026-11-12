---
feature: Teacher Portal Tidak Bisa Generate Report Siswa
slug: teacher-portal-cant-generate-report
status: draft
author: BBS Team
date: 2026-11-12
target_release: TBD
jam_url: https://jam.dev/c/71113a88-1a1f-48fc-9823-360b192c61cc
---

# Teacher Portal Tidak Bisa Generate Report Siswa

> **Status:** draft
> **Read-only:** Ya (analisis root cause, tidak ada perubahan kode)

## Ringkasan

Generate report siswa melalui teacher portal (`teacher.smartbag.binabangsaschool.com`) gagal padahal melalui admin portal berhasil untuk student yang sama (Kebon Jeruk Secondary, Secondary 2 Newton, AY 2026/2027). Penyebab: **gate kepemilikan homeroom teacher (`OnlyHomeroomTeacherAllowedToFillStudentReport`) aktif pada backend teacher portal (`bbs/api`) tetapi sudah dikomentari pada backend admin (`api_nest`)**.

## Rekaman Jam

- URL: [https://jam.dev/c/71113a88-1a1f-48fc-9823-360b192c61cc](https://jam.dev/c/71113a88-1a1f-48fc-9823-360b192c61cc)

## Aktual (Actual)

### Error yang muncul
`OnlyHomeroomTeacherAllowedToFillStudentReport` — reject saat generate report siswa.

### Alur yang terjadi di teacher portal
1. Frontend: `bbs/client-teacher/src/actions/fromApi.js` → `createReport()` → `makeApiRequestThunk(POST, /reports)`
2. Backend: `bbs/api/src/routers/reportRouter.ts` → `reportService.createReport({ callerUserId: req.user.id })`
3. Service: `bbs/api/src/services/report.service.ts:createReport()` — memvalidasi `teacher.id !== studentReport.classYear.homeroomTeacherId`, jika **tidak cocok** → melempar `OnlyHomeroomTeacherAllowedToFillStudentReport()`

### Path entry teacher portal
- `StudentDetailsReports.jsx` → `handlePreviewReport()` / `handlePreviewCambridgeReport()`
- `ClassReportViewer.jsx` → `resolveCambridgeReportWithTable()` → `fromApi.createReport()`

### Catatan dari Jam recording
- `POST /reports` justru **berhasil** (`201`) dalam rekaman; kegagalan terlihat di tahap **PDF preview/print** (`about:blank` setelah `window.open`) — ini adalah masalah terpisah dari gate homeroom teacher (lihat EC-04).

## Dibutuhkan (Expected)

- Teacher portal seharusnya bisa generate report untuk student (Secondary 2 Newton, Kebon Jeruk Secondary, AY 2026/2027) **seperti admin portal**.

### Perilaku yang diharapkan
- `POST /reports` → `bbs/api` → `createReport()` **harus** menghasilkan HTML report tanpa error gate homeroom teacher, selama pengguna sudah terautentikasi.
- Konsistensi perilaku dengan admin portal: keduanya via endpoint `POST /reports` yang sama secara konsep.

## Root Cause

### Perbedaan gate homeroom-teacher antara dua backend
| Backend | Lokasi | Gate homeroom-teacher |
|---------|--------|----------------------|
| `api_nest` (admin portal) | `api_nest/src/modules/report/report.service.ts:create()` | **Dikomentari** (baris 58–59) |
| `bbs/api` (teacher portal) | `bbs/api/src/services/report.service.ts:createReport()` | **Aktif** (baris 38–39) |

`api_nest` — `ReportController.create()` → `reportService.create(req.user.id, options)` — tidak mempertimbangkan role homeroom teacher, sehingga admin dapat generate.

`bbs/api` — `reportRouter.ts` → `reportService.createReport(...)` — memvalidasi bahwa user login adalah `homeroomTeacherId` dari classYear terkait. Guru yang bukan homeroom teacher kelas tersebut akan **ditolak** dengan error `OnlyHomeroomTeacherAllowedToFillStudentReport`.

### Alasan konsistensi office
- **Admin** adalah pengguna `api_nest`, di mana gate di-disable → generate berhasil.
- **Guru** adalah pengguna `bbs/api`, di mana gate aktif → generate gagal **ketika guru bukan `homeroomTeacherId` milik classYear** tersebut.

Jadi: bukan masalah API call, bukan masalah selisih role auth, tetapi **ketidak-konsistenan rule kepemilikan homeroom teacher antara dua service report**.

## Scope

### In Scope
- Dokumentasi akar masalah read-only dari kedua backend report
- Verifikasi alur frontend yang POST ke `/reports`

### Out of Scope
- Perbaikan kode
- Penyesuaian `homeroomTeacherId` di database
- Penyelesaian error PDF preview/print terpisah (EC-04)

## User Stories

### As a teacher
I want to generate a student report for my class (Secondary 2 Newton, Kebon Jeruk Secondary, AY 2026/2027)
So that I can preview/print the report just like the admin portal can

### As an administrator
I want teacher portal and admin portal to have consistent report generation behavior
So that there is no portal mismatch for the same student/class/term/report type

## Acceptance Criteria

- [ ] Dapat diidentifikasi dengan tepat lokasi bedanya: `bbs/api/src/services/report.service.ts` (gate aktif) vs `api_nest/src/modules/report/report.service.ts` (gate dikomentari)
- [ ] Dokumentasi alur panggilan: `fromApi.createReport` → `POST /reports` → `reportRouter` → `reportService.createReport`
- [ ] Dokumentasi alur admin: `POST /reports` → `ReportController.create` → `ReportService.create` (gate disable)
- [ ] Reproduksi error: guru yang bukan `homeroomTeacherId` classYear akan menerima `OnlyHomeroomTeacherAllowedToFillStudentReport`

## Impacted Files

| File | Lokasi | Peran |
|------|--------|-------|
| `bbs/api/src/services/report.service.ts` | `createReport()` | Gate aktif (penyebab utama) |
| `bbs/api/src/routers/reportRouter.ts` | Route `POST /` | Entri API teacher portal |
| `api_nest/src/modules/report/report.service.ts` | `create()` | Gate dikomentari |
| `api_nest/src/modules/report/report.controller.ts` | `POST /reports` | Entri API admin |
| `bbs/client-teacher/src/actions/fromApi.js` | `createReport()` thunk | POST ke `/reports` |
| `bbs/client-teacher/src/views/students/StudentDetails/StudentDetailsReports.jsx` | `handlePreviewReport()` / `handlePreviewCambridgeReport()` | UI preview report |
| `bbs/client-teacher/src/views/classrooms/viewer/ClassReportViewer.jsx` | `resolveCambridgeReportWithTable()` | UI preview laporan kelas |

## Error Handling

| Error | Condition | Backend |
|-------|-----------|---------|
| `OnlyHomeroomTeacherAllowedToFillStudentReport` | `teacher.id !== studentReport.classYear.homeroomTeacherId` | `bbs/api` |
| `NoTeacherFoundError` | `callerUserId` tidak ditemukan di DB | `bbs/api` |
| `NoStudentReportFoundError` | `studentReportId` tidak ditemukan | `bbs/api` / `api_nest` |
| `InvalidReportTermError` | Term bukan `[1,2,3,4]` | `bbs/api` / `api_nest` |

## Edge Cases

### EC-01: Teacher bukan homeroom teacher, admin bisa generate
| Opsi | Behavior |
|------|----------|
| (A) Hapus gate dari `bbs/api` (mirror `api_nest`) | Guru bisa generate semua class, konsisten admin |
| (B) Biarkan gate aktif, pastikan guru adalah `homeroomTeacherId` | Hanya homeroom teacher |
| (C) Biarkan gate aktif, tambahkan pengecualian role admin/coordinator | Guru non-homeroom tetap gagal |

**Decision:** _TBD_

### EC-02: Report sudah ada (term + reportType sama)
`api_nest` akan `update` report existing; `bbs/api` hanya `create` baru. Decision: _TBD_ (opsi A: sinkronkan update-existing).

### EC-03: Guru login tapi `Teacher` record tidak ditemukan
Error `NoTeacherFoundError` di `bbs/api`. Decision: _TBD_.

### EC-04: PDF preview gagal (`about:blank`) walau `POST /reports` = 201
Masalah **terpisah** dari gate homeroom teacher — terkait `generate(html)` / `htmlStringToPdf` / `window.open(url)` di `StudentDetailsReports.jsx`. Decision: _TBD_ (investigasi terpisah).

### EC-05: Admin bisa, guru juga bukan homeroom teacher dari classYear yang sama
Kedua portal tetap konsisten: admin berhasil (gate nonaktif), guru gagal (gate aktif). Pilihan: (A) nonaktifkan gate `bbs/api` atau (B) aktifkan gate `api_nest`. Decision: _TBD_.
