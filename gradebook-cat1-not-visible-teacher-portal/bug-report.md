# Bug Report: CAT1 Tidak Tampil di Gradebook Halaman Guru (Teacher Portal)

## Metadata
- **Reported by:** User (Jam dev recording)
- **Date:** 2026-10-29
- **Severity:** Major
- **Module:** Gradebook — Teacher Portal (client-teacher)
- **Environment:** Teacher portal, gradebook details page (PKN Primary 5, Hope Kebon Jeruk)
- **Jam:** https://jam.dev/c/d90b4284-0de6-4f6c-be06-a133a1134e42

## Deskripsi Bug
Admin telah mengatur opsi "show CAT1" pada gradebook settings (PKN Primary 5). Pada halaman cetak/print, CAT1 muncul. Namun, ketika guru membuka halaman gradebook details di Teacher Portal, kolom CAT1 tidak ditampilkan pada tabel.

## Steps to Reproduce
1. Buka halaman Gradebook Details di Teacher Portal untuk subjek PKN Primary 5 (Hope Kebon Jeruk).
2. Lihat tabel gradebook — kolom CAT1 tidak muncul.
3. Klik tombol Print/Generate cetak — CAT1 muncul di hasil cetak.

## Expected Behavior
Kolom CAT1 harus muncul di tabel gradebook di halaman guru (layar), sama seperti yang muncul saat print, karena CAT1 sudah diatur sebagai show di gradebook settings.

## Actual Behavior
- Layar guru: tabel tidak menampilkan kolom CAT1.
- Cetak PDF: kolom CAT1 muncul.

## Root Cause Analysis

### Frontend — `bbs/client-teacher/src/components/GradebookFlexibleTable.jsx`
Kolom yang dirender di layar berasal dari `gradebookTable` yang di-resolve pada baris ~155:

```js
const gradebookTable =
  currentGradebookTable ||
  getDefaultGradebookTable(parseInt(term), classYear.level, subject.category, subject.department);
```

1. Jika `currentGradebookTable` (tabel yang disimpan admin via gradebook settings) kosong atau tidak cocok dengan `term`, maka fallback `getDefaultGradebookTable` digunakan.
2. Fallback untuk subjek akademik tingkat dasar (`defaultGradebookTables.js`) memiliki `extraFields: []` (CAT dicomment/kosongkan), sehingga **CAT1 tidak muncul di layar**.

### Print handler — `GradebookFlexibleTable.jsx` baris ~828
`handlePrint` menyusun header cetak dari `fieldsName` — state yang diisi dari data grade yang dikirim backend. Karena data grade sudah mengandung CAT, maka **cetak menampilkan CAT1**, meski layar menggunakan konfigurasi default tanpa CAT.

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend | `bbs/client-teacher/src/components/GradebookFlexibleTable.jsx` | Layar tabel vs cetak tidak konsisten |
| Frontend | `bbs/client-teacher/src/views/gradebook/defaultGradebookTables.js` | Fallback default tidak menyertakan CAT untuk primary academic |

## Proposed Solution Options

### Option A + C (Recommended)
1. **Sinkronkan konfigurasi tabel:** pastikan `currentGradebookTable` (yang di-set admin) benar-benar didapat dan cocok dengan `term` + `subjectYearId` di halaman guru, sehingga CAT yang diatur admin muncul juga di layar — tidak hanya pakai fallback default.
2. **Cetak konsisten dengan layar:** ubah `handlePrint` agar header cetak berasal dari `fields` / `extraFields` yang sama dengan yang dirender di tabel, bukan dari `fieldsName` state terpisah.

Dengan ini, layar dan cetak selalu menampilkan kolom yang sama (CAT1 jika diatur admin).

## Notes
- Read-only analisis dari rekaman Jam; belum dilakukan perubahan kode.
- Koordinasi diperlukan dengan tim yang mengelola `defaultGradebookTables.js` jika opsi fallback juga ingin disertakan CAT.
