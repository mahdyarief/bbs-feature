# Edge Cases — PETALS Summary Report

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

## EC-01: Tidak ada data appraisal untuk campus tertentu

**Scenario:** Principal/HOD membuka halaman PETALS untuk campus yang belum memiliki data appraisal (misal awal tahun akademik).

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan empty state | Tabel kosong dengan pesan "No appraisal data found for this campus" — BBSNoItemCard. |
| (B) Sembunyikan halaman | Redirect ke halaman lain atau 404 — tidak informatif. |

**Decision:** _TBD_ — rekomendasi: **(A) Empty state** dengan pesan informatif.

---

## EC-02: Skor di luar range yang diharapkan

**Scenario:** Data appraisal memiliki skor P/E/T/A/L yang di luar rentang normal (misal P=15 padahal max 12).

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan apa adanya | Report hanya menampilkan data apa adanya — validasi dilakukan di input, bukan di report. |
| (B) Validasi di report | Report memvalidasi range dan menandai skor invalid dengan warna merah. |

**Decision:** _TBD_ — rekomendasi: **(A) Tampilkan apa adanya**. Validasi harus dilakukan di entry point (form input appraisal), bukan di report view.

---

## EC-03: CampusId tidak ditemukan di req.user

**Scenario:** User login (Principal/HOD) tidak memiliki campusId di token JWT.

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan dropdown campus | User bisa memilih campus dari list — fallback jika tidak ada default. |
| (B) Tolak dengan 400 | "Campus not found for user" — user harus diassign ke campus. |

**Decision:** _TBD_ — rekomendasi: **(A) Dropdown campus** sebagai fallback. Principal/HOD mungkin perlu melihat multiple campus.

---

## EC-04: Export Excel saat data kosong

**Scenario:** User mengklik Export to Excel ketika tidak ada data appraisal.

| Opsi | Behavior |
|------|----------|
| (A) Export file kosong | Download file Excel kosong dengan header kolom saja. |
| (B) Tolak dengan toast | Muncul toast "No data to export" — tidak ada file yang di-download. |

**Decision:** _TBD_ — rekomendasi: **(A) Export file kosong** dengan header. Konsisten dengan behavior umum export.

---

## EC-05: Banyak data (100+ guru)

**Scenario:** Campus besar dengan 100+ guru — tabel jadi panjang.

| Opsi | Behavior |
|------|----------|
| (A) Pagination server-side | `GET /api/v1/petals/report` dengan pagination — default pageSize 20. |
| (B) Scroll tanpa pagination | Load semua data — untuk report, scroll lebih natural. |

**Decision:** _TBD_ — rekomendasi: **(A) Pagination server-side** dengan ukuran page yang lebih besar (default 20-50) karena ini report, bukan form entry. Tapi untuk export Excel, ambil semua data.

---

## EC-06: Multiple academic year

**Scenario:** Ada data appraisal untuk beberapa tahun akademik — perlu filter tahun.

| Opsi | Behavior |
|------|----------|
| (A) Filter AY dropdown | Tambah filter Academic Year — default AY aktif. |
| (B) Tampilkan semua | Tampilkan semua data tanpa filter AY — report bisa campur aduk. |

**Decision:** _TBD_ — rekomendasi: **(A) Filter AY dropdown**. Appraisal dilakukan per AY, jadi harus bisa difilter. Default AY aktif.