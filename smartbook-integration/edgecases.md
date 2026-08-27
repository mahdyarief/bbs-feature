# Edge Cases — Smartbook Integration (Full Manage heyhi.sg)

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

## EC-01: SSO ke heyhi.sg gagal / token invalid
**Scenario:** Endpoint SSO (`sso_heyhi.php`) menolak token (legacy menampilkan "Access Denied, Username invalid!").

| Opsi | Behavior |
|------|----------|
| (A) Redirect ke halaman error | Tampilkan pesan akses ditolak + tombol kembali |
| (B) Redirect ke halaman login Smartbook | User login manual ke platform vendor |
| (C) Log error + retry | Catat kegagalan ke `smartbook_sso_log` dan izinkan retry |

**Decision:** _TBD_ — rekomendasi (A) + (C): tampilkan pesan & catat ke SSO log untuk audit.

---

## EC-02: Token SSO expire / berubah role
**Scenario:** Token HMAC yang di-generate sudah expire (misal session user berubah role / pindah campus).

| Opsi | Behavior |
|------|----------|
| (A) Regenerate token otomatis | Backend generate ulang saat user klik SSO Smartbook |
| (B) Tolak + minta login ulang | User diminta login ulang ke BBS |
| (C) Token statis per user | Token tetap valid selama user aktif (risk: replay attack) |

**Decision:** _TBD_ — rekomendasi (A) regenerate per request + expiry pendek (misal 5 menit).

---

## EC-03: Duplikat data enrollment (sinkronisasi)
**Scenario:** Sinkronisasi dari heyhi.sg mengirim baris yang sama dua kali (duplicate key).

| Opsi | Behavior |
|------|----------|
| (A) Unique constraint | Tabel `smartbook_enrollment` punya unique index (ay_id, student_id, subject_id) — insert conflict di-skip |
| (B) Dedup manual | Job sinkronisasi mengecek dan menghapus duplikat sebelum insert |
| (C) Upsert | Gunakan `INSERT ... ON CONFLICT DO UPDATE` |

**Decision:** _TBD_ — rekomendasi (C) upsert dengan unique constraint, paling robust untuk sinkronisasi berkala.

---

## EC-04: Update status payment — race condition
**Scenario:** Dua admin meng-update status payment siswa yang sama bersamaan.

| Opsi | Behavior |
|------|----------|
| (A) Last-write-wins | Status terakhir yang di-submit yang berlaku |
| (B) Optimistic lock | Simpan `updated_at`/version; jika sudah berubah, tolak update dengan konflik 409 |
| (C) Lock row | Lock row di DB selama update |

**Decision:** _TBD_ — rekomendasi (B) optimistic lock, konsisten dengan pola yang dipakai di modul lain `bbs`.

---

## EC-05: Export PDF dengan data besar
**Scenario:** Export laporan paid untuk seluruh campus (bisa ribuan siswa).

| Opsi | Behavior |
|------|----------|
| (A) Generate synchronous | PDF di-generate saat request, streaming response (bisa lambat) |
| (B) Async job + download link | PDF di-generate di background, user mendapat notifikasi/link saat siap |
| (C) Batasi jumlah baris | Export hanya untuk filter spesifik (wajib pilih campus/cohort) |

**Decision:** _TBD_ — rekomendasi (A) untuk versi awal (data Smartbook terbatas), upgrade ke (B) jika lambat.

---

## EC-06: Rentang tanggal SSO Log kosong / terlalu lebar
**Scenario:** User memilih rentang tanggal tanpa data, atau rentang terlalu lebar (misal 1 tahun → log 2MB+).

| Opsi | Behavior |
|------|----------|
| (A) Batasi maksimal 31 hari | Tolak rentang > 31 hari dengan pesan |
| (B) Paginasi ketat | Wajib paginasi, default 7 hari terakhir |
| (C) Tampilkan semua | Return semua data tanpa batas (risiko slow query) |

**Decision:** _TBD_ — rekomendasi (B) paginasi + default 7 hari, karena log legacy 2MB tanpa pagination (masalah performa).

---

## EC-07: Dashboard heyhi.sg tidak bisa diakses (timeout / region-lock)
**Scenario:** `clientreport.heyhi.sg` timeout dari jaringan BBS (seperti yang terjadi saat probe dari jaringan lokal).

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan pesan error embed | Halaman menampilkan placeholder "Dashboard tidak tersedia" |
| (B) Redirect ke dashboard di tab baru | Biarkan browser user membuka URL langsung (auth eksternal) |
| (C) Proxy server-side | Backend fetch dashboard dan render ulang (risiko CORS/SSRF) |

**Decision:** _TBD_ — rekomendasi (B) redirect/embed iframe langsung, karena (C) berisiko SSRF.

---

## EC-08: Leaps update kategori — validasi nilai
**Scenario:** User mengirim nilai kategori leaps yang tidak valid via `update_leap_cat.php`.

| Opsi | Behavior |
|------|----------|
| (A) Tolak nilai invalid | Return 400 "Invalid leaps category" — hanya terima LEADERSHIP/ACHIEVEMENTS/SERVICE |
| (B) Simpan apa pun | Simpan tanpa validasi (risiko data kotor) |
| (C) Map otomatis | Nilai asing di-map ke kategori terdekat |

**Decision:** _TBD_ — rekomendasi (A) validasi strict whitelist 3 kategori.

---

## EC-09: gettoken_nya dengan pasangan tkn/utp salah
**Scenario:** User mengakses halaman Tickets dengan token tidak valid (legacy: "Token Invalid! 1/3").

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan pesan invalid | Mirror legacy: "Token Invalid!" + log percobaan |
| (B) Redirect ke login | Arahkan ke halaman login ASD |
| (C) Silent 404 | Kembalikan 404 tanpa keterangan (anti-enumeration) |

**Decision:** _TBD_ — rekomendasi (C) silent 404 di sistem baru (lebih aman), dengan log di `smartbook_ticket`.

---

## EC-10: AY tidak aktif (past AY) pada member viewer
**Scenario:** Admin memilih AY yang sudah lewat (misal 25 = 2024/2025).

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan data historis | Viewer menampilkan data AY lama (read-only) |
| (B) Blokir akses | Tolak dengan pesan "Academic year not found/not active" |
| (C) Read-only + warning | Tampilkan data dengan banner "Data for past academic year" |

**Decision:** _TBD_ — rekomendasi (A) tampilkan data historis, karena legacy mendukung pilihan AY 25/26/27.

---

## EC-11: Campus tanpa data enrollment
**Scenario:** Admin memilih campus/cohort/status yang tidak memiliki data `smartbook_enrollment` untuk AY terpilih.

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan empty state | Muncul pesan "No smartbook data found for the selected filters" + tabel kosong |
| (B) Tampilkan 0 di statistik | Card statistik menampilkan `0/0 (0%)` untuk semua subject |
| (C) Sembunyikan card & tabel | Halaman hanya menampilkan filter bar tanpa konten |

**Decision:** _TBD_ — rekomendasi (A) empty state konsisten dengan pola `epetals-dashboard` (BBSNoItemCard).

---

## EC-12: Status "None" (belum ada enrollment)
**Scenario:** Siswa terdaftar di Smartbook tapi belum ada status enrollment sama sekali (belum Enrolled/Paid).

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan sebagai status `NONE` | Kolom Enrolled menampilkan badge "None" (sesuai legacy `status=2`) |
| (B) Sembunyikan dari daftar | Hanya siswa dengan status Enrolled/Paid/Not Paid yang muncul |

**Decision:** _TBD_ — rekomendasi (A) menampilkan "None" agar admin tahu siswa yang belum diproses.
