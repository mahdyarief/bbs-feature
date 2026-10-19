# Edge Cases — FYE Kosong Saat PUM Belum Di-Migrasi

> **Status: DECIDED 2026-10-19 (user confirmed)** — perilaku: fallback ke FYE manual saat `predicted_pum` kosong; otomatis bergeser ke nilai PUM setelah migrasi selesai. Implementasi kode menyusul di task terpisah.

## EC-01: FYE kosong di GradebookTeacher tapi ada nilai di grade table
**Scenario:** `grade` punya 11 row FYE (term 4, exam_type=4, score terisi), tapi sel FYE di frontend kosong.

| Opsi | Behavior |
|------|----------|
| (A) Frontend tetap ambil FYE dari `predicted_pum`; karena kosong → sel FYE kosong (perilaku sekarang) | Masalah dilaporkan guru; nilai manual tidak tampil. |
| (B) Frontend fallback: kalau `predicted_pum` kosong, tampilkan `score` dari `Grade` (exam_type=4, term=4) | Nilai manual tampil; setelah PUM di-migrasi, PUM jadi prioritas. |
| (C) Nilai manual FYE di `grade` di-override habis (diabaikan) | Data manual terbuang; tidak disarankan. |

**Decision:** **(B)** — `predicted_pum` masih NULL → fallback ke `score` dari `Grade` (exam_type=4, term=4). Setelah PUM di-migrasi, PUM jadi prioritas.

---

## EC-02: `predicted_pum` sudah terisi sebagian (sebagian siswa ada, sebagian tidak)
**Scenario:** Migrasi PUM bertahap — beberapa siswa sudah punya `predicted_pum`, sisanya masih NULL.

| Opsi | Behavior |
|------|----------|
| (A) Sel FYE per-siswa: yang ada PUM → tampilkan PUM; yang kosong → fallback ke nilai manual | Konsisten per-siswa; perlu logika per-row. |
| (B) Seluruh kolom FYE ambil dari satu sumber saja (PUM atau manual) | Sederhana tapi tidak akurat untuk siswa yang sudah punya PUM. |

**Decision:** **(A)** — aware per-siswa: kalau PUM siswa kosong, FYE manual yang dishow; kalau PUM ada, PUM yang dishow.

---

## EC-03: `score = 0` vs `score IS NULL` di grade FYE
**Scenario:** Ada 3 siswa dengan `score = 0` (21219, 21233, 22642).

| Opsi | Behavior |
|------|----------|
| (A) `0` diperlakukan sebagai nilai valid (ditampilkan) | Benar — 0 adalah nilai resmi, bukan "belum diisi". |
| (B) `0` dianggap kosong / tidak tampil | Salah — akan menyembunyikan nilai yang sah. |

**Decision:** **(A)** — `0` adalah nilai valid. Catatan: berdasarkan query final, siswa 21219 dan 22642 sebenarnya punya `score = -1` (ignored), bukan `0`.

---

## EC-04: PUM di-migrasi setelah fallback diterapkan
**Scenario:** `predicted_pum` terisi penuh dari migrasi AIS `enrollprelims` (aspum/apum) setelah fallback (EC-01 opsi B) diterapkan.

| Opsi | Behavior |
|------|----------|
| (A) FYE otomatis bergeser ke PUM (prioritas PUM > nilai manual) | Sel FYE update ke nilai PUM; nilai manual di grade tetap ada sebagai arsip. |
| (B) FYE tetap menampilkan nilai manual meski PUM sudah ada | Perlu manual switch; berisiko inkonsisten. |

**Decision:** **(A)** — begitu `predicted_pum` terisi, FYE langsung menampilkan PUM tanpa perubahan kode.

---

## EC-05: `exam_type` enum integer, bukan string
**Scenario:** Engineer query `grade` dengan `exam_type='FYE'` (string) → error `invalid input value for enum`.

| Opsi | Behavior |
|------|----------|
| (A) Query pakai angka literal (`exam_type=4`) atau cast (`exam_type::text='FYE'`) | Benar — enum adalah integer 1..15. |
| (B) Ubah enum jadi string di skema | Breaking change; tidak disarankan. |

**Decision:** **(A)** — dokumentasikan saja, jangan ubah skema.
