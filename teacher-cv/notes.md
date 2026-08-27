# Notes — Teacher CV

## Ringkasan Analisis

| Portal legacy | Temuan |
|---------------|--------|
| Teacher portal | Menu id **30** — di bawah modul "Staff & HR / Kepegawaian" (section 9 dari teacher portal feature inventory) → menunjuk ke `staff/asd_staff_cv.php` (halaman CV guru, printable document). |
| Portal ASD | `asd_staff.php` — halaman staff management (list/kelola data staff) — sumber data kepegawaian yang sama; varian yang fungsinya berbeda dari CV view. |
| Principals portal | Tidak ada halaman khusus CV yang terdokumentasi — akses principal diasumsikan melalui teacher portal (CV staff per campus) atau admin portal. **ASUMSI.** |

## Ketersediaan Referensi (PENTING)

- **Tidak ada HTML dump, tidak ada probe, tidak ada screenshot** untuk `staff/asd_staff_cv.php` pada saat brief ini ditulis.
- Yang tersedia **hanya**: menu id 30 → `staff/asd_staff_cv.php`, varian `asd_staff.php` di portal ASD, dan konsep "CV / riwayat hidup printable".
- **Konsekuensi:** seluruh detail layout, nama field, urutan seksi, dan struktur data di `spec.md` / `api-contract.md` adalah **ESTIMASI / PROPOSAL** — bukan fakta dari legacy.
- **Saran:** re-probe `staff/asd_staff_cv.php` (dan `asd_staff.php`) untuk konfirmasi layout CV, nama field, dan urutan seksi. Setelah ada dump HTML, perbarui brief ini dan tandai field yang terkonfirmasi.

## Asumsi yang Perlu Diverifikasi

| # | Asumsi | Status |
|---|--------|--------|
| A1 | CV menampilkan seksi: Data Pribadi, Pendidikan, Pengalaman Kerja, Kualifikasi/Sertifikasi, Pelatihan | ASUMSI (konseptual, umum untuk CV) |
| A2 | Urutan seksi: Data Pribadi → Pendidikan → Pengalaman Kerja → Kualifikasi/Sertifikasi → Pelatihan | ASUMSI |
| A3 | `employee-identity` menyimpan NIK, DOB, gender, alamat, kontak, foto | ASUMSI (pemetaan kolom proposal) |
| A4 | `employee-position` cukup untuk seksi Pengalaman/Kualifikasi; bila tidak → tabel baru `teacher_experience` | PERLU KONFIRMASI modul HR |
| A5 | Data pendidikan belum ada di api_nest → tabel opsional `teacher_education` | PERLU KONFIRMASI |
| A6 | Riwayat pelatihan bisa diambil dari modul `training-staff` sebagai seksi Pelatihan | PERLU KONFIRMASI integrasi |
| A7 | Principal dapat mengakses CV staff per campus-nya via teacher portal | ASUMSI |
| A8 | Print dilakukan via browser print dialog (client-side), bukan PDF server-side | ASUMSI (enhancement: export PDF server-side) |

## Keputusan: Perlu/Tidak Tabel Pendidikan & Pengalaman Baru

- **Posisi brief ini:** tidak ada tabel baru yang WAJIB untuk menampilkan CV — semua bisa di-compose dari entitas existing (`employee`, `employee-identity`, `employee-position`).
- **Jika** data pendidikan/pengalaman memang belum terstruktur di api_nest, maka diusulkan **dua tabel OPSIONAL**: `teacher_education` dan `teacher_experience` (detail kolom di `schema.md`).
- **Keputusan akhir ada di modul HR** — apakah akan (1) membaca dari tabel existing, (2) membuat tabel baru, atau (3) menunda seksi pendidikan/pengalaman sampai data tersedia. Jangan membuat tabel tanpa keputusan tersebut.
- Rekomendasi awal: rilis awal cukup menampilkan seksi yang sumber datanya sudah ada; seksi tanpa sumber → placeholder "—" / array kosong (lihat EC-01).

## Relasi ke Fitur Lain

| Fitur / Modul | Relasi dengan Teacher CV |
|---------------|--------------------------|
| `training-staff` | Riwayat pelatihan guru — **berpotensi menjadi sumber seksi "Pelatihan"** di CV (perlu konfirmasi integrasi). |
| `employee-identity` | Data identitas (NIK, DOB, gender, alamat, kontak, foto) — sumber seksi Data Pribadi. |
| Modul HR / staff management (setara `asd_staff.php` di portal ASD) | Sumber data kepegawaian yang dibaca CV; juga pemilik data pendidikan/pengalaman (input). |
| `employee-position` | Jabatan/posisi — dasar seksi Pengalaman Kerja & Kualifikasi. |

## Catatan Desain

- **Dual portal:** Admin Portal (`client/`) = pengguna utama (Admin/ASD lintas campus); Teacher Portal (`client-teacher/`) = mirroring (guru hanya CV sendiri; Principal per campus-nya).
- **Read-only:** tidak ada endpoint tulis di resource `teacher-cv`; edit data adalah tanggung jawab modul HR.
- **Placeholder:** data kosong ditampilkan "—" / array kosong — tidak error (EC-01).
- **Print multi-page:** CSS `@media print` + `page-break-inside: avoid` (EC-02).
- **Field names:** semua nama field di `api-contract.md` adalah **proposal** sampai ada dump HTML legacy.

## Konvensi Penamaan

| Folder slug | Deskripsi | Scope |
|-------------|-----------|-------|
| `teacher-cv` | Teacher CV — fitur ini | Read-only view + print/export dokumen CV per staff |
| `teacher-card` | Teacher Card (sibling) | Kartu identitas guru (fitur terpisah, hanya punya folder screenshots kosong) |
| `training-staff` | Training Staff (sibling) | Riwayat pelatihan guru — sumber potensial seksi Pelatihan di CV |
