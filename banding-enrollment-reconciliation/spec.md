---
feature: Banding Enrollment Reconciliation (Class 100024 PIK TEST)
slug: banding-enrollment-reconciliation
status: draft
author: System Analyst (dari temuan AI Operate Test TC-010)
date: 2026-09-30
target_release: TBD
---

# Banding Enrollment Reconciliation

## Overview

Alat dan proses untuk mendeteksi serta memulihkan **perbedaan enrollment siswa
pada `banding`** (junction `banding_students_student`) terhadap baseline yang
diketahui — khususnya class_year 100024 (P1 PIK TEST, AY 2026/2027) yang hasil
pengisian programatiknya (action 05, 2026-09-29: 16 banding × 3 siswa = 48
junction) kini terdeteksi 42 junction (4 banding kehilangan 6 siswa total).

## Problem / Motivation

Eksekusi TC-010 (2026-09-30) menemukan delta:

| banding id | subject_year | subject | siswa sekarang | baseline action 05 |
|---|---|---|---|---|
| 111981 | 112393 | Character First | 1 | 3 (−2) |
| 111985 | 112397 | Faith Builder | 2 | 3 (−1) |
| 111986 | 112398 | Pendidikan Pancasila dan Kewarganegaraan | 2 | 3 (−1) |
| 111987 | 112399 | Indonesian Studies | 1 | 3 (−2) |

Tidak ada mekanisme yang mencatat **siapa yang menghapus** junction tersebut
(tidak ada audit log pada pivot ini), sehingga tidak bisa dibedakan: perubahan
manual via UI oleh user, perubahan oleh proses lain, atau data yang memang harus
begitu. Tanpa rekonsiliasi, nilai LO siswa yang hilang dari banding berisiko
tidak ikut terisi di term berikutnya (banding = dasar pembagian grading per
subject).

## Scope

### In Scope
- Query deteksi delta enrollment banding terhadap baseline (snapshot/daftar eksplisit).
- Prosedur pemulihan via API (`PUT /api/v1/classYears/:id` tidak cukup granular —
  banding diatur lewat endpoint banding) atau DB write dengan persetujuan.
- Laporan rekonsiliasi: baris hilang/tambahan + sumber kebenaran yang dipakai.

### Out of Scope
- Audit log sistemik untuk semua pivot (fitur terpisah bila diminta PM).
- Perubahan skema `banding`/`banding_students_student`.
- Class lain selain yang disepakati per eksekusi.

## User Stories

### As an operator (AI operate)
I want to detect banding membership drift against a recorded baseline
So that programmatic setup results remain verifiable over time.

### As an admin (BBS)
I want a safe procedure to restore missing banding memberships
So that grading per subject covers all enrolled students.

## Acceptance Criteria

- [ ] Script/query menghasilkan laporan delta (hilang/tambahan) class 100024
      terhadap baseline action 05 tanpa write.
- [ ] Setiap langkah pemulihan menampilkan before/after dan butuh persetujuan eksplisit.
- [ ] Setelah pemulihan, junction = baseline (48) dan terverifikasi via query + API.
- [ ] `result.md` di `operate-smartbag` mencatat keputusan akhir (restore / keep).

## UI / UX Changes

Tidak ada perubahan UI (proses operasional via API/DB).

### Affected Portals
- [ ] Admin (client/)
- [ ] Student (client-student/)
- [ ] Teacher (client-teacher/)

## API Changes

Pemulihan idealnya via endpoint banding yang dipakai UI (perlu konfirmasi path —
`POST /api/v1/bandings` dipakai action 05 untuk create):

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/bandings?limit=50` | Verifikasi pasca-pemulihan (read-only) |
| POST | `/api/v1/bandings` | Re-create junction siswa (payload envelope JSON:API) — path persis per case |

## Database Changes

Tidak ada perubahan skema. Data:

```sql
-- deteksi (read-only)
SELECT b.id, b.subject_year_id, count(bs.student_id) AS n_siswa
FROM banding b
LEFT JOIN banding_students_student bs ON bs.banding_id = b.id
WHERE b.subject_year_id IN (SELECT id FROM subject_year WHERE class_year_id = 100024)
GROUP BY b.id, b.subject_year_id ORDER BY b.id;
```

## Business Rules / Validation

1. **Baseline yang dipakai** = hasil action 05 (16 banding × 3 siswa, id
   111975–111990) — dicatat di `operate-smartbag/actions/05-assign-banding-students/`.
2. Pemulihan hanya boleh menambah junction yang hilang; **jangan menghapus**
   junction "ekstra" tanpa konfirmasi (bisa jadi perubahan manual yang disengaja).
3. Duplikat check dulu sebelum insert (junction yang sama boleh sudah ada).
4. Semua write via API lebih disukai daripada DB langsung (jejak `updatedBy`).

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| Banding id tidak ada | 404 | Banding not found |
| Siswa sudah terdaftar (duplicate) | 400 | Student already assigned to this banding |
| Token tanpa permission | 403 | Fallback: verifikasi via DB read + eskalasi role |

## Dependencies

- `api_nest` module `banding` (controller/service path verifikasi saat eksekusi).
- `operate-smartbag` actions 05 (baseline) dan TC-010 (temuan).
- Token admin `.secret` (Super Admin).

## Open Questions (lihat edgecases.md)

- Apakah delta ini perubahan yang disengaja (user edit via UI) atau drift yang
  tidak diinginkan?
- Apakah perlu snapshot baseline otomatis setiap kali action operate selesai?
