# Edge Cases — Unassign All Submissions

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update `spec.md` jika ada perubahan behavior.

## EC-01: Bulk delete saat tidak ada submission sama sekali

**Skenario:** Teacher klik "Unassign All" padahal belum ada satupun `StudentAssignment` untuk assignment tersebut (mis. assignment baru dibuat, atau semua sudah di-unassign sebelumnya).

| Opsi | Behavior |
|------|----------|
| (A) Tetap panggil endpoint, backend return `{ data: [], count: 0 }`, UI refresh jadi kosong + toast info "0 submissions removed" | Aman, idempoten, tidak error. Cocok dengan service yang sudah return `0`. |
| (B) Disable tombol "Unassign All" saat list kosong (tidak bisa diklik) | UI lebih mencegah, tapi butuh guard `filteredStudentAssignments?.length`. |

**Decision:** _TBD_ — disarankan gabungkan keduanya: tombol disable saat list kosong (EC-01-B) SEKALIGUS endpoint aman untuk `count: 0` (EC-01-A) sebagai pertahanan terakhir.

---

## EC-02: Stale rows di Redux setelah bulk delete (race dengan MERGE)

**Skenario:** Endpoint return `data: []` dan `count: 28`, tapi `makeApiRequestThunk` memicu `studentAssignment/merge` dengan map kosong. Reducer tidak drop baris lama → UI tetap menampilkan 28 row usang.

| Opsi | Behavior |
|------|----------|
| (A) Panggil `studentAssignmentsApi?.refresh()` setelah dispatch bulk action | List di-fetch ulang dari server, menampilkan kosong. Paling andal. |
| (B) Gunakan `ACTION_TYPES.DELETE` dengan cara custom (kirim array id) | Butuh frontend tahu semua id dulu; lebih rumit dan tidak mendukung server-side filter. |
| (C) Biarkan MERGE kosong dan harap backend hapus via side-effect | TIDAK aman — reducer tidak dropped row. Ditolak. |

**Decision:** _TBD_ — disarankan **(A)**. Wajib `refresh()` pasca-bulk (lihat `api-contract.md` caveat).

---

## EC-03: Authorization — siapa yang boleh trigger bulk delete?

**Skenario:** Endpoint `DELETE /studentAssignments/assignment/:assignmentId` bisa dipanggil oleh teacher lain (bukan pembuat chapter/assignment), atau role non-teacher.

| Opsi | Behavior |
|------|----------|
| (A) Hanya gate di UI (`isOwnChapter`) — backend tidak ada ownership guard tambahan | Konsisten dengan rute single-delete yang sudah ada (tidak ada guard). Cepat, tapi backend tetap terbuka bagi caller yang tahu assignmentId. |
| (B) Tambah route-level teacher-role guard (mis. `@CheckPermissions` / cek `req.user` punya akses ke chapter) | Lebih aman, cegah cross-teacher bulk removal. Sedikit lebih banyak implementasi. |

**Decision:** _TBD_ — disarankan **(B)** sebagai hardening (rekomenasi di `spec.md` Business Rule #5), dengan catatan (A) tetap menerima karena sudah jadi konvensi rute yang ada. Perlu konfirmasi apakah module assignment punya permission guard yang bisa dipakai.

---

## EC-04: Concurrency — teacher klik "Unassign All" berkali-kali / double submit

**Skenario:** User klik tombol berkali-kali sebelum response pertama kembali (mirip bug spam Save di pickup-person). Atau dua teacher (jika EC-03 = A) menekan bersamaan.

| Opsi | Behavior |
|------|----------|
| (A) Disable tombol + loading spinner saat request in-flight | Cegah double submit di sisi UI. |
| (B) Biarkan backend idempoten (softRemove pada row yang mungkin sudah di-remove → `count` mengecil) | Aman secara data, tapi UI spam tetap perlu diantisipasi. |

**Decision:** _TBD_ — disarankan **(A)** + andalkan idempotensi service (EC-04-B sebagai safety net).

---

## EC-05: Submission yang sudah di-grade / sudah submit

**Skenario:** Sebagian `StudentAssignment` sudah `GRADED` atau `SUBMITTED`. Apakah bulk unassign boleh menghapusnya?

| Opsi | Behavior |
|------|----------|
| (A) Hapus semua tanpa peduli status (soft-delete semua) | Sederhana, cocok dengan tujuan "reset seluruh class". Grade/answer ikut tertutup bersama row. |
| (B) Blokir / minta konfirmasi ekstra jika ada yang sudah di-grade | Lebih hati-hati, tapi menambah kompleksitas dan tidak sejalan dengan niat "reset salah kirim ke seluruh class". |

**Decision:** _TBD_ — disarankan **(A)**: pesan konfirmasi sudah cukup explicit ("remove every student submission"). Soft-delete preserve data untuk audit/restore jika diperlukan nanti.
