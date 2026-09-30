---
title: Auth Impersonate — portalType yang tidak tersedia untuk employee mengembalikan HTTP 500 (seharusnya 400)
status: open
severity: minor
product: BBS LMS
portal: Admin
author: System Analyst
date: 2026-09-30
jam: (tidak tersedia — laporan berbasis eksekusi test API + code review)
---

# Auth Impersonate — `portalType` Mismatch Mengembalikan 500 dengan Pesan yang Sebenarnya Sudah Jelas

## Summary

Ditemukan saat eksekusi test case read-only TC-038 (folder
`operate-smartbag/test-cases/`, 2026-09-30). Endpoint
`POST /api/v1/auth/impersonate/login` menolak kombinasi employee × portal yang
tidak sah **dengan pesan yang tepat**, tetapi statusnya **HTTP 500**:

```json
{"statusCode":500,"code":500,"message":"Portal type 3 not available for this employee"}
```

Kasus repro: employee 100069 (`sample.teacher`, hanya portal TEACHER) dipaksa
`portalType: "STUDENT"`. Ini adalah **kesalahan input pemanggil**, bukan kegagalan
server → seharusnya **400**.

**Ekspektasi:** HTTP 400 dengan pesan yang sama (atau lebih ramah, mis.
"Portal type STUDENT not available for this employee").

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Buffy (AI agent) — eksekusi TC-038 |
| User ID (`selfUser`) | 35008 (`mahdy.klabs`, Super Admin) |
| Environment API | `https://api.binabangsaschool.dev` (production) |
| Data konteks | targetEmployeeId 100069 (`sample.teacher`); portalType `STUDENT`; dieksekusi 2026-09-30 |
| Browser / OS | curl |

---

## Steps to Reproduce

```bash
ADMIN=<admin token>
# 1. kontrak dua-tahap: cek portal yang tersedia (normal → 200, availablePortals: ["TEACHER"])
curl -s -X POST "https://api.binabangsaschool.dev/api/v1/auth/impersonate/check" \
  -H "Authorization: Bearer $ADMIN" -H "Content-Type: application/json" \
  -d '{"data":{"attributes":{"targetEmployeeId":100069}}}'
# 2. paksa login ke portal yang tidak tersedia
curl -s -w "\nHTTP:%{http_code}\n" -X POST \
  "https://api.binabangsaschool.dev/api/v1/auth/impersonate/login" \
  -H "Authorization: Bearer $ADMIN" -H "Content-Type: application/json" \
  -d '{"data":{"attributes":{"targetEmployeeId":100069,"portalType":"STUDENT"}}}'
# → HTTP 500, body: {"statusCode":500,"message":"Portal type 3 not available for this employee"}
```

**Actual Result:** HTTP 500. Pesan menyebut kode numerik enum (`Portal type 3`)
alih-alih nama (`STUDENT`).

**Expected Result:** HTTP 400 dengan pesan nama portal (`STUDENT not available ...`),
karena pemanggil diberi tahu portal tersedia lewat `impersonate/check`.

---

## Root Cause Analysis

### Bug — `throw new Error(...)` generik di service — `auth.service.ts:1057-1062`

```typescript
// api_nest/src/modules/auth/auth.service.ts:1057-1062
if (!availablePortals.includes(portalType)) {
  throw new Error(
    `Portal type ${portalType} not available for this employee`,
  );
}
```

`throw new Error` = exception internal → Nest `ExceptionFilter` memetakan ke 500.
Bandingkan dengan sibling di **controller** yang sudah benar:

```typescript
// api_nest/src/modules/auth/auth.controller.ts:461-463
if ('availablePortals' in result) {
  throw new BadRequestException('Portal type is required for login');  // ← 400
}
```

Perbaikan = ganti `new Error` → `new BadRequestException` (impor dari
`@nestjs/common`) — satu baris.

Faktor pendukung:
- Pesan memakai nilai enum numerik (`portalType 3`) karena DTO menerima value enum
  tanpa normalisasi ke nama — sekalian pakai nama (`STUDENT`) agar ramah dibaca.
- Respons 500 yang membocorkan pesan internal juga menghilangkan petunjuk bagi
  pemanggil sah (UI admin) untuk memperbaiki permintaannya.

---

## Bukti dari Test (tanpa Jam)

| Sumber | Temuan |
|--------|--------|
| **API probe (happy path)** | `impersonate/check` 100069 → 200 `availablePortals: ["TEACHER"]`; `impersonate/login` TEACHER → 200 token; `selfUser` dgn token guru → 200 id 100069 |
| **API probe (bug)** | `impersonate/login` portalType STUDENT → **500** + pesan `Portal type 3 not available for this employee` |
| **Code** | `auth.service.ts:1058-1061` `throw new Error(...)`; kontras: `auth.controller.ts:462` `BadRequestException` |
| **Rate limit** | 4× POST `impersonate/check` beruntun = 200 semua (catatan: pitfall lama "impersonate 502" tidak berlaku lagi — dicabut dari `operate-smartbag/actions/06-fill-grades/result.md`) |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/auth/auth.service.ts` (line 1058-1061) | `throw new Error` → 500 untuk user error |
| Backend DTO | `api_nest/src/modules/auth/dto/impersonate.dto.ts` (portalType) | enum numeric dikirim mentah ke pesan error |
| Pemanggil | UI admin switch-account / automation operate | Menerima 500 menyesatkan untuk input yang bisa diperbaiki |

---

## Proposed Solution Options

### Option A: Ganti ke `BadRequestException` (Recommended — 1 baris)

```typescript
// auth.service.ts
import { BadRequestException } from '@nestjs/common';
...
throw new BadRequestException(
  `Portal type ${portalType} not available for this employee`,
);
```

Tetap tampilkan nama enum (`portalType` dari request = nama, `STUDENT`) — hindari
mencetak nilai numerik. Perilaku lain tidak berubah.

### Option B: Validasi di DTO

`@IsEnum(AuthEntityTypeEnum)` sudah ada? bila portalType salah **value** (bukan
employee mismatch), DTO bisa menolak 400 lebih awal — tapi kasus ini adalah
**mismatch employee×portal** (STUDENT sah sebagai enum, hanya tidak tersedia untuk
guru), jadi tetap butuh perbaikan Option A di service.

### Option C: Audit serupa

Grep `throw new Error(` di service yang dipanggil dari controller publik — temukan
500-seharusnya-400 lain dengan pola sama (di luar scope laporan ini).

---

## Notes

- Sumber temuan: `operate-smartbag/test-cases/TC-038-impersonate-and-rate-limit/result.md`.
- Severity minor: endpoint admin-only, tidak ada dampak data. Namun berada di jalur
  impersonate yang dipakai automation operate — 500 memicu retry logic yang tidak
  perlu di script (`post_grades.py` dsb).
- Fakta pendukung dari eksekusi yang sama: rate limit 502 pada impersonate/check
  tidak terulang (4× beruntun = 200) — pitfall lama action 06 sudah dicabut.
