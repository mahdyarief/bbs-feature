# Edge Cases — Form Leave Workflow (Status Approval)

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

## EC-09: Transisi status tidak valid (state machine violation)

**Scenario:** Admin mencoba mengubah status dari `PENDING` langsung ke `APPROVED_BY_PRINCIPAL` (skip Admin approval), atau dari `REJECTED` ke `APPROVED_BY_ADMIN` (re-open setelah ditolak).

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 400 | Validasi state machine — "Invalid status transition from REJECTED to APPROVED_BY_ADMIN". |
| (B) Izinkan semua | Tidak ada validasi — siapa pun bisa set status apa pun ke nilai apa pun. |

**Decision:** _TBD_ — rekomendasi: **(A) Tolak dengan 400**. State machine adalah core business rule workflow approval. Tanpa validasi, workflow tidak bermakna. Daftar transisi valid:

| From | To | Valid? |
|------|----|--------|
| PENDING | APPROVED_BY_ADMIN | Ya |
| PENDING | REJECTED | Ya |
| APPROVED_BY_ADMIN | APPROVED_BY_PRINCIPAL | Ya |
| APPROVED_BY_ADMIN | REJECTED | Ya |
| (lainnya) | (lainnya) | Tidak → 400 |

---

## EC-10: Reject tanpa komentar

**Scenario:** Admin/Principal menolak pengajuan leave (`REJECTED`) tanpa mengisi komentar/comment.

| Opsi | Behavior |
|------|----------|
| (A) Wajib komentar | Validasi: jika `leaveStatus = REJECTED` dan `comment` kosong/null → 400 "Comment is required when rejecting a leave request". |
| (B) Opsional | Komentar tidak wajib — reject tanpa alasan diperbolehkan. |

**Decision:** _TBD_ — rekomendasi: **(A) Wajib komentar**. Reject tanpa alasan tidak informatif buat guru. Guru perlu tahu kenapa leave-nya ditolak agar bisa mengajukan ulang dengan benar.

---

## EC-11: Admin mengubah status leave milik guru di campus lain

**Scenario:** Admin dari campus A (campusId=4) mencoba mengubah status leave milik guru di campus B (campusId=7).

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 403 | Validasi scope campus — `campusId` record harus sama dengan `campusId` reviewer. |
| (B) Izinkan | Admin bisa ubah status leave campus mana pun — tidak ada scope. |

**Decision:** _TBD_ — rekomendasi: **(A) Tolak dengan 403**. Konsisten dengan fase 1 (mirroring Admin Portal per campus). Jika perlu akses lintas campus, itu enhancement dengan super-admin permission.

---

## EC-12: Mengubah status record yang sudah di-soft-delete

**Scenario:** Admin mencoba mengubah status leave yang sudah dihapus (`activeStatus = INACTIVE`).

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 404 | Record tidak ditemukan (filter `activeStatus = ACTIVE` di query). |
| (B) Izinkan | Record soft-deleted tetap bisa diubah statusnya — tidak konsisten. |

**Decision:** _TBD_ — rekomendasi: **(A) Tolak dengan 404**. Record yang sudah dihapus tidak bisa diubah statusnya. Jika perlu dibatalkan penghapusan, itu enhancement (restore).

---

## EC-13: Dua Admin mengubah status secara bersamaan (race condition)

**Scenario:** Dua Admin berbeda mengakses halaman yang sama dan mengubah status record yang sama dalam waktu bersamaan.

| Opsi | Behavior |
|------|----------|
| (A) Optimistic locking | Validasi di service: `updatedAt` atau version field cocok dengan yang di-fetch — jika tidak cocok, tolak 409. |
| (B) Last-write-wins | Tidak ada cek — perubahan terakhir yang menang. Risiko overwrite. |

**Decision:** _TBD_ — rekomendasi: **(B) Last-write-wins** di fase 2. Transisi status di state machine sudah mencegah overwrite destructive (misal APPROVED_BY_PRINCIPAL tidak bisa di-overwrite ke PENDING oleh Admin lain). Optimistic locking bisa ditambahkan di enhancement jika terbukti perlu.

---

## EC-14: Guru melihat status di Teacher Portal saat record di-soft-delete

**Scenario:** Record leave milik guru dihapus (soft delete) oleh guru sendiri. Record tetap ada di DB (`activeStatus = INACTIVE`). Apakah status approval-nya masih perlu ditampilkan?

| Opsi | Behavior |
|------|----------|
| (A) Jangan tampilkan | Filter `activeStatus = ACTIVE` di query Teacher Portal — record soft-deleted tidak muncul. |
| (B) Tampilkan dengan label "Deleted" | Record soft-deleted tetap muncul di list guru dengan status "Deleted" dan badge khusus. |

**Decision:** _TBD_ — rekomendasi: **(A) Jangan tampilkan**. Konsisten dengan fase 1 (AC-6: soft delete → item hilang dari list). Guru bisa melihat riwayat lengkap (termasuk deleted) di enhancement "History/Archive".