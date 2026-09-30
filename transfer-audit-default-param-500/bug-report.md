---
title: Transfer Audit — 500 tanpa query page/limit eksplisit (default param tidak diterapkan) + statistics body rusak
status: open
severity: major
product: BBS LMS
portal: Admin
author: System Analyst
date: 2026-09-30
jam: (tidak tersedia — laporan berbasis eksekusi test API + code review)
---

# transfer-audit — default `@Query()` pagination tidak bekerja → 500; `/statistics` 200 dengan body sampah

## Summary

Ditemukan saat TC-039/TC-040 (2026-09-30). Controller
`class-year/transfer-audit.controller.ts` memakai **default parameter fungsi
TypeScript** pada `@Query()`:

```ts
async getStudentTransferHistory(
  @Param('studentId') studentId: number,
  @Query('page') page: number = 1,     // ← TIDAK pernah terisi 1
  @Query('limit') limit: number = 10,  // ← TIDAK pernah terisi 10
) { ... }
```

Dekorator Nest meng-inject nilai dari query string; kalau query tidak ada,
nilainya `undefined` — default `= 1` **tidak pernah diterapkan**. `page` dan
`limit` → `undefined` → TypeORM `skip: NaN` → **500**
`Provided "skip" value is not a number`.

Kena 4 route: `/student/:id`, `/user/:id`, `/campus/:id`, `/failed`.
Dengan `?page=1&limit=10` **eksplisit** semua route → 200.

Bonus temuan: `GET /api/transfer-audit/statistics` → **200** tapi body-nya
`{"data":[{"type":"object","attributes":{"type":"object"}}]}` — serialisasi
JSON:API salah objek (meng-serialize tipe, bukan isi).

Catatan posisi URI: seluruh modul melayani di **`/api/` tanpa `/v1`**
(`@Controller('transfer-audit')` non-version) — tidak masalah fungsional, tapi
frontend yang selalu menambah `/v1` akan 404. Lihat `TC-040-versioning-audit/`.

**Ekspektasi:** tanpa query pagination, endpoint memakai default page 1 limit 10
dan mengembalikan 200.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Buffy (AI agent) — eksekusi TC-039/TC-040 |
| User ID | 35008 (`mahdy.klabs`, Super Admin) |
| Environment API | `https://api.binabangsaschool.dev` (production) |
| Browser / OS | curl |

## Steps to Reproduce

```bash
# 500:
curl -s "https://api.binabangsaschool.dev/api/transfer-audit/student/100215" \
  -H "Authorization: Bearer $TOKEN" -o /dev/null -w "%{http_code}\n"
# → 500  {"message":"Provided \"skip\" value is not a number..."}

# 200 (workaround):
curl -s "https://api.binabangsaschool.dev/api/transfer-audit/student/100215?page=1&limit=10" \
  -H "Authorization: Bearer $TOKEN" -o /dev/null -w "%{http_code}\n"

# statistics body rusak:
curl -s "https://api.binabangsaschool.dev/api/transfer-audit/statistics" \
  -H "Authorization: Bearer $TOKEN"
# → {"statusCode":200,"data":[{"type":"object","attributes":{"type":"object"}}],...}
```

## Actual vs Expected

| Route (tanpa query) | Actual | Expected |
|---|---|---|
| `/transfer-audit/student/:id` | 500 | 200 (default page=1, limit=10) |
| `/transfer-audit/user/:id` | 500 | 200 |
| `/transfer-audit/campus/:id` | 500 | 200 |
| `/transfer-audit/failed` | 500 | 200 |
| `/transfer-audit/statistics` | 200 tapi body `{"type":"object"}` | 200 dengan statistik nyata |

## Impact

- Semua konsumen yang tidak mengirim pagination eksplisit kena 500.
- `statistics` mengembalikan data kosong terselubung sukses — bisa menyesatkan
  monitoring/dashboard.

## Rekomendasi

1. Parse + default eksplisit: `page: Number(page ?? 1)`, `limit: Number(limit ?? 10)`
   (atau DTO `@Type(() => Number) @IsOptional()`).
2. Perbaiki serialisasi `statistics` (return objek statistik nyata).
3. Pertimbangkan daftarkan modul di `version: '1'` agar konsisten dengan pola
   `/api/v1/` mayoritas (atau pastikan frontend mengetahui path non-version).
