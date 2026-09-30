---
feature: Pagination Page-Base Mismatch — PageOptionsDto 0-based vs service 1-based
slug: pagination-page-base-mismatch
status: draft
author: System Analyst (dari temuan AI Operate Test TC-039)
date: 2026-09-30
target_release: TBD
related_bug_reports:
  - transfer-audit-default-param-500
---

# Pagination Page-Base Mismatch (sveStudentGrades OFFSET negative)

## Overview

Brief perbaikan kode (untuk engineer backend) atas **ketidakselarasan basis
`page`** antara `PageOptionsDto` global (0-based) dan service yang menghitung
offset manual dengan asumsi 1-based. Kasus nyata: `GET /api/v1/sveStudentGrades`
tanpa `page` → **400** `OFFSET must not be negative`.

## Problem / Motivation

TC-039 (2026-09-30) menemukan:

- `PageOptionsDto` global (`src/common/dto/page-options.dto.ts`):
  `page = 0` (default), `pageSize = 5` — **0-based**. Mayoritas service memakai
  `findOptionsHelper` yang menghitung `offset = page * pageSize` → konsisten.
- `sve-student-grade.service.ts` menghitung manual:
  `skip((page - 1) * pageSize)` — **asumsi 1-based**.

Akibatnya tanpa query `page`, default DTO `page=0` menghasilkan
`skip = (0-1)*10 = -10` → TypeORM menolak:
`400 {"code":"2201X","message":"OFFSET must not be negative"}`.
Dengan `?page=1` endpoint normal (200). Pola ini juga akar masalah bug
`transfer-audit` (ditangani brief terpisah, plus bug default-paramnya).

Grep `page - 1` di seluruh `modules/**/*.service.ts` → hanya **2 file**:
`transfer-audit.service.ts` dan `sve-student-grade.service.ts`. Artinya mismatch
ini terisolasi, tapi kontrak pagination jadi **tidak seragam**: dua endpoint
ini memakai `page` mulai dari 1, sisanya codebase mulai dari 0.

## Scope

### In Scope
- `sve-student-grade.service.ts`: selaraskan basis page (lihat EC-01).
- Panduan seragam: satu basis `page` untuk seluruh codebase (terdokumentasi
  di PageOptionsDto), agar service baru tidak mengulangi pola manual.

### Out of Scope
- `transfer-audit` (brief terpisah `transfer-audit-pagination-fix`, mencakup
  default-param + statistics).
- Refactor `findOptionsHelper` / `PageMetaDto` (meta math — terkait quirk
  `hasNextPage` di TC-034, di luar masalah ini).

## User Stories

### As an API consumer
I want `page=0` and omitted `page` to behave the same on every list endpoint
So that pagination is predictable across the API.

## Acceptance Criteria

- [ ] `GET /api/v1/sveStudentGrades` (tanpa query) → 200, setara `page` pertama,
      bukan 400 OFFSET negative.
- [ ] `?page=0` dan `?page=1` punya semantik yang jelas dan sama dengan endpoint
      list lain (salah satu basis dipilih global).
- [ ] Tidak ada service lain yang menghitung offset manual dengan basis berbeda
      (verifikasi: grep `skip((page` / `page - 1` → nol selain yang disepakati).

## API Changes

| Method | Path | Perubahan |
|--------|------|-----------|
| GET | `/api/v1/sveStudentGrades` | offset = `page * pageSize` (0-based, mengikuti konvensi global) atau DTO khusus bila kontrak 1-based dipertahankan |

## Database Changes

Tidak ada.

## Business Rules / Validation

1. Basis page yang menang: **0-based** (konvensi mayoritas via `PageOptionsDto`
   + `findOptionsHelper`) — EC-01 memutuskan.
2. Frontend `client/` yang memanggil `sveStudentGrades` saat ini dengan
   `?page=1` harus dicek dampaknya setelah perubahan basis (offset geser).
3. Dokumentasikan kontrak pagination di satu tempat (comment PageOptionsDto /
   wiki `systems.md`).

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| `page` negatif (mis. `?page=-1`) | 400 | Validation `@Min(0)` sudah ada di DTO — pertahankan |
| OFFSET negative dari perhitungan internal | — | Tidak boleh mungkin lagi |

## Dependencies

- Repo `smartbag/api_nest` (implementasi oleh tim backend).
- Bukti: `operate-smartbag/test-cases/TC-039-full-controller-path-sweep/result.md`
  (quirk sveStudentGrades), TC-034 (meta math quirk).
- Verifikasi pasca-deploy: re-run probe `sveStudentGrades` tanpa query.

## Open Questions (lihat edgecases.md)

- EC-01: 0-based global vs bungkus DTO 1-based khusus dua modul ini.
- EC-02: dampak ke pemanggil yang selama ini pakai `?page=1`.
