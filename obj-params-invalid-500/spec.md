---
feature: Obj-Params Query Builder — input buruk → 500 + kebocoran pesan internal
slug: obj-params-invalid-500
status: draft
author: System Analyst (dari temuan AI Operate Test TC-043)
date: 2026-09-30
target_release: TBD
related_bug_reports:
  - list-sortby-invalid-500
related_test_cases:
  - operate-smartbag/test-cases/TC-043-dynamic-qb-params
---

# Obj-Params Query Builder — 500 untuk Input Buruk + Info Leak

## Overview

Brief perbaikan kode (untuk engineer backend) atas keluarga parameter generik
`PageOptionsDto` ber-objek JSON (`relationsObj`, `wheresObj`, `orderObj`,
`selectsObj`) yang mengubah **input buruk menjadi 500** dan **membocorkan pesan
error internal TypeORM** ke client. Satu keluarga dengan bug `sortBy` invalid
→ 500 (TC-034, `list-sortby-invalid-500`).

## Problem / Motivation

TC-043 (2026-09-30) di `GET /api/v1/campuses` (berlaku global — dipakai hampir
semua list endpoint):

| Input | Respons | Harusnya |
|---|---|---|
| `relationsObj={"branches":true}` (relasi tidak ada) | **500** `Property "branches" was not found in "Campus". Make sure your query is correct.` | 400 tanpa detail internal |
| `wheresObj={"name":` (JSON rusak) | **500** `Unexpected end of JSON input` | 400 invalid JSON |
| `orderObj={"(SELECT 1)":"desc"}` (kunci aneh) | **500** `Property "(SELECT 1)" was not found in "Campus". Make sure your query is correct.` | 400 + whitelist kunci |

Akar: `@Transform` di `PageOptionsDto` melakukan `JSON.parse` tanpa try/catch,
dan `dynamic-qb`/`findOptionsHelper` meneruskan kunci apa pun ke TypeORM
(`Property ... was not found` = error TypeORM yang tidak dibungkus). Selain
berisiko (pesan error mengungkap struktur entity), monitoring 500 terpolusi
oleh request buruk biasa.

## Scope

### In Scope
- Bungkus `JSON.parse` transform dengan try/catch → input rusak ditolak 400.
- Validasi kunci `relationsObj`/`orderObj`/`selectsObj`/`wheresObj` terhadap
  metadata entity (whitelist) atau minimal catch error TypeORM "Property not
  found" → 400 generik.
- Konsisten dengan fix `sortBy` (`list-sortby-invalid-500`) — sebaiknya satu
  PR keluarga.

### Out of Scope
- Fitur query-builder baru.
- `breakthrough` flag (akses/role — dievaluasi terpisah, lihat Open Questions
  TC-043).

## User Stories

### As an API consumer
I want malformed obj-params to return 400 with a clear message
So that I can fix my request instead of guessing a server bug.

### As a security reviewer
I want TypeORM internal messages not to reach clients
So that entity structure is not disclosed.

## Acceptance Criteria

- [ ] `wheresObj` JSON rusak → **400** (bukan 500), pesan generik.
- [ ] `orderObj`/`relationsObj` dengan kunci yang bukan properti entity → **400**.
- [ ] Request valid (wheresObj filter, orderObj sort) tetap bekerja seperti
      sebelumnya (regresi TC-043 Step 1).
- [ ] Tidak ada lagi `Unexpected end of JSON input` atau
      `Property "..." was not found in "..."` di response client.

## API Changes

Tidak ada perubahan kontrak sukses — hanya mapping error (500→400).

## Database Changes

Tidak ada.

## Business Rules / Validation

1. Semua transform `PageOptionsDto` (`relations`, `relationsObj`, `selectsObj`,
   `orderObj`, `wheresObj`) wajib try/catch → `BadRequestException`.
2. Kunci kolom/relasi divalidasi dari metadata entity sebelum masuk TypeORM.
3. Pesan error ke client generik: `invalid <param>: <reason singkat>` tanpa
   nama internal TypeORM.

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| JSON rusak di obj-param | 400 | invalid wheresObj: malformed JSON |
| Kunci bukan properti entity | 400 | invalid orderObj: unknown column |
| Error TypeORM tak terduga | 500 | generic internal error (tanpa detail) |

## Dependencies

- Repo `smartbag/api_nest` (implementasi oleh tim backend).
- Bukti: `operate-smartbag/test-cases/TC-043-dynamic-qb-params/result.md`.
- Bug saudara: `list-sortby-invalid-500/` (sortBy) — gabungkan fix.
- Verifikasi pasca-deploy: re-run TC-043 Step 2 (negative).

## Open Questions

- Apakah kunci wheresObj perlu whitelist operator juga (mis. `contains`, `in`)
  atau cukup try/catch? (bisa menyusul setelah 400 mapping aman).
