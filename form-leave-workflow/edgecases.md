# Edge Cases — Form Leave Workflow (Approval & Teacher Leave Management)

> **Status: APPROVED** — diselaraskan langsung dengan fungsionalitas Principal Portal (`ais_legacy/principals_tool`), alur pembatalan, email notification queue, dan sorting record canceled di urutan terbawah.

---

## EC-09: Pembatalan Pengajuan Cuti (Cancel Request Behavior & Data Retention)

**Scenario:** Guru membatalkan cuti yang belum diproses atau Principal membatalkan cuti guru (`leaveStatus = CANCELED`).

| Opsi | Behavior |
|------|----------|
| (A) Soft delete (activeStatus = 0) | Data tersembunyi dari list guru dan reviewer — riwayat permohonan hilang. |
| (B) Hard delete (DELETE row) | Baris database dihapus — merusak audit trail dan integritas data sekolah. |
| (C) Ubah status CANCELED & order paling bawah (Disetujui) | Status berubah menjadi `CANCELED`, `activeStatus` tetap `ACTIVE = 1`. Data tetap tersimpan di database dan ditampilkan pada tabel list, namun **selalu berada di urutan paling bawah** agar tidak mengganggu antrean pengajuan aktif. |

**Decision:** **(C) Ubah status CANCELED & order paling bawah**. Sesuai arahan spesifik: saat pembatalan datanya tidak dihapus, statusnya menjadi canceled, tetap ada di list namun order paling bawah.

---

## EC-10: Penolakan atau Pembatalan Tanpa Komentar Reviewer

**Scenario:** Reviewer (Principal) mengubah status cuti menjadi `DECLINED` atau `CANCELED` tanpa mengisi catatan komentar/alasan.

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 400 Bad Request | Validasi backend & form modal: komentar wajib diisi saat memilih Decline atau Cancel — pesan: "Comment is required when declining or canceling a leave request". |
| (B) Izinkan tanpa komentar | Status tersimpan tanpa alasan — guru bingung kenapa permohonannya ditolak atau dibatalkan. |

**Decision:** **(A) Tolak dengan 400 Bad Request**. Guru berhak mengetahui pertimbangan operasional mengapa izin tidak disetujui atau dibatalkan oleh pihak sekolah.

---

## EC-11: Guru Biasa Mencoba Mengakses Menu "Form Leave Teacher"

**Scenario:** Guru yang bukan merupakan Principal atau Vice Principal mencoba membuka URL `/form-leave-teacher` di Teacher Portal.

| Opsi | Behavior |
|------|----------|
| (A) Blokir dengan 403 Forbidden / Redirect | Hook `usePrincipalOrHod.js` mendeteksi `isPrincipalOrVp === false`, rute diblokir dan diarahkan ke `/dashboard` atau `/teacher-leave`. Endpoint backend `/v1/leaves/:id/status` mengembalikan error 403 Forbidden. |
| (B) Tampilkan halaman kosong | Halaman terbuka tanpa data — pengalaman pengguna membingungkan. |

**Decision:** **(A) Blokir dengan 403 Forbidden / Redirect**. Menu "Form Leave Teacher" adalah supervisi khusus Principal / VP untuk memproses izin guru sekolahnya.

---

## EC-12: Pengiriman Notifikasi Email Mengalami Kegagalan (Mail Server / SES Timeout)

**Scenario:** Saat permohonan dibuat atau status diperbarui, koneksi ke AWS SES / SMTP server terputus atau throttling.

| Opsi | Behavior |
|------|----------|
| (A) Gagalkan transaksi DB (Rollback) | Pengajuan/update status gagal disimpan hanya karena gangguan email — menghambat operasional sekolah. |
| (B) Antrean Asinkron (Bull Queue with Exponential Backoff) | Transaksi DB berhasil di-commit seketika. Job pengiriman email dimasukkan ke antrean Bull Queue `SEND_EMAIL` dengan `attempts: 3` dan backoff exponential. Jika gagal 3 kali, error di-log tanpa membatalkan status cuti. |

**Decision:** **(B) Antrean Asinkron (Bull Queue with Exponential Backoff)**. Menjaga responsivitas API sistem sekolah dan ketahanan pengiriman email notification.

---

## EC-13: Akses Admin Portal (Supervisi ASD / HR) Melakukan Mutasi Status

**Scenario:** Administrator / ASD di Admin Portal (`client/`) membuka permohonan guru dan mencoba mengubah status cuti.

| Opsi | Behavior |
|------|----------|
| (A) Izinkan Admin mengubah status | Admin dapat mengubah status — melanggar wewenang persetujuan yang berada pada Principal sekolah. |
| (B) View Only berbasis RBAC | Halaman Admin Portal hanya bersifat **View Only** (`ACLTypeEnum.READ` pada `ModulesTypeEnum.LEAVE`). Tabel menampilkan badge status dan riwayat komentar tanpa tombol aksi/dropdown perubahan status. |

**Decision:** **(B) View Only berbasis RBAC**. Sesuai requirement: di admin ada juga untuk view only yang dapat diakses berdasarkan RBAC.

---

## EC-14: Mengubah Status Record Inaktif (`activeStatus = INACTIVE`)

**Scenario:** Permintaan update status dikirimkan untuk record leave yang sebelumnya berstatus `activeStatus = INACTIVE`.

| Opsi | Behavior |
|------|----------|
| (A) Tolak dengan 404 Not Found | Service mengecek `activeStatus === StatusTypeEnum.ACTIVE`, jika tidak aktif kembalikan "Teacher leave not found or inactive". |
| (B) Reaktifkan otomatis | Menimbulkan ketidakkonsistenan data historis. |

**Decision:** **(A) Tolak dengan 404 Not Found**. Record inaktif tidak boleh dimutasi status alurnya.
