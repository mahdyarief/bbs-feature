---
feature: FTP Evaluation — Skip Terms Without Conduct Data (Mid-Year Entry)
slug: ftp-evaluation-skip-empty-term
status: draft
author: BBS Team
date: 2026-08-10
target_release: TBD
---

# FTP Evaluation — Skip Terms Without Conduct Data (Mid-Year Entry)

## Overview

Enhancement pada kalkulasi **FTP (E-Badges) Evaluation score** agar term yang **tidak memiliki data conduct sama sekali** (0 rows di `student_ftp_conduct`) tidak dihitung dan tidak dibuatkan evaluation row-nya. Saat ini, student yang masuk di term 3 (mid-year entry) tetap mendapat evaluation row untuk term 1 & 2 dengan score yang **dihitung dari data conduct term 3/4** — hasilnya salah secara semantik dan menyesatkan di report.

## Problem / Motivation

- Student yang masuk di **term 3** (misal student id 103033, academic year 2025/2026) **tidak punya data conduct term 1/2 sama sekali** (0 rows), tapi sistem **tetap membuat evaluation row term 1 & 2** dengan score 21 dan 22.
- Root cause: `ftpEvaluation` (`api_nest/src/helpers/reports/ftp-evaluation.ts:62-64`) memfilter `studentFtpConducts` **hanya oleh `ftpReportTableId`**, bukan oleh term. Karena `groupByTerm` hanya meng-iterasi term yang punya rows, score term 1/2 dihitung dari conduct term 3/4 yang table-nya cocok dengan setting term 1/2.
- Score tersebut **bukan 0** — bukan pula "badges 0 → +1 point" — melainkan akumulasi point dari data term lain yang secara semantik salah.
- Evaluation row yang sudah terlanjur dibuat akan **terus di-recompute** setiap generate report karena `termsToEvaluate = unique([...existingEvalTerms, term])` (`api_nest/src/helpers/reports/ftp-report.helper.ts:172-174`).
- `syncFtpEvaluation` (`api_nest/src/modules/student-ftp-conduct/v2/student-ftp-conduct-v2.service.ts:111-128`) melakukan batch-generate untuk **semua** student di class year tanpa cek keberadaan data per student — memperluas dampak ke seluruh kelas.
- Tidak ada konsep "term masuk" (entry term) di skema: `student_report`, `student_ftp_conduct`, dan `student_ftp_conduct_evaluation` tidak memiliki field tanggal/term masuk.

## Scope

### In Scope
- **Titik kalkulasi** `ftpEvaluation` (`api_nest/src/helpers/reports/ftp-evaluation.ts`): term tanpa conduct data tidak boleh menyumbang point.
- **Pembentukan daftar term evaluasi** di `createStudentFtpReport` (`api_nest/src/helpers/reports/ftp-report.helper.ts`): `termsToEvaluate` harus dibatasi hanya pada term yang benar-benar punya data conduct untuk student tsb.
- **Batch evaluasi** `syncFtpEvaluation` (`api_nest/src/modules/student-ftp-conduct/v2/student-ftp-conduct-v2.service.ts`): tidak membuat evaluation row untuk student yang tidak punya conduct data pada term yang di-sync.
- **Pendefinisian "term kosong"**: term yang tidak memiliki **row `student_ftp_conduct` sama sekali** untuk student_report tsb (bukan badges = 0).
- Menyertakan data trace dari database production (lihat `notes.md`) agar engineer dapat memverifikasi sebelum/sesudah implementasi.

### Out of Scope
- Perubahan penamaan FTP → E-Badges (isu terpisah, sudah di-approve sebagai perbedaan istilah teknis vs user-facing).
- Normalisasi/proporsionalisasi threshold mark (≥42 EXCEEDING / ≥25 MEETING) untuk student partial-year — butuh keputusan bisnis terpisah.
- Penambahan field `entry_term` / tanggal masuk di `student_report` (opsi alternatif, belum diputuskan — lihat notes.md).
- Perubahan mekanisme input grading FTP di frontend (`FTPModalGrading.jsx`).
- Hapus/cleanup data evaluation row term 1/2 yang sudah terlanjur salah di production (perlu keputusan terpisah, lihat notes.md).

## User Stories

### As a homeroom teacher
I want FTP evaluation for mid-year entry students (e.g. joining in term 3) to only count terms where the student actually has badge data
So that the evaluation score and report reflect the student's real period of attendance.

### As a school administrator
I want FTP evaluation reports to not show fabricated Term 1/Term 2 scores for students who were not yet enrolled
So that reports sent to parents are accurate and defensible.

### As an engineer
I want the empty-term rule to be deterministic and traceable against production data
So that I can verify the fix with the documented student case before and after implementation.

## Acceptance Criteria

- [ ] Student yang **tidak punya row `student_ftp_conduct`** untuk suatu term → term tersebut **tidak ikut dihitung** di `ftpEvaluation` (tidak menambah point ke score).
- [ ] Student yang **tidak punya row conduct term 1/2** → `termsToEvaluate` di `createStudentFtpReport` **tidak memasukkan term 1/2**; evaluation row term 1/2 **tidak dibuat** (atau di-skip saat render jika sudah ada).
- [ ] `syncFtpEvaluation` untuk suatu term → student tanpa conduct data di term tsb **tidak dibuatkan evaluation row**.
- [ ] Verifikasi dengan data produksi student 103033 (student_report 3758, academic year 2025/2026): setelah fix, evaluation row term 1/2 **tidak muncul** di report FTP Evaluation (lihat `notes.md` untuk data trace).
- [ ] Student full-year dengan data conduct lengkap term 1-4 → score dan evaluation row **tidak berubah** (regression check).
- [ ] Student yang punya conduct rows di semua Character/Traits/Values dengan badges 0 (form disimpan kosong) — **tetap dihitung sebagai point minimum** (lihat EC-02 di `edgecases.md`): hanya term yang **tidak punya row sama sekali (null)** yang diabaikan.
- [ ] Tidak ada perubahan API contract dan tidak ada perubahan skema database wajib.

## UI / UX Changes

Tidak ada perubahan UI. Perubahan hanya pada logika kalkulasi dan isi report FTP Evaluation (PDF).

### Affected Portals
- [ ] Admin (client/)
- [ ] Student (client-student/)
- [ ] Teacher (client-teacher/)

(No portal changes — backend report generation only)

## API Changes

Tidak ada endpoint baru.

| Method | Path | Description |
|--------|------|-------------|
| — | — | Tidak ada perubahan API. Perubahan internal pada `ftpEvaluation`, `createStudentFtpReport`, dan `syncFtpEvaluation` |

## Database Changes

### New Tables
- None

### Modified Tables
- None (opsional: jika keputusan bisnis memilih field `entry_term`, lihat `notes.md`)

### Migrations
- Tidak wajib. Solusi utama berbasis data existing (`student_ftp_conduct.term` + `student_report_id`).

## Business Rules / Validation

1. **Definisi term kosong (null = diabaikan)**: term dianggap kosong jika **tidak ada row** `student_ftp_conduct` dengan `student_report_id` yang bersangkutan pada term tersebut. Term tanpa rows → 0 kontribusi ke score, dan tidak termasuk dalam `termsToEvaluate`.
2. **Row ada dengan badges 0/null = tetap dihitung**: row `student_ftp_conduct` yang ADA di term tersebut, walaupun badges-nya 0/null di **semua** Character/Traits/Values, **tetap dihitung** sebagai point minimum (`getConductPoint(0)` → 1 per (table, term)). Hanya **ketiadaan row (null)** yang diabaikan — bukan badges 0. (EC-02, keputusan: opsi B)
3. **Filter evaluasi**: `filteredFtpConducts` di `ftpEvaluation` tetap memfilter `ftpReportTableId`, namun iterasi `groupByTerm` harus memastikan hanya term yang valid untuk student yang berkontribusi point.
4. **Daftar term evaluasi**: `termsToEvaluate = unique([...existingEvalTerms, term])` harus di-filter terhadap term yang punya conduct data (atau term `>= minTerm`, dengan `minTerm` = term terkecil yang punya conduct rows untuk student tsb).
5. **Batch sync**: `syncFtpEvaluation` hanya memanggil generate untuk student yang punya minimal satu conduct row pada term yang di-sync (atau meneruskan rule yang sama ke `createStudentFtpReport`).
6. **Threshold mark tidak berubah**: ≥42 EXCEEDING, ≥25 MEETING, else BELOW — tetap berlaku untuk student dengan data lengkap; normalisasi untuk partial-year di luar scope brief ini.

## Error Handling

| Error | Condition | Behavior |
|-------|-----------|----------|
| Student tanpa conduct data sama sekali di semua term | `studentFtpConducts` kosong (tidak ada row di term manapun) | Tidak ada evaluation row yang dibuat; report FTP Evaluation menampilkan kosong (atau skenario TBD — lihat EC-05) |
| Term tanpa rows sama sekali (null) | Tidak ada row `student_ftp_conduct` untuk term tsb | Term diabaikan — tidak menyumbang point, tidak dibuat eval row |
| Term punya rows tapi badges 0/null | Row ada, `totalBadges = 0` di semua table evaluasi | **Tetap dihitung** point minimum 1 per (table, term) — bukan di-skip (EC-02, keputusan: opsi B) |
| Eval row term invalid sudah ada di DB | Row `student_ftp_conduct_evaluation` untuk term tanpa conduct data | Rows tetap di DB; **nilai akan diperbarui (recompute) saat report di-update** setelah implementasi kalkulasi baru — term tanpa conduct data tidak lagi dihitung (EC-03, keputusan: opsi C; tidak ada script cleanup terpisah) |
| `ftpGradeSetting` tidak ditemukan untuk term | Query `FtpGradeSetting` return null | `getConductPoint` return 0 (perilaku existing, tidak berubah) |

## Dependencies

- **FTP evaluation calculation**: `api_nest/src/helpers/reports/ftp-evaluation.ts` — `getConductPoint` (line 11-27), filter (line 62-64), `groupByTerm` loop (line 66-83)
- **FTP report orchestration**: `api_nest/src/helpers/reports/ftp-report.helper.ts` — `termsToEvaluate` (line 165-194)
- **Batch evaluation sync**: `api_nest/src/modules/student-ftp-conduct/v2/student-ftp-conduct-v2.service.ts` — `syncFtpEvaluation` (line 111-128)
- **FTP evaluation report render**: `api_nest/src/helpers/reports/ftp-evaluation-report.helper.ts` — `createStudentFtpEvaluationReport` (menampilkan semua eval rows per student_report)
- **Entities**: `student_ftp_conduct.entity.ts`, `student_ftp_conduct_evaluation.entity.ts`, `ftp_evaluation_setting.entity.ts`, `ftp_grade_setting.entity.ts`
- **Data trace (production)**: lihat `notes.md` untuk query dan hasil verifikasi student 103033
