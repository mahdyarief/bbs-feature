---
feature: Security Deposit — status benar (404, bukan 500) untuk siswa tanpa deposit
slug: security-deposit-no-deposit-500
status: draft
author: System Analyst (dari temuan AI Operate Test TC-039)
date: 2026-09-30
target_release: TBD
related_bug_reports:
  - security-deposit-no-deposit-500
---

# Security Deposit — 404 untuk Siswa Tanpa Deposit

## Overview

Brief perbaikan kode (untuk engineer backend) atas error handling modul
`security-deposit` (`api_nest/src/modules/security-deposit/`). Dokumen ini HANYA
spesifikasi — implementasi dikerjakan tim backend.

## Problem / Motivation

TC-039 (2026-09-30): `GET /api/v1/securityDeposits/:studentId` untuk siswa yang
memang **tidak punya** security deposit mengembalikan **500**, padahal itu
kondisi bisnis normal. Service:

```ts
const securityDeposit = await SecurityDeposit.findOne({ where: { studentId }, ... });
if (!securityDeposit) {
  throw new Error('No security deposit found for student');   // → HTTP 500
}
```

`throw new Error` = failure internal → 500. Kondisi "resource tidak ada" adalah
user error → semestinya **404** (`NotFoundException`), persis klasifikasi **A**
dalam audit `generic-error-throw-audit` (24 lokasi user-error di api_nest).
Modul ini adalah tambahan konkret ke daftar audit itu.

Dampak nyata: normal flow UI ("buka deposit siswa") untuk mayoritas siswa
menampilkan error server, dan monitoring 500 terpolusi alarm palsu.

## Scope

### In Scope
- `getStudentSecurityDeposit()`: ganti `throw new Error` → `NotFoundException`
  (atau return `data: null` — lihat EC-01).
- Cek 2 helper error lain di modul ini (`RefundSecurityDepositError`,
  `SecurityDepositInsufficientFundError` di `errors/ResourceError`): pastikan
  dipetakan ke 4xx yang tepat (400/409), bukan 500.

### Out of Scope
- Perubahan skema / tabel `security_deposit`, `security_deposit_transaction_history`.
- Fitur refund/top-up baru.
- Audit penuh 37 lokasi `throw new Error` — sudah terdokumentasi di
  `generic-error-throw-audit/audit.md`; modul ini tambahan baru di luar grep itu.

## User Stories

### As an admin (finance)
I want the deposit page for a student without a deposit to show an empty state
So that normal usage does not look like a server failure.

### As a backend engineer
I want resource-not-found to be mapped to 404 consistently
So that monitoring 500s indicate real bugs only.

## Acceptance Criteria

- [ ] `GET /api/v1/securityDeposits/:studentId` untuk siswa tanpa deposit → **404**
      (atau 200 `data:null` sesuai keputusan EC-01) dengan pesan jelas.
- [ ] Siswa dengan deposit → 200 envelope berisi deposit + transaksinya (perilaku
      lama dipertahankan).
- [ ] `securityDeposits/:studentId/unpaidBillings` untuk siswa tanpa unpaid billing
      → 200 `data:[]` (bukan 500) — verifikasi kebalikannya juga benar.
- [ ] Re-run probe TC-039: `/securityDeposits/31001` tidak lagi 500.

## API Changes

| Method | Path | Perubahan |
|--------|------|-----------|
| GET | `/api/v1/securityDeposits/:studentId` | tidak-punya-deposit → 404 (bukan 500) |

## Database Changes

Tidak ada.

## Business Rules / Validation

1. "Tanpa deposit" = tidak ada baris `security_deposit` untuk `studentId` itu
   (bukan saldo 0 — saldo 0 tetap 200 dengan data).
2. Pesan error dipertahankan: `No security deposit found for student`.
3. Konsisten dengan konvensi modul lain di api_nest yang memakai
   `NotFoundException` untuk resource yang tidak ada.

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| Siswa tidak punya deposit | **404** | No security deposit found for student |
| Siswa tidak ada sama sekali | 404 | Student not found (bila divalidasi) |
| Refund melebihi saldo | 400/409 | Insufficient fund (via ResourceError yang tepat) |

## Dependencies

- Repo `smartbag/api_nest` (implementasi oleh tim backend).
- Bukti & reproduksi: `security-deposit-no-deposit-500/bug-report.md`.
- Konteks pola: `generic-error-throw-audit/audit.md` (klasifikasi A).
- Verifikasi pasca-deploy: re-run probe TC-039 path ini.

## Open Questions (lihat edgecases.md)

- EC-01: 404 vs 200 dengan `data: null`.
- EC-02: perlukah sibling route `/unpaidBillings` di-audit juga.
