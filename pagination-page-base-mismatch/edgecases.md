# Edge Cases — Pagination Page-Base Mismatch

## EC-01: Basis `page` yang dipakai global?
**Scenario:** Dua service menghitung offset manual asumsi 1-based
(`skip((page-1)*pageSize)`) sementara konvensi global `PageOptionsDto` 0-based
(`page=0` default, offset `page*pageSize`).

| Opsi | Behavior |
|------|----------|
| (A) Global 0-based; ubah 2 service manual | `sveStudentGrades?` tanpa page = baris pertama; konsisten `findOptionsHelper`; FE yang kini kirim `?page=1` bergeser ke halaman kedua |
| (B) Pertahankan 1-based untuk kedua modul; DTO khusus | Kontrak endpoint ini tidak berubah bagi pemanggil; tapi dua kontrak pagination hidup berdampingan |

**Decision:** _TBD_ (rekomendasi (A) — satu kontrak; catat pergeseran FE di EC-02).

---

## EC-02: Pemanggil yang selama ini kirim `?page=1` ke sveStudentGrades
**Scenario:** Setelah basis diseragamkan ke 0-based, `page=1` lama = halaman
kedua (offset 10) — data yang tampil bergeser.

| Opsi | Behavior |
|------|----------|
| (A) Audit pemanggil (grep `client/` untuk `sveStudentGrades`) + sesuaikan | FE dan BE berpindah bareng; tidak ada duplikat/lompat data |
| (B) Wrapper compatibility (terima page 1-based hanya di endpoint ini) | Tidak ada perubahan FE; menambah kompleksitas jangka panjang |

**Decision:** _TBD_

---

## EC-03: `pageSize=0` (unpaged)
**Scenario:** Service meng-guard `if (pageSize > 0)` sebelum skip/take — artinya
`pageSize=0` dimaknai "tanpa paging" (semua baris), sedangkan DTO mem-bolehkan
`@Min(0)`.

| Opsi | Behavior |
|------|----------|
| (A) Pertahankan semantik `pageSize=0` = unpaged | Dokumentasikan eksplisit; hati-hati tabel besar |
| (B) Wajibkan pageSize ≥ 1 | Konsisten unbounded-pageSize lesson dari TC-034 |

**Decision:** _TBD_

---

## EC-04: Module lain diam-diam menghitung offset manual
**Scenario:** Grep `page - 1` hanya menemukan 2 file, tapi pola lain mungkin
`(options.page ?? 1)` dsb.

| Opsi | Behavior |
|------|----------|
| (A) Audit sekalian: grep `skip(` manual di services | Daftar lengkap titik perhitungan offset; satukan ke helper |
| (B) Fix yang diketahui saja | Minimal, risiko temuan baru nanti |

**Decision:** _TBD_ (rekomendasi (A) — pekerjaan kecil, mencegah kejadian ulang).
