# Edge Cases — PETALS Lesson Observation Input

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.

## EC-OBS-01: Guru belum pernah diobservasi sama sekali

**Scenario:** Principal/HOD membuka form observasi untuk guru yang belum pernah diobservasi.

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan form kosong | Form dengan 18 item all mark 0, Strength/Areas of Concern kosong, status DRAFT. |
| (B) Tampilkan pesan | "No observation record found. Create new observation?" dengan tombol Start. |

**Decision:** _TBD_ — rekomendasi: **(A) Form kosong** langsung bisa diisi. Observer tinggal men-dropdown mark.

---

## EC-OBS-02: Guru pindah campus di tengah AY

**Scenario:** Guru pindah campus (misal PIK-S ke KJ-P) di tengah AY. Observasi sebelumnya sudah ada di campus lama.

| Opsi | Behavior |
|------|----------|
| (A) Observasi tetap di campus lama | Data observasi fix di campus tempat observasi dilakukan — tidak terpengaruh perpindahan. |
| (B) Observasi ikut pindah | Semua observasi mengikuti campus terbaru guru — data bisa campur aduk. |

**Decision:** _TBD_ — rekomendasi: **(A) Observasi tetap di campus lama**. `campus_id` di-set saat observasi dibuat, tidak diubah.

---

## EC-OBS-03: Multiple observer untuk guru yang sama di AY yang sama

**Scenario:** Principal dan HOD keduanya mengobservasi guru yang sama di AY yang sama.

| Opsi | Behavior |
|------|----------|
| (A) Diperbolehkan | Kombinasi unik `(teacher_id, academic_year_id, observer_id)` — observer berbeda diperbolehkan. |
| (B) Ditolak | Hanya satu observasi per guru per AY. |

**Decision:** _TBD_ — rekomendasi: **(A) Diperbolehkan**. Principal dan HOD bisa observasi guru yang sama secara independen. Report PETALS menampilkan rata-rata atau observasi terbaru.

---

## EC-OBS-04: Semua mark 0 (belum diisi)

**Scenario:** Observer membuka form baru (semua mark 0) lalu klik Save tanpa mengubah apa pun.

| Opsi | Behavior |
|------|----------|
| (A) Simpan sebagai DRAFT | Mark 0 tetap disimpan, status DRAFT. Observer bisa lanjut nanti. |
| (B) Tolak dengan toast | "At least one item must be marked" — tidak bisa simpan mark all 0. |

**Decision:** _TBD_ — rekomendasi: **(A) Simpan sebagai DRAFT**. Observer mungkin ingin menyimpan progres parsial.

---

## EC-OBS-05: Submit observasi dengan Strength/Areas of Concern kosong

**Scenario:** Observer mengisi semua mark tapi tidak menulis Strength atau Areas of Concern.

| Opsi | Behavior |
|------|----------|
| (A) Allow submit | Strength/Areas of Concern opsional — bisa submit tanpa keduanya. |
| (B) Wajib diisi minimal satu | Minimal Strength atau Areas of Concern harus diisi. |

**Decision:** _TBD_ — rekomendasi: **(A) Opsional**. Di legacy, kedua field bisa kosong.

---

## EC-OBS-06: Conflict mark saat save bersamaan

**Scenario:** Dua observer (atau admin + observer) menyimpan mark untuk observasi yang sama di waktu bersamaan.

| Opsi | Behavior |
|------|----------|
| (A) Last write wins | Timestamp-based — data terakhir yang disimpan yang dipakai. |
| (B) Optimistic locking | Gunakan `version` column (integer) — conflict → 409. |

**Decision:** _TBD_ — rekomendasi: **(B) Optimistic locking**. Karena observasi satu user, conflict jarang terjadi. Tapi data integrity penting.