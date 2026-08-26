# Edge Cases — E-PETALS Dashboard & Petal Chart

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.

## EC-EP-01: Campus tanpa data di chart

**Scenario:** Satu atau beberapa campus tidak memiliki data PETALS untuk AY terpilih.

| Opsi | Behavior |
|------|----------|
| (A) Skip campus | Campus tanpa data tidak muncul di chart — hanya campus dengan data yang tampil. |
| (B) Tampilkan 0 | Campus tetap muncul di chart dengan nilai 0 — user bisa melihat campus mana yang belum diobservasi. |

**Decision:** _TBD_ — rekomendasi: **(B) Tampilkan 0** dengan warna berbeda/grey. Lebih informatif: Principal HQ bisa langsung melihat campus mana yang belum meng-input observasi. Tapi perlu konfirmasi (query legacy hanya mengembalikan campus dengan data).

---

## EC-EP-02: Campus id tidak valid di query

**Scenario:** User mengirim `campusId=999` (tidak ada di database) ke `/v1/epetals/summary`.

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 400 | "Campus not found" — standar validasi. |
| (B) Return empty list | Data kosong — tidak error. |

**Decision:** _TBD_ — rekomendasi: **(A) Tolak dengan 400**. Konsisten dengan error handling modul lain (petals report EC invalid campus → 400).

---

## EC-EP-03: AY tidak ditemukan / tidak valid

**Scenario:** User mengirim `ay=999` (tidak ada di tabel academic_year).

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 400 | "Academic year not found". |
| (B) Return empty chart | Chart kosong + empty state. |

**Decision:** _TBD_ — rekomendasi: **(A) Tolak dengan 400** di backend; frontend dropdown AY hanya menampilkan AY valid.

---

## EC-EP-04: Tidak ada data sama sekali untuk AY terpilih

**Scenario:** AY terpilih belum ada observasi PETALS sama sekali (misal awal tahun akademik baru).

| Opsi | Behavior |
|------|----------|
| (A) Empty state chart | Chart kosong dengan pesan "No PETALS data found for the selected academic year" — BBSNoItemCard. |
| (B) Chart semua 0 | Semua bar = 0 — misleading (seolah skor 0, padahal belum diobservasi). |

**Decision:** _TBD_ — rekomendasi: **(A) Empty state**. Bedakan "belum diobservasi" dari "skor 0".

---

## EC-EP-05: Akses lintas campus untuk Principal campus tunggal

**Scenario:** Principal biasa (bukan HQ) mengakses E-PETALS dashboard yang menampilkan semua campus.

| Opsi | Behavior |
|------|----------|
| (A) Batasi ke campus miliknya | Principal biasa hanya melihat campus-nya sendiri; shell hanya menampilkan 1 campus. |
| (B) Izinkan semua | Principal bisa melihat semua campus. |

**Decision:** _TBD_ — rekomendasi: **(A) Batasi ke campus miliknya**. E-PETALS lintas campus hanya untuk Principal HQ / Super Admin / Admin (`PETALS_MANAGE`). Principal campus biasa tetap pakai PETALS Summary Report single-campus.

---

## EC-EP-06: Banyak campus dengan data di chart (overcrowding)

**Scenario:** Semua 11 campus punya data — label sumbu X jadi sempit.

| Opsi | Behavior |
|------|----------|
| (A) Rotate label | Rotate label 45° dan tooltip per bar — tetap tampil semua campus. |
| (B) Horizontal scroll | Chart bisa di-scroll horizontal. |

**Decision:** _TBD_ — rekomendasi: **(A) Rotate label 45° + tooltip**. 11 bar masih muat di satu layar; rotasi label cukup.