# Notes — Form Leave Workflow (Approval & Teacher Leave Management)

> Status: APPROVED — diselaraskan langsung dengan fungsionalitas Principal Portal (`https://zone.binabangsaschool.com/ais/principals/home.php?vmenu=form_leave_teacher&vname=Form%20Leave%20Teacher`), capture `ais_legacy/principals_tool`, dan codebase `smartbag` (`api_nest` & `bbs`).

---

## Ringkasan Investigasi & Keputusan Kunci

### 1. Sumber dan Alur Modul Legacy
- **Sumber Utama:** `https://zone.binabangsaschool.com/ais/principals/home.php?vmenu=form_leave_teacher&vname=Form%20Leave%20Teacher`
- **Capture File:** `ais_legacy/principals_tool/leaves/approval_iframe.html` (100KB) dan `home_vmenu_form_leave_teacher.html` (52KB).
- **Akses Role:** Di legacy berada pada Principal tools (misal akun `bbs_linawati`). Di arsitektur baru, diimplementasikan di **Teacher Portal (`client-teacher`)** pada menu **\"Form Leave Teacher\"** yang diproteksi menggunakan hook `usePrincipalOrHod` (`isPrincipalOrVp === true`).
- **Akses Admin Portal (`client`):** Di Admin Portal modul ini disediakan sebagai **View Only** berdasarkan RBAC (`ModulesTypeEnum.LEAVE`, `ACLTypeEnum.READ`). Admin tidak melakukan perubahan status.

### 2. Nilai Dropdown Status (Sesuai `approval_iframe.html`)
Pilihan status pada legacy dropdown (`select#changestatusReqLeave_[id]`):
- `0` → `Pending` (`PENDING`)
- `1` → `Approve ( unpaid leave )` (`APPROVED_UNPAID`)
- `2` → `Approve ( paid leave )` (`APPROVED_PAID`)
- `3` → `Decline` (`DECLINED`)
- `6` → `Cancel` (`CANCELED`)

### 3. Aturan Pembatalan & Ordering Khusus (Cancel Behavior & Query Order)
- Saat permohonan dibatalkan (Cancel Request), **data TIDAK DIHAPUS** (baik soft delete maupun hard delete, `activeStatus` tetap `ACTIVE = 1`).
- Status permohonan diubah menjadi `CANCELED`.
- Data permohonan dengan status `CANCELED` **selalu berada di urutan paling bawah (order paling bawah)** pada tabel frontend dan query backend.
- Implementasi query SQL / TypeORM:
  ```sql
  ORDER BY (CASE WHEN leave.leave_status = 'CANCELED' THEN 1 ELSE 0 END) ASC, leave.created_at DESC
  ```

### 4. Notifikasi Email Otomatis (Queue Bull `mailer` via AWS SES)
- **Saat Pengiriman Baru (Submission):** Sistem mengirimkan notifikasi email dari pengirim (guru) ke alamat email Principal unit/campus terkait (`employee.email`).
- **Saat Pergantian Status (Status Transition):** Setiap kali status diubah oleh Principal (Pending → Approve Paid/Unpaid, Decline, Cancel), notifikasi email otomatis dikirim ke guru pemohon memuat status baru dan catatan komentar peninjau.
- Menggunakan Bull Queue `SEND_EMAIL` di `api_nest/src/modules/mailer/` dengan exponential backoff dan retry 3x.

---

## Log Keputusan Desain

| # | Keputusan | Deskripsi & Justifikasi |
|---|-----------|-------------------------|
| D-01 | **Portal Peninjauan** | Menu peninjauan status ditempatkan di Teacher Portal (`client-teacher`) dengan hak akses Principal/VP (`usePrincipalOrHod`), bukan guru biasa. |
| D-02 | **Admin Portal Mode** | Admin Portal (`client/`) beroperasi dalam mode **View Only** berbasis RBAC (`ModulesTypeEnum.LEAVE`, `ACLTypeEnum.READ`). |
| D-03 | **Status Approval Enum** | Mengadopsi 5 status legacy: `PENDING`, `APPROVED_UNPAID`, `APPROVED_PAID`, `DECLINED`, `CANCELED`. |
| D-04 | **Penanganan Pembatalan** | Pembatalan (`CANCELED`) tidak menghapus baris data. Data tetap disimpan utuh untuk kepentingan audit trail. |
| D-05 | **Urutan Record Canceled** | Record dengan status `CANCELED` dipaksa selalu berada di bagian bawah list query (`ORDER BY CASE WHEN status='CANCELED' THEN 1 ELSE 0 END ASC, createdAt DESC`). |
| D-06 | **Notifikasi Email 2 Arah** | Integrasi asinkron dengan modul mailer: Guru → Email Principal saat submit, Principal → Email Guru saat status berubah. |
| D-07 | **Mandatory Comment** | Komentar/alasan wajib diisi saat peninjau memilih status `DECLINED` atau `CANCELED`. |

---

## Referensi File Terkait di Codebase Smartbag

### Backend (`api_nest`)
- `src/modules/teacher-leave/leave.controller.ts` (Controller REST API)
- `src/modules/teacher-leave/leave.service.ts` (Business logic, custom ordering, status mutation)
- `src/modules/teacher-leave/entities/leave.entity.ts` (Entity TypeORM)
- `src/types/enums/leave-status.ts` (Enum status permohonan cuti)
- `src/modules/mailer/mailer.service.ts` & `mailer-queue.processor.ts` (Queue Bull email notification)

### Frontend (`bbs`)
- `client-teacher/src/hooks/usePrincipalOrHod.js` (Guard role Principal & VP)
- `client-teacher/src/containers/_nav.jsx` (Sub-menu \"Form Leave Teacher\")
- `client-teacher/src/views/formLeaveTeacher/` (Halaman persetujuan & modal komentar Principal)
- `client/src/views/formLeave/` (Halaman Admin View-Only dengan RBAC)
