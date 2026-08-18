---
title: CCA In Year — CCA Option tidak menampilkan CCA Code, membingungkan user ketika nama sama
status: open
severity: minor
product: BBS LMS
portal: Admin
author: System Analyst
date: 2026-08-18
jam: ~
---

# CCA In Year — CCA Option tidak menampilkan CCA Code, membingungkan user ketika nama sama

## Summary

Di halaman Admin portal `/cca-in-year/add` (Add CCA In Year) dan `/cca-in-year/:id/edit` (Edit CCA In Year), field **CCA** (dropdown selector) hanya menampilkan nama CCA saja, tanpa kode. Ketika ada beberapa CCA dengan nama yang mirip atau sama tetapi kode berbeda (mis. "Basketball Boys" vs "Basketball Girls" dengan kode "BSK-B" dan "BSK-G"), user tidak bisa membedakannya di dropdown. User harus menebak atau keluar dari halaman ini untuk cek kode CCA terlebih dahulu.

**Ekspektasi:** Dropdown CCA menampilkan nama **dan** kode, contoh: `Basketball Boys (BSK-B)` — sehingga user bisa langsung membedakan CCA yang satu dengan yang lain tanpa keluar dari halaman.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | System Analyst |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | — |
| Portal URL | https://admin.smartbag.binabangsaschool.com/cca-in-year/add |
| Environment API | admin.smartbag.binabangsaschool.com |
| Data konteks (class ID / daId / tanggal) | — |
| Browser / OS | — |

---

## Steps to Reproduce

1. Login ke Admin portal, buka halaman `/cca-in-year/add`.
2. Klik field **CCA** untuk membuka dropdown.
3. Ketik nama CCA yang ada duplikatnya (mis. "Basketball").

**Actual Result:** Dropdown hanya menampilkan nama CCA saja (mis. "Basketball Boys", "Basketball Girls"). Jika ada CCA yang namanya identik dengan kode berbeda, user tidak bisa membedakannya.

**Expected Result:** Dropdown menampilkan nama **dan** kode (mis. "Basketball Boys (BSK-B)", "Basketball Girls (BSK-G)") — sehingga user bisa langsung membedakan.

---

## Root Cause Analysis

### Bug #1 — `CCAsSelector` hanya menggunakan `name` sebagai label, tidak menyertakan `code` — `CCAsSelector.jsx` (line 84-96)

`CCAsSelector` component membangun option label hanya dari `resource[resourceLabel]` (default: `"name"`). Padahal CCA entity memiliki field `code` yang bisa dipakai untuk membedakan CCA yang namanya sama.

**Kode saat ini (sebelum fix):**

```javascript
// path: bbs/client/src/selectors/CCAsSelector.jsx:84-87
...resources.map((resource) => ({
  value: resource[valueKey]?.toString(),
  label: `${resource[resourceLabel]}`   // hanya nama
}))

// path: CCAsSelector.jsx:93-96 (untuk single resource yang di-fetch by ID)
{
  value: resource[valueKey]?.toString(),
  label: `${resource[resourceLabel]}`   // hanya nama
}
```

**CCA entity memiliki field `code`:**

```typescript
// path: api_nest/src/modules/cca/entities/cca.entity.ts:22-23
@Column()
code: string;
```

---

## Bukti dari Jam

Tidak ada Jam recording — ini adalah enhancement request berdasarkan user feedback bahwa dropdown membingungkan ketika ada CCA dengan nama yang sama.

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend | `bbs/client/src/selectors/CCAsSelector.jsx` (line 84-96) | Dropdown CCA hanya tampil nama, tidak tampil kode |

---

## Proposed Solution Options

### Option A: Tambahkan code ke label di `CCAsSelector.jsx` (Recommended — sudah diimplementasi)

Ubah label construction di `CCAsSelector.jsx` untuk menyertakan `resource.code` jika ada, dengan format `"Name (CODE)"`.

```javascript
// FIXED — CCAsSelector.jsx:84-87
...resources.map((resource) => ({
  value: resource[valueKey]?.toString(),
  label: resource.code
    ? `${resource[resourceLabel]} (${resource.code})`
    : `${resource[resourceLabel]}`
}))

// FIXED — CCAsSelector.jsx:93-96 (single resource)
{
  value: resource[valueKey]?.toString(),
  label: resource.code
    ? `${resource[resourceLabel]} (${resource.code})`
    : `${resource[resourceLabel]}`
}
```

**Kelebihan:**
- Satu perubahan di `CCAsSelector.jsx` berlaku untuk semua halaman yang menggunakan component ini (`/cca-in-year/add` dan `/cca-in-year/:id/edit`).
- Format `"Name (CODE)"` familier dan mudah dibaca.
- Aman — jika `code` tidak ada (null/empty), label tetap hanya nama (tidak error).

### Option B: Pass custom `optionLabelWithSeparator` atau formatter prop

Pass prop khusus ke `CCAsSelector` dari `CCAInYearFormCreate.jsx` untuk menentukan format label.

**Kekurangan:** Lebih verbose, perlu update setiap caller. Option A lebih simpel karena code selalu ada di CCA entity.

---

## Notes

- Enhancement ini **frontend-only**, tidak ada perubahan backend/API.
- `CCAsSelector.jsx` digunakan di `/cca-in-year/add` (line 156-162) dan `/cca-in-year/:id/edit` (line 186-192) — keduanya benefit dari fix ini.
- Field `code` pada CCA entity adalah required (`@Column()` tanpa `nullable: true`), jadi selalu ada nilai untuk setiap CCA.
- Perubahan ini **backward-compatible** — tidak ada payload yang berubah, hanya tampilan label di UI.
