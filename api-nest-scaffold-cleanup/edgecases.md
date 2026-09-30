# Edge Cases — API Nest Scaffold Cleanup

## EC-01: Keputusan per modul — implement atau un-expose?
**Scenario:** 4 modul scaffold tidak selesai harus diputuskan satu per satu
(bisa beda keputusan antar modul).

| Modul | Opsi (A) Implement | Opsi (B) Un-expose |
|---|---|---|
| `student-billing` | Implement service findAll + daftarkan `version: '1'` (fitur billing list memang ada kebutuhan bisnis? konfirmasi PM) | Hapus dari `app.module` — frontend dapat 404 yang jujur, bukan 500 |
| `ccaYearCoordinators` | Uncomment + lengkapi `findAll()` (service `CcaYearProgrammesProgrammeService` sudah ada) | Comment-out route `@Get()` atau hapus modul |
| `cca-grade` | Spesifikasi fitur dulu (tidak ada requirement tertulis) | Hapus modul dari `app.module` |
| `ftp-evaluation-setting` | Spesifikasi fitur dulu | Hapus modul dari `app.module` |

**Decision:** _TBD_ (per modul; rekomendasi: un-expose dulu semua, implement
kemudian sesuai kebutuhan produk — pengerjaan paling kecil, menghapus 500 dari
produksi seketika).

---

## EC-02: Frontend masih memanggil path yang di-un-expose
**Scenario:** Ada halaman admin yang masih memanggil `/api/v1/ccaYearCoordinators`
(walau selama ini selalu menerima 500) — setelah un-expose jadi 404 dan UI bisa
berubah perilaku.

| Opsi | Behavior |
|------|----------|
| (A) Audit pemanggil dulu | Grep `client/` untuk path-path ini sebelum un-expose; koordinasikan halaman yang terdampak |
| (B) Langsung un-expose | 404 diterima sebagai "fitur tidak ada"; tangani di FE belakangan |

**Decision:** _TBD_

---

## EC-03: Scaffold ternyata dipakai internal (bukan via HTTP)
**Scenario:** Service scaffold dipakai modul lain via DI (bukan route HTTP), jadi
menghapus modul mem-break build.

| Opsi | Behavior |
|------|----------|
| (A) Pertahankan service, un-expose hanya controller | Hapus controller dari `controllers: [...]` di module, service tetap |
| (B) Refactor pemakaian internal | Pindahkan logic ke modul pemakai |

**Decision:** _TBD_ (cek dulu import-nya: `cca-grade-term-setting.service` dsb.)

---

## EC-04: Implementasi student-billing ditunda tanpa batas waktu
**Scenario:** PM memilih "implement" tapi tidak masuk sprint mana pun — endpoint
tetap 500 berbulan-bulan.

| Opsi | Behavior |
|------|----------|
| (A) Interim: un-expose sampai implement | 500 hilang seketika; saat implement tinggal daftarkan ulang |
| (B) Biarkan 500 dengan tiket terbuka | Monitoring terus terpolusi noise yang dikenal |

**Decision:** _TBD_ (rekomendasi: (A) — tiket tetap terbuka, produksi bersih).
