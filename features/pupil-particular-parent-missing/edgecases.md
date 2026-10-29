## EC-01: Siswa tanpa parent sama sekali

**Scenario:** Siswa tidak memiliki `parents_ids` atau data parent kosong.

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan tanda "-" / kosong di kolom parent di PDF | PDF masih bisa dibuka, kolom parent kosong |
| (B) Tampilkan pesan "Tidak ada parent tercatat" | Informatif bagi user bahwa memang tidak ada parent |

**Decision:** TBD

---

## EC-02: Siswa memiliki lebih dari satu parent (ayah dan ibu)

**Scenario:** `parentsIds` berisi 2 parent dengan role `father` dan `mother`.

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan keduanya (Father dan Mother) dalam satu PDF | Lengkap, cocok untuk opsi "Both" |
| (B) Hanya tampilkan satu parent berdasarkan `selectedParentRole` | Membutuhkan pemilihan parent terlebih dahulu |

**Decision:** TBD

---

## EC-03: Nilai `parentRole` di database tidak standar (bukan `father`/`mother`)

**Scenario:** Database menyimpan `parent_role` dengan casing atau format berbeda (misal `FATHER`, `Father`, `father_role`, `1`, `0`), atau `NULL`.

| Opsi | Behavior |
|------|----------|
| (A) Normalisasi ke lowercase, strip suffix `_role`, dan handle `NULL`/`undefined` sebelum dibandingkan | Robust terhadap variasi casing/format dan `NULL`. `NULL` ditampilkan sebagai "-" |
| (B) Terpaksa cocok exact dengan kunci `PARENT_ROLE_ENUM` | Rentan gagal jika data DB berubah casing atau `NULL` |

**Decision:** (A) — implemented via `normalizeParentRole()` in `pupil-particular.jsx`. `NULL`/`undefined` returns `"-"`.

---

## EC-05: Sebuah siswa memiliki beberapa parent, sebagian muncul sebagian tidak

**Scenario:** Siswa memiliki lebih dari satu parent di mana beberapa `parent_role` valid (misal `father`) dan yang lain `NULL` atau casing tidak standar.

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan semua parent yang role-nya dapat dinormalisasi, sisanya tampil "-" | Parent yang valid muncul dengan nama/role, parent `NULL` tetap muncul sebagai "-" |
| (B) Hilangkan parent yang `parent_role`-nya `NULL` dari daftar | Hanya parent dengan role valid yang muncul |

**Decision:** (A) — implemented via `normalizeParentRole()` which returns `"-"` for `NULL`/unrecognized values, so all parents remain visible.

---

## EC-04: `selectedParentRole` tidak dikirim (undefined) dari StudentDetails

**Scenario:** `pupilParticularDocument` dipanggil tanpa parameter `selectedParentRole`.

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan semua parent yang terkait | Tidak memerlukan pemilihan parent |
| (B) Tampilkan pesan error / tidak generate PDF | Memaksa pemilihan parent terlebih dahulu |

**Decision:** TBD

---