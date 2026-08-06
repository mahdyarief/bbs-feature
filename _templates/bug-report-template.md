---
title: # Bug title (e.g., CCA Registration — Gender filter tidak berfungsi)
status: open # open | in-progress | fixed | verified | closed
severity: # critical | major | minor | cosmetic
product: BBS LMS
portal: # Admin | Student | Teacher
author: # who wrote this (e.g., System Analyst)
date: # YYYY-MM-DD
jam: # https://jam.dev/c/<jam-id> (link ke recording bug)
---

# {Bug Title — singkat, sebutkan fitur + masalah}

## Summary

<!-- 2-4 kalimat: apa bug-nya, di halaman/portal mana, dan dampaknya ke user. -->
<!-- Sebutkan jumlah masalah jika lebih dari satu (mis. Bug #1, Bug #2). -->

**Ekspektasi:** <!-- apa yang seharusnya terjadi -->

---

## Test Identity / Akun Akses

<!-- Data identitas user yang mereproduksi bug — untuk mempermudah recreate & testing oleh QA. -->
<!-- Diisi dari Jam: author + info dari console log (mis. selfUser ID) + data konteks (class, daId, tanggal). -->

| Field | Nilai |
|-------|-------|
| Reporter / Tester | |
| Email (Jam account) | |
| Jam author ID | |
| User ID (dari console log, mis. `selfUser`) | |
| Portal URL | |
| Environment API | |
| Data konteks (class ID / daId / tanggal) | |
| Browser / OS | |

---

## Steps to Reproduce

<!-- Langkah konkret untuk mereproduksi bug, dengan data spesifik (URL, ID, nilai awal). -->

1. 
2. 
3. 

**Actual Result:** <!-- apa yang terjadi saat ini (termasuk evidence: error message, data salah) -->

**Expected Result:** <!-- apa yang seharusnya terjadi -->

---

## Root Cause Analysis

<!-- Analisis akar masalah, referensikan file + line number. Pisahkan per bug jika lebih dari satu. -->

### Bug #N — {Nama singkat} — `{file}.ts` (line x-y)

<!-- Jelaskan akar masalah + potongan kode yang relevan. -->

```typescript
// path:line — deskripsi singkat
```

<!-- Faktor pendukung lain (DTO, entity default, dsb). -->

---

## Bukti dari Jam ({jam-url})

<!-- Tabel bukti dari Jam: video, network requests, console logs, user events. -->

| Sumber | Temuan |
|--------|--------|
| **Video (t=...)** | |
| **Network** | |
| **Console** | |
| **User events** | |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | | |
| Backend DTO | | |
| Backend Entity | | |
| Frontend | | |

---

## Proposed Solution Options

<!-- Minimal 2 opsi solusi, tandai yang direkomendasikan. -->

### Option A: {Nama} (Recommended)

<!-- Deskripsi + potongan kode/perubahan yang disarankan. -->

### Option B: {Nama}

<!-- Deskripsi singkat. -->

---

## Notes

<!-- Informasi tambahan: environment, koordinasi dengan tim terkait, langkah lanjutan. -->

- 
