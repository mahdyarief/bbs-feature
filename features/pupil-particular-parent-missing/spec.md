---
title: Pupil Particular — Data Parent Tidak Muncul pada Generate PDF
status: open
severity: major
product: BBS LMS
portal: Admin
author: System Analyst (AI)
date: 2026-10-29
jam: https://jam.dev/c/f75be0ff-7858-4d10-b741-92cf711c94a9
---

# Pupil Particular — Data Parent Tidak Muncul pada Generate PDF

## Summary

Di halaman Student List, nama parent (panggilan) siswa Wu Xinnian Hanna Ng sudah terlihat. Namun, ketika generate **Pupil Particular PDF**, data parent (Father — Wu Pei) tidak tercetak di dokumen. Dampaknya: dokumen Pupil Particular tidak lengkap karena kolom parent kosong meskipun relasi parent-siswa sudah tersimpan di database.

**Ekspektasi:** Ketika generate Pupil Particular PDF, parent yang terhubung dengan siswa (berdasarkan `parentsIds` dan tabel `parents`) harus muncul di dokumen dengan nama dan role yang benar.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | User pelapor (Jam recording) |
| Jam author ID | — |
| User ID (dari console log) | — |
| Portal URL | Admin portal — Student List / Student Details |
| Environment API | api_nest |
| Data konteks | Student Wu Xinnian Hanna Ng, id `25423`, student_id `WXN-2020-0170` |
| Browser / OS | — |

---

## Steps to Reproduce

1. Masuk ke halaman Student List di Admin portal.
2. Cari siswa "Wu Xinnian Hanna Ng" (student_id `WXN-2020-0170`).
3. Pastikan nama parent sudah muncul di daftar student list.
4. Klik tombol ikon dokumen (generate Pupil Particular PDF).
5. Cek hasil PDF yang dibuka.

**Actual Result:** PDF dibuka, tetapi kolom parent (Father/Mother/Both) kosong — data parent tidak muncul.

**Expected Result:** PDF menampilkan parent yang terkait, mis. **Father — Wu Pei**.

---

## Root Cause Analysis

### Bug #1 — Parent tidak muncul di Pupil Particular PDF

Berdasarkan pemeriksaan database (read-only via `db-access-tools`):

- Student `Wu Xinnian Hanna Ng` (id `25423`) memiliki `parents_ids = [5326]`.
- Parent `Wu Pei` (id `5326`) memiliki `parent_role = 'father'`, terkait student_id `25423`.
- Data relasi parent-siswa **valid dan lengkap** di database.

Masalah ada di sisi frontend PDF generator:

| File | Masalah |
|------|---------|
| `bbs/client/src/utils/pdf-generator/pupil-particular.jsx` | Filter parent berdasarkan `parentRole` dengan enum `PARENT_ROLE_ENUM`. Jika nilai di database (`'father'`, lowercase) tidak cocok dengan kunci enum (misal `'FATHER'` atau format berbeda), maka `PARENT_ROLE_ENUM?.[parent?.parentRole]` menghasilkan `undefined`, sehingga `relationship` jadi `"-"` dan parent tidak cocok filter. |
| `bbs/client/src/views/students/StudentDetails.jsx` | `pupilParticularDocument()` dipanggil tanpa parameter `selectedParentRole`, sehingga nilainya `undefined`. Dengan `undefined`, logika filter kemungkinan mengembalikan semua parent (harusnya), tetapi jika `relationship` sudah `undefined`/`"-"`, parent tidak akan muncul sama sekali. |

**Kesimpulan:** Data parent ada di DB dan dikirim ke frontend, tetapi mapping `parentRole` dari database ke label enum di PDF generator gagal, sehingga parent difilter/keluarkan dari dokumen PDF.

---

## Bukti dari Jam

| Sumber | Temuan |
|--------|--------|
| **Video** | Di halaman student list, nama parent Wu Pei sudah terlihat di baris siswa Wu Xinnian Hanna Ng. |
| **Network** | — |
| **Console** | — |
| **User events** | — |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend (PDF Generator) | `bbs/client/src/utils/pdf-generator/pupil-particular.jsx` | Parent tidak muncul di PDF karena filter `parentRole` gagal cocok |
| Frontend (Student Details) | `bbs/client/src/views/students/StudentDetails.jsx` | Memanggil `pupilParticularDocument` tanpa `selectedParentRole` |
| Database | `students`, `parents` | Data valid, tidak perlu perubahan |

---

## Proposed Solution Options

### Option A: Normalisasi `parentRole` sebelum filter (Recommended)

Di `pupil-particular.jsx`, ubah logika pemetaan `parentRole` agar case-insensitive dan tidak bergantung pada casing kunci enum. Contoh:

```js
const relationship = (parent?.parentRole || "").toString().toLowerCase().replace(/_role$/, "");
if (isFather && relationship !== "father") return false;
if (isMother && relationship !== "mother") return false;
```

### Option B: Gunakan `selectedParentRole` sebagai fallback

Pastikan `StudentDetails.jsx` melewati `selectedParentRole` saat memanggil `pupilParticularDocument`, dan tambahkan fallback di PDF generator jika `relationship` tidak dikenali oleh enum.

---

## Notes

- Database: `binabangsa_prod_mig_v01` (RDS production), tabel `students` dan `parents`.
- Perbaikan hanya di frontend (`bbs/`), tidak ada perubahan backend/api_nest maupun database.
- Setelah implementasi, uji generate Pupil Particular PDF untuk siswa Wu Xinnian Hanna Ng dan pastikan **Father — Wu Pei** muncul di dokumen.