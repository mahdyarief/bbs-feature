---
title: Format Margin Nasional Report Card New vs Old
slug: national-report-margin
---

# Brief — Format Margin Nasional Report Card (New vs Old)

Diukur langsung dari PDF hasil generate legacy (html2pdf / TCPDF 5.0.002). Ukuran kertas A4 210 × 297 mm (595.28 × 841.89 pt).

## Nasional Report Card New (`report_indo_nas_new.php`)

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

## Nasional Report Card Old (`report_indo_nas.php`)

PDF varian old diunduh dari legacy (`report_indo_nas.php?classid=MjAwNQ==`, class 2005 Sec 3 Taylor) via script `ais_legacy/admin_tools/23_capture_nasional_old.py`, tersimpan di `ais_legacy/admin_tools/downloads/nasional_old_2005_sec3_taylor_exemple01.pdf`. Ukuran dan print area sama-sama A4:

| Varian | Halaman | Margin L | Margin T | Margin R (dari tepi) | Margin B (dari tepi) |
|--------|---------|----------|----------|----------------------|----------------------|
| Old (2005 Sec 3 Taylor) | 20 | ~10.8 mm | ~48.1 mm | ~17.5 mm | ~32.3 mm |
| New (2005 Sec 3 Taylor) | 20 | ~10.8 mm | ~48.1 mm | ~30.6 mm | ~31.0 mm |

**Perbandingan new vs old (class yang sama):** kiri dan atas identik (L ~10.8 mm, T ~48.1 mm), halaman sama-sama 20. Perbedaan ada di margin kanan — new ~30.6 mm vs old ~17.5 mm — artinya konten teks new lebih sempit di sisi kanan (~13 mm lebih sempit), sedangkan margin bawah hampir sama (~31 vs ~32 mm).
