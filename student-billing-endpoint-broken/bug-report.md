---
title: Finance — Endpoint List Student Billing rusak (500) karena Service kosong & tanpa versi URI
status: open
severity: minor
product: BBS LMS
portal: Admin
author: System Analyst
date: 2026-09-30
jam: (tidak tersedia — laporan berbasis eksekusi test API + code review)
---

# Student Billing — `GET /student-billing` mengembalikan 500 (`findAll is not a function`) dan Terdaftar di Path Tanpa `/v1`

## Summary

Ditemukan saat eksekusi test case read-only (TC-015, folder
`operate-smartbag/test-cases/`, 2026-09-30). Modul `student-billing` di api_nest
adalah **scaffold NestJS yang tak pernah diimplementasi**: controller punya 4 route
lengkap (POST/GET/GET :id/DELETE), tetapi **semua method di service masih di-comment**.
Akibatnya:

1. `GET {base}/api/student-billing` → **HTTP 500** dengan pesan internal
   `this.studentBillingService.findAll is not a function`
2. Endpoint **tidak melayani di `/api/v1/`** karena `@Controller('student-billing')`
   tanpa `version: '1'` — frontend yang selalu menambah `v1` (pola
   `makeApiRequest.js`) tidak akan pernah bisa memakainya (404).

Dua masalah ini membuat modul terlihat "hidup" tapi sepenuhnya mati, dan 500
dengan pesan internal mengekspos detail implementasi ke caller.

**Ekspektasi:** endpoint list student billing berfungsi di `/api/v1/student-billing`
mengembalikan daftar billing siswa (envelope JSON:API), atau — bila memang belum
jadi — seluruh route modul tidak boleh terekspos.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Buffy (AI agent) — eksekusi TC-015 |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | 35008 (`mahdy.klabs`, Super Admin) |
| Portal URL | — (dites langsung ke API) |
| Environment API | `https://api.binabangsaschool.dev` (production) |
| Data konteks (class ID / daId / tanggal) | query `?limit=1`; dieksekusi 2026-09-30 |
| Browser / OS | curl (bukan browser) |

---

## Steps to Reproduce

1. Dapatkan token admin (Bearer) yang punya permission READ BILLING.
2. Hit endpoint sesuai konvensi API app (semua endpoint lain memakai `/api/v1/`):

```bash
curl -s "https://api.binabangsaschool.dev/api/v1/student-billing?limit=1" \
  -H "Authorization: Bearer $TOKEN" -o /dev/null -w "%{http_code}\n"
# → 404 (route tidak terdaftar di versi 1)

curl -s "https://api.binabangsaschool.dev/api/student-billing?limit=1" \
  -H "Authorization: Bearer $TOKEN"
# → 500 {"statusCode":500,"code":500,"message":"this.studentBillingService.findAll is not a function"}
```

**Actual Result:**
- `/api/v1/student-billing` → 404 (kontrak frontend tidak terpenuhi)
- `/api/student-billing` → 500 dengan pesan error internal

**Expected Result:**
- List billing tersedia di `/api/v1/student-billing` (200 + envelope), atau route
  dimatikan sampai implementasi selesai (tidak boleh 500).

---

## Root Cause Analysis

### Bug #1 — Service kosong (scaffold belum diimplementasi) — `student-billing.service.ts` (seluruh file)

```typescript
// api_nest/src/modules/student-billing/student-billing.service.ts
@Injectable()
export class StudentBillingService {
  // create(createStudentBillingDto: CreateStudentBillingDto) { ... }
  //
  // findAll() {
  //   return `This action returns all studentBilling`;
  // }
  // ... (semua method masih comment)
}
```

Controller memanggil method yang tidak ada:

```typescript
// api_nest/src/modules/student-billing/student-billing.controller.ts:31-40
@Get()
@CheckPermissions([{ action: ACLTypeEnum.READ, subject: ModulesTypeEnum.BILLING }])
findAll() {
  return this.studentBillingService.findAll();  // ← runtime error: not a function
}
```

### Bug #2 — Controller tanpa `version: '1'` — `student-billing.controller.ts:16`

```typescript
@Controller('student-billing')   // ← tanpa { version: '1', path: ... }
```

Bandingkan dengan modul lain, mis. `billing-report.controller.ts:10`:

```typescript
@Controller({ version: '1', path: 'billingReports' })
```

`main.ts` mengaktifkan URI versioning `VERSIONING_PREFIX = 'v1'` — controller tanpa
properti `version` tidak pernah dilayani di `/api/v1/*`. Karena semua portal
menyusun URL sebagai `base + '/api/' + 'v1' + path`, endpoint ini mustahil dijangkau
dari frontend.

### Konteks

Modul `billing` yang sesungguhnya berfungsi ada di `modules/billing/` dan
`modules/invoice/` (keduanya 200 saat TC-015). Pembuatan billing siswa dari admin
kemungkinan memakai jalur admission (`addBilling`, lihat wiki
`student-admission.md`) atau `billingReports` — bukan modul scaffold ini.

---

## Bukti dari Test (tanpa Jam)

| Sumber | Temuan |
|--------|--------|
| **API probe** | `GET /api/v1/student-billing?limit=1` → **404**; `GET /api/student-billing?limit=1` → **500** `{"statusCode":500,"code":500,"message":"this.studentBillingService.findAll is not a function"}` |
| **Code** | `student-billing.service.ts`: seluruh method di-comment; `student-billing.controller.ts:16`: `@Controller('student-billing')` tanpa version; `student-billing.controller.ts:20,31,42,67`: 4 route aktif (POST, GET, GET :id, DELETE) |
| **Endpoint lain (sehat)** | `invoices` 200, `wallets` 200, `masterProducts` 200 (TC-015) |
| **app.module.ts** | `StudentBillingModule` terdaftar (baris 409) — modul aktif meski kosong |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/student-billing/student-billing.service.ts` | Semua method commented — runtime "not a function" |
| Backend Controller | `api_nest/src/modules/student-billing/student-billing.controller.ts` | 4 route terekspos tanpa implementasi; path tanpa `version: '1'` |
| Backend Module | `api_nest/src/modules/student-billing/student-billing.module.ts` | Terdaftar di `app.module.ts:409` sehingga route aktif |
| Frontend | semua portal (pola `makeApiRequest.js` menambah `v1`) | Endpoint tidak terjangkau bila dipakai |

---

## Proposed Solution Options

### Option A: Matikan route sampai implementasi (Recommended — cepat & aman)

Kosongkan controller (komentari 4 route) atau hapus modul dari `app.module.ts`
sampai fitur benar-benar diimplementasi. Efek: tidak ada lagi 500 yang bocor;
tidak ada perubahan perilaku yang dirasakan user (endpoint memang tidak terpakai
portal mana pun). Prioritas rendah karena tidak ada dampak user aktif.

### Option B: Implementasi penuh `findAll` (Recommended bila fitur dibutuhkan)

1. Tambah `version: '1'` pada `@Controller` agar konsisten konvensi:
   `@Controller({ version: '1', path: 'student-billing' })`.
2. Implementasi `StudentBillingService.findAll()` dengan pagination + filter
   (`campusId`, `academicYearId`, `studentId`) dan **eager-load minimal**
   (pelajaran dari bug `student-list-no-filter-timeout` dan
   `topics-list-eager-loading-timeout` di folder ini).
3. Ikuti pola `billing-report` / `invoice` untuk envelope JSON:API.

### Option C: Validasi preventif modul scaffold lain

Audit modul lain yang polanya sama (controller memanggil service kosong) dengan
grep `is not a function` pattern — cegah 500 serupa sebelum ditemukan user.

---

## Notes

- Sumber temuan: `operate-smartbag/test-cases/TC-015-finance-billing/result.md`
  (AI Operate Test, 2026-09-30).
- Tidak ada indikasi fitur ini dipakai portal mana pun saat ini → severity **minor**,
  tapi 500 dengan pesan internal adalah kebocoran informasi yang tidak perlu.
- Terkait: wiki `finance.md` menyebut pembuatan billing via Finance nav — jalur itu
  sehat (invoices 200); modul ini hanya scaffold Nest CLI yang terlupa.
