---
feature: GPA Threshold Config Fix
slug: gpa-threshold-config
status: draft
author: BBS Team
date: 2025-07-15
target_release: TBD
---

# GPA Threshold Config Fix

## Overview

Sistem threshold GPA saat ini tidak direlasikan dengan `academic_year_id`. Konfigurasi threshold menggunakan prinsip "berlaku dari tahun X sampai tahun Y" — satu baris config diberlakukan lintas beberapa tahun sekaligus. Karena perhitungan GPA selalu melihat config 3 tahun ke belakang, mekanisme `effective_from`-`effective_to` yang overlap menyebabkan **nilai GPA selalu salah**.

Fix: tambah `academic_year_id` sebagai FK utama di tabel threshold, sehingga setiap AY punya config sendiri-sendiri.

## Problem / Motivation

> Satu baris konfigurasi threshold tidak boleh berlaku lintas tahun akademik. Setiap academic year harus punya threshold-nya sendiri, dan harus direlasikan dengan `academic_year_id`.

### Masalah yang Terjadi

1. Config `2022-2024` aktif di SEMUA tahun 2022, 2023, 2024 — bukan khusus untuk AY manapun
2. Perhitungan GPA untuk AY 2025 tidak bisa menentukan config mana yang benar
3. Ketika aturan threshold berubah di salah satu tahun, tidak bisa diakomodasi tanpa mengubah config tahun lain
4. `status` 1/0 hanya mengaktifkan/menonaktifkan satu baris, bukan per tahun akademik

## Scope

### In Scope
- Tambah kolom `academic_year_id` di tabel `threshold`
- Dropdown Academic Year di UI
- Logic copy data dari AY sebelumnya (deep copy)
- Perhitungan GPA: lookup via `academic_year_id`
- Migrasi data existing
- Update UI konfigurasi threshold & add syllabus

### Out of Scope
- Perubahan aturan "3 tahun ke belakang" (sudah benar)
- Perubahan struktur tabel lain yang tidak terkait threshold
- Perubahan logic hitung GPA itu sendiri (hanya perubahan lookup)

## User Stories

### As a Admin
I want to configure GPA threshold per academic year
So that each year's grading uses the correct threshold data

### As a Admin
I want existing threshold data to be auto-copied when creating a new academic year
So that I don't have to re-enter data for historical years

## Acceptance Criteria

- [ ] Tabel `threshold` punya kolom `academic_year_id` (INT FK ke `academic_year.id`)
- [ ] Dropdown di UI menampilkan Academic Year, user pilih AY dulu baru bisa kelola threshold
- [ ] Saat buat config untuk AY baru, data dari AY sebelumnya otomatis di-copy (deep copy)
- [ ] Edit di AY baru tidak mempengaruhi data di AY sebelumnya
- [ ] Perhitungan GPA menggunakan `WHERE academic_year_id = [AY yang dihitung]`
- [ ] Data existing yang belum punya `academic_year_id` tetap bisa diakses (nullable FK)

## UI / UX Changes

### Affected Portals
- [x] Admin (client/)
- [ ] Student (client-student/)
- [ ] Teacher (client-teacher/)

### Changes
- Dropdown di halaman threshold: ganti dari range tahun (from-to) → pilih Academic Year
- Tabel threshold menampilkan kolom Academic Year sebagai konteks utama
- Modal add syllabus: user pilih Academic Year ID dulu, lalu tentukan parameter per tahun
- Tidak ada validasi `effective_from = effective_to` — karena beda AY boleh punya tahun yang sama

## Business Rules / Validation

### Pola 3 Tahun ke Belakang

Setiap AY menggunakan avg_value dari tahun sebelumnya. Avg_value adalah rerata dari 3 tahun ke belakang:

| Academic Year | Menggunakan Avg Value Dari | Avg Itu Sendiri Dari Tahun |
|---|---|---|
| AY 2024/2025 (ID 24) | Tahun 2024 | 2023, 2022, 2021 |
| AY 2025/2026 (ID 25) | Tahun 2025 | 2024, 2023, 2022 |
| AY 2026/2027 (ID 26) | Tahun 2026 | 2025, 2024, 2023 |

### Copy Logic (Deep Copy)

Ketika user membuat config untuk AY baru (misal AY 26):
- Copy data dari AY sebelumnya (AY 25) untuk tahun yang relevan
- Tahun terbaru (belum ada di AY sebelumnya) tidak di-copy — user isi sendiri
- Data yang di-copy adalah **entry baru** (bukan reference)
- Edit di AY baru tidak mempengaruhi AY sebelumnya

Contoh:
```
AY 25 existing: year 2022, 2023, 2024
AY 26 (baru, auto-copy): year 2023 (copy), 2024 (copy), 2025 (isi sendiri)
```

## Dependencies

- `academic-year` module (existing FK target)
- `threshold` table (existing, to be altered)
- `grade` module (GPA calculation lookup)
- Admin panel views (client/)
