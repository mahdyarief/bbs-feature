---
title: CCA Registration — CCA dengan level spesifik masih tampil untuk semua level siswa
status: open
severity: major
product: BBS LMS
portal: Student Portal
author: System Analyst
date: 2025-07-31
---

# CCA Registration Level Filter — CCA Tidak Terfilter Berdasarkan Level Siswa

## Summary

Ketika suatu CCA-Year di-set hanya untuk level tertentu (misal: `masterLevelIds = [Secondary 1]`), CCA tersebut **masih tampil** di halaman registrasi Student Portal untuk **semua siswa** dari level lain (misal: siswa Secondary 2).

**Ekspektasi:** CCA yang hanya ditujukan untuk level tertentu **hanya tampil** untuk siswa dengan level tersebut — siswa dari level lain tidak boleh melihat atau bisa mendaftar ke CCA itu.

---

## Steps to Reproduce

1. Login sebagai **Student** ke Student Portal
2. Buka halaman **CCA Registration**
3. Admin sebelumnya membuat CCA-Year dengan konfigurasi:
   - `masterLevelIds = [id_secondary_1]` (hanya untuk Secondary 1)
   - Academic Year aktif
4. Amati daftar CCA yang tampil untuk siswa **Secondary 2**

**Actual Result:** CCA Secondary 1 tersebut **tampil** untuk siswa Secondary 2

**Expected Result:** CCA Secondary 1 tersebut **hanya tampil untuk siswa Secondary 1**. Siswa Secondary 2 tidak boleh melihat CCA tersebut di daftar.

---

## Root Cause Analysis

### Lapisan 1 — Backend `cca-year.service.ts:findAll()` (line 234-409)

Method `findAll()` membangun `whereFilters` (line 296-385) dengan filter untuk:
- `activeStatus`, `ccaName`, `ccaType`, `academicYearId`
- `programmeId`, `teacherId`, `teacherName`
- `campusIds`, `campusId`, `ids`

**TIDAK ADA FILTER untuk `masterLevelId` atau `masterLevelIds`.** Tidak ada logika yang membaca level student yang sedang login dan memfilter CcaYears yang sesuai.

DTO `GetCcaYearsDto` juga **tidak memiliki properti `masterLevelId`**, sehingga frontend tidak bisa mengirim parameter level filter.

### Lapisan 2 — Backend `cca-registration.service.ts:create()` (line 58-166)

Method `create()` melakukan beberapa validasi (gender quota, window check, max CCA per student), tetapi **TIDAK ADA VALIDASI** yang mengecek apakah level student (`student.currentClassYear.masterLevelId`) cocok dengan level CcaYear (`ccaYear.masterLevelIds`).

Method `assertWindowOpen()` (line 675-686) hanya mengecek window berdasarkan `ccaYear.masterLevelId` (level dari CCA), **bukan level student**. Ini berarti: jika registration window dikonfigurasi untuk Secondary 1, student Secondary 2 yang mendaftar ke CCA Secondary 1 tetap lolos.

### Lapisan 3 — Frontend `StudentCCARegistration.jsx`

Frontend memiliki akses ke level student (`studentClassYear?.level` di line 103-105) tetapi **tidak menggunakannya** untuk memfilter daftar CCA. Pemanggilan API di line 60-67 hanya mengirim `academicYearId`, `ccaName`, `page`, `pageSize`, dan `activeStatus` — **tidak ada `masterLevelId`**.

---

## Business Rules

1. **Master level bersifat authoritative** — CCA-Year hanya boleh diikuti oleh siswa yang levelnya tercantum di `masterLevelIds`
2. `masterLevelIds = [Secondary 1]` → hanya siswa Secondary 1 boleh lihat & daftar
3. `masterLevelIds = [Secondary 1, Secondary 2]` → siswa Secondary 1 & 2 boleh lihat & daftar
4. `masterLevelIds = null/[]` → fallback ke semua level (tidak ada pembatasan level)

### Yang Harusnya Terjadi

| Config | Secondary 1 Student | Secondary 2 Student |
|--------|-------------------|-------------------|
| `masterLevelIds=[S1]` | Bisa lihat & daftar | **Tidak bisa lihat** |
| `masterLevelIds=[S2]` | **Tidak bisa lihat** | Bisa lihat & daftar |
| `masterLevelIds=[S1,S2]` | Bisa lihat & daftar | Bisa lihat & daftar |
| `masterLevelIds=null` | Bisa lihat & daftar | Bisa lihat & daftar |

---

## Data Model

### Level Student

Student tidak memiliki kolom level langsung. Level didapatkan melalui relasi:
```
Student.currentClassYearId → ClassYear.masterLevelId → MasterLevel
```

### Level CCA

CcaYear menyimpan level target di kolom:
- `masterLevelId` (number) — kolom tunggal untuk backward compatibility
- `masterLevelIds` (number[]) — array of int, full set of targeted levels

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/cca-year/cca-year.service.ts` | `findAll()` tidak filter by student level |
| Backend DTO | `api_nest/src/modules/cca-year/dto/get-cca-years.dto.ts` | Tidak ada parameter `masterLevelId` |
| Backend Service | `api_nest/src/modules/cca-registration/cca-registration.service.ts` | `create()` tidak validasi level student vs CCA level |
| Frontend Student | `bbs/client-student/src/views/ccaRegistration/StudentCCARegistration.jsx` | Tidak filter berdasarkan level student |

---

## Proposed Solution Options

### Option A: Backend Filter (Recommended)

1. **`cca-year.service.ts:findAll()`** — Tambahkan filter untuk mengecek `masterLevelIds`:
   - Jika student login, ambil `student.currentClassYear.masterLevelId`
   - Filter CcaYears yang `masterLevelIds` mengandung level student tersebut, ATAU `masterLevelIds` kosong/null (berlaku untuk semua level)

2. **`cca-registration.service.ts:create()`** — Tambahkan validasi:
   - Cek apakah `student.currentClassYear.masterLevelId` termasuk dalam `ccaYear.masterLevelIds`
   - Jika tidak, throw error

### Option B: Backend + DTO Enhancement

1. **`get-cca-years.dto.ts`** — Tambah field `masterLevelId` opsional
2. **Frontend** — Kirim `masterLevelId` dari student yang login ke API
3. **Backend** — Filter berdasarkan `masterLevelId` dari request

### Option C: Frontend-Only Filter

1. **`StudentCCARegistration.jsx`** — Filter hasil response di frontend berdasarkan level student
2. **Risiko:** Data tidak terfilter dari API (masalah keamanan/eksposur data), dan validasi di `create()` tetap diperlukan

---

## Notes

- Gunakan `bbs-feature/cca-registration-level-filter/` untuk spec implementasi
- Data level student sudah tersedia di frontend melalui `studentClassYear` (line 103-105 `StudentCCARegistration.jsx`)
- Backend entity `CcaYear` sudah memiliki kolom `masterLevelIds` (array), tinggal dimanfaatkan
- Jalur data level student: `Student.currentClassYear → ClassYear.masterLevelId → MasterLevel`