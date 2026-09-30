---
title: CCA — Endpoint ccaYearCoordinators 500 karena findAll dikosongkan (scaffold rusak)
status: open
severity: minor
product: BBS LMS
portal: Admin
author: System Analyst
date: 2026-09-30
jam: (tidak tersedia — laporan berbasis eksekusi test API + code review)
---

# ccaYearCoordinators — `GET /api/v1/ccaYearCoordinators` → 500 (method kosong tanpa return)

## Summary

Ditemukan saat TC-039 full controller path sweep (2026-09-30). Controller
`cca-year-coordinator/cca-year-coordinator.controller.ts` line 38–41:

```ts
@Get()
findAll() {
  // return this.ccaYearProgrammesProgrammeService.findAll();
}
```

Body method **seluruhnya komentar** — method dieksekusi, tidak melempar error,
dan tidak me-return apa-apa → NestJS merespons **500**. Route tetap teregistrasi
sehingga endpoint terlihat "hidup" di katalog, tapi mati total. Pola identik
dengan `student-billing` (lihat `student-billing-endpoint-broken/`): scaffold
NestJS yang tidak pernah diselesaikan, namun terekspos di produksi.

**Ekspektasi:** endpoint mengembalikan daftar coordinator (200 envelope), atau
route tidak terekspos sampai implementasinya selesai (jangan biarkan 500).

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Buffy (AI agent) — eksekusi TC-039 sweep |
| User ID | 35008 (`mahdy.klabs`, Super Admin) |
| Environment API | `https://api.binabangsaschool.dev` (production) |
| Browser / OS | curl |

## Steps to Reproduce

```bash
curl -s "https://api.binabangsaschool.dev/api/v1/ccaYearCoordinators?limit=1" \
  -H "Authorization: Bearer $TOKEN" -o /dev/null -w "%{http_code}\n"
# → 500
```

## Actual vs Expected

| | Nilai |
|---|---|
| Actual | **500** untuk GET list |
| Expected | 200 dengan envelope `{data:[...]}`, atau 404/403 jika memang belum diimplementasi |

## Impact

- Frontend/klien yang memakai endpoint ini menerima 500 tanpa pesan bermakna.
- Katalog API tampak menyediakan fitur yang sebenarnya tidak ada.

## Rekomendasi

1. Implementasi `findAll()` (service-nya sudah ada), atau
2. Comment-out / hapus route `@Get()` sampai fitur jadi, atau
3. Hapus modul dari `app.module` bila fitur dibatalkan.

Sekaligus audit modul scaffold lain yang ditemukan senada di sweep yang sama:
`cca-grade` & `ftp-evaluation-setting` (controller tanpa route sama sekali),
`ccaYearCoordinators` (route ada, method kosong), `student-billing` (route ada,
service kosong) — 4 titik scaffold tidak selesai.
