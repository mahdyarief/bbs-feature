# Edge Cases — Appraisal Lock/Unlock

## EC-AL-01: Lock saat guru sedang aktif mengedit

**Skenario:** Admin mengunci (lock) tab ACADEMIC guru "Devie Lana" tepat saat guru tersebut sedang membuka form skor dan mengetik.

**Penanganan:** Lock berjalan di sisi backend (`appraisal_lock.is_locked = true`) dan langsung berlaku — setiap PUT skor setelah lock ditolak 409, apapun yang sedang guru lihat. Frontend guru harus mengecek status lock sebelum submit dan menampilkan error "Appraisal entry is locked. Ask admin to unlock to edit." jika gagal. **Keputusan: enforce di backend; frontend hanya memberikan feedback (tidak boleh andalkan state lokal).** Jika ingin lebih ketat, kirim ulang status lock via polling/reload saat form dibuka.

## EC-AL-02: Unlock lalu guru re-submit skor

**Skenario:** Admin meng-unlock tab ACADEMIC, guru mengoreksi skor, lalu menyubmit kembali.

**Penanganan:** Setelah unlock, edit diperbolehkan seperti biasa (status `is_locked = false`). Guru yang mengoreksi skor harus menyubmit ulang; jika review-nya bertipe EPMS (`teacher_review`), status kembali ke SUBMITTED setelah submit ulang. **Keputusan: unlock hanya membuka akses edit — status SUBMITTED/DRAFT review tidak diubah otomatis oleh unlock, submit ulang yang mengubahnya.**

## EC-AL-03: Lock lintas tab — apakah mengunci ACADEMIC mengunci CCA juga?

**Skenario:** Admin hanya mengunci tab ACADEMIC. Apakah guru masih bisa mengubah skor CCA?

**Penanganan:** Tab bersifat independen — lock ACADEMIC hanya menolak edit pada tab ACADEMIC. Guru masih bisa mengubah CCA/REMARKS. Untuk mengunci semua sekaligus, gunakan tab khusus `APPRAISAL` (menyeluruh) atau lock tiap tab. **Keputusan: 1 baris `appraisal_lock` per (teacher_id, academic_year_id, tab); tab `APPRAISAL` bersifat master yang menolak edit semua tab.**

## EC-AL-04: Guru pindah campus di tengah AY

**Skenario:** Guru di-lock di campus PIK-S pada Semester 1, lalu pindah ke campus lain pada Semester 2.

**Penanganan:** Baris `appraisal_lock` menyimpan `campus_id` saat lock dibuat. Setelah pindah, baris lama tetap valid untuk data AY tersebut (riwayat skor campus lama). Principal campus baru dapat membuat baris lock baru untuk guru di campus barunya. **Keputusan: baris lock mengikuti campus saat aksi dilakukan; tidak otomatis bermigrasi mengikuti perpindahan guru.**

## EC-AL-05: Academic Year berganti

**Skenario:** Admin melihat lock view untuk AY yang sudah tidak aktif (mis. AY 2024/2025 yang closed) atau AY baru yang belum dibuka.

**Penanganan:** Hanya AY dengan `activeStatus = ACTIVE` yang dapat di-lock/unlock — jika user memfilter AY non-aktif, return 400 "Academic year not found or inactive." Data lock AY lama tetap dapat dibaca (read-only) untuk audit/review. **Keputusan: AY non-aktif read-only untuk lock view; AY aktif bisa diubah.**

## EC-AL-06: Admin tanpa permission mencoba lock

**Skenario:** User dengan role teacher biasa (atau admin tanpa permission appraisal) mencoba memanggil `PUT /v1/appraisal/locks/:teacherId`.

**Penanganan:** Backend enforce permission via `@CheckPermissions` — return 403 "You don't have permission to lock appraisal." Frontend menyembunyikan tombol toggle bagi user tanpa hak (read-only). **Keputusan: enforce di backend; frontend hanya menyembunyikan UI.**

## EC-AL-07: Conflict saat 2 admin mengubah lock bersamaan

**Skenario:** Admin A membuka lock view (status guru X = UNLOCKED). Admin B meng-lock guru X lebih dulu. Admin A kemudian meng-unlock guru X berdasarkan data lama.

**Penanganan:** Gunakan optimistic concurrency: frontend mengirim `expectedUpdatedAt` (atau versi) dari baris saat list dimuat. Jika baris sudah berubah (admin B menulis duluan), backend return 409 "Lock status was changed by another admin. Reload and try again." — admin A reload dan melihat status terbaru. **Keputusan: optimistic locking via `updated_at`; 409 jika ada race.**

## EC-AL-08: Audit trail — siklus lock/unlock berulang

**Skenario:** Guru di-lock → di-unlock → di-lock lagi dalam 1 AY (koreksi berulang). Admin ingin melihat riwayat lengkap.

**Penanganan:** Setiap aksi menulis 1 baris baru di `appraisal_lock_audit` (append-only). Baris `appraisal_lock` hanya menyimpan state terakhir (`locked_by/locked_at` atau `unlocked_by/unlocked_at`), sedangkan seluruh siklus tercatat di audit dengan `from_status`/`to_status` dan `action` LOCK/UNLOCK. **Keputusan: audit append-only, tidak ada update/delete — riwayat lengkap selalu tersedia via `GET /v1/appraisal/locks/audit`.**
