# Edge Cases — Subject Folder Auto-Load (Filter as Visibility Only)

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

## EC-01: Guru mengganti filter level setelah data awal dimuat
**Scenario:** Halaman terbuka, semua subject/files guru (AY berjalan) sudah tampil. User memilih Cohort Level di dropdown `masterLevelId`.

| Opsi | Behavior |
|------|----------|
| (A) Filter client-side saja | Data yang sudah dimuat di-hide/show sesuai level — tanpa request baru, instant |
| (B) Refetch dengan `masterLevelId` | Request baru `GET /subjectFile?masterLevelId=...` — data sesuai filter server-side |
| (C) Hybrid | Refetch untuk tabel files, tetapi pills subjects difilter client-side dari data awal |

**Decision:** _TBD_

---

## EC-02: Mengklik subject pill yang sama dua kali (toggle off)
**Scenario:** User mengklik subject pill "English Language" → tabel menunjukkan files subject itu. User mengklik pill yang sama lagi.

| Opsi | Behavior |
|------|----------|
| (A) Toggle off | `activeSubject` di-reset ke undefined → tabel kembali menampilkan SEMUA files (kondisi awal) |
| (B) Tetap aktif | Klik kedua tidak berubah — pill harus dibersihkan via tombol "clear" terpisah |
| (C) Toggle off + tombol clear | Pill toggle off seperti (A), ditambah tombol "Clear filter" eksplisit |

**Decision:** _TBD_

---

## EC-03: Penentuan "academic year berjalan" untuk fetch awal
**Scenario:** Data awal harus mencakup subject guru di AY berjalan — bagaimana AY itu dikirim ke backend?

| Opsi | Behavior |
|------|----------|
| (A) Ikut session/global AY state | Ambil `academicYearId` dari state Redux/global yang sudah dipakai halaman lain (pola eksisting teacher portal) |
| (B) Tanpa `academicYearId` | Backend return semua — biar server yang scope; front-end hanya filter by `teacherId` |
| (C) URL query `academicYearId` | Tambahkan ke schema `useQueryString`, default AY aktif |

**Decision:** _TBD_

---

## EC-04: Guru mengajar banyak subject — performa & tampilan pills
**Scenario:** Seorang guru mengajar 10+ subject. Fetch awal (`pageSize: 0`) mengembalikan banyak files.

| Opsi | Behavior |
|------|----------|
| (A) Tetap `pageSize: 0` | Ambil semua sekali, filter client-side; pills tampil semua (wrap) |
| (B) Fetch all + scroll/pagination pills | Data lengkap, pills di-scroll horizontal |
| (C) Fetch all files, paginasi tabel tetap server-side | Files di-fetch per halaman saat filter berubah (kembali ke refetch) |

**Decision:** _TBD_

---

## EC-05: Kondisi awal vs kondisi "filter aktif" — empty state
**Scenario:** Guru membuka halaman, tidak ada files sama sekali, ATAU kombinasi filter level+subject menghasilkan 0 files.

| Opsi | Behavior |
|------|----------|
| (A) Empty state dibedakan | Pesan berbeda: "No subject files yet" (data awal kosong) vs "No files match the current filters" (filter aktif) |
| (B) Satu empty state | Pesan generik "There's No Subject File" untuk kedua kondisi |

**Decision:** _TBD_

---

## EC-06: Reset `activeSubject` saat level dropdown berubah
**Scenario:** User pilih level "Primary 4" → klik subject "English". Lalu ganti level ke "Secondary 1". `activeSubject` masih "English" (yang mungkin tidak ada di level baru).

| Opsi | Behavior |
|------|----------|
| (A) Reset subject saat level berubah | `activeSubject` di-reset → tampilkan semua files level baru |
| (B) Pertahankan subject | Filter tetap "English" → mungkin 0 hasil di level baru |
| (C) Reset hanya jika subject tidak ada di level baru | Cek ke data subjects terfilter dulu |

**Decision:** _TBD_
