# Edge Cases — EPMS Work Review

## EC-EP-01: Guru pindah campus di tengah AY

**Skenario:** Guru mengajar di campus A pada Semester 1, lalu pindah ke campus B pada Semester 2.

**Penanganan:** `teacher_review.campus_id` diisi campus saat review dibuat. Principal bisa membuat review terpisah per campus per semester (2 review untuk 1 guru di 1 AY). Atau cukup 1 review di campus saat review dibuat — catatan di Interschool/JOB Desc field bisa dipakai. **Keputusan: review di campus saat ini, notasinya di field interschool.**

## EC-EP-02: Principal baru di tengah AY — review sisa dari Principal sebelumnya

**Skenario:** Principal A sudah mengisi review Semester 1, lalu Principal B menggantikan di Semester 2.

**Penanganan:** Kombinasi unik `(teacher_id, academic_year_id, reviewer_id)` — setiap Principal punya review sendiri. Namun di legacy hanya ada 1 review per guru per AY. **Keputusan: 1 review per (teacher_id, academic_year_id) — reviewer_id di-update jika berubah.**

## EC-EP-03: Section 1-5 belum lengkap, tapi user mencoba submit

**Skenario:** User klik Submit padahal beberapa item di Section 1-5 masih kosong.

**Penanganan:** Backend wajib validasi semua item Section 1-5 terisi (minimal 1 semester). Jika ada yang kosong, return 400 dengan daftar item yang belum diisi. Section 6 & 7 opsional.

## EC-EP-04: Teacher's Comments vs Reporting Officer's Comments — siapa yang mengisi?

**Skenario:** Di legacy, form hanya memiliki textarea untuk Reporting Officer's Comments (`s72`, `s74`). Teacher's Comments tidak memiliki textarea di HTML (hanya `<td></td>` kosong).

**Penanganan:** Di sistem baru, sediakan textarea untuk keduanya. Reporting Officer's Comments diisi oleh Principal. Teacher's Comments bisa diisi oleh guru sendiri (jika akses read-only nanti) atau diisi oleh Principal atas nama guru. **Keputusan: Principal mengisi semua kolom (Teacher's Comments sebagai proxy untuk guru).**

## EC-EP-05: Academic Year yang tidak aktif

**Skenario:** User memilih AY yang sudah tidak aktif (mis. AY 2024/2025 yang sudah closed).

**Penanganan:** Backend hanya menampilkan data untuk AY dengan `activeStatus = ACTIVE`. Jika user memfilter AY yang tidak aktif, return 400 dengan pesan "Academic year not found or inactive."

## EC-EP-06: Campus tanpa data guru

**Skenario:** Principal membuka halaman EPMS untuk campus yang tidak memiliki guru (mis. campus baru).

**Penanganan:** List guru kosong → tampilkan BBSNoItemCard dengan pesan "No teachers found for this campus." Tidak ada tombol Review yang muncul.

## EC-EP-07: Review sudah SUBMITTED — user coba edit

**Skenario:** User membuka review yang sudah di-submit dan mencoba mengubah skor.

**Penanganan:** Backend reject PUT /scores dan PUT /comments dengan 409 "Review is already submitted." Admin dengan `PETALS_MANAGE` bisa override (buka kunci). Frontend disabled form jika status = SUBMITTED.

## EC-EP-08: Skor melebihi batas wajar

**Skenario:** User memasukkan skor 200 pada item yang seharusnya 0-100.

**Penanganan:** Backend validasi skor sesuai range yang didefinisikan di `epms_review_item`. Default: 0-100. Return 400 "Score must be between 0 and 100."

## EC-EP-09: Hubungan EPMS dengan PETALS — duplikasi data?

**Skenario:** Developer bertanya apakah EPMS membaca data dari PETALS (mis. skor P/E/T/A/L dimasukkan ke EPMS).

**Penanganan:** Tidak ada. EPMS adalah instrumen independen. Data PETALS (lesson observation) tidak masuk ke EPMS, dan sebaliknya. Keduanya hanya sama-sama bagian dari modul "Appraisal & Performance" di sidebar. **Keputusan: tidak ada relasi database antara EPMS dan PETALS.**

## EC-EP-10: Multiple reviewer untuk satu guru di AY yang sama

**Skenario:** Principal dan HOD sama-sama mengisi review untuk guru yang sama di AY yang sama.

**Penanganan:** Di legacy, hanya Principal yang mengisi EPMS (menu id 391 `principal_teacher_review.php`). HOD memiliki menu terpisah (`asd_staff_app_hod_new.php` id 3219). Untuk EPMS, hanya 1 reviewer per guru per AY. HOD punya appraisal terpisah (New Appraisal). **Keputusan: 1 reviewer per guru per AY untuk EPMS.**
