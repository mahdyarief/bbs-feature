# Notes — Appraisal Summary Report

## Ringkasan Analisis

| Portal legacy | Temuan |
|---------------|--------|
| Portal ASD (portal admin) | Menu Appraisal Summary Report (322) → `staff/appraisal_summary_asd.php?campus=1&cname=KJP` — varian dengan param campus + cname; Appraisal Data Analysis (325), Appraisal Raw Data (326) |
| Teacher portal | Menu id 341 (Appraisal Summary Report) → `staff/appraisal_summary.php`; 342 (HOD Appraisal Summary) → `staff/appraisal_summary_hod.php` |
| Principals portal | Sidebar "Appraisal/PETALS/Audit Per Group" — akses langsung `staff/...` (tanpa proxy) dengan param `?campus=0` (semua campus) |

## Alur Legacy (reverse-engineer)

1. Principal/HOD login → menu Appraisal Summary Report (id 341) atau HOD Appraisal Summary (id 342) di portal teacher.
2. Halaman load → `staff/appraisal_summary.php` (teacher) / `staff/appraisal_summary_hod.php` (HOD) — render PHP server-side langsung menampilkan tabel.
3. Header halaman: "Appraisal Summary Report for {Campus} Campus" / "HOD Appraisal Summary Report for {Campus} Campus"; shell title "BBS Staff Database".
4. Toolbar atas (div `#print`): tombol Print `<input name="print" value="Print" onclick="print_div()">` + link "Export to Excel File" → `print_appraisal_summary.php` (teacher) / `print_appraisal_summary_hod.php` (HOD).
5. `print_div()`: `document.getElementById('print').style.visibility='collapse'; window.print(); self.close();` — menyembunyikan tombol lalu memicu print dialog.
6. Tabel `<table id="dataTable">` — kolom #, User ID, Name, dimensi..., TOTAL (TOTAL di-bold via `font-weight: bold`).

## Perbedaan 17 Dimensi Teacher vs 9 Dimensi HOD

| Aspek | Teacher summary (`appraisal_summary.php`) | HOD summary (`appraisal_summary_hod.php`) |
|-------|-------------------------------------------|--------------------------------------------|
| Jumlah dimensi | 17 | 9 |
| Skala | Maks per dimensi berbeda (14, 14, 8, 6, 6, 5, 5, 4, 3, 3, 5, 5, 4, 4, 4, 4, 4) → total teoretis ±99 | Seragam 0-5 |
| TOTAL | Sum 17 dimensi (mis. Muhammad Affan 76) | Rata-rata 9 dimensi (mis. Chandi Wijaya 4.61, Mazlinda Salleh Huddin 3.31) |
| Set dimensi | Overlap dgn PETALS/EPMS Teaching Competencies & Professional Qualities | Khusus kompetensi manajemen HOD (Leadership/Vision, Strategic Planning, dsb.) |
| Jumlah baris (probe PIK-S) | 45 guru | 4 HOD |

## Hubungan dengan appraisal-new (gateway)

- **Appraisal Summary Report TIDAK meng-input skor.** Skor di-input lewat gateway `features/appraisal-new/` (New Appraisal / Staff Database — daftar guru + tombol Appraisal). Summary report adalah **aggregation view** read-only di atas tabel skor gateway (`staff_appraisal` / `staff_appraisal_score`).
- Jika skor diubah/dikunci di gateway, summary otomatis mencerminkannya pada render berikutnya — tidak ada sinkronisasi manual.
- Grade A/B/C/D (mis. 86.04(B)) yang tampil di gateway dihitung dari total; summary report untuk fase 1 menampilkan total mentah (grade bisa jadi enhancement).

## Konvensi Penamaan

| Folder slug | Deskripsi | Scope |
|-------------|-----------|-------|
| `appraisal-summary` | Appraisal Summary Report — fitur ini | Report agregat 17 (teacher) / 9 (HOD) dimensi per campus + export/print |
| `appraisal-new` | New Appraisal / Staff Database gateway | Input skor appraisal (daftar guru + form 18 dimensi) — sumber data summary |
| `petals` | PETALS Summary Report | Report view observasi PETALS (Lesson Observation) |
| `petals-observation` | PETALS Lesson Observation Input | Input skor observasi 18 item (mark 0-4) |
| `epetals-dashboard` | E-PETALS Dashboard & Petal Chart | Multi-campus dashboard + chart |

## Open Questions

- **NQ-01:** Apakah report teacher summary hanya menampilkan skor yang sudah **di-lock**? Di legacy, skor tersimpan langsung saat input gateway (state param `update_appraisal_new.php`). Perlu konfirmasi apakah summary hanya menampilkan record dengan status tertentu (mis. COMPLETED) — sementara asumsi: tampilkan semua record skor valid untuk AY+campus.
- **NQ-02:** Varian Principal (tingkatan ketiga) — apakah summary teacher mencakup Principal juga? Legacy punya `asd_staff_app_principal_new.php`; probe summary teacher hanya menampilkan guru. Untuk fase 1: hanya TEACHER dan HOD (sesuai probe), Principal di luar scope.
- **NQ-03:** Export format — legacy memakai `print_appraisal_summary.php` (terlihat menghasilkan file, kemungkinan HTML→Excel via `library/html2excel`). Sistem baru pakai `exceljs` (lihat D-03 di `features/petals/notes.md`).

## Enhancement Ideas (di luar scope fase 1)

- Dril-down: klik nama guru → lihat rincian skor per dimensi + history per observer.
- Grade otomatis A/B/C/D dari total.
- Chart distribusi skor per dimensi (referensi `features/epetals-dashboard/`).
- History trend antar AY untuk guru yang sama.
- Perbandingan antar campus untuk Super Admin (referensi `appraisal_summary_asd.php`).
