---
title: CCA — Dukung 3 Moderator Teacher (Coordinator) per CCA
status: draft
type: task
product: BBS LMS
portal: Admin (client)
author: System Analyst
date: 2025-08-04
target_release: TBD
---

# Task: CCA Mendukung 3 Moderator Teacher (Coordinator)

## Overview

Saat ini sebuah CCA hanya dapat memiliki **maksimal 2 moderator teacher** (disebut **coordinator** di kode — `CcaYearCoordinator`). Kebutuhan bisnis: CCA harus dapat memuat **hingga 3 moderator teacher**.

Batas 2 saat ini **bukan batasan schema/DB** — data model sudah mendukung N coordinator (array rows). Batas tersebut hanyalah **konstanta policy** yang di-enforce di backend (1 file) dan frontend (3 file).

## Problem / Motivation

- User membutuhkan hingga 3 moderator teacher per CCA untuk pembagian peran
- Saat ini ketika user mencoba menambahkan coordinator ke-3 melalui form, sistem menolak dengan error "at most 2 coordinators"
- 3 moderator diperlukan agar semua peran (setter/vetter/expert) bisa ditangani oleh guru berbeda

## Scope

### In Scope

- Ubah batas coordinator dari **2 → 3** di backend dan frontend
- Update pesan error yang menyebut "at most 2 coordinators" menjadi "3"
- Pastikan form create & update merender **3 slot** coordinator
- Pastikan validasi yup schema menerima **3 coordinator**

### Out of Scope

- Perubahan schema/database (TIDAK diperlukan — data model sudah N rows)
- Perubahan entity (`Cca`, `CcaYear`, `CcaYearCoordinator`) — struktur sudah mendukung
- Perubahan DTO (sudah menerima array tanpa limit)
- Perubahan tampilan list/detail (`CCAInYears.jsx`, `CCAInYearDetails.jsx`) — sudah `.map()` semua coordinator

## Acceptance Criteria

- [ ] User dapat meng-assign **3 moderator teacher** saat create CCA
- [ ] User dapat meng-assign **3 moderator teacher** saat update CCA
- [ ] Ke-3 moderator tersimpan di DB (3 rows `CcaYearCoordinator`)
- [ ] Menambahkan coordinator ke-4 **ditolak** dengan pesan error yang benar ("at most 3 coordinators")
- [ ] Form tidak boleh memilih teacher yang sama di 2 slot berbeda (duplicate check tetap berlaku)
- [ ] Detail & list CCA tetap menampilkan semua coordinator (termasuk yang ke-3)

## UI / UX Changes

### Affected Portals

- [x] Admin (client/)
- [ ] Student (client-student/)
- [ ] Teacher (client-teacher/)

Form coordinator di `CCAInYearFormCreate.jsx` / `CCAInYearFormUpdate.jsx` menampilkan **3 slot** `BBSResourceSelect` dengan label "Coordinators (max 3)".

## API Changes

TIDAK ADA perubahan API — endpoint yang ada sudah menerima array `ccaYearCoordinators` tanpa limit.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/v1/ccaYears` | Sudah terima `ccaYearCoordinators[]` (loop create) |
| POST | `/v1/ccaYearCoordinators` | Sudah per-slot |
| PATCH | `/v1/ccaYearCoordinators/:id` | Sudah per-slot |
| DELETE | `/v1/ccaYearCoordinators/:id` | Sudah per-slot |

## Database Changes

**TIDAK ADA** — batas 2 adalah policy, bukan schema. `CcaYearCoordinator` sudah berupa tabel child (N rows per CcaYear).

### Konfirmasi Struktur Data

| Kolom | Tipe | Keterangan |
|-------|------|------------|
| `ccaYearId` | int | FK ke CcaYear |
| `teacherId` | int | FK ke **Employee** — ini "moderator teacher" |
| `programmeId` | int (nullable) | Legacy, tidak dipakai frontend baru |

## Business Rules / Validation

1. Maksimal **3** coordinator per CCA-Year (sebelumnya 2)
2. Coordinator harus **teacher** (`employeeType: "TEACHER"`) — di-filter frontend
3. Satu teacher **tidak boleh** dipilih di 2 slot berbeda dalam CCA yang sama
4. Coordinator ke-4 ke atas → error **400** "A CCA can have at most 3 coordinators"

## File Changes Required

Semua perubahan hanyalah mengubah konstanta batas 2 → 3:

| # | File | Baris | Perubahan |
|---|------|-------|-----------|
| 1 | `api_nest/src/modules/cca-year-coordinator/cca-year-coordinator.service.ts` | 17 | `MAX_CCA_YEAR_COORDINATORS = 2` → `3` |
| 2 | `api_nest/src/errors/ResourceError.ts` | 806-812 | Pesan error "at most 2 coordinators" → "at most 3 coordinators" |
| 3 | `bbs/client/src/views/ccaInYears/components/CCACoordinatorsField.jsx` | 5 | `MAX_COORDINATORS = 2` → `3` |
| 4 | `bbs/client/src/views/ccaInYears/CCAInYearFormUpdate.jsx` | 28 | `MAX_COORDINATORS = 2` → `3` |
| 5 | `bbs/client/src/views/ccaInYears/ccaInYearSchema.js` | 45 | `.max(2, "...")` → `.max(3, "...")` |

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| Coordinator ke-4 | 400 | "A CCA can have at most 3 coordinators." |

## Dependencies

- Konstanta `MAX_CCA_YEAR_COORDINATORS` di backend harus **sinkron** dengan `MAX_COORDINATORS` di frontend (2 file) dan schema yup — jika tidak, salah satu layer akan menolak/tidak menyimpan slot ke-3
- `CCAInYearFormUpdate.jsx` mengambil `MAX_COORDINATORS` dari konstanta lokal (line 28) — pastikan sinkron dengan komponen field, jika tidak slot ke-3 tidak tersimpan meskipun backend menerimanya

## Notes

- Istilah "moderator teacher" di kode bernama **"coordinator"** (`CcaYearCoordinator`)
- Coordinator terhubung ke entity **`Employee`** (dengan filter `employeeType: "TEACHER"` di frontend), bukan entity `Teacher`
- `programmeId` pada coordinator bersifat legacy dan tidak dipakai frontend baru (field hanya mengirim `teacherId`)
- Detail view (`CCAInYearDetails.jsx` line 255-282) dan list view (`CCAInYears.jsx` line 168-191) sudah render semua coordinator tanpa limit — otomatis tampil 3 setelah perubahan ini