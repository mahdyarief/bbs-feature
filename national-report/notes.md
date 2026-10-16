---
feature: National Report Card
slug: national-report
---

# Notes — National Report Card

## Sumber Data

Feature brief ini didasarkan pada:

1. **PDF Nasional Report Card New** yang di-download dari legacy (`report_indo_nas_new.php?classid=<base64>`) — 5 file PDF untuk AY 2024/2025 (KJ-S): Secondary 3 Taylor, Secondary 3 Pascal, JC2 Mendel, JC2 Forbes, Secondary 4 Pascal. File PDF asli tersimpan di `reference/`:
   - `nasional-report-card-sec3-taylor-ay2024-2025.pdf` (Sec 3 Taylor)
   - `nasional-report-card-sec4-pascal-ay2024-2025.pdf` (Sec 4 Pascal)
   - `nasional-report-card-jc2-mendel-ay2024-2025.pdf` (JC2 Mendel)
   - `nasional-report-card-jc2-forbes-ay2024-2025.pdf` (JC2 Forbes)
2. **OCR hasil PDF** (JC2 Mendel, 42 halaman) — mengonfirmasi struktur: Surat Keterangan Lulus (SKL) + Transkrip Nilai per siswa.
3. **Clue formula dari bbs-notes** — GP Value × Unit, level logic (JC2/SEC4/lainnya), transcript Dec/June, NAS Ranking.
4. **Report Viewer** (`choose_ay_p.php` di teacher portal) — mengonfirmasi reporttype=24 = "Nasional".
5. **Menu Class in Year** di portal admin (`ais_new`) — mengonfirmasi kelompok menu Report Card dengan URL ke `library/html2pdf/examples/`.

## Key Findings

### URL dan Parameter

- Endpoint: `library/html2pdf/examples/report_indo_nas_new.php?classid=<base64>`
- `classid` adalah **base64 dari class id integer** (contoh: `MjAwNQ==` = 2005)
- Base64 decoding: `python -c "import base64; print(base64.b64decode('MjAwNQ==').decode())"` → 2005
- Output: PDF download (html2pdf library)

### Data AY 2024/2025 (KJ-S)

12 class dengan classid 2001-2012:
| classid | Class |
|---------|-------|
| 2001 | Secondary 1 Pascal |
| 2002 | Secondary 1 Newton |
| 2003 | Secondary 2 Taylor |
| 2004 | Secondary 2 Pascal |
| 2005 | Secondary 3 Taylor |
| 2006 | Secondary 3 Pascal |
| 2007 | Secondary 4 Pascal |
| 2008 | Secondary 4 Newton |
| 2009 | JC1 Mendel |
| 2010 | JC1 Forbes |
| 2011 | JC2 Mendel |
| 2012 | JC2 Forbes |

### Kelompok Menu Report Card (per class)

Setiap class di menu Class in Year memiliki aksi:
- Progress Report (T1, T3)
- FTP (T1, T2, T3, T4)
- Report Card (Sem 1, Sem 2)
- E-Report Card (Sem 1, Sem 2)
- GPA Mid
- Medal
- Form Ijazah, Check Ijazah, Ijazah Excel
- **Nasional Report Card New**, **Nasional Report Card Old**
- Leaps
- FTP Evaluation
- Award Descriptor (P1-P2, P3-P6, Secondary, JC)

## Asumsi yang Perlu Diverifikasi

1. **Data PUM/FYA/SA1/IGCSE** — belum dikonfirmasi apakah sudah tersimpan di smartbag. Jika belum, implementasi National Report tidak bisa menggunakan formula level logic yang benar.
2. **GP Value × Unit formula** — unit disimpan per AY (`secondary-gpa-subject-unit`). CCA di-average. Ada Range CCA General. Perlu divalidasi.
3. **Nomor SKL/Transkrip** — format nomor di legacy adalah `{nomor}/CJCE/{campus}/{tahun}` untuk SKL dan `{nomor}/TN/SMA/{campus}/{tahun}` untuk Transkrip. Perlu dikonfirmasi apakah format ini dipertahankan.
4. **Peminatan/Jurusan** — ada di Transkrip legacy tapi tidak terlihat di semua data OCR. Perlu dikonfirmasi sumber datanya.
5. **Tanda tangan digital** — legacy tidak menggunakan tanda tangan digital (hanya teks "Kepala Sekolah: Richard, M.Fin, MBS"). Smartbag mungkin perlu implementasi signature digital.

## Dependensi & Cross-Reference

- **FEATURE_COMPARISON.md** — gap #1 (report analysis) sangat besar, National Report adalah bagian dari gap ini.
- **FEATURE_IMPROVEMENT_RECOMMENDATIONS.md** — National Report direkomendasikan di fase 1 (high priority).
- **GPA Calculator** (`gpa_calculation_view.php`) — sumber data GP Value, terhubung ke `secondary-gpa-grading-scale` dan `secondary-gpa-subject-unit`.
- **Moderation** (`moderate_component_prelim_new.php`) — terkait dengan data PUM/IGCSE/A Level.
- **Transcript** — belum ada feature brief, tapi National Report mencakup Transkrip. Jika Transcript ingin dipisah jadi feature brief terpisah, perlu koordinasi.

## Catatan Implementasi

### Endpoint vs Direct Download

Legacy menggunakan `report_indo_nas_new.php` yang langsung mendownload PDF. Di smartbag, disarankan:
- Endpoint REST `/api/v1/national-reports/{classId}/pdf` yang mengembalikan file PDF
- Bisa juga ditambahkan endpoint `/api/v1/national-reports/{classId}/preview` untuk preview HTML sebelum download

### Format PDF

PDF di legacy menggunakan `html2pdf` (PHP library). Untuk smartbag (NestJS + TypeScript), rekomendasi:
- **Puppeteer** — render HTML → PDF, hasil paling akurat
- Atau PDFKit untuk generate langsung

### Dual Portal

- Admin Portal: akses semua campus, semua AY
- Teacher Portal: akses terbatas ke campus sendiri, mungkin perlu permission tambahan
- Student Portal: tidak ada aksi generate, hanya menerima dokumen

## Format Margin (New vs Old)

Diukur langsung dari PDF hasil generate legacy (html2pdf / TCPDF 5.0.002), ukuran kertas A4 210 × 297 mm (595.28 × 841.89 pt).

### Nasional Report Card New (`report_indo_nas_new.php`)

5 file PDF AY 2024/2025 (Sec 3 Taylor, Sec 3 Pascal, Sec 4 Pascal, JC2 Mendel, JC2 Forbes) semua dicetak A4. Print area teks (bukan border rias) per halaman:

| File | Halaman | Margin L | Margin T | Margin R (dari tepi) | Margin B (dari tepi) |
|------|---------|----------|----------|----------------------|----------------------|
| sec3-taylor | 20 | ~10.8 mm | ~48.1 mm | ~30.6 mm | ~31.0 mm |
| sec3-pascal | 22 | ~10.8 mm | ~48.1 mm | ~30.6 mm | ~17.6 mm |
| sec4-pascal | 43 | ~5.0 mm* | ~8.2 mm* | ~-1.6 mm* (full-bleed) | ~12.3 mm |
| jc2-mendel | 42 | ~10.0 mm | ~48.1 mm | ~10.4 mm | ~15.2 mm |
| jc2-forbes | 35 | ~10.8 mm | ~48.0 mm | ~28.8 mm | ~18.7 mm |

\* Halaman Sec 4 Pascal elemen gambar/tabel melebar ke tepi kertas (full-bleed), bukan teks — margin teksnya tetap konsisten di halaman lain.

**Rangkuman New:** Margin kiri konsisten ~10–10.8 mm, atas ~48 mm (tahan spasi besar untuk header SKL), kanan ~28–31 mm, bawah ~15–31 mm tergantung panjang isi halaman. PDF di-generate per class, satu PDF berisi banyak siswa (SKL + Transkrip), mencakup 3 tahun terakhir.

### Nasional Report Card Old (`report_indo_nas.php`)

PDF varian old diunduh dari legacy (`report_indo_nas.php?classid=MjAwNQ==`, class 2005 Sec 3 Taylor) via script `ais_legacy/admin_tools/23_capture_nasional_old.py`, tersimpan di `ais_legacy/admin_tools/downloads/nasional_old_2005_sec3_taylor_exemple01.pdf`. Ukuran dan print area sama-sama A4:

| Varian | Halaman | Margin L | Margin T | Margin R (dari tepi) | Margin B (dari tepi) |
|--------|---------|----------|----------|----------------------|----------------------|
| Old (2005 Sec 3 Taylor) | 20 | ~10.8 mm | ~48.1 mm | ~17.5 mm | ~32.3 mm |
| New (2005 Sec 3 Taylor) | 20 | ~10.8 mm | ~48.1 mm | ~30.6 mm | ~31.0 mm |

**Perbandingan new vs old (class yang sama):** kiri dan atas identik (L ~10.8 mm, T ~48.1 mm), halaman sama-sama 20. Perbedaan ada di margin kanan — new ~30.6 mm vs old ~17.5 mm — artinya konten teks new lebih sempit di sisi kanan (~13 mm lebih sempit), sedangkan margin bawah hampir sama (~31 vs ~32 mm).

## Daftar Asumsi (Marked as Assumption)

| # | Asumsi | Dampak jika salah |
|---|--------|-------------------|
| A-1 | Data PUM/FYA/SA1/IGCSE sudah atau akan ada di smartbag | Implementasi tidak bisa menggunakan formula yang benar |
| A-2 | GP Value × Unit formula sudah benar di implementasi GPA | Nilai final salah |
| A-3 | Format nomor SKL/Transkrip mengikuti pola legacy | Dokumen tidak sesuai standar |
| A-4 | 13 mapel Transkrip konsisten untuk semua level | Beberapa level mungkin punya mapel berbeda |
| A-5 | Class in Year menu akan diimplementasi di smartbag | Tidak ada entry point untuk generate PDF |