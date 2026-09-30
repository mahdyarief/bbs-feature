---
title: Students API — GET /students tanpa filter campus selalu 502 (eager-load relasi berat + permit scan semua campus)
status: open
severity: major
product: BBS LMS
portal: Admin | Teacher
author: System Analyst
date: 2026-09-30
jam: (tidak tersedia — laporan berbasis eksekusi test API + code review)
---

# Students — `GET /api/v1/students?limit=1` Tanpa Filter Campus Mengembalikan 502 Bad Gateway

## Summary

Ditemukan saat eksekusi test case read-only (TC-006, folder
`operate-smartbag/test-cases/`, 2026-09-30). `GET /api/v1/students?limit=1`
**tanpa filter campus** mengembalikan **502 Bad Gateway (nginx)** secara konsisten
(2 dari 2 percobaan, termasuk setelah delay 8 detik). Dengan filter
`?campusId=23&limit=5` endpoint sehat (**200**), dan detail `GET /students/100215`
juga **200**.

**Ekspektasi:** list students dengan `limit=1` seharusnya merespons cepat tanpa
memedulikan ada/tidaknya filter campus. Timeout terjadi karena kombinasi
**eager-load relasi berat** di service dan (indikasi kuat) **pemindaian permit
campus untuk seluruh database** sebelum limit diterapkan — bukan sekadar "data
banyak".

Ini risiko performa produksi, bukan error kontrak: nginx memutus koneksi (gateway
timeout) sebelum Nest selesai menyusun response.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Buffy (AI agent) — eksekusi TC-006 |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | 35008 (`mahdy.klabs`, Super Admin) |
| Portal URL | — (dites langsung ke API) |
| Environment API | `https://api.binabangsaschool.dev` (production) |
| Data konteks (class ID / daId / tanggal) | query `?limit=1`; dieksekusi 2026-09-30 |
| Browser / OS | curl (bukan browser) |

---

## Steps to Reproduce

1. Dapatkan token admin (Super Admin, semua campus diizinkan).

```bash
# Tanpa filter → 502 konsisten
curl -s -m 30 -o /dev/null -w "%{http_code}\n" \
  "https://api.binabangsaschool.dev/api/v1/students?limit=1" \
  -H "Authorization: Bearer $TOKEN"
# → 502 (HTML nginx Bad Gateway), 2x percobaan dengan delay 8s antar percobaan

# Dengan filter campus → 200 cepat
curl -s -o /dev/null -w "%{http_code}\n" \
  "https://api.binabangsaschool.dev/api/v1/students?campusId=23&limit=5" \
  -H "Authorization: Bearer $TOKEN"
# → 200

curl -s -o /dev/null -w "%{http_code}\n" \
  "https://api.binabangsaschool.dev/api/v1/students/100215" \
  -H "Authorization: Bearer $TOKEN"
# → 200 (detail endpoint sehat)
```

**Actual Result:** 502 Bad Gateway (HTML nginx, bukan JSON error) untuk request
list tanpa filter campus.

**Expected Result:** 200 dengan envelope JSON:API + meta pagination.

---

## Root Cause Analysis

### Bug #1 (akar) — eager-load relasi sangat berat di `findAll` — `student.service.ts` (line 468-505)

```typescript
// api_nest/src/modules/student/student.service.ts:468-505
async findAll(options: GetStudentsDto) {
  const findOpts: FindManyOptions<Student> = {
    ...findOptionsHelper<Student>(options, {
      campus: true,
      parents: true,
      ccaYears: true,
      bandings: true,
      classYears: { classroom: true, programme: true, academicYear: true },
      currentClassYear: { classroom: true, programme: true, masterLevel: true, academicYear: true },
      studentReports: { classYear: true, reports: true },
      studentAdmissions: { placementTestSchedules: true, masterLevel: true, masterProgramme: { programmes: true } },
      masterDiscounts: true,
      familyBankInformation: true,
    }),
  };
  // ...
```

Relasi yang di-load: **studentReports → reports** (setiap student bisa punya
banyak report, DB berisi ±49 ribu baris `report`), admissions beserta placement
test schedule, CCA years, bandings, dsb. Untuk domain penuh (semua campus) ini
menghasilkan ribuan query / join besar — waktu eksekusi melampaui timeout nginx.

### Bug #2 (pendukung) — permit campus memfilter `In(campusIds)` untuk Super Admin = semua campus — `student.service.ts` (line 521-524)

```typescript
// api_nest/src/modules/student/student.service.ts:521-524
if (options.breakthrough === undefined || !options.breakthrough) {
  const campusIds = await checkCampusesPermit(this.req.user);
  whereFilters.campusId = In(campusIds);
}
```

Untuk Super Admin, `campusIds` = seluruh campus (±15+) → tidak ada penyempitan
data. Limit (`limit=1`) diterapkan pada query student, tetapi query count + join
relasi tetap bekerja atas seluruh domain sebelum limit efektif. (Perlu verifikasi
SQL EXPLAIN oleh BE untuk memastikan titik tepat pemborosan; yang pasti, filter
campus eksplisit membuat query sehat.)

### Pola yang sama (sistemik)

`GET /topics?limit=1` juga **502 konsisten** (3x, TC-009) karena
`topic.service.ts:128+` selalu eager-load `subjects→subjectYears→classYear`,
`masterLevel→classYears`, `academicYears`, `lessons→materials`, teachers, file —
lihat bug report `topics-list-eager-loading-timeout` di folder ini.

---

## Bukti dari Test (tanpa Jam)

| Sumber | Temuan |
|--------|--------|
| **API probe #1** | `GET /students?limit=1` → **502** (HTML nginx) |
| **API probe #2** | setelah sleep 8s, `GET /students?limit=1` → **502** lagi |
| **API probe #3** | `GET /students?campusId=23&limit=5` → **200** |
| **API probe #4** | `GET /students/100215` → **200** |
| **Code** | `student.service.ts:468-505` (eager-load), `:521-524` (permit In semua campus) |
| **Konteks DB** | campus 23 punya 116 student live (TC-006); domain penuh jauh lebih besar |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/student/student.service.ts` | `findAll` (line 468+): eager-load berat; permit scan semua campus |
| Backend DTO | `GetStudentsDto` | Tidak memaksa filter wajib (campus/AY) untuk list |
| Infra | nginx (production) | Memutus request > timeout → 502 tanpa informasi bagi user |
| Frontend | halaman list student admin | Tidak terdampak saat ini (selalu mengirim campus/filter) — risiko bila ada jalur tanpa filter |

---

## Proposed Solution Options

### Option A: Terapkan filter campus/AY default + batasi eager-load (Recommended)

1. Wajibkan minimal satu filter domain pada `findAll` (campus **atau**
   academicYear **atau** classYear) — tolak dengan `400` bila tidak ada, kecuali
   `breakthrough` eksplisit untuk job background.
2. Pecah eager-load: relasi berat (studentReports.reports, studentAdmissions.*)
   hanya di-load pada endpoint detail, bukan list.
3. Pertimbangkan `select` eksplisit kolom yang dipakai tabel admin.

### Option B: Query builder + pagination count yang murah

Ganti `findOptionsHelper` eager-load dengan query builder yang menghitung count
terpisah (tanpa join relasi berat) dan mengambil halaman data dengan join minimal.

### Option C: Hardening infra (pelengkap, bukan pengganti)

Naikkan `proxy_read_timeout` nginx untuk endpoint list berat — memperpanjang
waktu tunggu, tetapi tidak menyembuhkan query lambat. Tidak direkomendasikan
sendirian.

---

## Notes

- Sumber temuan: `operate-smartbag/test-cases/TC-006-students-status/result.md`
  (AI Operate Test, 2026-09-30).
- Satu keluarga dengan `topics-list-eager-loading-timeout` (TC-009) — pertimbangkan
  solusi sistemik: konvensi "list endpoint wajib filter domain + eager-load minimal".
- Workaround operasional saat ini: selalu sertakan `campusId` saat list students
  (tercatat di `operate-smartbag/_shared/api.md` bila diperlukan).
