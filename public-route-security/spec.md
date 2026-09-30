---
feature: Keamanan Route Publik — sync routes tanpa guard & static token integrasi attendance
slug: public-route-security
status: draft
author: System Analyst (dari temuan AI Operate Test TC-042)
date: 2026-09-30
target_release: TBD
related_bug_reports: []
related_test_cases:
  - operate-smartbag/test-cases/TC-042-public-auth-surface
---

# Keamanan Route Publik — Sync Routes & Static Token Integrasi

## Overview

Brief perbaikan kode (untuk engineer backend) atas permukaan `@Public()` yang
dipetakan penuh TC-042: ±45 route publik, mayoritas sah (auth, katalog master),
tetapi ada **2 area berisiko**: (1) ±18 route mutasi `sync*`/`checkOverdues`
ber-flag `@Public` tanpa proteksi terlihat, (2) integrasi attendance
gate/hardware memakai static token hardcoded di source.

## Problem / Motivation

TC-042 (2026-09-30) menemukan:

1. **Static token hardcoded** — `attendance.service.ts:1314`:

```ts
if (!options.token || options.token !== 'e5N7Zq4Szsw3TbFp5o8rNh')
  AuthLinkTokenError();
```

`POST /attendances/integration` & `/integration/bulk` menulis attendance dari
gate hardware; satu-satunya proteksi = token literal di source. Siapa pun yang
bisa membaca repo (atau bundle FE bila bocor) bisa menulis attendance siswa.

2. **Route `sync*` @Public tanpa guard terlihat** — `grades/sync`,
`subjects/syncMasterLevels`, `topics/syncMasterLevels`,
`gradeWeights/syncMasterLevels`, `ftp-grade-setting/syncMasterLevels`,
`gradebookTables/syncMasterLevelAndSubject`, `reportSettings/syncMasterLevels`,
`learningOutcomeAnalyses/syncGrades`, `learningIndicatorGrades/syncMark`,
`studentFtpConducts/sync`, `studentCcaGrades/sync`,
`students/syncCurrentClassYears`, `employees/syncUsers`,
`employees/syncProfiles`, `masterSubjects/syncMasterProgrammes`,
`billing/checkOverdues`. Pembanding yang benar: `admin/cronTest` memakai
`validateCronPassword(options.password)` — pattern yang sudah ada di codebase.

3. **Info leak ringan**: `campuses/:id` dan `files` (metadata) bisa dibaca tanpa
token — perlu keputusan produk apakah disengaja.

## Scope

### In Scope
- Rotasi + pemindahan token integrasi attendance ke env secret (atau ganti
  mekanisme HMAC/signature + timestamp).
- Tambah `validateCronPassword` (atau guard setara) ke semua route `sync*`
  yang dipanggil scheduler/cron.
- Keputusan eksplisit untuk `campuses/:id` & `files` GET publik.

### Out of Scope
- Auth portal utama (login/refresh — sudah sehat).
- Rate limiting global (brief infrastruktur terpisah bila diminta).
- Menguji eksekusi sync routes di produksi (write — tidak dilakukan AI).

## User Stories

### As a security reviewer
I want mutation endpoints to require a rotating secret or auth
So that attendance and sync data cannot be written by anyone.

### As an integration device (gate hardware)
I want a documented, rotatable credential flow
So that hardware integration keeps working after rotation.

## Acceptance Criteria

- [ ] Tidak ada lagi secret literal di source (`grep -rn "e5N7Zq4" src/` → 0).
- [ ] Semua route `sync*` menolak tanpa password cron (401/403/400 dengan
      `validateCronPassword` atau guard setara), kecuali ada alasan dokumentasi.
- [ ] Rotasi token tidak mem-break device gate (koordinasi + jendela overlap).
- [ ] Re-run TC-042 Step 1: inventaris publik hanya memuat route yang sah.

## API Changes

| Method | Path | Perubahan |
|--------|------|-----------|
| POST | `/api/v1/attendances/integration` (+ `/bulk`) | token dari env (`ATTENDANCE_INTEGRATION_TOKEN`), rotatable; atau HMAC header |
| GET/POST | semua route `sync*` (±18) | tambah query `password` wajib (pola `cronTest`) atau header secret |

## Database Changes

Tidak ada.

## Business Rules / Validation

1. Secret di env/secret manager, TIDAK di repo; rotasi berkala.
2. Error salah token tetap 400/401 generik — jangan bocorkan mana yang salah.
3. Device gate: minta tim hardware meng-update konfigurasi saat rotasi
   (jendela overlap maksimal 1 minggu, dua token diterima sementara).

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| Token/password kosong | 400 | credentials required |
| Token/password salah | 401 | invalid credentials (pesan generik) |

## Dependencies

- Repo `smartbag/api_nest` + tim hardware gate (untuk rotasi token).
- Bukti: `operate-smartbag/test-cases/TC-042-public-auth-surface/result.md`.
- Pattern yang sudah ada: `admin/cronTest` + `validateCronPassword`.

## Open Questions (lihat edgecases.md)

- EC-01: static-token-plus-env vs HMAC signature untuk attendance integration.
- EC-02: route sync mana yang benar-benar dipanggil cron eksternal (perlu
  konfirmasi devops — jangan mengunci route yang dipakai scheduler produksi).
- EC-03: campuses/:id & files publik — disengaja atau dikunci?
