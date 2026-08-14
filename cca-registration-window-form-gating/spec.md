---
feature: CCA Registration Window — Form Gating & Data Integrity
slug: cca-registration-window-form-gating
status: draft
author: System Analyst
date: 2026-08-14
target_release: TBD
---

# CCA Registration Window — Form Gating & Data Integrity

## Overview

Fix perilaku CCA Registration Window agar:

1. **Form registrasi CCA di Student Portal TIDAK muncul** ketika tidak ada registration window yang dikonfigurasi untuk scope siswa (academic year + campus + level).
2. Menutup 4 celah integritas data yang ditemukan saat review:
   - tidak ada validasi `opensAt < closesAt`
   - tidak ada pengecekan window ganda/overlap di scope yang sama
   - window ber-status `INACTIVE` diperlakukan sebagai "tidak dikonfigurasi" → registrasi malah terbuka
   - `findOne(id)` tidak menangani id yang tidak ditemukan (tidak melempar 404)

## Problem / Motivation

Saat ini `CcaRegistrationWindowService.isRegistrationOpen()` (line 114-164) punya default **backward-compatible**: jika tidak ada window sama sekali untuk scope, hasilnya `true` (registrasi terbuka). Akibatnya:

- Siswa selalu melihat form & daftar CCA di Student Portal meskipun admin **belum pernah** mengatur registration window — ini berlawanan dengan ekspektasi bisnis: tanpa window, tidak ada periode registrasi, jadi form tidak boleh tampil.
- Frontend tidak bisa membedakan "registrasi terbuka karena window sedang berjalan" vs "registrasi terbuka karena tidak ada window dikonfigurasi", karena `CcaYearDto.isRegistrationOpen` default `true` (line 6-13 `cca-year.dto.ts`).

Selain itu, review menemukan celah integritas di service window (`cca-registration-window.service.ts`) yang membuat state window tidak deterministik.

### Potensi Masalah yang Ditemukan (Review)

| # | Issue | Lokasi | Dampak |
|---|-------|--------|--------|
| 1 | Tidak ada validasi `opensAt < closesAt` | `create()` / `update()` di `cca-registration-window.service.ts` | Window dengan rentang terbalik (closesAt < opensAt) tersimpan → `isRegistrationOpen` tidak pernah `true` di periode tsb |
| 2 | Tidak ada pengecekan overlap/duplikasi window di scope yang sama `(academicYearId, campusId, masterLevelId)` | `create()` / `update()` | `findOne()` → hasil non-deterministik jika 2 window aktif di scope sama; siswa bisa dapat hasil berbeda antar request |
| 3 | Window `INACTIVE` dihitung sebagai "tidak dikonfigurasi" | `isRegistrationOpen()` tail (line 150-164) & `isWindowConfigured()` (line 169-195) | Admin menonaktifkan window → registrasi malah terbuka (fallback `return true`) |
| 4 | `findOne(id)` tidak handle id tidak ada | `findOne()` | Return null/undefined alih-alih `NotFoundException` → error handling tidak konsisten |

## Scope

### In Scope

- Backend: ekspos sinyal "window dikonfigurasi" (mis. `isWindowConfigured`) ke Student Portal, atau ubah semantik default untuk konteks student.
- Backend: validasi `opensAt < closesAt` pada create/update window.
- Backend: deteksi overlap/duplikasi window pada scope yang sama.
- Backend: semantik `INACTIVE` window (keputusan di `edgecases.md` EC-02).
- Backend: `findOne(id)` melempar `NotFoundException`.
- Frontend: gating form registrasi CCA di `StudentCCARegistration.jsx` — sembunyikan form/daftar saat tidak ada window untuk scope siswa.

### Out of Scope

- Perubahan logika level filter CCA (`masterLevelIds` vs level siswa) — sudah ditangani oleh spec terpisah `bbs-feature/cca-registration-level-filter/` (catatan: `assertWindowOpen` di `cca-registration.service.ts:688-699` memakai `ccaYear.masterLevelId` bukan level siswa; dianggap dependensi/known issue terpisah).
- Perubahan tampilan UI selain gating form (tidak ada redesain).
- Migrasi data (tidak ada perubahan skema tabel; optional index unik dibahas di Database Changes).

## User Stories

### Sebagai siswa
Saya ingin form registrasi CCA tidak muncul ketika admin belum mengatur registration window untuk level & campus saya, sehingga saya tidak bingung mencoba mendaftar di luar periode resmi.

### Sebagai admin
Saya ingin sistem menolak window yang rentang waktunya terbalik atau tumpang tindih dengan window lain di scope yang sama, sehingga perilaku registrasi deterministik dan mudah dijelaskan.

### Sebagai admin
Saya ingin menonaktifkan sebuah window untuk menutup registrasi (bukan malah membukanya), sehingga kontrol periode registrasi bisa dipercaya.

## Acceptance Criteria

- [ ] **AC-1:** Siswa membuka halaman CCA Registration di Student Portal, dan tidak ada window (ACTIVE maupun INACTIVE) untuk scope `(academicYearId, campusId, level)`-nya → form registrasi & daftar CCA **tidak ditampilkan**; tampilkan empty state/informasi "registrasi belum dibuka".
- [ ] **AC-2:** Siswa memiliki window ACTIVE yang sedang berjalan (`opensAt <= now <= closesAt`) → form tampil normal.
- [ ] **AC-3:** Siswa memiliki window ACTIVE tetapi di luar periode (`now < opensAt` atau `now > closesAt`) → form tidak tampil (perilaku `isRegistrationOpen = false` yang sudah ada, tetap dipertahankan).
- [ ] **AC-4:** Admin membuat window dengan `closesAt <= opensAt` → API menolak dengan 400 dan pesan yang jelas.
- [ ] **AC-5:** Admin membuat window kedua yang tumpang tindih waktunya di scope `(academicYearId, campusId, masterLevelId)` yang sama → API menolak dengan 400.
- [ ] **AC-6:** Admin menonaktifkan satu-satunya window ACTIVE di suatu scope → registrasi untuk scope tsb **tertutup** (form tidak tampil), bukan terbuka.
- [ ] **AC-7:** `GET /v1/ccaRegistrationWindows/:id` dengan id tidak ada → 404 `NotFoundException`.
- [ ] **AC-8:** Perilaku lama (backward compatible) yang masih valid tetap berjalan: window ACTIVE campus-wide (`masterLevelId IS NULL`) tetap berlaku untuk semua level yang tidak punya window level spesifik.

## UI / UX Changes

### Affected Portals

- [ ] Admin (client/)
- [x] Student (client-student/)
- [ ] Teacher (client-teacher/)

### Deskripsi

Halaman `StudentCCARegistration.jsx`:

- Saat `isWindowConfigured === false` untuk scope siswa (academic year aktif + campus siswa + level siswa):
  - Jangan render form registrasi & daftar CCA (section yang berisi `ccaYears` cards, form pilihan, tombol daftar).
  - Render empty state, mis. "Pendaftaran CCA belum dibuka untuk level Anda. Silakan kembali lagi nanti."
- Perilaku lain (kuota, gender, `canRegisterForMore`, dsb.) tidak berubah.

## API Changes

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v1/ccaYears` (existing) | **Modified** — response item `CcaYearDto` ditambah field `isWindowConfigured: boolean` (true bila ada window dikonfigurasi untuk scope CCA tsb, ACTIVE atau INACTIVE sesuai keputusan EC-02) |
| POST | `/v1/ccaRegistrationWindows` (existing) | **Modified** — tambah validasi `opensAt < closesAt` + cek overlap scope |
| PATCH | `/v1/ccaRegistrationWindows/:id` (existing) | **Modified** — validasi sama seperti create |
| GET | `/v1/ccaRegistrationWindows/:id` (existing) | **Modified** — throw `NotFoundException` jika id tidak ada |

> Tidak ada endpoint baru. Sinyal gating dikirim lewat DTO `CcaYearDto` (bukan endpoint terpisah) agar satu kali fetch, konsisten dengan pola `isRegistrationOpen` yang sudah ada.

## Database Changes

### New Tables

Tidak ada.

### Modified Tables

Tidak ada perubahan kolom.

### Migrations

Opsional (disarankan, untuk enforce di level DB):

- **Index unik parsial** untuk mencegah duplikasi window aktif di scope yang sama:
  - `UNIQUE (academic_year_id, campus_id, master_level_id)` — hanya jika keputusan bisnis adalah "maksimal 1 window per scope" (lihat EC-05). Jika mengizinkan beberapa window non-overlap per scope, gunakan constraint `EXCLUDE`/cek overlap via aplikasi, bukan unique index.
  - Catatan: `master_level_id` nullable — pastikan partial index (`WHERE master_level_id IS NOT NULL`) untuk scope campus-wide.

> Keputusan final mengikuti hasil `edgecases.md` EC-05. Tanpa migrasi pun, validasi di service sudah cukup untuk MVP.

## Business Rules / Validation

1. **Tanpa window = tertutup** — Jika tidak ada window yang dikonfigurasi untuk scope `(academicYearId, campusId, masterLevelId)`, form registrasi CCA siswa TIDAK ditampilkan (ini perubahan dari perilaku lama "open by default").
2. **Window berlaku untuk scope**: level-spesifik (`masterLevelId = X`) menang atas campus-wide (`masterLevelId IS NULL`). Jika window level-spesifik ada tapi tidak sedang open → registrasi tertutup (tidak fallback ke campus-wide). Ini perilaku `isRegistrationOpen()` yang sudah ada (line 114-164) — dipertahankan.
3. **`opensAt < closesAt`** — wajib divalidasi di create & update.
4. **Tidak boleh ada window yang tumpang tindih** di scope yang sama (keputusan overlap policy di EC-05).
5. **`INACTIVE` = sengaja ditutup** (rekomendasi, konfirmasi di EC-02) — window INACTIVE tetap dihitung sebagai "dikonfigurasi", sehingga registrasi tidak fallback ke open.
6. **Backward compatible untuk kasus valid** — window campus-wide tetap berlaku untuk semua level tanpa window level-spesifik; window level-spesifik yang sedang open tetap membuka registrasi untuk level tsb.

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| `closesAt <= opensAt` | 400 | `Registration window closesAt must be after opensAt.` |
| Overlap window di scope sama | 400 | `An active registration window already exists for this academic year, campus and level.` |
| Window id tidak ditemukan | 404 | `Registration window with id {id} not found.` |
| (Backend guard, existing) Registrasi di luar window | 400 | `CCA registration is currently closed for your campus and level.` (tetap, dari `assertWindowOpen`) |

## Dependencies

- `bbs-feature/cca-registration-level-filter/` — related known issue: `assertWindowOpen()` memakai `ccaYear.masterLevelId` (level CCA) bukan level siswa. Gating form di spec ini tidak menggantikan validasi level; keduanya perlu diimplementasikan.
- Frontend `useFromApi`/`useResourceMapper` pattern di `bbs/client-student` — gating mengikuti pola data loading yang sudah ada (line 56-105 `StudentCCARegistration.jsx`).
- Entity `CcaRegistrationWindow` + migration `1782360000004-CreateCcaRegistrationWindow` — skema sudah tersedia, tidak berubah.
