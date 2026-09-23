---
title: Class Report — Kolom Principal kosong pada Combined Report print dari Teacher Portal
status: open
severity: minor
product: BBS LMS
portal: Teacher
author: Ariel Wirawan Saputra
date: 2026-09-22
jam: https://jam.dev/c/1c40877a-5e54-4d2d-9605-ab1655076d23
---

# Class Report — Kolom Principal kosong pada print Combined Report dari Teacher Portal

## Summary

Saat melakukan **print** pada **Class Report — Combined Report** melalui **Teacher Portal** (teacher.smartbag.binabangsaschool.com), hasil print **menampilkan kolom "Principal" kosong**. Sedangkan apabila report yang sama di-generate dari tempat lain (portal lain / jalur generate lain), kolom Principal **terisi nama principal** dengan benar.

Ini mengindikasikan data principal tersedia di sisi data/API, tetapi jalur print Combined Report di Teacher Portal tidak mengisi/meneruskan nilai principal tersebut ke template print.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Ariel Wirawan Saputra |
| Email (Jam account) | arielwirawansaputra@gmail.com |
| Portal URL | https://teacher.smartbag.binabangsaschool.com |
| Environment API | api.binabangsaschool.dev (staging) |
| Browser / OS | Chrome 152.0.7977.83 / macOS (arm) 26.3.0 |
| Tanggal temuan | 2026-09-22 |

---

## Steps to Reproduce

1. Login ke **Teacher Portal** (https://teacher.smartbag.binabangsaschool.com).
2. Masuk ke menu **Class Report**.
3. Pilih **Combined Report**.
4. Klik **Print / Generate** report.
5. Amati kolom **Principal** pada hasil print.

**Actual Result:** kolom "Principal" **kosong** pada hasil print.

**Expected Result:** kolom "Principal" menampilkan **nama principal** yang sesuai, konsisten dengan hasil generate dari jalur lain.

---

## Perbandingan Perilaku

| Jalur Generate | Kolom Principal |
|----------------|-----------------|
| Teacher Portal (Combined Report print) | ❌ Kosong |
| Tempat lain (portal/jalur generate lain) | ✅ Terisi nama principal |

---

## Bukti dari Jam (https://jam.dev/c/1c40877a-5e54-4d2d-9605-ab1655076d23)

| Sumber | Temuan |
|--------|--------|
| **Video (15 detik)** | Hasil print Combined Report dari Teacher Portal menampilkan kolom Principal kosong |
| **Network** | 66 request (50 GET, 16 OPTIONS) — semua sukses (200/204), 35 request ke api.binabangsaschool.dev. Tidak ada request gagal |
| **Console** | 298 log (semua level `log`), 0 error / 0 warn — tidak ada runtime failure |

Karena semua request sukses dan tidak ada error console, masalah ini kemungkinan bukan kegagalan fetch, melainkan:
- payload/response API yang dipakai jalur Teacher Portal tidak menyertakan field principal, atau
- komponen print di Teacher Portal tidak me-mapping field principal ke template.

---

## Root Cause Analysis (Hipotesis)

### Bug #1 — Jalur print Combined Report Teacher Portal tidak mengisi field principal

Perbedaan perilaku antar jalur generate menunjukkan data principal tersedia, tetapi jalur Teacher Portal salah satu dari:

1. **Frontend Teacher Portal** — komponen print combined report tidak membaca/meneruskan field `principal` (atau nama field berbeda, mis. `principalName`, `headmaster`) saat menyusun data untuk template print.
2. **API** — endpoint yang dipanggil dari Teacher Portal (versi/parameter query berbeda) tidak meng-include relasi principal, sementara endpoint yang dipakai jalur lain meng-include-nya.

**Langkah verifikasi lanjutan:**
- Bandingkan request/response API pada Jam ini (teacher portal) dengan request/response saat generate dari jalur yang benar — cek endpoint & payload yang berbeda.
- Cek komponen print combined report di frontend teacher: pemetaan field principal ke template.

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | Perlu verifikasi — kemungkinan endpoint studentReports/combined report versi teacher | Field principal tidak ter-include pada response |
| Frontend | Komponen print Combined Report (Teacher Portal) | Kolom Principal kosong pada hasil print |

---

## Proposed Solution Options

### Option A: Perbaiki mapping di frontend Teacher Portal (jika data tersedia di response)

Pastikan komponen print combined report membaca field principal dari response API dan meneruskannya ke template print, konsisten dengan implementasi di jalur lain.

### Option B: Samakan endpoint/parameter API (jika response tidak menyertakan principal)

Samakan endpoint atau parameter query yang dipakai Teacher Portal dengan jalur lain sehingga response menyertakan data principal.

---

## Notes

- Environment: staging (api.binabangsaschool.dev), Chrome 152 / macOS (arm) 26.3.0.
- Tidak ada console error dan semua network request sukses — issue ini murni data/mapping, bukan kegagalan request.
- Perlu perbandingan langsung payload API antara Teacher Portal vs jalur lain untuk menentukan root cause definitif (frontend mapping vs endpoint).
