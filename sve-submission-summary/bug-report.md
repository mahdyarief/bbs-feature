---
title: SVE Submission Summary — Judul 'Submission Summary' Hilang dan Konten Tabel Terpotong di PDF
status: open
severity: major
product: BBS LMS
portal: Admin
author: System Analyst
date: 2026-09-30
jam: (tidak tersedia — laporan berbasis investigasi PDF + code review)
---

# SVE Submission Summary — Title Hilang & Konten Kepotong Saat Ada File

## Summary

Pada halaman **SVE Submission Summary** (`bbs/client/src/views/sves/submission-summary/`), file PDF hasil download (`SVE SUBMISSION SUMMARY_2025_2026_FYE_Primary.pdf`, 5 halaman, A2 landscape) mengalami **dua cacat**:

1. **Judul "Submission Summary" tidak tampil.** Header PDF hanya menampilkan `Primary Final Year Examination` + `AY: ...`. Ekspektasi user: ada baris judul `Submission Summary` di atas tulisan `Primary Final Year Examination`.
2. **Konten tabel kepotong di sisi kanan saat ada file.** Jika sel berisi nama file (mis. `25-26_FYE_P3...`, `...Fbp`, `LC_TP_FBP.pd...`), kolom kanan terpotong di tepi kertas. Jika sel hanya berisi `---` (tidak ada konten), tabel tampil utuh dan aman.

**Ekspektasi:**
- Header PDF berurutan: `Submission Summary` (paling atas) → `Primary Final Year Examination` → `AY: ...`.
- Tabel selalu muat penuh di lebar kertas A2 landscape baik saat ada file maupun saat kosong (`---`); teks panjang seharusnya wrap, bukan kepotong.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | — (laporan dari user) |
| Portal URL | SVE → Submission Summary (portal Admin, `SveSubmissionSummary.jsx`) |
| File contoh | `features/sve-submission-summary/SVE SUBMISSION SUMMARY_2025_2026_FYE_Primary.pdf` (5 halaman) |
| Environment | — |
| Browser / OS | — |

---

## Steps to Reproduce

### Issue A — Title hilang
1. Buka halaman SVE Submission Summary, pilih AY + Programme + Exam Type (mis. Primary + FYE).
2. Klik download/preview PDF (`generatePreview` di `SveSubmissionSummary.jsx`).
3. Perhatikan header halaman: hanya ada `Primary Final Year Examination` dan `AY: ...`.

**Actual Result:**
- Tidak ada tulisan `Submission Summary` di atas `Primary Final Year Examination`.

**Expected Result:**
- Ada judul `Submission Summary` di atas tulisan `Primary Final Year Examination`.

### Issue B — Konten kepotong saat ada file
1. Pada filter yang sama, pastikan ada submission yang sudah upload file (nama file panjang, mis. `25-26_FYE_P3...`).
2. Download PDF.
3. Perhatikan kolom-kolom kanan tabel.

**Actual Result:**
- Teks di tepi kanan terpotong (mis. header `Paper` hanya terbaca `Pape`; sel `25-26_FYE_P3...`, `Fbp` melewati batas kertas).
- Baris tanpa file (isi `---`) tampil utuh.

**Expected Result:**
- Seluruh tabel terlihat penuh di kertas baik ada file maupun tidak; teks panjang wrap ke baris baru.

---

## Root Cause Analysis

### Issue A — Header tidak pernah mencetak "Submission Summary"

`SveSubmissionSummaryPDFLayout.jsx` fungsi `printingDetailHeader(year, masterProgramme, masterSveExamType)`:

```jsx
text:
  masterProgramme && masterSveExamType
    ? `${masterProgramme} ${sveExamTypeMapper?.[masterSveExamType] || masterSveExamType} `
    : "Submission Summary",
```

Teks `"Submission Summary"` hanya dipakai sebagai **fallback** saat `masterProgramme`/`masterSveExamType` kosong. Pada alur normal (`generatePreview` selalu mengirim `masterProgramme?.name` dan `masterSveExamType?.name`), cabang yang terpakai adalah `${masterProgramme} ${examType}` — sehingga kata `Submission Summary` tidak pernah dicetak. Perlu baris judul tersendiri di atasnya.

### Issue B — `tableAutoSize: true` membuat tabel lebih lebar dari kertas

`SveSubmissionSummary.jsx` → `generatePreview()`:

```js
const content = htmlToPdfmake(
  getHtmlSveSubmissionSummary(...),
  { tableAutoSize: true }
);
```

`tableAutoSize: true` pada `html-to-pdfmake` (≈ v2.5.13) mengeset lebar tiap kolom ke `auto` (= mengikuti isi sel terlebar). Tabel dibuat oleh `SveSubmissionSummaryHtml.jsx` (`<table data-pdfmake='{"headerRows":1}'>`, tanpa `widths`) sehingga total lebar tabel = jumlah lebar konten. Begitu ada nama file panjang, total melebihi lebar kertas → terpotong di kanan. Saat semua sel `---`, konten pendek sehingga tabel muat — sesuai gejala "kalau ga ada konten aman".

**Bukti geometri dari PDF contoh** (diukur dengan pymupdf, bukan tebakan visual):

| Metrik | Nilai |
|--------|-------|
| Ukuran halaman | A2 landscape, `page.rect` width **1683.8 pt** × height 1190.6 pt |
| Margin (`SVESubmissionSummaryDefaultPDFSetting.pageMargins`) | `[20, 170, 20, 30]` → area konten ≈ **1643.8 pt** |
| Lebar tabel aktual (garis border pdfmake, `drawings max x1`) | **1968.2 pt** → melewati tepi kanan ≈ **305 pt** |
| Teks melewati batas kanan | 6 kata (hal. 1), 10 kata (hal. 2); contoh: `25-26_FYE_P3` x1=1686.2 > 1683.8, `Fbp` x1=1684.1, header `Pape` (terpotong dari `Paper`) |
| Teks melewati batas kiri | 0 |
| `defaultStyle.fontSize` | 14 (relatif besar untuk tabel berkolom banyak) |

Kombinasi `tableAutoSize` + nama file panjang + fontSize 14 menjelaskan mengapa tabel selebar 1968 pt dipaksa masuk ke area 1644 pt.

---

## Bukti dari File PDF

- File: `features/sve-submission-summary/SVE SUBMISSION SUMMARY_2025_2026_FYE_Primary.pdf` (603.6 KB, 5 halaman).
- Halaman 1–3: kolom kanan (`Paper`, sel berisi `25-26_FYE_P3...` / `Fbp` / `LC_TP_FBP.pd...`) terpotong di tepi kanan.
- Baris yang seluruh selnya `---` tampil utuh.

---

## Dampak

- **Dokumen resmi cacat**: PDF submission summary yang dikirim/diarsipkan kehilangan judul identitas (`Submission Summary`) dan kehilangan sebagian nama file di kolom kanan — tidak bisa dipakai sebagai bukti submission yang lengkap.
- **Intermiten dan menipu**: saat belum ada yang upload (`---` semua) PDF terlihat normal, sehingga bug baru ketahuan setelah banyak file masuk (saat paling dibutuhkan).
- **Scope**: setiap kombinasi AY/Programme/ExamType dengan nama file panjang berpotensi kena; bukan data-spesifik satu Primary saja.

---

## Rekomendasi Perbaikan

> Catatan: rekomendasi di bawah ini untuk tim dev; **tidak ada perubahan codebase yang dilakukan dalam sesi ini** (atas instruksi user, hanya bug report).

### Option A: Tambah judul + paksa tabel muat (Recommended)

1. **Header** (`SveSubmissionSummaryPDFLayout.jsx` → `printingDetailHeader`): jadikan `Submission Summary` baris judul permanen (fontSize 30, bold, center), lalu di bawahnya `${masterProgramme} ${examType}` (mis. fontSize 24), lalu `AY: ...`. Jangan jadikan `Submission Summary` sekadar fallback else.
2. **Lebar tabel**: setelah `htmlToPdfmake(...)`, override `table.widths` menjadi `*` (star, dibagi rata) sejumlah kolom, atau definisikan `widths` eksplisit via `data-pdfmake` pada `<table>` di `SveSubmissionSummaryHtml.jsx`. Dengan `*`, pdfmake membagi lebar area yang tersedia dan teks panjang wrap otomatis.
3. Opsional pendukung: kecilkan `defaultStyle.fontSize` untuk tabel ini saja (mis. 10–11), dan/atau pertimbangkan `pageSize` tetap A2 landscape (sudah benar) — jangan mengecilkan ke A4/A3 karena kolom banyak.

### Option B: Pertahanan di generator HTML

- Di `SveSubmissionSummaryHtml.jsx`, tambahkan atribut `data-pdfmake` dengan `widths` pada `<table>` sejak awal (mis. kolom pertama `auto`, kolom file `*`), sehingga tidak bergantung pada post-processing hasil `htmlToPdfmake`.
- Render nama file dengan word-break (spasi/zero-width) agar string panjang tanpa spasi (`25-26_FYE_P3_...pdf`) bisa wrap di pdfmake.

### Verifikasi yang disarankan setelah fix

1. Download PDF untuk filter yang **ada file** (kasus Primary FYE ini) → pastikan tidak ada teks dengan x1 > lebar halaman dan semua nama file terbaca utuh (wrap, bukan potong).
2. Download PDF untuk filter yang **kosong** (`---` semua) → pastikan tetap tampil utuh seperti sebelumnya.
3. Pastikan header tiga baris tampil: `Submission Summary` / `Primary Final Year Examination` / `AY: ...`.
4. Ulangi untuk kombinasi exam type lain (MYE) dan programme lain.

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend (Admin) | `bbs/client/src/views/sves/submission-summary/SveSubmissionSummaryPDFLayout.jsx` (`printingDetailHeader`, `SVESubmissionSummaryDefaultPDFSetting`) | header tanpa judul `Submission Summary`; setting A2 landscape + margin 20/170/20/30 |
| Frontend (Admin) | `bbs/client/src/views/sves/submission-summary/SveSubmissionSummary.jsx` (`generatePreview`, `htmlToPdfmake(..., { tableAutoSize: true })`) | `tableAutoSize` → kolom `auto` → tabel 1968 pt > area 1644 pt |
| Frontend (Admin) | `bbs/client/src/views/sves/submission-summary/SveSubmissionSummaryHtml.jsx` (`getHtmlSveSubmissionSummary`, `<table data-pdfmake='{"headerRows":1}'>`) | tabel tanpa `widths`; sel file berisi nama panjang yang tidak wrap |
| Library | `html-to-pdfmake@^2.5.13` + `pdfmake@^0.2.12` (lihat `bbs/client/package.json`) | perilaku `tableAutoSize` → `auto` widths |
| Data contoh | `features/sve-submission-summary/SVE SUBMISSION SUMMARY_2025_2026_FYE_Primary.pdf` | bukti 5 halaman, kolom kanan kepotong |

---

## Notes

- Tidak ada perubahan kode yang dilakukan — file di `smartbag/bbs/client/...` dikembalikan ke kondisi semula setelah investigasi (revert `fitTableWidths` dan edit header yang sempat dicoba).
- File temp diagnostik (`bbs_sve_pdf.py` di Temp) sudah dihapus.
- Library `node_modules` tidak tersedia di lokal sehingga source `html-to-pdfmake` tidak dibaca langsung; analisis perilaku `tableAutoSize` didasarkan pada output PDF terukur + pemanggilan di `SveSubmissionSummary.jsx`.
