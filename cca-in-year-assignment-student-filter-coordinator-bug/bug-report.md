---
title: CCA In Year Assignment — List Student tidak terfilter cabang & kode cabang coordinator kosong
status: open
severity: major
product: BBS LMS
portal: Admin Portal
author: System Analyst
date: 2026-08-13
---

# CCA In Year Assignment — Student List Tidak Terfilter Cabang & Coordinator Branch Code Kosong

## Summary

Pada fitur **CCA In Year Assignment** di Admin Portal, ditemukan 2 bug:

1. **List Student pada menu Assign Student tidak terfilter berdasarkan cabang (programme)** — Ketika membuat CCA In Year, admin sudah menginput programme (cabang). Namun pada halaman Assign Student, kolom "List Student In ..." menampilkan **seluruh student dari semua cabang**, tidak terfilter sesuai programme CCA tersebut.
2. **Kode cabang coordinator kosong** — Pada kolom Coordinator di laman CCA Year Assignment, format tampilan adalah `Nama Teacher - ()`. Seharusnya di dalam kurung terdapat **kode cabang** coordinator tersebut (contoh: `GPA Teacher - (PT-S)`), namun aktual saat ini **kosong** (`GPA Teacher - ()`).

**Ekspektasi:**
- List student pada Assign Student hanya menampilkan siswa yang belongs ke cabang/programme yang sama dengan CCA In Year tersebut.
- Kolom coordinator menampilkan nama teacher beserta kode cabangnya dalam kurung, mis. `GPA Teacher - (PT-S)`.

---

## Steps to Reproduce

### Bug #1 — List Student Tidak Terfilter Cabang

1. Login sebagai **Admin** ke Admin Portal (`admin.smartbag.binabangsaschool.com`)
2. Buat **CCA In Year** dengan memilih programme/cabang tertentu (mis. cabang `PT-S`)
3. Buka halaman **CCA In Year Assignment** untuk CCA In Year tersebut
4. Masuk ke menu **Assign Student**
5. Amati kolom **List Student In ...**

**Actual Result:**
- List student menampilkan **seluruh siswa dari semua cabang**, tidak terfilter sesuai programme CCA In Year.

**Expected Result:**
- List student **hanya menampilkan siswa dari cabang/programme** yang dipilih saat membuat CCA In Year (mis. hanya siswa cabang `PT-S`).

---

### Bug #2 — Kode Cabang Coordinator Kosong

1. Login sebagai **Admin** ke Admin Portal
2. Buka laman **CCA Year Assignment**
3. Amati kolom **Coordinator**

**Actual Result:**
- Tampilan coordinator: `Nama Teacher - ()` — kode cabang di dalam kurung **kosong**.
- Contoh aktual: `GPA Teacher - ()`

**Expected Result:**
- Tampilan coordinator: `Nama Teacher - (XX-X)` — kode cabang coordinator terisi dengan benar.
- Contoh ekspektasi: `GPA Teacher - (PT-S)`

---

## Root Cause Analysis

### Bug #1 — Student List Tidak Terfilter Programme

**Root cause:** `programmeId` yang digunakan saat ini **tidak aware terhadap branch/cabang**. Ketika filter student dijalankan menggunakan `programmeId`, query tidak memperhitungkan relasi antara programme dengan branch, sehingga hasil yang dikembalikan adalah **seluruh student dari semua cabang** tanpa memandang cabang CCA In Year tersebut.

Detail:
- `programmeId` di CCA In Year mereferensikan programme tertentu, namun query list student tidak memetakan `programmeId` → `branchId` (atau tidak menggunakan relasi programme-branch) sebagai filter.
- Akibatnya, filter berdasarkan `programmeId` saja tidak cukup untuk membatasi student hanya dari cabang yang sesuai — semua student tetap muncul.
- **Yang seharusnya terjadi:** `programmeId` harus di-resolve ke `branchId` (atau query harus join ke relasi programme-branch), sehingga student yang dikembalikan hanya yang belongs ke branch tersebut.

### Bug #2 — Branch Code Coordinator Kosong

Kemungkinan penyebab:
- Data coordinator di-relate ke teacher, namun **branch code tidak di-join / tidak di-include** saat mengambil data coordinator.
- Frontend merender format `Teacher Name - (Branch Code)` namun field branch code yang diterima dari API bernilai `null` / `undefined` / empty string.
- Kemungkinan relasi `teacher → branch` belum di-eager load di query, atau field branch code belum di-expose di DTO/response.

---

## Affected Components

| Layer | Area | Impact |
|-------|------|--------|
| Backend API | Endpoint list student untuk CCA Assign Student | `programmeId` tidak di-resolve ke `branchId` — filter tidak aware branch |
| Backend API | Endpoint get CCA Year Assignment (coordinator) | Tidak menyertakan branch code coordinator di response |
| Frontend Admin | Halaman CCA In Year Assignment → Assign Student | Menampilkan seluruh student tanpa filter cabang |
| Frontend Admin | Halaman CCA Year Assignment → kolom Coordinator | Merender branch code kosong karena data tidak diterima dari API |

---

## Proposed Solution Options

### Bug #1 — Filter Student by Programme (Aware Branch)

**Option A: Resolve programmeId → branchId di Backend (Recommended)**
- Saat menerima request list student, backend resolve `programmeId` dari CCA In Year ke `branchId` yang sesuai
- Tambahkan filter `branchId` pada query student, sehingga hanya student dari branch tersebut yang dikembalikan
- Frontend tetap meneruskan `programmeId` (atau CCA In Year ID), backend yang handle mapping ke branch

**Option B: Frontend Filter**
- Backend tetap kirim semua student, frontend filter berdasarkan branch
- **Tidak disarankan** — boros bandwidth dan rawan inkonsistensi

### Bug #2 — Include Branch Code pada Coordinator

**Option A: Backend — Include Branch Relation (Recommended)**
- Pada query coordinator, lakukan join/include ke tabel `branch` untuk mengambil `branchCode`
- Expose field `branchCode` di response DTO coordinator

**Option B: Frontend Fallback**
- Jika branch code kosong, tampilkan hanya nama teacher tanpa format kurung
- **Tidak disarankan** — hanya menyembunyikan gejala, tidak menyelesaikan akar masalah

---

## Notes

- Kedua bug ini berdampak pada akurasi data assignment CCA — student dari cabang lain bisa ter-assign ke CCA yang bukan cabangnya
- Koordinator perlu teridentifikasi dengan jelas beserta cabangnya untuk keperluan monitoring & reporting
- Perlu koordinasi dengan BE Engineer untuk investigasi query dan relasi data
- Setelah keputusan final, buat spec implementasi di folder fitur ini (ikuti `_templates/spec-template.md`)
