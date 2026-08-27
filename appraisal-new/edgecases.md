# Edge Cases — New Appraisal / Staff Database

## EC-AP-01: Guru belum pernah di-appraise ("Never!" / score 0)

**Skenario:** Guru baru (mis. bergabung pertengahan tahun) belum pernah di-appraise sama sekali. Di legacy, Principal view menampilkan "Never!" dan Teacher view menampilkan "0()".

**Penanganan:** Jika tidak ada record `staff_appraisal` untuk guru tersebut di AY aktif (atau score = 0 dan status = INCOMPLETE), backend mengembalikan `score: 0`, `grade: null`, `neverAppraised: true`. Frontend menampilkan "Never!" (Principal view) / "0()" (Teacher view). Tombol Appraisal tetap tersedia untuk memulai appraisal baru.

## EC-AP-02: Grade A/B/C/D — boundary nilai

**Skenario:** Score 89.99 vs 90.00 — apakah keduanya grade B atau A? Score persis di threshold juga perlu diputuskan.

**Penanganan:** Threshold inklusif di batas atas: A >= 90, B >= 75, C >= 60, D < 60. Jadi score 90.00 = A, score 89.99 = B, score 75.00 = B, score 74.99 = C. **Keputusan: batas bawah inklusif (>=), kecuali D yang < 60.** Threshold final perlu diverifikasi dengan data legacy sebelum implementasi.

## EC-AP-03: Weighting 80/20 Teacher + HOD

**Skenario:** Guru sudah di-appraise oleh Teacher (score 91.979) dan oleh HOD (score 90.625). Total yang ditampilkan 91.708 — bagaimana hitungannya?

**Penanganan:** `combinedScore = teacherScore * 0.8 + hodScore * 0.2` → `91.979 * 0.8 + 90.625 * 0.2 = 73.5832 + 18.125 = 91.708`. Jika salah satu komponen belum ada (mis. HOD belum menilai), combined score tidak dihitung — tampilkan komponen yang ada saja dengan status Incomplete. **Keputusan: combined score hanya dihitung jika kedua komponen tersedia.**

## EC-AP-04: Filter campus kosong (All Campus)

**Skenario:** User memilih "All Campus" pada filter campus — seluruh ~640 guru dari 13 campus ditampilkan.

**Penanganan:** Query tanpa `campusId` → backend tidak memfilter campus. Pagination wajib aktif (default pageSize 50) agar performa tetap baik untuk 640+ baris. Empty state hanya muncul jika tidak ada guru sama sekali (bukan karena campus kosong).

## EC-AP-05: Guru pindah campus di tengah AY

**Skenario:** Guru mengajar di campus A pada Semester 1, lalu pindah ke campus B pada Semester 2. Appraisal lama tercatat di campus A.

**Penanganan:** `staff_appraisal.campus_id` diisi campus saat appraisal dibuat (atau saat submit terakhir). Guru muncul di daftar kedua campus — di campus B dengan status belum di-appraise (belum ada record untuk campus B), di campus A dengan status lama. **Keputusan: record appraisal tetap di campus saat appraisal dibuat; histori tidak dipindahkan.**

## EC-AP-06: Multiple reviewer untuk satu guru di AY yang sama

**Skenario:** Teacher di-appraise oleh diri sendiri (TEACHER), oleh HOD (HOD), dan oleh Principal (PRINCIPAL) di AY yang sama — masing-masing dengan score berbeda.

**Penanganan:** Kombinasi unik `(teacher_id, academic_year_id, appraisal_type, reviewer_id)` memungkinkan record per tipe. Untuk HOD view, `teacher_score` diambil dari record `appraisal_type = TEACHER` dan `hod_score` dari record `appraisal_type = HOD` untuk guru yang sama. **Keputusan: satu record per (teacher, AY, appraisal_type, reviewer); agregasi HOD view memadukan dua record.**

## EC-AP-07: Appraisal sudah submit — user mencoba edit/menilai ulang

**Skenario:** Appraisal berstatus COMPLETED, tapi Principal ingin mengubah nilai setelah koreksi (mis. salah input di observasi PETALS).

**Penanganan:** Backend mengizinkan PUT status kembali ke INCOMPLETE hanya oleh reviewer pemilik record atau admin (permission `APPRAISAL_MANAGE`). Saat status diubah ke INCOMPLETE, `date_submit` di-reset ke NULL. Setelah submit ulang, `date_submit` diperbarui. Frontend menampilkan tombol "PDF Report" hanya saat COMPLETED, dan form PETALS dalam mode view-only saat COMPLETED (edit memerlukan unlock via Lock/Unlock — menu id 392 di teacher portal).

## EC-AP-08: Academic year yang tidak aktif

**Skenario:** User memilih AY yang sudah closed (mis. AY 2024/2025) — data appraisal lama masih tampil tapi AY tidak lagi aktif.

**Penanganan:** Backend menolak filter AY non-aktif dengan 400 "Academic year not found or inactive". Untuk kebutuhan histori, enhancement: tampilkan read-only AY non-aktif (tanpa aksi edit/Appraisal). Default list selalu menggunakan AY aktif.
