---
title: Topics API — GET /topics selalu 502 karena findAll eager-load seluruh relasi tanpa syarat filter
status: open
severity: major
product: BBS LMS
portal: Admin | Teacher
author: System Analyst
date: 2026-09-30
jam: (tidak tersedia — laporan berbasis eksekusi test API + code review)
---

# Topics — `GET /api/v1/topics?limit=1` Mengembalikan 502 Bad Gateway Secara Konsisten

## Summary

Ditemukan saat eksekusi test case read-only (TC-009, folder
`operate-smartbag/test-cases/`, 2026-09-30). `GET /api/v1/topics?limit=1`
mengembalikan **502 Bad Gateway (nginx)** pada **3 dari 3 percobaan** — termasuk
dengan delay 5 detik dan dengan filter `subjectYearId`. Endpoint `lessons` dan
`assignments` yang se-kluster sehat (200).

**Ekspektasi:** list topics dengan limit kecil harus responsif seperti endpoints
saudaranya. Akar masalah: `TopicService.findAll` **selalu** eager-load jaringan
relasi lebar (subjects → subjectYears → classYear, masterLevel → classYears,
academicYears, lessons → materials, teachers, file) tanpa syarat filter — membuat
query melampaui timeout gateway bahkan untuk 1 baris.

Ini pola yang sama dengan `student-list-no-filter-timeout` (folder ini) —
indikasi kebutuhan konvensi list-endpoint yang sistemik.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Buffy (AI agent) — eksekusi TC-009 |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | 35008 (`mahdy.klabs`, Super Admin) |
| Portal URL | — (dites langsung ke API) |
| Environment API | `https://api.binabangsaschool.dev` (production) |
| Data konteks (class ID / daId / tanggal) | query `?limit=1` dan `?limit=1&subjectYearId=112388`; dieksekusi 2026-09-30 |
| Browser / OS | curl (bukan browser) |

---

## Steps to Reproduce

1. Dapatkan token admin (Super Admin).

```bash
# Percobaan 1 — limit kecil
curl -s -m 30 -o /dev/null -w "%{http_code}\n" \
  "https://api.binabangsaschool.dev/api/v1/topics?limit=1" \
  -H "Authorization: Bearer $TOKEN"
# → 502

# Percobaan 2 — setelah delay 5s
# → 502

# Percobaan 3 — dengan filter subjectYearId
curl -s -m 45 -o /dev/null -w "%{http_code}\n" \
  "https://api.binabangsaschool.dev/api/v1/topics?limit=1&subjectYearId=112388" \
  -H "Authorization: Bearer $TOKEN"
# → 502

# Pembanding sehat (se-kluster)
curl -s -o /dev/null -w "%{http_code}\n" \
  "https://api.binabangsaschool.dev/api/v1/lessons?limit=1" \
  -H "Authorization: Bearer $TOKEN"
# → 200
```

**Actual Result:** 502 Bad Gateway (HTML nginx) pada semua varian request topics.

**Expected Result:** 200 + `{ data: [...], count }`.

---

## Root Cause Analysis

### Bug (akar) — eager-load tanpa syarat — `topic.service.ts` (line 128-148)

```typescript
// api_nest/src/modules/topic/topic.service.ts:128-148
async findAll(options: GetTopicsDto) {
  const findOpts: FindManyOptions<Topic> = {
    ...findOptionsHelper<Topic>(options, {
      subjects: {
        subjectYears: {
          classYear: true,        // → subjectYears → classYears ...
        },
      },
      masterLevel: {
        classYears: true,
      },
      academicYears: true,
      lessons: {
        materials: true,          // → lessons → materials
      },
      createdByTeacher: true,
      updatedByTeacher: true,
      file: true,
    }),
  };
```

Jaringan join ini dieksekusi **untuk setiap request list** — tidak ada jalur
"list ringan". Kombinasi `subjects→subjectYears→classYear` dan
`masterLevel→classYears` menghasilkan fan-out join besar pada seluruh domain
topics, sehingga bahkan `limit=1` tidak menyelamatkan (count + join tetap berat).

### Faktor pendukung

- Tidak ada filter domain wajib pada `GetTopicsDto` — request tanpa filter sah.
- nginx memutus koneksi sebelum Nest selesai → 502 tanpa pesan bermakna bagi
  caller (pola yang sama dilihat pada `/students`).

---

## Bukti dari Test (tanpa Jam)

| Sumber | Temuan |
|--------|--------|
| **API probe #1–3** | `GET /topics?limit=1` → **502** (3x: 2 tanpa filter, 1 dengan `subjectYearId=112388`) |
| **Pembanding** | `lessons` → 200, `assignments` → 200, `sow` → 200 (path singular) |
| **Code** | `topic.service.ts:128-148` eager-load; `topic.controller.ts:56-62` `@Get()` tanpa filter wajib |
| **Data** | tabel `sow` 155 baris; kluster lesson/topic hidup (TC-009) |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/topic/topic.service.ts` | `findAll` (line 128+) eager-load berat tanpa syarat |
| Backend Controller | `api_nest/src/modules/topic/topic.controller.ts` | `@Get()` menerima request tanpa filter domain |
| Infra | nginx (production) | 502 tanpa pesan bermakna |
| Frontend | halaman yang memakai list topics | Risiko timeout saat data membesar (saat ini halaman UI kemungkinan memakai filter sempit) |

---

## Proposed Solution Options

### Option A: Pisahkan jalur "list ringan" vs "detail lengkap" (Recommended)

1. `findAll` untuk list: eager-load minimal (mis. hanya `subjects` nama) +
   wajibkan filter domain (campus/AY/subjectYear) — 400 bila kosong.
2. Relasi berat (`lessons.materials`, `subjectYears.classYear`,
   `masterLevel.classYears`) hanya di-load pada `findOne`/detail.
3. Tambah opsi `include=` untuk kebutuhan khusus bila UI butuh relasi tertentu
   pada list.

### Option B: Query builder + count murah

Ganti ke query builder dengan count terpisah (tanpa join) dan halaman data join
minimal — pola yang bisa dipakai bersama `student-list-no-filter-timeout`.

### Option C: Endpoint khusus UI

Bila UI tertentu butuh topic + lessons + materials sekaligus, buat endpoint
khusus dengan scope sempit (per subjectYear) daripada melonggarkan findAll.

---

## Notes

- Sumber temuan: `operate-smartbag/test-cases/TC-009-sow-lessons/result.md`
  (AI Operate Test, 2026-09-30).
- Keluarga bug yang sama: `student-list-no-filter-timeout` — pertimbangkan satu
  spec perbaikan sistemik "list endpoint hygiene" untuk keduanya.
- Workaround operasional: hindari `GET /topics` tanpa filter; gunakan detail per
  subjectYear lewat endpoint lain bila memungkinkan.
