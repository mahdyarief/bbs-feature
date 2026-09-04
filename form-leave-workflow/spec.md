---
feature: Form Leave Workflow (Approval & Teacher Leave Management)
slug: form-leave-workflow
status: approved
author: System Analyst (deep analysis dari ais_legacy principals_tool & smartbag codebase)
date: 2026-09-02 (revised 2026-09-04)
target_release: TBD
reference_url: https://zone.binabangsaschool.com/ais/principals/home.php?vmenu=form_leave_teacher&vname=Form%20Leave%20Teacher
depends_on: form-leave (modul Leave base di api_nest: create/list/detail/soft delete)
---

# Form Leave Workflow — Approval & Teacher Leave Management

## Overview

Fitur **Form Leave Workflow** mereplikasi modul manajemen cuti/izin guru legacy dari:
`https://zone.binabangsaschool.com/ais/principals/home.php?vmenu=form_leave_teacher&vname=Form%20Leave%20Teacher`
sebagaimana dianalisis pada `ais_legacy/principals_tool` (`approval_iframe.html`, `home_vmenu_form_leave_teacher.html`, dan handler `savestatusLeave.php`).

Fitur ini melengkapi entitas dasar `Leave` (`api_nest/src/modules/teacher-leave/`) dengan arsitektur portal terintegrasi:
1. **Teacher Portal (`bbs/client-teacher`) — Menu "Form Leave Teacher":**
   - Menu terdedikasi khusus untuk **Principal / Vice Principal** yang diproteksi menggunakan hook `usePrincipalOrHod` (`isPrincipalOrVp === true`).
   - Principal dapat melihat seluruh permohonan leave guru di sekolahnya, mengubah status permohonan (`Pending`, `Approve ( unpaid leave )`, `Approve ( paid leave )`, `Decline`, `Cancel`), dan menuliskan komentar peninjauan secara interaktif.
2. **Admin Portal (`bbs/client`) — View Only (RBAC):**
   - Ditujukan untuk Administrator / ASD / HR sebagai monitoring read-only permohonan cuti guru lintas unit/campus.
   - Diproteksi berbasis RBAC (`ModulesTypeEnum.LEAVE`, action `ACLTypeEnum.READ`). Admin tidak mengubah status di sini, melainkan melakukan supervisi dan rekap data.
3. **Aturan Pembatalan (Cancelation & Ordering Rule):**
   - Saat permohonan dibatalkan (baik oleh guru sebelum diproses maupun oleh reviewer/Principal), data **TIDAK DIHAPUS** dari database.
   - Status permohonan berubah menjadi `CANCELED` (`activeStatus` tetap `ACTIVE = 1`).
   - Pada query list di database maupun tampilan frontend, record berstatus `CANCELED` **selalu diurutkan di urutan paling bawah (order paling bawah)** menggunakan klausa custom sorting:
     `ORDER BY (CASE WHEN leave.leave_status = 'CANCELED' THEN 1 ELSE 0 END) ASC, leave.created_at DESC`.
4. **Notifikasi Email Otomatis (AWS SES via Bull Queue `mailer`):**
   - **Saat Pengiriman Baru (Submission):** Notifikasi email otomatis dikirim dari sistem atas nama pengirim (guru) ke alamat email Principal campus terkait (`employee.email`).
   - **Saat Pergantian Status (Status Transition):** Setiap kali reviewer mengubah status (misal Pending → Approve Paid / Unpaid / Decline / Cancel), notifikasi email konfirmasi otomatis dikirimkan ke email guru pemohon beserta catatan komentar peninjau.

## Problem / Motivation

1. **Kebutuhan Persetujuan Cuti Terpusat:** Pengajuan cuti guru harus diverifikasi langsung oleh Principal/VP unit bersangkutan untuk memastikan kelancaran operasional kelas dan ketersediaan guru pengganti (substitute).
2. **Audit Trail & Transparansi Status:** Membedakan dengan jelas izin berbayar (*paid leave*) dan izin tanpa gaji (*unpaid leave*), serta mencatat alasan penolakan atau pembatalan secara permanen tanpa menghilangkan jejak audit.
3. **Data Integrity pada Pembatalan:** Penghapusan hard-delete berisiko menghilangkan histori pengajuan. Dengan status `CANCELED` dan perlakuan sorting di urutan paling bawah, data historis tetap utuh dan tidak mengganggu antrean permohonan aktif.
4. **Kecepatan Koordinasi via Email:** Email otomatis menjamin Principal segera mengetahui adanya pengajuan izin baru tanpa harus terus-menerus mengecek portal secara manual, dan guru segera memperoleh kepastian permohonannya.

## Referensi Analisis (Berdasarkan `ais_legacy/principals_tool`)

Hasil audit langsung terhadap `approval_iframe.html` (100KB) dan `home_vmenu_form_leave_teacher.html` (52KB):

| Komponen Legacy | Detail Implementasi Legacy | Mapping & Penyelarasan Smartbag |
|---|---|---|
| **Halaman Legacy** | `home.php?vmenu=form_leave_teacher&vname=Form Leave Teacher` | Menu Teacher Portal `Form Leave Teacher` (`/form-leave-teacher`) |
| **Akses Role** | Akun Principal (`bbs_linawati`), sesi level Principal | Teacher Portal hook `usePrincipalOrHod` (`isPrincipalOrVp`) |
| **Dropdown Status** | `select#changestatusReqLeave_[id]` (values: 0=Pending, 1=Approve unpaid, 2=Approve paid, 3=Decline, 6=Cancel) | Enum `LeaveStatusEnum` (`PENDING`, `APPROVED_UNPAID`, `APPROVED_PAID`, `DECLINED`, `CANCELED`) |
| **Modal Komentar** | `#ModalComments` muncul otomatis saat status bernilai 1, 2, 3, atau cancel | Modal komentar interaktif sebelum konfirmasi simpan status |
| **Handler Update** | POST `services/savestatusLeave.php` (`{comment, status, rec, tipe: '1'}`) | `PATCH /api/v1/leaves/:id/status` dengan JWT token auth |
| **Filter Header** | Filter Campus, Status (Pending, Approve, Decline, Cancel), Leave Type | Filter query params `campusId`, `leaveStatus`, `leaveType`, `year` |
| **Tampilan Admin** | `home.php?vmenu=form_leave_teacher` di ASD Portal | Admin Portal (`client/src/views/formLeave/`) dengan mode View-Only (RBAC `READ`) |

## Scope

### In Scope
1. **Teacher Portal (`client-teacher/src/views/formLeaveTeacher/`):**
   - Sub-menu khusus di sidebar `_nav.jsx` berlabel **"Form Leave Teacher"** dengan rute `/form-leave-teacher`.
   - Proteksi route dan view menggunakan hook `usePrincipalOrHod.js` (`isPrincipalOrVp === true`). Jika bukan Principal/VP, akses ditolak (403 / redirect).
   - Tabel permohonan cuti guru seluruh campus/unit yang dipimpin dengan kolom: No, Campus, Tanggal Cuti, Nama Guru, Posisi, Departemen, Jenis Cuti, Alasan, Lampiran Surat Dokter/Dokumen, Status Dropdown, dan Komentar Review.
   - Kontrol pembaruan status interaktif:
     * Dropdown pilihan status: `Pending`, `Approve ( unpaid leave )`, `Approve ( paid leave )`, `Decline`, `Cancel`.
     * Modal dialog komentar review yang otomatis terbuka saat memilih Approve/Decline/Cancel untuk memasukkan catatan peninjau.
2. **Admin Portal (`client/src/views/formLeave/` — View Only):**
   - Tampilan antarmuka pemantauan data permohonan cuti guru bagi HRD dan Manajemen ASD.
   - Diproteksi berbasis RBAC (`ModulesTypeEnum.LEAVE`, `ACLTypeEnum.READ`).
   - Tampilan tabel bersifat **View Only** (read-only): menampilkan badge status dan riwayat komentar tanpa adanya dropdown pengubah status maupun aksi mutasi.
3. **Backend Modul `teacher-leave` (`api_nest`):**
   - **Schema Extension:** Menambahkan kolom status approval, komentar reviewer, dan audit trail pada tabel `leave`:
     * `leave_status`: Enum `PENDING`, `APPROVED_UNPAID`, `APPROVED_PAID`, `DECLINED`, `CANCELED`.
     * `reviewer_comment`: Text nullable (menyimpan catatan feedback dari Principal/Reviewer).
     * `status_changed_by`: FK ke `employee.id` (pencatat user pengubah status).
     * `status_changed_at`: Timestamp perubahan status.
   - **Business Rules Status & Sorting:**
     * Status default permohonan baru adalah `PENDING`.
     * Pembatalan cuti (`CANCELED`) tidak menghapus baris data (`activeStatus` tetap `ACTIVE = 1`).
     * Query data list di database menerapkan aturan urutan khusus: record dengan status `CANCELED` **selalu berada di urutan paling bawah**:
       ```sql
       ORDER BY (CASE WHEN leave.leave_status = 'CANCELED' THEN 1 ELSE 0 END) ASC, leave.created_at DESC
       ```
   - **Notifikasi Email (Queue-based Mailer):**
     * Terintegrasi dengan modul `mailer` (`api_nest/src/modules/mailer/` dan queue `SEND_EMAIL`).
     * Event 1: Email ke Principal saat guru pertama kali mengirimkan form leave (`LEAVE_SUBMITTED_NOTIFICATION`).
     * Event 2: Email ke Guru pemohon saat statusnya diperbarui oleh Principal (`LEAVE_STATUS_CHANGED_NOTIFICATION`).
4. **RBAC & Endpoint Security:**
   - `GET /v1/leaves`: Mengambil daftar cuti dengan filter dinamis dan ordering khusus status Canceled.
   - `PATCH /v1/leaves/:id/status`: Endpoint khusus untuk memperbarui status dan komentar permohonan cuti guru.

### Out of Scope
- Sinkronisasi otomatis ke mesin absensi biometric fisik pihak ketiga.
- Pemotongan jatah cuti tahunan (payroll cut-off calculation engine) — payroll dilakukan di modul finance terpisah.
- Modifikasi isi konten formulir cuti (tanggal dan alasan) oleh Principal (Principal hanya berhak approve/decline/cancel, bukan mengubah tanggal izin guru).

## User Stories

### As a Teacher (Pemohon)
- Saya ingin mengajukan form permohonan cuti/izin lengkap dengan lampiran surat keterangan dokter/dokumen pendukung.
- Saya ingin mendapatkan konfirmasi email otomatis bahwa permohonan saya telah diterima oleh Principal.
- Saya ingin membatalkan permohonan yang sudah dibuat jika terjadi perubahan rencana, di mana data saya tetap tercatat sebagai Canceled dan tidak hilang.
- Saya ingin menerima notifikasi email saat permohonan cuti saya telah disetujui (paid/unpaid) atau ditolak oleh Principal.

### As a Principal / Vice Principal (Reviewer di Teacher Portal)
- Saya ingin mengakses menu khusus "Form Leave Teacher" di Teacher Portal untuk meninjau seluruh permohonan cuti guru di campus saya.
- Saya ingin memperbarui status permohonan cuti menjadi `Approve ( unpaid leave )`, `Approve ( paid leave )`, `Decline`, atau `Cancel` disertai alasan/komentar yang jelas.
- Saya ingin permohonan yang dibatalkan (`CANCELED`) tetap tersimpan dalam sistem namun otomatis bergeser ke baris paling bawah agar antrean permohonan aktif tetap rapi dan terfokus.

### As an Admin / ASD Management (View Only di Admin Portal)
- Saya ingin melihat seluruh rekapan permohonan cuti guru dari seluruh campus secara read-only sesuai hak akses RBAC saya.
- Saya ingin menyaring daftar berdasarkan Campus, Status Permohonan, Jenis Cuti, dan Rentang Tahun tanpa risiko mengubah data yang telah diputuskan oleh Principal.

## Acceptance Criteria

- [ ] **AC-1 (Menu & Akses Teacher Portal):** Sub-menu "Form Leave Teacher" (`/form-leave-teacher`) tampil di Teacher Portal dan hanya dapat dibuka jika `usePrincipalOrHod` bernilai `isPrincipalOrVp === true`. User non-Principal diarahkan keluar (403/redirect).
- [ ] **AC-2 (Tampilan Admin View-Only):** Halaman Form Leave di Admin Portal (`client/`) beroperasi dalam mode View-Only berdasarkan permission `ModulesTypeEnum.LEAVE` (`READ`), menampilkan data status dan komentar tanpa kontrol perubahan data.
- [ ] **AC-3 (Pembaruan Status & Komentar):** Principal dapat mengubah status record cuti melalui endpoint `PATCH /v1/leaves/:id/status` dengan opsi status: `PENDING`, `APPROVED_UNPAID`, `APPROVED_PAID`, `DECLINED`, `CANCELED`.
- [ ] **AC-4 (Mandatory Comment on Decline/Cancel):** Pengubahan status menjadi `DECLINED` atau `CANCELED` mewajibkan pengisian kolom komentar/alasan.
- [ ] **AC-5 (Aturan Pembatalan Canceled):** Permohonan yang dibatalkan (`CANCELED`) tidak di-soft-delete ataupun di-hard-delete (`activeStatus = 1`). Data tetap ada di database dan riwayat.
- [ ] **AC-6 (Sorting Record Canceled di Urutan Terbawah):** Pada endpoint `GET /v1/leaves` maupun tabel frontend, seluruh record dengan status `CANCELED` wajib diurutkan pada posisi paling bawah, diikuti urutan tanggal pembuatan terbaru (`createdAt DESC`) untuk status lainnya.
- [ ] **AC-7 (Notifikasi Email Pengiriman Form):** Saat guru mengirimkan permohonan cuti baru, sistem secara asinkron (queue processor) mengirimkan email notifikasi ke alamat email Principal campus pemohon (`employee.email`).
- [ ] **AC-8 (Notifikasi Email Perubahan Status):** Saat Principal memperbarui status cuti guru, sistem mengirimkan notifikasi email ke guru pemohon yang memuat status baru (`Approved Paid`, `Approved Unpaid`, `Declined`, `Canceled`) dan catatan komentar peninjau.
- [ ] **AC-9 (Audit Log Reviewer):** Kolom `status_changed_by` secara otomatis mencatat ID Principal yang melakukan eksekusi (dari `req.user.id`) beserta timestamp `status_changed_at`.
- [ ] **AC-10 (Proteksi Record Inaktif):** Record permohonan yang berstatus `activeStatus = INACTIVE` (0) tidak dapat diubah statusnya (mengembalikan error 404/400).

## Screenshots & Reference UI

1. **Menu Form Leave Teacher — Principal POV (`principals_tool`):**  
   *Tampilan iframe `vmenu/form_leave_teacher.php` dengan filter Campus/Status/Type, data guru, lampiran file dokter, dan dropdown status per baris:*  
   ![Admin View All Request](screenshots/admin-view-all-request.png)

2. **Modal Komentar Review (`#ModalComments`):**  
   *Modal interaktif pengisian komentar reviewer saat pemilihan status cuti:*  
   ![Admin Comment Modal](screenshots/admin-comment-modal.png)

3. **Form Permohonan Guru (Teacher View):**  
   *Formulir pengajuan cuti guru:*  
   ![Teacher Form Filled](screenshots/teacher-form-filled.png)


**Setelah submit (Submission List milik guru):**
![Teacher - After Submit](screenshots/teacher-form-after-submit.png)

### Referensi HTML Approval (modal komentar + handler JS)

File `D:\Work\BBS\requirement\ais_legacy\bbs_tools\leave_form.html` (Admin dashboard shell) — berisi elemen UI approval yang tidak ada di screenshot di atas (modal komentar di dalam iframe):

| Baris | Elemen | Bukti approval workflow |
|-------|--------|--------------------------|
| 882–888 | Modal `#ModalComments` + textarea `#commentsleave` + hidden `#recleave`/`#statleave` | Form komentar Admin saat approve/reject |
| 1179–1212 | Handler `[id^="changestatusReqLeave_"]` (change) → `$.post('services/savestatusLeave.php', {status, rec, tipe:'1'})` | **Dropdown status per record** (id `changestatusReqLeave_<rec>`) |
| 1214–1231 | `#submitcommentleave` → `$.post('services/savestatusLeave.php', {comment, status, rec, tipe:'1'})` | Simpan komentar + status Admin sekaligus |
| 1233–1246 | Blok di-comment out: `dataMap['field'] = 'comments_principal'; tipe: '2'` | Komentar Principal terpisah (`comments_principal`) |
| 1275–1279 | `<form id="form_param">` base64 `user_id`, `fullname`, `campus`, `campus_name` | Konteks reviewer yang mengubah status |
