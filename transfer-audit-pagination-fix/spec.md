---
feature: Transfer Audit — perbaikan default pagination & statistics serialization
slug: transfer-audit-pagination-fix
status: draft
author: System Analyst (dari temuan AI Operate Test TC-039/TC-040)
date: 2026-09-30
target_release: TBD
related_bug_reports:
  - transfer-audit-default-param-500
---

# Transfer Audit — Perbaikan Default Pagination & Statistics

## Overview

Brief perbaikan kode (untuk engineer backend) atas modul `transfer-audit`
(`api_nest/src/modules/class-year/transfer-audit.{controller,service}.ts`).
Dokumen ini HANYA spesifikasi — implementasi dikerjakan tim backend.

## Problem / Motivation

TC-039/TC-040 (2026-09-30) menemukan dua kegagalan pada modul yang melayani
audit log student transfer:

1. **Default pagination tidak pernah diterapkan.** Controller memakai default
   parameter TypeScript pada decorator query:

   ```ts
   async getStudentTransferHistory(
     @Param('studentId') studentId: number,
     @Query('page') page: number = 1,     // ← TIDAK pernah terisi
     @Query('limit') limit: number = 10,  // ← TIDAK pernah terisi
   )
   ```

   Nest meng-inject `undefined` bila query tidak dikirim; default `= 1` pada
   parameter fungsi tidak dieksekusi. `skip: (page-1)*limit` → `NaN` →
   TypeORM menolak: **500** `Provided "skip" value is not a number`.
   Terdampak 4 route: `/student/:id`, `/user/:id`, `/campus/:id`, `/failed`.

2. **`/statistics` mengembalikan body salah serialisasi.** HTTP 200 tapi data
   berisi `{"type":"object","attributes":{"type":"object"}}` — objek statistik
   di-serialize sebagai tipe JSON:API, bukan isinya.

## Scope

### In Scope
- Default pagination yang benar (DTO ber-tipe atau koersi eksplisit) di 4 route.
- Perbaikan return value `getStatistics()` agar body JSON:API berisi data nyata.
- Konsistensi tipe `page`/`limit` (number, bukan string) di semua method service
  terkait (`getTransferHistory`, `getAllAudits`, `getAuditsByUser`,
  `getAuditsByCampus`, `getFailedAudits`).

### Out of Scope
- Migrasi modul ke `version: '1'` (keputusan versioning terpisah — lihat
  EC-03; saat ini live di `/api/` tanpa `/v1`).
- Fitur audit baru / retensi log / indexing baru.
- Modul lain dengan pola pagination serupa (sudah terverifikasi hidup via
  PageOptionsDto; modul ini satu-satunya yang memakai default-param manual).

## User Stories

### As an integrator (FE/monitoring)
I want transfer-audit endpoints to work without explicit pagination query
So that standard API calls do not return 500.

### As an admin
I want /statistics to return real transfer statistics
So that dashboards show meaningful data.

## Acceptance Criteria

- [ ] `GET /api/transfer-audit/student/:id` (tanpa query) → 200 dengan page=1,
      limit=10 (default).
- [ ] Idem untuk `/user/:id`, `/campus/:id`, `/failed`.
- [ ] `GET /api/transfer-audit/statistics` → 200 dengan body statistik nyata
      (mis. total transfer, sukses/gagal, per periode) — bukan `{"type":"object"}`.
- [ ] `?page=2&limit=5` tetap bekerja (koersi string→number aman).
- [ ] Verifikasi ulang sweep TC-039: keempat route tanpa query → 200.

## API Changes

| Method | Path | Perubahan |
|--------|------|-----------|
| GET | `/api/transfer-audit/student/:id` | default page=1 limit=10 diterapkan |
| GET | `/api/transfer-audit/user/:id` | idem |
| GET | `/api/transfer-audit/campus/:id` | idem |
| GET | `/api/transfer-audit/failed` | idem |
| GET | `/api/transfer-audit/statistics` | return objek statistik nyata |

## Database Changes

Tidak ada perubahan skema. Tabel `transfer_audit_log` sudah ada (dengan kolom
`sourceStudentId`, `targetStudentId`, `campusId`, `userId`).

## Business Rules / Validation

1. Default pagination: **page 1, limit 10** (sesuai nilai default yang sudah
   tertulis di kode — hanya tidak pernah diterapkan).
2. Koersi eksplisit: `Number(page ?? 1)`, `Number(limit ?? 10)` — atau DTO
   `@Type(() => Number) @IsOptional()` ala `PageOptionsDto` yang dipakai modul lain.
3. Validasi batas: `limit` dibatasi maksimum wajar (mis. 100) agar tidak jadi
   unbounded query.
4. `statistics` harus mengembalikan agregat dari tabel audit, bukan konstanta.

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| `page`/`limit` non-numerik (mis. `?page=abc`) | 400 | Validation error ala DTO |
| `skip`/`take` negatif | 400 | (tidak boleh sampai ke TypeORM lagi) |

## Dependencies

- Repo `smartbag/api_nest` (implementasi oleh tim backend).
- Bug report bukti & reproduksi: `transfer-audit-default-param-500/`.
- Verifikasi pasca-deploy: re-run probe TC-039 pada 5 path ini (tanpa dan
  dengan query).

## Open Questions (lihat edgecases.md)

- EC-01: Migrasi ke `/api/v1/` sekalian atau tidak?
- EC-02: Bentuk isi body `/statistics` yang diinginkan produk.
