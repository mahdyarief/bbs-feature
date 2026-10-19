---
feature: National Report Card — Nomor sequence per Sekolah (se-KJ) per AY
slug: national-report-sequence
status: draft
author: OpenClaude
date: 2026-10-19
target_release: TBD
---

# National Report Card — Penomoran Nomor Report (se-KJ/AY, urut Abjad)
**Context:** skrip-generasi Surat Keterangan Lulus (SKL) + Transkrip Nilai per siswa, bagian dari National Report Card (`features/national-report/`).

## Overview
Di legacy, report nasional (SKL + Transkrip) dibangkitkan per class, dan setiap siswa mendapat nomor laporan dengan format:
- SKL: `NNN/CJCE/KJ/2025`
- Transkrip: `NNN/TN/SMA/KJ/2025`

Terkait `nomor report` yang muncul di bawah text "Surat Keterangan Lulus." Nomor itu di-generate seq sesuai urutan siswa.

## Problem / Motivation
Dari PDF referensi (`nasional-report-card-jc2-mendel-ay2024-2025.pdf`, AY 2024/2025, sekolah SMA Bina Bangsa School Kebon Jeruk / KJ-S):
1. **Nomor lompat antar siswa**, bukan urut per class: 002, 004, 005, 009, 010, 011, 013, 014, ... 033.
2. **Urutan siswa = urutan abjad nama** (mulai nama depan), contoh: Audrey Tiara Gozali 002, Calvin Mateo Susanto 004, ... Velove Maria 033.
3. Nomor di SKL dan Transkrip **sama** untuk siswa yang sama: 002/CJCE/KJ/2025 (SKL) = 002/TN/SMA/KJ/2025 (Transkrip).
4. Ada **data test**: "Sample Student S3A Taylor" dengan nomor 41/CJCE/KJ/2025 — bukan bagian dari 21 siswa yang valid; bukan bagian dari sequence yang digunakan.
5. Sequence **tidak dimulai dari 001** di setiap kelas. Nomor bersifat global per sekolah per AY (linier), sesuai urutan abjad siswa.

## Scope
### In Scope
- Dokumentasi format nomor: `SKL NNN/CJCE/KJ/YYYY`, `TN NNN/TN/SMA/KJ/YYYY`.
- Dokumentasi urutan: berdasarkan abjad nama siswa, global per sekolah per AY.
- Pengembangan function sequence generator (standalone, idempotent).
- Persist nomor per siswa per AY (agar stabil kalau data siswa berubah / siswa pindah class).
- Catatan: data test "Sample Student S3A Taylor" (nomor 41) dianggap **data test**, tidak dimasukkan ke sequence produksi.
- Reference document (SKL 21 siswa + format + example) sebagai panduan implementasi.
### Out of Scope
- Implementasi lengkap generate SKL/Transkrip PDF (sudah ada `national-report` feature — ini hanya sequence nomor).
- Perubahan ke format legacy yang sudah ada.
- Menyentuh `report_indo_nas_new.php` (file legacy unchanged).

## User Stories
### As an Admin (Class in Year → National Report)
I want each student to have a unique, stable report number (sequence by abjad nama)
So that setiap SKL/Transkrip bisa dikenali secara unik dan tidak bergantung pada urutan generate yang berubah-ubah.

### As a Wakil Kurikulum
I want nomor SKL & Transkrip terstruktur: `NNN/CJCE/KJ/2025` (SKL) and `NNN/TN/SMA/KJ/2025` (TN)
So that dokumen resmi cocok dengan template dan dapat diverifikasi.

## Acceptance Criteria
- [ ] Function generator nomor menghasilkan sequence `002, 004, 005, 009, 010, ...` (3 digit zero-padded) per sekolah per AY.
- [ ] Urutan siswa = abjad nama (kapitalisasi normal: "audrey" = "Audrey", "velove" = "Velove").
- [ ] Nomor ditambah **persisted** di database (tabel baru atau kolom di student) & idempotent: jika generate ulang dengan data siswa sama, nomor tetap sama.
- [ ] Siswa dengan data test ("Sample Student S3A Taylor") **tidak** ikut sequence.
- [ ] Nomor SKL dan Nomor TN sama untuk siswa yang sama dalam satu AY (format berbeda).
- [ ] Schema & migration ditambahkan (drop minimal — migration reversible).
- [ ] Unit test sederhana untuk generator sequence (test set acak, cek urutan abjad & format).

## Example (from PDF JC2 Mendel AY2024-2025)
```
Audrey Tiara Gozali    → SKL: 002/CJCE/KJ/2025   TN: 002/TN/SMA/KJ/2025
Calvin Mateo Susanto   → SKL: 004/CJCE/KJ/2025   TN: 004/TN/SMA/KJ/2025
...
Velove Maria           → SKL: 033/CJCE/KJ/2025   TN: 033/TN/SMA/KJ/2025
```
> Siswa nomor 41 ("Sample Student S3A Taylor") adalah data test, tidak termasuk sequence produksi.

## UI / UX Changes
- Tidak ada perubahan UI di frontend. Hanya generator backend baru + tabel/persist.
- Disediakan untuk modul National Report Card yang akan di-stub di `features/national-report` (format-margin-nasional.md, notes.md, api-contract.md, reference/, screenshots/, schema.md, spec.md).

## API Changes
- Endpoints (proposal): `POST /report/nasional/:classId/generate-sequence` — menjalankan sequence per class/AY.
- Request: `{ classId: number, academicYearId: number }`
- Response: `{ generated: [{ studentId, name, sequence, sklNumber, tnNumber }] }`

## Database Changes
- **Tabel baru** (proposal): `report_number_sequence` — mapping idempoten per siswa/AY:
  - `id` (serial PK)
  - `student_id` (FK)
  - `academic_year_id` (FK)
  - `class_id` (FK, null jika sequence global sekolah)
  - `skl_number` (varchar)
  - `tn_number` (varchar)
  - unique constraint (student_id, academic_year_id)
- Migration rollup (TypeORM): create table (proposal), reversible, safe.

## Business Rules / Validation
1. Sequence **global per sekolah**, bukan per class: nomor lompat ketika ada student dari class lain yang masuk urutan abjad.
2. Urutan = **abjad nama lengkap** (case-insensitive), bukan NISN, bukan student_id.
3. Nomor **persisted** → idempotent, stabil.
4. SKL nomor = TN nomor (format berbeda: `NNN/CJCE/KJ/YYYY` vs `NNN/TN/SMA/KJ/YYYY`).
5. Data test ("Sample Student S3A Taylor") **dibuang** saat sequence (bukan bagian dari dataset lintas-kelas se-KJ).
6. Data-nilai (GP Value, Unit Subject, PUM, dll.) hanyalah inputnya, bukan bagian dari nomor ini.

## Error Handling
- Student di class tidak memiliki `name` (NULL): dilewati / diberi waktu exception; log + lanjut generate berikutnya.
- Duplicate generation: idempotent, hanya regenerate untuk student yang belum punya nomor (upsert).
- Akademik year tidak aktif: reject.

## Dependencies
- `api_nest/src/modules/student/student.service.ts` (student data)
- `api_nest/src/modules/class-year/class-year.service.ts` (class/AY)
- `api_nest/src/modules/academic-year/academic-year.service.ts` (AY existence)
- `api_nest/src/modules/report-setting/report-setting.service.ts` (report setting)

## Notes
- Referensi PDF asli (5 file) disimpan di `features/national-report/reference/`.
- Sample text "Sample Student S3A Taylor" (nomor 41) adalah placeholder/test data — ditolak dari sequence produksi.
- Format nomor SKL (`CJCE` = sekolah program), sedangkan nomor Transkrip (`TN`) agak berbeda (contoh: `002/TN/SMA/KJ/2025`).
