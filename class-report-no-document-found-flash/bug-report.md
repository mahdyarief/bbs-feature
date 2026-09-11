---
title: Class Report Viewer — "No Document Found!" tampil sementara data masih loading (false empty state)
status: open
severity: minor
product: BBS LMS
portal: Admin
author: System Analyst
date: 2026-09-10
jam: https://jam.dev/c/46637b21-713c-45ce-a28f-58aee800e11c
---

# Class Report Viewer — Flash "No Document Found!" sebelum data report dimuat

## Summary

Pada halaman **Class Report** (combined view, print mode) di portal Admin, saat user membuka URL `/class-report/100932/1/combined?print=true`, halaman **menampilkan "No Document Found!" secara sesaat** sebelum data report dimuat. Urutan visual di video: **"No Document Found!" → loading spinner → document rows tampil**. Perilaku ini menyesatkan — user yang membuka halaman untuk print/PDF bisa mengira report kelas kosong, padahal data sebenarnya tersedia (semua network request sukses 200).

**Ekspektasi:** selama fetch data report masih berlangsung, halaman harus menampilkan **loading indicator / skeleton**, bukan empty state. "No Document Found!" hanya layak tampil setelah response sukses dan mengembalikan list kosong.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Mahdy Arief |
| Email (Jam account) | mahdy.arief.torche.indonesia@gmail.com |
| Jam author ID | 7ebe5b87-970c-4f3d-95f5-c4f39b901a03 |
| User ID (dari console log, mis. `selfUser`) | — (console kosong) |
| Portal URL | https://admin.smartbag.binabangsaschool.com/class-report/100932/1/combined?print=true |
| Environment API | api.binabangsaschool.com (staging/admin) |
| Data konteks (class ID / daId / tanggal) | classYearId=100932, reportTerm=1, masterLevelId=4, 14 students (studentIds: 25121, 20590, 22163, 22658, 26683, 24988, 103442, 22624, 26036, 22820, 21430, 25081, 22937, 22615) |
| Browser / OS | Brave 150.0.0.0 / Windows 11 |

---

## Steps to Reproduce

1. Login ke portal Admin (https://admin.smartbag.binabangsaschool.com).
2. Buka URL langsung: `/class-report/100932/1/combined?print=true` (atau via menu Class Report, pilih kelas 100932, term 1, combined).
3. Amati layar saat page load (durasi recording 8 detik).

**Actual Result:** halaman menampilkan "No Document Found!" terlebih dahulu (~t=0–1.5s), kemudian loading spinner (~t=2s), lalu document rows tampil dengan sukses (~t=4s+). Data report **tersedia** — semua API request sukses 200 (studentReports, reportSettings, dll.), jadi empty state ini adalah false positive.

**Expected Result:** halaman menampilkan loading spinner/skeleton selama fetch data, dan "No Document Found!" HANYA tampil jika response sukses mengembalikan document kosong.

---

## Root Cause Analysis

### Bug #1 — False empty state: viewer me-render "No Document Found!" ketika data report belum di-set — `ClassReportViewer` (frontend admin)

Class Report viewer menginisiasi state dengan array documents **kosong** (kemungkinan `documents: []` atau `null`). Logika render tidak membedakan antara `loading` (masih fetch) dan `empty` (fetch selesai, hasil kosong) — sehingga saat state masih berupa default/initial, kondisi empty state langsung terpenuhi dan "No Document Found!" tampil. Ketika data report dikirim dari API, barulah spinner/rows menggantikan tampilan tersebut.

```
// path: (frontend admin — komponen ClassReportViewer / halaman class report)
// Indikasi: render berdasarkan `documents.length === 0` tanpa gate `isLoading`
if (documents.length === 0) {
  return <EmptyState title="No Document Found!" />; // tampil ketika data belum selesai di-fetch
}
```

**Faktor pendukung:** pada Jam, semua network request sukses dan data siap dalam < 4 detik, jadi window "No Document Found!" hanya muncul singkat — namun cukup untuk membuat user ragu terhadap ketersediaan data, terutama pada print mode.

---

## Bukti dari Jam (https://jam.dev/c/46637b21-713c-45ce-a28f-58aee800e11c)

| Sumber | Temuan |
|--------|--------|
| **Video (t=0–1.5s)** | Halaman menampilkan "No Document Found!" sebelum data dimuat |
| **Video (t≈2s)** | Loading spinner tampil |
| **Video (t≈4s)** | Document rows tampil (data siap) |
| **Network** | Semua request API sukses 200 — termasuk `GET /api/v1/studentReports?studentIds=...&classYearId=100932&reportTerm=1` (response 142894 bytes) dan `GET /api/v1/reportSettings/letter` (5417 bytes) — data tersedia, sehingga "No Document Found!" adalah false empty state |
| **Console** | 0 log/errors — tidak ada runtime failure, hanya masalah timing/state rendering |
| **User events** | Navigasi ke URL print view (`onCommitted` @1569ms), tidak ada interaksi lanjutan |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | — (tidak ada perubahan backend) | — |
| Backend DTO | — | — |
| Backend Entity | — | — |
| Frontend | Class Report Viewer (halaman class-report combined/print) — komponen render empty state saat loading | False empty state "No Document Found!" flash saat page load |

---

## Proposed Solution Options

### Option A: Gate empty state dengan flag `isLoading` (Recommended)

Tambahkan state `isLoading` (atau menunggu hingga fetch data selesai) sebelum me-render empty state:

```typescript
// path: (frontend admin — ClassReportViewer render)
if (isLoading) {
  return <LoadingSpinner />; // atau skeleton rows
}
if (documents.length === 0) {
  return <EmptyState title="No Document Found!" />;
}
return <ReportRows documents={documents} />;
```

Dengan cara ini "No Document Found!" HANYA tampil setelah fetch selesai dan hasilnya kosong — selama loading selalu ditampilkan spinner/skeleton.

### Option B: Inisialisasi state dengan `null` dan render loading sampai data di-set

Inisialisasi `documents` dengan `null` (belum di-fetch) dan hanya render empty state jika `documents !== null && documents.length === 0`. Lebih defensive untuk kasus fetch yang gagal atau respon tidak terduga.

---

## Notes

- Environment: staging admin (admin.smartbag.binabangsaschool.com), browser Brave 150 / Windows 11.
- Recording Jam hanya 8 detik — window visual singkat; network & console menjadi bukti utama untuk memastikan root cause.
- Relevan untuk UX print mode: user yang membuka `?print=true` mengharapkan report siap cetak, bukan pesan "No Document Found!".