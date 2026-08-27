# Edge Cases — Appraisal Summary Report

## EC-AS-01: Campus tanpa data appraisal

**Skenario:** Principal membuka halaman Appraisal Summary untuk campus yang belum punya data skor appraisal (mis. campus baru, atau belum ada guru yang di-appraise di AY terpilih).

**Penanganan:** Backend return `{ data: [], count: 0 }` (HTTP 200). Frontend tampilkan BBSNoItemCard dengan pesan "No appraisal data found for this campus." Tombol Export Excel tetap aktif tapi menghasilkan file dengan header kolom saja (tanpa baris data).

## EC-AS-02: Academic Year yang tidak aktif

**Skenario:** User memilih AY yang sudah tidak aktif / closed (mis. AY 2024/2025) pada dropdown filter.

**Penanganan:** Backend hanya menampilkan data untuk AY dengan `activeStatus = ACTIVE`. Jika user memfilter AY yang tidak aktif, return 400 dengan pesan "Academic year not found or inactive." Dropdown frontend sebaiknya hanya menampilkan AY aktif.

## EC-AS-03: Skor null / guru belum di-appraise

**Skenario:** Sebagian guru di campus belum di-appraise (tidak ada record skor, atau semua skor dimensi null).

**Penanganan:** Guru dengan skor null / tidak ada record skor **tidak muncul** di report (AC-8). Hanya guru yang punya setidaknya satu record skor valid di AY+campus tersebut yang dirender. Untuk dimensi yang nilainya null tapi guru muncul (sebagian dimensi kosong), tampilkan sebagai `-` atau `0`? **Keputusan: tampilkan `-` (dash) untuk dimensi yang null dan hitung total dari nilai yang ada; jika seluruh dimensi null, guru tidak dirender sama sekali.**

## EC-AS-04: Total melebihi batas maks

**Skenario:** Karena data dari gateway tidak valid, total sum 17 dimensi melebihi total teoretis (99) — mis. skor salah input di gateway.

**Penanganan:** Backend hitung total apa adanya (sum), tapi **validasi** menggunakan config `appraisal_dimension.max_mark`: jika skor suatu dimensi > max_mark, backend log warning dan tampilkan flag (mis. `invalid: true`) agar QA bisa menelusuri ke gateway. Frontend tidak memblokir tampilan — report bersifat read-only, root cause ada di input gateway (`features/appraisal-new/`).

## EC-AS-05: HOD summary hanya untuk HOD

**Skenario:** Teacher biasa membuka varian HOD summary (menu 342) atau Principal membuka HOD summary untuk campus yang tidak punya HOD.

**Penanganan:** Permission: type=HOD hanya bisa diakses oleh role HOD/Principal/Super Admin/Admin. Teacher biasa → 403 "You don't have permission to access HOD appraisal summary." Jika campus tidak punya HOD yang di-appraise → empty state (EC-AS-01).

## EC-AS-06: Filter campus kosong / tidak dipilih

**Skenario:** User tidak memilih campus (filter kosong) — khususnya di Admin Portal yang punya akses lintas campus.

**Penanganan:** Di Teacher Portal, `campusId` default diambil dari `req.user` (tidak boleh kosong — 400 "Campus is required" jika tidak resolvable). Di Admin Portal, jika `campusId` kosong, **wajib diisi** (dropdown campus) → return 400 "Campus is required" agar tidak tanpa sengaja meng-agregasi semua campus.

## EC-AS-07: Export Excel kosong / campus tanpa data

**Skenario:** User mengklik "Export to Excel File" padahal tabel kosong (campus tanpa data, EC-AS-01).

**Penanganan:** Tetap generate file Excel yang valid dengan baris header (#, User ID, Name, 17/9 dimensi, TOTAL) dan nol baris data — jangan return error. Frontend tampilkan toast "No data to export." saat tabel kosong sebelum memanggil endpoint export.
