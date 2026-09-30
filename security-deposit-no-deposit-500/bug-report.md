---
title: Security Deposit — 500 bukan 404 saat siswa tidak punya deposit (throw new Error untuk user error)
status: open
severity: minor
product: BBS LMS
portal: Admin
author: System Analyst
date: 2026-09-30
jam: (tidak tersedia — laporan berbasis eksekusi test API + code review)
---

# securityDeposits — `GET /api/v1/securityDeposits/:studentId` → 500 untuk siswa tanpa deposit

## Summary

Ditemukan saat TC-039 full controller path sweep (2026-09-30). Service
`security-deposit/security-deposit.service.ts`:

```ts
const securityDeposit = await SecurityDeposit.findOne({ where: { studentId }, ... });
if (!securityDeposit) {
  throw new Error('No security deposit found for student');   // → HTTP 500
}
```

"Student tidak punya security deposit" adalah **kondisi bisnis normal**
(sebagian besar siswa memang tidak punya deposit), tapi di-map ke **500**.
Pola persis klasifikasi **A** di audit `generic-error-throw-audit`
(user error / resource tidak ada → harusnya 404 via `NotFoundException`).

**Ekspektasi:** siswa tanpa deposit → **404** (atau 200 dengan `data: null`),
bukan 500.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Buffy (AI agent) — eksekusi TC-039 sweep |
| User ID | 35008 (`mahdy.klabs`, Super Admin) |
| Environment API | `https://api.binabangsaschool.dev` (production) |
| Data konteks | siswa PIK TEST 31001 (tanpa deposit); tabel `security_deposit` berisi 5.948 baris, student_id min 50 |
| Browser / OS | curl |

## Steps to Reproduce

```bash
curl -s "https://api.binabangsaschool.dev/api/v1/securityDeposits/31001" \
  -H "Authorization: Bearer $TOKEN" -o /dev/null -w "%{http_code}\n"
# → 500
```

## Actual vs Expected

| | Nilai |
|---|---|
| Actual | **500** `Internal Server Error` |
| Expected | **404** via `NotFoundException` (pola modul lain), atau 200 `data:null` |

## Impact

- Normal flow UI ("cek deposit siswa yang tidak punya deposit") memunculkan
  error server, bukan state kosong — berisiko alarm palsu di monitoring
  (500 biasanya berarti bug server).

## Rekomendasi

Ganti `throw new Error(...)` → `throw new NotFoundException('No security deposit
found for student')`. Sekalian audit 3 sibling di modul ini
(`RefundSecurityDepositError`, `SecurityDepositInsufficientFundError` di
`errors/ResourceError` — cek status code masing-masing).
