# Edge Cases — Training Staff

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

## EC-01: datefrom > dateto (rentang tanggal tidak valid)
**Scenario:** User mengisi tanggal mulai (datefrom) lebih besar dari tanggal selesai (dateto) — mis. datefrom=2026-08-20, dateto=2026-08-10.

| Opsi | Behavior |
|------|----------|
| (A) Tolak di frontend (datepicker min/max + validasi form) | User tidak bisa submit; pesan error langsung sebelum request dikirim. |
| (B) Tolak di backend (400 "Date from must be before or equal to date to") | Backend selalu validasi; frontend hanya menampilkan error dari API. |
| (C) A+B (frontend & backend) | Validasi berlapis — UX cepat di frontend, keamanan di backend. |

**Decision:** _TBD_ — rekomendasi awal: (C). Backend WAJIB validasi (data dapat masuk via API langsung); frontend validasi untuk UX. Perilaku legacy tidak terdokumentasi (tidak ada bukti validasi di `savetraining.php`).

---

## EC-02: Training tanpa staff assigned
**Scenario:** User menyimpan pelatihan tanpa memilih staff sama sekali (`staff_id` / `staffIds` kosong).

| Opsi | Behavior |
|------|----------|
| (A) Tolak — minimal 1 staff wajib | 400 "At least one staff must be assigned"; record tidak dibuat. |
| (B) Izinkan simpan tanpa staff | Record master tersimpan; `staff_training_staff` kosong; nanti bisa di-assign via edit. |

**Decision:** _TBD_ — rekomendasi awal: (A) minimal 1 staff, karena fitur ini adalah riwayat training *per staff* dan filter campus bergantung pada staff ter-assign. Catatan: legacy `#form-field-tags` bersifat opsional di HTML, namun tujuan fitur (riwayat per staff) menuntut minimal 1.

---

## EC-03: country_id kosong
**Scenario:** User tidak memilih negara pada form (`country_id` kosong) — field opsional di legacy.

| Opsi | Behavior |
|------|----------|
| (A) Simpan NULL | `country_id = NULL`; tampil kosong di list; dropdown tetap bisa dipilih saat edit. |
| (B) Default ke negara Indonesia | Isi otomatis `country_id` default sistem (mis. Indonesia) saat create; user bisa ubah. |

**Decision:** _TBD_ — rekomendasi awal: (B) default ke negara sistem (umumnya Indonesia untuk mayoritas pelatihan), konsisten dengan praktik `employee` yang memakai default country. Perlu konfirmasi apakah default sistem tersedia di modul `country`.

---

## EC-04: Training lintas campus (staff dari campus berbeda di satu training)
**Scenario:** Satu record pelatihan di-assign ke staff dari campus berbeda (mis. 2 staff PIK-S + 1 staff BDG-P). Legacy memfilter list per campus, sehingga record ini harus muncul di kedua filter campus.

| Opsi | Behavior |
|------|----------|
| (A) Tanpa kolom campus — derived dari staff | Filter per campus memakai EXISTS subquery ke `staff_training_staff` join employee.campus_id; record muncul di semua campus staff-nya. Tidak ada "campus utama". |
| (B) Kolom `campus_id` eksplisit (keputusan brief ini) | Simpan campus staff pertama/utama; filter list pakai `campus_id`. Record lintas campus hanya muncul di campus utama saat list — detail tetap menampilkan semua staff. |
| (C) Kolom `campus_id` eksplisit + derived fallback | Filter = `campus_id` utama; jika user butuh semua, tambahkan toggle "tampilkan training lintas campus yang melibatkan campus ini". |

**Decision:** _TBD_ — brief ini memilih (B) sebagai keputusan awal (lihat `schema.md` catatan desain): kolom `campus_id` eksplisit untuk performa filter list, dengan catatan bahwa record lintas campus ter-grouping di campus utama. Opsi (C) adalah enhancement jika kebutuhan bisnis menuntut visibility lintas campus penuh.

---

## EC-05: Duplicate training (judul + tanggal sama)
**Scenario:** User membuat pelatihan baru dengan judul dan rentang tanggal yang identik dengan record yang sudah ada (untuk kombinasi staff yang sama).

| Opsi | Behavior |
|------|----------|
| (A) Tolak (409) | "Training with same title and date range already exists"; user harus mengubah judul/tanggal. |
| (B) Izinkan | Tidak ada duplicate check — bisa ada record kembar (perilaku legacy, tidak ada bukti cek duplikasi). |
| (C) Tolak hanya jika kombinasi staff sama persis | Duplicate check dengan memperhitungkan staffIds — record dengan judul+tanggal sama tapi staff berbeda diizinkan (mis. batch pelatihan regional). |

**Decision:** _TBD_ — rekomendasi awal: (C), karena satu judul pelatihan bisa diulang untuk batch/campus berbeda. Implementasi: cek title + date range overlap + staffIds tumpang tindih.

---

## EC-06: Staff nonaktif dalam training yang sudah ada
**Scenario:** Seorang staff di-nonaktifkan (employee `active_status = INACTIVE`) tetapi masih ter-assign di record training lama.

| Opsi | Behavior |
|------|----------|
| (A) Pertahankan assignment, tandai di UI | Staff tetap muncul di detail/list dengan badge "Nonaktif"; tidak bisa ditambahkan ke training baru. |
| (B) Hapus assignment otomatis | Relasi di `staff_training_staff` dihapus saat staff nonaktif — riwayat training staff tersebut hilang dari CV. |
| (C) Pertahankan, sembunyikan di filter | Assignment dipertahankan (histori CV aman); staff nonaktif disembunyikan dari dropdown assign, tapi tetap tampil di record lama. |

**Decision:** _TBD_ — rekomendasi awal: (C) (mirip A dengan penyembunyian dari dropdown). Historis CV harus tetap utuh — menghapus assignment (B) merusak riwayat pengembangan profesional.

---

## EC-07: Delete training yang sudah direferensikan (CV/appraisal)
**Scenario:** Admin menghapus (soft delete) sebuah pelatihan yang sudah tampil di CV guru (`features/teacher-cv`) atau menjadi bahan konteks appraisal (`features/appraisal-summary`).

| Opsi | Behavior |
|------|----------|
| (A) Soft delete, referensi tetap valid | `active_status = INACTIVE`; CV menampilkan hanya ACTIVE, referensi yang sudah dibuat (snapshot CV) tetap utuh. |
| (B) Soft delete + peringatan referensi | Sebelum delete, backend mengecek jumlah referensi (CV/appraisal) dan menampilkan konfirmasi "Pelatihan ini direferensikan di N CV. Lanjutkan?" |
| (C) Hard delete | Tidak disarankan — menghapus histori permanen. |

**Decision:** _TBD_ — rekomendasi awal: (B) dengan soft delete (A sebagai mekanisme). Karena fitur CV/appraisal belum punya relasi FK ke `staff_training` (masih konseptual), pengecekan referensi dilakukan saat integrasi dibangun.

---

## EC-08: Inline update subject legacy — dipertahankan atau tidak
**Scenario:** Legacy mendukung inline update satu field dari tabel (`subjectchange_` → `services/update_subjectais.php`, param `recid` + `isi`). Apakah smartbag mempertahankan mekanisme ini?

| Opsi | Behavior |
|------|----------|
| (A) Dipertahankan | Field tertentu (mis. title, participation) dapat diedit inline di tabel via `PATCH /v1/training-staff/:id` partial; UX cepat. |
| (B) Tidak — semua edit lewat form | Hapus inline update; semua perubahan melalui form add/edit penuh; lebih sederhana, konsisten. |
| (C) Dipertahankan hanya untuk kolom tertentu | Inline update terbatas pada kolom "aman" (mis. comments, venue); kolom penting (title, tanggal, staff) wajib via form. |

**Decision:** _TBD_ — rekomendasi awal: (B), karena smartbag tidak wajib mereplikasi mekanisme legacy yang tidak terdokumentasi jelas (`update_subjectais.php` menangani "subject" yang ambig — kemungkinan untuk data mata pelajaran, bukan field training). Form edit penuh sudah mencakup kebutuhan. Jika ingin UX cepat, adopsi (C) dengan daftar kolom eksplisit.
