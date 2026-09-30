# Edge Cases — Security Deposit 404 Fix

## EC-01: 404 atau 200 dengan `data: null`?
**Scenario:** UI membuka halaman deposit siswa yang tidak punya deposit. Kedua
kontrak bisa dipakai FE.

| Opsi | Behavior |
|------|----------|
| (A) 404 `NotFoundException` | Konsisten pola modul lain & audit klasifikasi A; FE harus tangani error state |
| (B) 200 `data: null` | FE render empty state langsung tanpa try/catch; tapi menyimpang dari konvensi resource-not-found di api_nest |

**Decision:** _TBD_ (rekomendasi (A) — konsistensi konvensi codebase menang).

---

## EC-02: Route sibling `/unpaidBillings` ikut diaudit?
**Scenario:** `GET /securityDeposits/:studentId/unpaidBillings` tidak melempar
error untuk kondisi kosong (return array kosong) — tapi perlu dipastikan juga
untuk siswa yang tidak ada sama sekali (studentId ngawur).

| Opsi | Behavior |
|------|----------|
| (A) Sekalian validasi student existence | Siswa ngawur → 404 "Student not found", bukan `data:[]` yang menyesatkan |
| (B) Biarkan | Array kosong untuk siapa pun; risiko kecil karena read-only |

**Decision:** _TBD_ (rekomendasi (A) bila modif dibuka; (B) bila ingin perubahan
minimal).

---

## EC-03: Siswa dengan deposit soft-deleted
**Scenario:** Baris `security_deposit` ada tapi soft-deleted. Entity default
TypeORM menyaring soft-delete — siswa ini akan dianggap "tidak punya deposit".

| Opsi | Behavior |
|------|----------|
| (A) 404 (perlakuan sama dengan tidak-punya) | Sederhana dan cukup untuk UI |
| (B) 200 dengan data history | Berguna untuk audit finance, butuh `withDeleted` + kebijakan akses |

**Decision:** _TBD_ (rekomendasi (A) untuk sekarang; (B) jadi fitur terpisah bila
finance memintanya).
