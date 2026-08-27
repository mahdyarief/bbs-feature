---
feature: National Report Card
slug: national-report
---

# Edge Cases — National Report Card

## EC-01: Class tanpa data nilai

**Skenario:** Admin generate PDF untuk class yang tidak memiliki data nilai (siswa baru atau belum di-assessment).

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Tampilkan error "No grade data" (422) | Admin harus mengisi data nilai terlebih dahulu |
| **B** | Generate PDF dengan nilai kosong/placeholder (-) | PDF tetap terbit tapi tidak valid |
| **C** | Generate PDF hanya untuk siswa yang punya data | PDF tidak lengkap, siswa lain hilang |

**Decision:** _TBD — rekomendasi Opsi A (error) karena SKL/Transkrip adalah dokumen resmi, tidak boleh kosong._

---

## EC-02: Siswa pindah di tengah tahun

**Skenario:** Siswa pindah masuk/keluar di tengah AY, sehingga data nilai tidak lengkap.

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Sertakan siswa dengan data yang ada | PDF tidak lengkap |
| **B** | Keluarkan siswa dari list PDF | Perlu validasi admin |
| **C** | Sertakan dengan catatan "Pindah" | Transparan tapi tidak resmi |

**Decision:** _TBD — rekomendasi Opsi A dengan catatan di PDF._

---

## EC-03: AY tidak memiliki class

**Skenario:** Admin memilih AY yang tidak memiliki class (misal AY terlalu baru).

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Tampilkan tabel kosong "No classes available" | Informasi jelas |
| **B** | Redirect ke AY terdekat yang punya class | Membingungkan |
| **C** | Tampilkan error | Tidak user-friendly |

**Decision:** _TBD — rekomendasi Opsi A (tabel kosong dengan pesan)._

---

## EC-04: Varian Old vs New

**Skenario:** Admin memilih varian "Old" untuk format lama.

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Implementasi kedua varian secara terpisah | Maintenance lebih berat |
| **B** | Hanya implementasi "New", abaikan "Old" | Breaking change untuk legacy users |
| **C** | "Old" adalah alias/template berbeda dari "New" | Kode reuse |

**Decision:** _TBD — rekomendasi Opsi C (gunakan template berbeda, reuse data layer)._

---

## EC-05: Nilai tidak sesuai level logic

**Skenario:** Level siswa tidak jelas (misal campuran JC2 dan Sec 4 dalam satu class).

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Deteksi level per siswa dan terapkan formula berbeda | Kompleks tapi akurat |
| **B** | Gunakan satu formula untuk semua siswa di class | Sederhana tapi tidak akurat |
| **C** | Minta admin memilih level per siswa | Manual dan lambat |

**Decision:** _TBD — rekomendasi Opsi A (deteksi otomatis per siswa)._

---

## EC-06: PDF terlalu besar

**Skenario:** Class dengan banyak siswa (40+) menghasilkan PDF puluhan MB.

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Batasi 1 PDF per class (unlimited) | Bisa besar |
| **B** | Split per N siswa (misal 20 per file) | Perlu handling multiple files |
| **C** | Kompresi PDF (image quality rendah) | Kualitas turun |

**Decision:** _TBD — rekomendasi Opsi A dengan optimasi ukuran file._

---

## EC-07: Download gagal

**Skenario:** Proses generate PDF gagal di tengah (timeout, server error).

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Return error 500 dengan retry button | User-friendly |
| **B** | Async job dengan notifikasi | Lebih kompleks |
| **C** | Coba lagi otomatis 1x | Risk of double load |

**Decision:** _TBD — rekomendasi Opsi A (sync dengan retry)._

---

## EC-08: Akses lintas campus

**Skenario:** Admin dari campus A mencoba generate PDF untuk class di campus B.

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Blok akses (403) — hanya campus sendiri | Aman, sesuai role |
| **B** | Izinkan untuk superadmin | Fleksibel |
| **C** | Izinkan semua (tidak ada filter campus) | Risk of data leakage |

**Decision:** _TBD — rekomendasi Opsi A + B (superadmin bypass)._

---

## EC-09: Data PUM/FYA/SA1 belum ada di smartbag

**Skenario:** Smartbag belum memiliki data PUM/FYA/SA1 yang dibutuhkan untuk formula Final Mark.

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Tunda implementasi sampai data nilai tersedia | Menunda fitur |
| **B** | Gunakan GP Value × Unit saja (tanpa level logic) | Tidak akurat untuk JC2/Sec 4 |
| **C** | Implementasi parsial: hanya untuk level yang datanya ada | Gradual |

**Decision:** _TBD — rekomendasi Opsi C (implementasi bertahap)._

---

## EC-10: Nomor SKL/Transkrip duplikat

**Skenario:** Nomor SKL atau Transkrip terduplikasi karena regenerate.

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Generate nomor unik setiap kali | Tidak bisa regenerate identik |
| **B** | Simpan nomor pertama, reuse untuk regenerate | Konsisten |
| **C** | Append versi (v1, v2) | Melacak revisi |

**Decision:** _TBD — rekomendasi Opsi B (nomor tetap, data diperbarui)._