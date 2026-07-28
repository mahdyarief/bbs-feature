---
title: CCA Registration — CCA dengan gender quota spesifik masih tampil untuk semua gender
status: open
severity: major
product: BBS LMS
portal: Student Portal
author: System Analyst
date: 2025-07-28
---

# CCA Registration Quota — Gender Filter Tidak Berfungsi

## Summary

Ketika suatu CCA-Year hanya memiliki quota untuk satu gender (misal: `femaleQuota = 10`, `maleQuota = null`), CCA tersebut **masih tampil** di halaman registrasi Student Portal untuk **semua siswa**, termasuk siswa dengan gender yang tidak memiliki quota.

**Ekspektasi:** CCA yang hanya memiliki quota untuk gender tertentu **hanya tampil** untuk siswa dengan gender tersebut — siswa dengan gender lain tidak boleh melihat atau bisa mendaftar ke CCA itu.

---

## Steps to Reproduce

1. Login sebagai **Student** ke Student Portal
2. Buka halaman **CCA Registration**
3. Admin sebelumnya membuat CCA-Year dengan konfigurasi:
   - `femaleQuota = 10`
   - `maleQuota = null` (atau 0)
   - `quota = null` (gender quota bersifat authoritative)
4. Amati daftar CCA yang tampil

**Actual Result:** CCA tersebut tampil untuk **siswa male** maupun **siswa female**

**Expected Result:** CCA tersebut **hanya tampil untuk siswa female**. Siswa male tidak boleh melihat CCA tersebut di daftar.

---

## Root Cause Analysis

### Backend — `cca-year.service.ts` (line 234-409)

Method `findAll()` di `CcaYearService` **tidak melakukan filter berdasarkan gender student** sama sekali. Tidak ada parameter `gender` atau `studentGender` di `GetCcaYearsDto`, dan tidak ada `where` clause yang mengecek `maleQuota`/`femaleQuota`.

### Frontend — `StudentCCARegistration.jsx`

Halaman registrasi memanggil `fromApi.getCcaYears()` tanpa info gender. Setelah data diterima, component `CCACard` merender semua CCA tanpa filter berdasarkan gender student.

### Validasi di `cca-registration.service.ts` (line 127-128)

Validasi gender quota hanya terjadi **saat registrasi (create)**, bukan saat **menampilkan daftar**:

```typescript
// enforce gender quota on registration, not on listing
if (this.hasGenderQuota(ccaYear)) {
  await this.validateGenderQuota(ccaYear, student);
}
```

Akibatnya:
1. Siswa male bisa **melihat** CCA yang hanya punya femaleQuota
2. Siswa male bisa **mendaftar** (create) — validasi `validateGenderQuota()` akan return early karena `maleQuota = null`
3. Baru error terjadi jika maleQuota diisi > 0 dan sudah penuh

### Gender Quota Logic — `validateGenderQuota()` (line 786-814)

```typescript
if (genderQuota === null || genderQuota === undefined || genderQuota <= 0) {
  return;  // early return — No quota for this gender = no restriction
}
```

Ini berarti: `maleQuota = null` dibaca sebagai **"tidak ada batasan untuk male"** — bukan **"tidak boleh male mendaftar"**.

---

## Business Rules

1. **Gender quota bersifat authoritative** — jika diisi, mengoverride total quota
2. `maleQuota = 10, femaleQuota = null` → hanya siswa male boleh daftar
3. `maleQuota = 10, femaleQuota = 10` → kedua gender boleh daftar
4. `maleQuota = null, femaleQuota = null` → fallback ke `quota` total, semua gender boleh

### Yang Harusnya Terjadi

| Config | Male Student | Female Student |
|--------|-------------|---------------|
| `maleQuota=10, femaleQuota=null` | Bisa lihat & daftar | **Tidak bisa lihat** |
| `maleQuota=null, femaleQuota=10` | **Tidak bisa lihat** | Bisa lihat & daftar |
| `maleQuota=10, femaleQuota=10` | Bisa lihat & daftar | Bisa lihat & daftar |
| `quota=10` (gender null) | Bisa lihat & daftar | Bisa lihat & daftar |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/cca-year/cca-year.service.ts` | `findAll()` tidak filter gender |
| Backend Service | `api_nest/src/modules/cca-year/dto/cca-year.dto.ts` | `CcaYearDto` tidak expose maleQuota/femaleQuota |
| Backend Service | `api_nest/src/modules/cca-year/dto/get-cca-years.dto.ts` | Tidak ada filter gender |
| Frontend Student | `bbs/client-student/src/views/ccaRegistration/StudentCCARegistration.jsx` | `CCACard` tidak filter gender di render |

---

## Proposed Solution Options

### Option A: Backend Filter (Recommended)

Tambahkan filter di `cca-year.service.ts:findAll()` untuk exclude CCA-Year yang student's gender quota-nya null/0, ketika gender quota di CCA-Year tersebut terkonfigurasi (salah satu gender punya quota > 0).

**Logic:**
```
IF ccaYear.hasGenderQuota() AND studentGender quota is null/0
  THEN exclude from result
```

### Option B: Backend + DTO Enhancement

Backend:
- Kirim `maleQuota` dan `femaleQuota` dalam response `getCcaYears`
Frontend:
- Filter di `CCACard` berdasarkan gender student yang login

### Option C: Validate Gender Visibility at Registration

Backend `cca-registration.service.ts`:
- Ubah `validateGenderQuota()` agar `maleQuota = null` (saat `femaleQuota > 0`) berarti **tidak boleh male mendaftar** (bukan "no restriction")
- Frontend tetap perlu filter agar CCA tidak tampil

---

## Notes

- Gunakan `bbs-feature/` untuk spec implementasi setelah keputusan final
- Perlu koordinasi dengan FE Engineer untuk frontend filtering
- Backend sudah ada helper `hasGenderQuota()` yang bisa dipakai
- Gender student tersedia di `selfUser` (student profile) di frontend
