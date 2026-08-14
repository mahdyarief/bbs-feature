# Edge Cases — CCA Registration Window Form Gating & Data Integrity

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

## EC-01: Tidak ada window sama sekali untuk scope siswa
**Scenario:** Admin belum pernah membuat registration window untuk kombinasi `(academicYearId, campusId, masterLevelId)` milik siswa. Saat ini `isRegistrationOpen()` fallback ke `return true` (line 163 `cca-registration-window.service.ts`), dan `CcaYearDto.isRegistrationOpen` default `true`.

| Opsi | Behavior |
|------|----------|
| (A) Sinyal baru `isWindowConfigured` (Recommended) | Backend menambahkan field `isWindowConfigured` di `CcaYearDto` (dari `isWindowConfigured()` service yang sudah ada). Frontend menyembunyikan form saat `false`. `isRegistrationOpen` tetap backward-compatible untuk konsumen lain. |
| (B) Ubah default `isRegistrationOpen` → `false` | Tanpa window, `isRegistrationOpen` bernilai `false` di semua konsumen. Lebih sederhana tapi mengubah kontrak API global — berisiko memengaruhi Admin/Teacher yang bergantung pada perilaku lama. |
| (C) Gating murni frontend | Frontend memanggil `getCcaRegistrationWindows` dan menghitung sendiri. Tidak ada perubahan backend, tapi duplikasi logika & rawan tidak sinkron. |

**Decision:** _TBD_ — rekomendasi (A): ekspos `isWindowConfigured` lewat DTO, satu fetch, konsisten dengan pola `isRegistrationOpen`.

---

## EC-02: Window ber-status `INACTIVE`
**Scenario:** Admin menonaktifkan window (mis. salah konfigurasi). `isRegistrationOpen()` dan `isWindowConfigured()` saat ini hanya menghitung `activeStatus: ACTIVE` → window INACTIVE dianggap "tidak ada" → registrasi fallback terbuka. Ini kontra-intuitif: menonaktifkan window seharusnya menutup registrasi.

| Opsi | Behavior |
|------|----------|
| (A) INACTIVE = sengaja ditutup (Recommended) | `isWindowConfigured` menghitung window ACTIVE **maupun** INACTIVE (yang penting ada record untuk scope). Tanpa record → form disembunyikan; ada record INACTIVE → form disembunyikan juga (karena `isRegistrationOpen` sudah `false`). |
| (B) INACTIVE = tidak dikonfigurasi | Perilaku sekarang dipertahankan: window INACTIVE dianggap tidak ada, fallback ke open. Konsisten dengan `isWindowConfigured()` saat ini, tapi menonaktifkan window justru membuka registrasi. |
| (C) INACTIVE = hard close | Window INACTIVE dianggap ada DAN menutup paksa — sama hasilnya dengan (A) untuk gating form, bedanya (A) lebih eksplisit soal record vs status. |

**Decision:** _TBD_ — rekomendasi (A). Implikasi: `isWindowConfigured()` perlu diubah untuk menghitung window tanpa filter `activeStatus` (atau menambah param `includeInactive`).

---

## EC-03: Window ACTIVE tapi di luar periode (belum buka / sudah tutup)
**Scenario:** Ada window ACTIVE untuk scope siswa, tapi `now < opensAt` (belum buka) atau `now > closesAt` (sudah tutup). `isRegistrationOpen()` sudah mengembalikan `false` untuk kasus ini.

| Opsi | Behavior |
|------|----------|
| (A) Form tetap disembunyikan, tampilkan info "belum dibuka" (Recommended) | `isWindowConfigured = true` tapi `isRegistrationOpen = false` → form tidak tampil; bisa tampilkan pesan berbeda ("pendaftaran dibuka {opensAt}") jika diinginkan. |
| (B) Tampilkan form tapi nonaktif (disabled) | Form terlihat tapi tombol daftar disabled. Lebih informatif tapi menambah kompleksitas UI dan berisiko siswa melihat CCA yang seharusnya belum/tidak tersedia. |

**Decision:** _TBD_ — rekomendasi (A). Tidak ada perubahan kode untuk kasus ini: gating form cukup `isWindowConfigured && isRegistrationOpen`.

---

## EC-04: Window level-spesifik vs campus-wide
**Scenario:** Siswa level X. Ada window campus-wide (`masterLevelId IS NULL`) ACTIVE, tapi tidak ada window level-spesifik untuk X. Sebaliknya: ada window level-spesifik X tapi tidak open, sementara window campus-wide open.

| Opsi | Behavior |
|------|----------|
| (A) Pertahankan hierarki yang ada (Recommended) | Level-spesifik menang: jika window level X ada (dalam periode) → open; jika ada tapi di luar periode → tutup (tanpa fallback campus-wide); jika tidak ada window level X → fallback ke campus-wide; jika tidak ada sama sekali → `isWindowConfigured = false` → form disembunyikan. |
| (B) Level-spesifik tidak pernah override campus-wide | Campus-wide selalu berlaku, window level hanya "menambah" bukaan. Ini mengubah kontrak `isRegistrationOpen()` yang sudah ada — perlu konfirmasi bisnis. |

**Decision:** _TBD_ — rekomendasi (A), mempertahankan perilaku `isRegistrationOpen()` yang sudah ada (line 114-164). Gating form mengikuti hasil hierarki ini.

---

## EC-05: Dua window tumpang tindih di scope yang sama
**Scenario:** Admin membuat window kedua untuk scope `(academicYearId, campusId, masterLevelId)` yang sama dengan rentang waktu tumpang tindih (atau identik) dengan window pertama. Saat ini tidak ada pencegahan → `findOne()`/query window bisa mengembalikan hasil non-deterministik.

| Opsi | Behavior |
|------|----------|
| (A) Tolak overlap saat create/update (Recommended) | Validasi di service: tolak jika ada window lain di scope sama yang rentangnya overlap (`existing.opensAt < new.closesAt AND new.opensAt < existing.closesAt`). Boleh ada window non-overlap di scope sama (mis. window musim gugur & musim semi). |
| (B) Maksimal 1 window per scope (unique) | Hanya 1 window per `(academicYearId, campusId, masterLevelId)` — enforce dengan unique index parsial + validasi service. Paling sederhana, tapi tidak bisa representasi beberapa periode dalam 1 tahun ajaran. |
| (C) Izinkan overlap, pilih window paling spesifik/terbaru | Tidak menolak data, tapi resolusi window harus deterministik (mis. urutkan, ambil yang paling baru). Lebih fleksibel tapi kompleks dan mudah salah. |

**Decision:** _TBD_ — rekomendasi (A): tolak overlap saat create/update; tanpa migrasi unique index, validasi di service cukup untuk MVP (lihat spec Database Changes).

---

## EC-06: `GET /v1/ccaRegistrationWindows/:id` dengan id tidak ada
**Scenario:** Client request detail window dengan id yang tidak ada di database.

| Opsi | Behavior |
|------|----------|
| (A) Throw `NotFoundException` (Recommended) | `findOne()` melempar `NotFoundException('Registration window with id {id} not found.')` → HTTP 404. Konsisten dengan pola service lain di `api_nest`. |
| (B) Return null | Perilaku sekarang. Client harus handle null; tidak konsisten dengan endpoint lain. |

**Decision:** _TBD_ — rekomendasi (A).
