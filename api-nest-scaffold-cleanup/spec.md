---
feature: API Nest Scaffold Cleanup — 4 modul setengah jadi di produksi
slug: api-nest-scaffold-cleanup
status: draft
author: System Analyst (dari temuan AI Operate Test TC-039/TC-040)
date: 2026-09-30
target_release: TBD
related_bug_reports:
  - cca-year-coordinators-empty-500
  - student-billing-endpoint-broken
---

# API Nest Scaffold Cleanup — 4 Modul Setengah Jadi

## Overview

Brief perbaikan kode (untuk engineer backend) atas **4 modul scaffold NestJS yang
tidak pernah diselesaikan** namun teregistrasi di `app.module` produksi. Dokumen
ini HANYA spesifikasi — implementasi dikerjakan tim backend di repo
`smartbag/api_nest`, bukan oleh AI operate.

## Problem / Motivation

Sweep TC-039 (2026-09-30) menemukan 4 titik scaffold tidak selesai:

| # | Modul | Kondisi kode | Dampak produksi |
|---|---|---|---|
| 1 | `student-billing` (`student-billing/`) | 4 route lengkap (POST/GET/:id/DELETE), **seluruh method service di-comment**; `@Controller('student-billing')` tanpa `version: '1'` | `GET /api/student-billing` → **500** `findAll is not a function`; frontend yang selalu pakai `/v1` dapat 404 |
| 2 | `ccaYearCoordinators` (`cca-year-coordinator/`) | `@Get() findAll()` body-nya hanya komentar `// return this.service.findAll();` | `GET /api/v1/ccaYearCoordinators` → **500** (method tanpa return) |
| 3 | `cca-grade` (`cca-grade-term-setting/`) | Controller `@Controller('cca-grade')` **tanpa satu route pun** (hanya constructor) | 404 di semua URI; modul mati terekspos di katalog |
| 4 | `ftp-evaluation-setting` (`ftp-evaluation-setting/`) | Controller `@Controller('ftp-evaluation-setting')` **tanpa satu route pun** | 404 di semua URI (koreksi dugaan awal TC-028 bahwa modul ini hidup) |

Pola bersama: scaffold generator NestJS dibiarkan setengah jadi, tapi modul tetap
di-`import` di `app.module.ts` sehingga terlihat "ada" di katalog API padahal
tidak berfungsi. Ini menyesatkan konsumen API dan menghasilkan 500/404 yang
mengotori monitoring.

## Scope

### In Scope
- Keputusan per modul: **implement** atau **un-expose** (lihat edgecases).
- Un-expose = hapus/comment modul dari `app.module.ts` + (opsional) hapus folder
  modul, atau tambahkan deklarasi route yang benar bila implementasi ditunda.
- Bila implement `student-billing`: daftarkan `version: '1'` agar melayani di
  `/api/v1/student-billing` (konsisten pola mayoritas).

### Out of Scope
- Fitur bisnis baru di luar yang sudah terkontrak scaffold (mis. fitur billing
  lanjutan, CCA grade term setting yang belum dispesifikasi produk).
- Refactor pola versioning global (`main.ts`) — brief terpisah bila diminta.
- Modul lain yang sudah berfungsi.

## User Stories

### As a backend engineer
I want a clear decision per unfinished scaffold (implement or un-expose)
So that the production API catalog contains only working endpoints.

### As an API consumer (frontend/FE)
I want no 500 from modules that were never implemented
So that error monitoring is not polluted by known-dead endpoints.

## Acceptance Criteria

- [ ] Tidak ada lagi 500 yang berasal dari `student-billing` dan `ccaYearCoordinators`.
- [ ] Katalog API (Swagger/route list) tidak lagi memuat modul yang memang tidak
      jadi (`cca-grade`, `ftp-evaluation-setting`) ATAU keduanya sudah implement penuh.
- [ ] `GET /api/v1/student-billing` mengembalikan 200 envelope, ATAU route-nya
      tidak terekspos sama sekali.
- [ ] Verifikasi ulang sweep TC-039 pada path-path ini: tidak ada perubahan status
      yang tidak terjelaskan.

## API Changes

| Opsi | Method + Path | Perubahan |
|---|---|---|
| Implement student-billing | `GET /api/v1/student-billing` | Service diimplement (findAll paginated + filter), controller diberi `version: '1'` |
| Un-expose | `DELETE` dari routing | Hapus `StudentBillingModule`, `CcaGradeTermSettingModule` (cca-grade), `FtpEvaluationSettingModule` dari `app.module.ts` imports |
| Implement ccaYearCoordinators | `GET /api/v1/ccaYearCoordinators` | Uncomment/implement `findAll()` di controller + service |

## Database Changes

Tidak ada perubahan skema. Tabel terkait sudah ada:
`student_billing` (FK payment→billing), `cca_year_coordinator`.

## Business Rules / Validation

1. Sebelum implement `student-billing`, konfirmasi ke PM: apakah fitur billing
   list ini memang dibutuhkan (scaffold bisa jadi sisa eksperimen).
2. Un-expose tidak boleh menghapus tabel/data — hanya route/registrasi modul.
3. Bila ada frontend yang masih memanggil path-path ini (walau 500/404),
   koordinasikan dulu sebelum menghapus route.

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| Modul dihapus tapi masih dipanggil | 404 | Standard NestJS not found |
| Implement tapi data kosong | 200 | `{data: [], meta: {...}}` — bukan 500 |

## Dependencies

- Repo `smartbag/api_nest` (implementasi oleh tim backend).
- Bug reports terkait (bukti & reproduksi): `cca-year-coordinators-empty-500/`,
  `student-billing-endpoint-broken/`.
- Verifikasi pasca-deploy: re-run TC-039 Step sweep pada 4 path ini.

## Open Questions (lihat edgecases.md)

- EC-01: implement vs un-expose per modul (4 keputusan).
- EC-02: apa yang terjadi pada frontend yang masih memanggil path ini.
