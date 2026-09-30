---
feature: Billing Reports — batasi eager-load & wajibkan filter (502 gateway timeout)
slug: billing-reports-eager-load-502
status: draft
author: System Analyst (dari temuan AI Operate Test TC-039)
date: 2026-09-30
target_release: TBD
related_bug_reports:
  - student-list-no-filter-timeout
  - topics-list-eager-loading-timeout
---

# Billing Reports — 502 karena Eager-Load Seluruh Billings per MasterProduct

## Overview

Brief perbaikan kode (untuk engineer backend) atas `GET /api/v1/billingReports`
yang mengembalikan **502 gateway timeout** konsisten. Ini anggota keluarga bug
list-endpoint yang sama dengan `student-list-no-filter-timeout` (`/students`
tanpa filter) dan `topics-list-eager-loading-timeout` (`/topics`) — sekarang
ada 3 anggota terdokumentasi dengan 3 varian penyebab.

## Problem / Motivation

TC-039 (2026-09-30): `GET /api/v1/billingReports?limit=1` → **502**.

Akar di `billing-report.service.ts` `findAll()`:

```ts
const rawMasterProducts = await MasterProduct.find({
  relations: { billings: true },           // ← eager-load SEMUA billing
  where: {
    branch: { campuses: { id: In(campusIds) } },
    billings: { student: { id: Not(IsNull()) } },
  },
});
```

Lalu dilanjutkan DynamicQueryBuilder dengan `pageSize=1`. Dua beban tidak
proporsional:

1. **Pre-query eager-load semua `billings`** per MasterProduct di lingkup
   campus permitted — jumlah baris tidak dibatasi `pageSize`.
2. `masterProductsIds` hasil pre-query dipakai `IN (...)` di query utama —
   daftar id bisa puluhan ribu → query besar.

`pageSize=1` tidak menolong karena beban utamanya di pre-query, bukan di
result set.

## Scope

### In Scope
- Ganti pre-query `relations: { billings: true }` → **subquery EXISTS / join
  id-only** untuk mendapatkan daftar masterProductId yang punya billing siswa.
- Pastikan limit `masterProductsIds` tidak menghasilkan `IN` raksasa (chunk /
  join langsung di QB utama).
- Perilaku respons dipertahankan (bentuk data report per masterProduct).

### Out of Scope
- Perbaikan `/students` dan `/topics` (brief masing-masing sudah ada).
- Solusi infrastruktur timeout gateway (nginx/proxy) — dampak sistemik, bukan
  akar masalah endpoint ini.
- Excel export di modul ini (route download) — dievaluasi terpisah bila ikut
  lambat.

## User Stories

### As an admin (finance)
I want the billing report list to load within the gateway timeout
So that I can view billing summaries per product.

## Acceptance Criteria

- [ ] `GET /api/v1/billingReports?limit=1` → **200** (bukan 502) dengan meta benar.
- [ ] Query pre-load tidak lagi mengambil seluruh entitas billing; hanya id
      yang dibutuhkan (EXISTS subquery atau join id-only).
- [ ] Query utama tidak menghasilkan `IN` dengan puluhan ribu parameter.
- [ ] Respons body sama bentuknya dengan sebelumnya (tidak breaking untuk FE).

## API Changes

Tidak ada perubahan kontrak — hanya implementasi (performance fix).

## Database Changes

Tidak ada perubahan skema. **Rekomendasi index** (verifikasi `EXPLAIN` dulu):
- `billing(master_product_id)` bila belum ada (kemungkinan sudah via FK).
- Composite `billing(master_product_id, student_id)` bila planner masih scan.

## Business Rules / Validation

1. Lingkup data tetap dibatasi campus permit (`checkCampusesPermit`) — jangan
   hilangkan filter ini saat refactor.
2. Billing yang dihitung tetap yang punya `student` (bukan billing tanpa siswa).
3. `pageSize` tetap efektif di query utama (DynamicQueryBuilder).

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| Timeout tetap terjadi (data tumbuh) | 502→504 | Kembalikan saran filter (branchId/campusId wajib) — lihat EC-02 |
| Campus permit kosong | 200 | `{data: []}` — bukan 500 |

## Dependencies

- Repo `smartbag/api_nest` (implementasi oleh tim backend).
- Bukti: `operate-smartbag/test-cases/TC-039-full-controller-path-sweep/result.md`.
- Keluarga bug serupa: `student-list-no-filter-timeout/`,
  `topics-list-eager-loading-timeout/` — pertimbangkan satu pattern-guide
  "list endpoint" untuk ketiganya (lihat Rekomendasi COVERAGE.md #2).
- Verifikasi pasca-deploy: re-run probe `billingReports?limit=1`.

## Open Questions (lihat edgecases.md)

- EC-01: strategi refactor pre-query (EXISTS vs join id-only).
- EC-02: wajibkan filter (campusId/branchId) bila dataset global tetap berat.
- EC-03: pattern-guide bersama untuk 3 endpoint bermasalah.
