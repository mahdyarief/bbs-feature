---
title: List API — Query param sortBy yang tidak valid mengembalikan HTTP 500 (bukan 400)
status: open
severity: minor
product: BBS LMS
portal: Admin | Teacher | Student
author: System Analyst
date: 2026-09-30
jam: (tidak tersedia — laporan berbasis eksekusi test API + code review)
---

# List API — `sortBy=kolomNgawur` Mengembalikan 500 Padahal Seharusnya 400

## Summary

Ditemukan saat eksekusi test case read-only TC-034 (folder
`operate-smartbag/test-cases/`, 2026-09-30). Setiap list endpoint yang memakai
`findOptionsHelper` menerima query `sortBy` **tanpa validasi nama kolom**. Nilai
yang tidak cocok dengan kolom mana pun diteruskan ke TypeORM sebagai `order` →
query gagal → exception tidak tertangani → **HTTP 500**.

Repro satu baris pada salah satu endpoint paling stabil:

```bash
GET /api/v1/branches?page=0&pageSize=5&sortBy=kolomNgawur
→ 500
```

**Ekspektasi:** input yang tidak valid = kesalahan pemanggil → HTTP **400** dengan
pesan bermakna (atau diabaikan dengan fallback sort default) — bukan 500 yang
muncul di log error dan membingungkan pemanggil.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Buffy (AI agent) — eksekusi TC-034 |
| User ID (`selfUser`) | 35008 (`mahdy.klabs`, Super Admin) |
| Environment API | `https://api.binabangsaschool.dev` (production) |
| Data konteks | `GET /branches?page=0&pageSize=5&sortBy=kolomNgawur`; dieksekusi 2026-09-30 |
| Browser / OS | curl |

---

## Steps to Reproduce

```bash
TOKEN=<admin token>
curl -s -o /dev/null -w "%{http_code}\n" \
  "https://api.binabangsaschool.dev/api/v1/branches?page=0&pageSize=5&sortBy=kolomNgawur" \
  -H "Authorization: Bearer $TOKEN"
# → 500
```

**Actual Result:** HTTP 500 (internal server error) untuk input query yang salah.

**Expected Result:** HTTP 400 (`sortBy must reference a sortable column`) — atau
200 dengan fallback ke sort default.

---

## Root Cause Analysis

### Bug — `sortBy` diteruskan mentah ke TypeORM — `find-options-helper.ts` + `utils.helper.ts`

```typescript
// api_nest/src/helpers/find-options-helper.ts:68-76
...(options.sortBy
  ? {
      order: sortAttributeHelper(options.sortBy),   // ← tidak ada validasi kolom
    }
  : { ... }),
```

```typescript
// api_nest/src/helpers/utils.helper.ts:93-114
export const sortAttributeHelper = (sortBy: string, sortShape?: any) => {
  if (!sortBy) return null;
  let sortKey = sortBy;            // ← 'kolomNgawur' masuk apa adanya
  let sortOrder: SortOrder = 'ASC';
  if (sortBy.startsWith('-')) { sortKey = sortBy.slice(1); sortOrder = 'DESC'; }
  const buildNestedSort = (keys, order) =>
    keys.reduceRight((acc, key) => ({ [key]: acc }), order);
  // → order = { kolomNgawur: 'ASC' } → TypeORM QueryFailedError
```

`sortShape` (parameter ke-2) tampaknya dirancang sebagai whitelist tapi tidak
pernah dipakai oleh pemanggil — semua service hanya mengirim `sortBy`.

Faktor pendukung: exception dari TypeORM tidak dipetakan ke `BadRequestException`
di layer mana pun → 500 generik.

---

## Bukti dari Test (tanpa Jam)

| Sumber | Temuan |
|--------|--------|
| **API probe** | `GET /branches?...&sortBy=kolomNgawur` → **500** |
| **Pembanding** | endpoint sama tanpa sortBy → 200 (TC-034 S1/S2/S4) |
| **Code** | `find-options-helper.ts:68-76` meneruskan sortBy; `utils.helper.ts:93-114` membangun order object tanpa whitelist |
| **Cakupan** | berlaku untuk SEMUA list endpoint yang memakai `findOptionsHelper` (mayoritas modul) |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Helper | `api_nest/src/helpers/find-options-helper.ts` | Meneruskan `sortBy` tanpa validasi |
| Backend Helper | `api_nest/src/helpers/utils.helper.ts` | `sortAttributeHelper` tidak memakai `sortShape` (whitelist) |
| Backend (semua list controller) | modul yang memakai `findOptionsHelper` | Permukaan 500 dari satu titik utilitas |

---

## Proposed Solution Options

### Option A: Validasi + 400 di helper (Recommended — satu titik perbaikan)

Di `sortAttributeHelper`, bila `sortShape` diberikan: tolak key di luar whitelist
(`throw new BadRequestException(\`sortBy '${key}' is not sortable\`)`).
Kemudian tambahkan `sortShape` di service yang mau ketat. Minimal: bungkus
pembangunan order dengan try/catch → 400 generik "invalid sortBy".

### Option B: Sanitasi diam (fallback)

Key yang tidak dikenal diabaikan (order di-drop) + log warning. UX paling halus,
tapi menyembunyikan typo pemanggil — kurang disiplin untuk API internal.

### Option C: Validasi di DTO

Tambah whitelist per-DTO (`@ApiPropertyOptional({ enum: [...] })`). Paling eksplisit
tapi menyentuh banyak file — lebih mahal daripada Option A.

---

## Notes

- Sumber temuan: `operate-smartbag/test-cases/TC-034-pagination-boundary/result.md`.
- Severity minor: butuh token sah + hanya merusak request yang memang salah;
  tidak ada data yang berubah. Tapi satu titik perbaikan (helper) menutup seluruh
  permukaan 500 di semua list endpoint — ROI perbaikan tinggi.
- Temuan sekunder di TC-034 (quirk `hasNextPage=true` saat data habis pada
  pageSize besar, `page-meta.dto.ts:34`) — minor, sekadar dicatat.
