# Notes — Training Staff

## Ringkasan Analisis

| Sumber | Temuan |
|--------|--------|
| HTML dump | `ais/teachers/trainnig_staff.php` — halaman **"Traning Staff Viewer"** (typo legacy), campus selector 25 opsi, tombol Find (`#search`) + Add Training (`#addtraining`), container `#studentlist` |
| Screenshot | `screenshots/training_staff.png` — referensi visual halaman legacy |
| Endpoint legacy | `get_trainnig_staff.php` (list), `get_add_trainnig_staff.php` (form), `services/savetraining.php` (save/upsert), `services/update_subjectais.php` (inline update) |
| Modul | Staff & HR / Kepegawaian (teacher portal) |
| Status di smartbag | **Belum ada module** — FEATURE_COMPARISON **gap #13** ("Training Staff / Recruitment"); `api_nest` hanya punya `employee`, `employee-identity`, `teacher` |

## Typo Legacy (wajib diperbaiki di smartbag)

1. Nama file: `trainnig_staff.php` (dua "n" — seharusnya `training`). Endpoint pendukung juga `get_trainnig_staff.php` / `get_add_trainnig_staff.php` (dua "n").
2. Header halaman: **"Traning Staff Viewer"** (satu "n" hilang — seharusnya "Training").

Smartbag memakai penamaan bersih: `training-staff`, entity `StaffTraining`, endpoint `/v1/training-staff`.

## Keputusan campus_id (derived vs kolom eksplisit) — PERLU FINALISASI

Legacy memfilter list per campus (campus selector), namun data form tidak menyimpan campus secara eksplisit — campus tersirat dari staff ter-assign (`#form-field-tags`).

**Opsi yang dipertimbangkan:**

| Opsi | Deskripsi | Kelebihan | Kekurangan |
|------|-----------|-----------|------------|
| Derived only | Tidak ada kolom campus; filter per campus via EXISTS ke join table → employee.campus_id | Normalisasi bersih, tidak ada data redundan; record lintas campus muncul di semua filter | Query filter lebih kompleks (subquery); perlu index join table; list training lintas campus ambigu |
| **Kolom eksplisit `campus_id`** (dipilih brief ini) | Kolom FK campus.id di `staff_training`, diisi dari campus staff pertama saat create | Filter list cepat & sederhana (INDEX campus_id); konsisten dengan skema legacy yang memfilter per campus | Redundan jika semua staff satu campus; perlu update saat assignment berubah; record lintas campus ter-grouping di campus utama (EC-04) |

**Keputusan awal brief: kolom eksplisit `campus_id`** — didokumentasikan di `schema.md` dan `edgecases.md` EC-04. Finalisasi: tanya PM/apakah kebutuhan bisnis menuntut record lintas campus muncul di semua filter (→ derived/opsi C di EC-04).

## Relasi ke Fitur Lain (cross-reference, tanpa duplikasi konten)

- **`features/teacher-cv/`** — riwayat training staff berpotensi menjadi seksi "Training / Pengembangan Profesional" di CV guru. Data dibaca dari `staff_training_staff` (list training per employee). Perlu sinkronisasi saat CV dibangun.
- **`features/appraisal-summary/`** — riwayat pelatihan sebagai konteks pengembangan profesional dalam penilaian kinerja. Integrasi ini **Out of Scope** pada brief ini (enhancement).
- **`features/teacher-card/`** — profil staff tempat riwayat training ditampilkan.

## Saran Probe

`get_trainnig_staff.php` belum diprobed langsung (hanya HTML dump + screenshot). Saran: probe endpoint `get_trainnig_staff.php` (dengan param campus) untuk mengonfirmasi **kolom tabel hasil** yang sebenarnya di-render (apakah hanya title/date, atau termasuk venue/staff). Jika response HTML tersedia, update `spec.md` bagian UI/UX kolom tabel. Endpoint lain (`get_add_trainnig_staff.php`, `services/savetraining.php`) juga bisa diprobe untuk konfirmasi struktur form dan format request `staff_id` (array vs comma-separated).

## Daftar Asumsi

1. **`country_id` opsional** di form legacy — dianggap opsional, dengan default negara sistem (EC-03).
2. **`staff_id` minimal 1** — asumsi fitur riwayat per staff menuntut minimal 1 staff (EC-02), meski legacy tidak memvalidasi.
3. **Delete = soft delete** — legacy tidak memiliki delete; smartbag memakai `active_status = INACTIVE` agar histori CV aman (EC-07).
4. **Inline update (`subjectchange_`)** — kemungkinan menangani kolom "subject" yang tidak jelas kaitannya dengan field training; smartbag tidak wajib mereplikasi (EC-08, rekomendasi tidak dipertahankan).
5. **Response `{ edited: '1' }`** di legacy — smartbag memakai response wrapper `{ data }` standar; frontend menampilkan toast sukses berdasarkan HTTP status, bukan flag string.
6. **Dual portal** — Admin Portal pengguna utama (lintas campus); Teacher Portal mirroring (Principal per campus, guru riwayat sendiri).
7. **Endpoint `get_trainnig_staff.php` menerima `campus`** sebagai param; di smartbag menjadi query param `campusId` (0 = All Campus).
