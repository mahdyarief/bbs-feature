# Edge Cases — Form Leave (Teacher Leave)

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

## EC-01: Tanggal `dateTo` sebelum `dateFrom`

**Scenario:** Guru memasukkan Date To yang lebih awal dari Date From (misal dari 2026-09-05 sampai 2026-09-01).

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 400 | Validasi `dateTo >= dateFrom` — pesan "Date To must be greater than or equal to Date From". |
| (B) Auto-swap | Server menukar kedua tanggal — user tidak sadar datanya berubah. |

**Decision:** _TBD_ — rekomendasi: **(A) Tolak dengan 400** + validasi yup di frontend (blocking sebelum submit). Auto-swap berisiko user salah input tanpa sadar.

---

## EC-02: Cuti overlapping dengan pengajuan lain

**Scenario:** Guru sudah punya leave aktif (misal 2026-09-01 sampai 2026-09-02), lalu mengajukan leave baru yang tumpang tindih (misal 2026-09-02 sampai 2026-09-05).

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 409 | Cek overlap di service — pesan "You already have a leave request in this date range". Mencegah double claim. |
| (B) Izinkan | Tidak cek overlap — guru bisa punya banyak leave tumpang tindih (teacher web legacy kemungkinan tidak cek). |

**Decision:** _TBD_ — rekomendasi: **(A) Tolak dengan 409** di fase 1 karena lebih aman dan mencegah data anomali. Catatan: teacher web legacy tampaknya tidak melakukan cek ini (belum terverifikasi), jadi opsi (B) bisa jadi fallback jika ingin replicate persis.

---

## EC-03: Guru mengirim `attachmentFileId` yang bukan miliknya / bukan PDF

**Scenario:** Guru menyertakan `attachmentFileId` yang menunjuk file milik user lain atau file non-PDF.

| Opsi | Behavior |
|------|----------|
| (A) Validasi ownership + tipe | Backend memvalidasi file milik `req.user.id` dan bertipe PDF — selain itu 400. |
| (B) Hanya validasi tipe | Cek MIME PDF saja, tanpa cek ownership — berisiko akses file orang lain. |

**Decision:** _TBD_ — rekomendasi: **(A)**. Validasi `attachmentFileId` mengarah ke file yang valid (eksis, PDF, milik user) untuk mencegah data silang.

---

## EC-04: Upload PDF via modul `file` yang menerima banyak MIME

**Scenario:** `POST /v1/files` (file.controller.ts:67) menerima banyak MIME (gambar, dokumen, dll) — bukan PDF-only.

| Opsi | Behavior |
|------|----------|
| (A) Endpoint `file` khusus leave | Tambah route `POST /v1/files/teacherLeave` dengan `FileTypeValidator` PDF-only (pola uploadSow di file.controller.ts:174). |
| (B) Validasi MIME di service leave | Setelah upload generic, service `teacher-leave` cek MIME file → 400 jika bukan PDF. |

**Decision:** _TBD_ — rekomendasi: **(A)** endpoint khusus `POST /v1/files/teacherLeave` (atau `leaveAttachment`) dengan validator PDF-only, konsisten dengan pola endpoint per-entityType di `FileController` (uploadImage, uploadSow, uploadSve, dll).

---

## EC-05: Guru tidak punya campus / data teacher tidak lengkap

**Scenario:** `req.user` tidak membawa `campusId` (atau employee record tidak punya campus assignment).

| Opsi | Behavior |
|------|----------|
| (A) Default campusId null | Simpan leave tanpa campus — reporting per campus jadi tidak akurat. |
| (B) Ambil dari employee record | Service fetch campus dari relasi Employee → jika tidak ada, tolak dengan 400 "Campus not found for teacher". |

**Decision:** _TBD_ — rekomendasi: **(B)**. Konsisten dengan pola lesson-plan (`campusId` dari `req.user` campus teacher). Jika tidak tersedia, lebih baik tolak daripada simpan data tanpa campus.

---

## EC-06: Cuti melewati akhir tahun / lintas tahun akademik

**Scenario:** Guru mengajukan leave 2026-12-28 sampai 2027-01-03 (lintas tahun).

| Opsi | Behavior |
|------|----------|
| (A) Izinkan | Simpan apa adanya — tanggal adalah date range natural, tidak terikat AY. |
| (B) Pecah per tahun | Membuat dua record (satu per tahun) — kompleks, tidak ada kebutuhan bisnis jelas. |

**Decision:** _TBD_ — rekomendasi: **(A) Izinkan**. Form leave berbasis tanggal, bukan term/week; validasi lintas tahun tidak relevan (berbeda dari lesson-plan yang terikat AY).

---

## EC-07: Guru menghapus leave yang sudah lewat / masa lalu

**Scenario:** Guru menghapus (soft delete) leave yang tanggalnya sudah lewat (misal cuti bulan lalu).

| Opsi | Behavior |
|------|----------|
| (A) Izinkan hapus kapan saja | Soft delete tanpa batas waktu — simple, konsisten dengan teacher web (tidak ada constraint). |
| (B) Blokir jika sudah lewat | Hanya bisa hapus sebelum `dateFrom` — butuh logika tambahan tanpa kebutuhan jelas. |

**Decision:** _TBD_ — rekomendasi: **(A) Izinkan kapan saja**. Teacher web tidak punya constraint ini; soft delete tetap menyimpan riwayat (hanya `activeStatus = INACTIVE`), jadi aman.

---

## EC-08: Submission List pagination / jumlah data banyak

**Scenario:** Guru sudah punya banyak pengajuan leave (puluhan atau ratusan) — panel kanan jadi panjang.

| Opsi | Behavior |
|------|----------|
| (A) Pagination server-side | `GET /v1/teacher-leaves` dengan `page`/`pageSize` + scroll/pagination di UI. |
| (B) Ambil semua tanpa pagination | List kecil, ambil semua sekaligus — sederhana tapi tidak scale. |

**Decision:** _TBD_ — rekomendasi: **(A) Pagination server-side** dengan `PageOptionsDto` — konsisten dengan konvensi api_nest; default pageSize 10.
