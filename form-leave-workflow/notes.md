# Notes — Form Leave Workflow (Status Approval)

> Status: DRAFT — open questions, decisions, discussion notes.

## Open Questions

### NQ-03: Apakah perlu approval workflow (HOD/Principal)? — **TERJAWAB (ADA, dari analisis ais_legacy)**

**Pertanyaan dari `form-leave/notes.md`:** Nama endpoint `savestatusLeave` (save **status** leave) + field `HOD`/`as_code` di varian Admin mengindikasikan ada konsep status/approval.

**Jawaban dari analisis `ais_legacy`:**

| Bukti | Lokasi | Detail |
|-------|--------|--------|
| Dropdown status per record di daftar pengajuan | `bbs_tools/leave_form.html:836` (iframe `vmenu/form_leave_teacher.php`) | Setiap baris pengajuan punya dropdown status |
| POST ke `savestatusLeave.php` dengan `{status, rec, tipe: '1'}` | `leave_form.html:1194-1212` | Handler JS ganti status |
| POST ke `savestatusLeave.php` dengan `{comment, status, rec, tipe: '1'}` | `leave_form.html:1214-1231` | Handler `#submitcommentleave` — simpan komentar + status |
| Blok di-comment out: `dataMap['field'] = 'comments_principal'; tipe: '2'` | `leave_form.html:1233-1246` | Indikasi komentar Principal terpisah |
| `form_param` base64: `user_id`, `fullname`, `campus` | `leave_form.html:1275-1279` | Konteks reviewer |
| Endpoint `savestatusLeave.php` dan `save_form_teacher_leave.php` | `principals_tool/endpoint_analysis.json` + `bbs_tools/endpoint_analysis.json` | Kedua endpoint terkonfirmasi ada |

**Implikasi:** Approval workflow **harus ada** sebagai fitur lanjutan (`form-leave-workflow`). Fase 1 tetap create/list/soft delete tanpa ubah status. Fase 2 (feature ini) menambahkan status approval + komentar.

### NQ-05: Nilai status legacy di dropdown `vmenu/form_leave_teacher.php`?

**Belum terverifikasi.** Halaman iframe `vmenu/form_leave_teacher.php` tidak tertangkap di `ais_legacy` — hanya shell dashboard-nya (`leave_form.html`). Nilai-nilai dropdown status legacy (misal: 0/1/2 untuk Pending/Approved/Rejected atau string lain) **tidak diketahui**. Yang pasti:

- Ada dropdown status per record (dari referensi HTML).
- Endpoint `savestatusLeave.php` menerima parameter `status` (nilai belum diketahui).
- Ada field `tipe: '1'` (Admin) dan `tipe: '2'` (Principal) — menunjukkan komentar per role.

**Rekomendasi:** Saat implementasi, konfirmasi ke stakeholder nilai status yang dipakai di legacy. Fase 2 menggunakan enum baru (`PENDING`, `APPROVED_BY_ADMIN`, `APPROVED_BY_PRINCIPAL`, `REJECTED`) — tidak perlu mapping 1:1 ke legacy.

### NQ-06: Siapa yang bisa mengubah status? Admin vs Principal vs HOD?

Legacy menunjukkan dua role:
- **Admin** (`tipe: '1'`, field `commentsleave`) — bisa ubah status + komentar Admin.
- **Principal** (`tipe: '2'`, field `comments_principal` — di-comment out di legacy) — komentar Principal terpisah.

**Yang belum jelas:**
- Apakah HOD (Head of Department) juga punya wewenang? Legacy tidak menunjukkan HOD-specific handler.
- Apakah Principal harus approve setelah Admin? Atau bisa langsung?
- Apakah ada urutan wajib (Admin → Principal) atau bisa langsung Principal?

**Rekomendasi:** Fase 2 mengadopsi urutan 2-step: `PENDING → APPROVED_BY_ADMIN → APPROVED_BY_PRINCIPAL` (dengan opsi reject di setiap tahap). Urutan ini bisa disesuaikan dengan konfirmasi stakeholder. Role HOD bisa ditambahkan di enhancement jika diperlukan.

### NQ-07: Screenshot UI approval legacy — **TERJAWAB (capture sukses via login admin `bbs_mng`)**

Capture awal halaman Admin `home.php?vmenu=form_leave_teacher&vname=Form Leave Teacher` **redirect ke halaman login** (file PNG 110KB = halaman login, bukan "View All Request") karena sesi tidak ter-autentikasi.

**Koreksi:** Setelah login fresh dengan akun admin **`bbs_mng` / `NewBBS2025!`** (dari `bbs_tools/01_login_test.py:7-8` dan `06_probe_menu_leave.py:8-9`), halaman `home.php?vmenu=form_leave_teacher` **berhasil di-capture** dan menampilkan **UI approval asli**:

- File: `ais_legacy/bbs_tools/approval_form_leave_teacher.png` (152KB) + `approval_form_leave_teacher.html` (98KB).
- Isi: tabel daftar SEMUA pengajuan leave dengan kolom No, Campus, Date, Teacher, Position, Department, Type, Reason, Comment + filter "All Campus / All Status / All Type" + aksi View per row.
- Data riil: Maria Tiarani (SMG-S, Mathematics), Narlita Garcia (SMG-PS, Preschool), Linawati Lauw (PIK-P, Principal), Stewart James Spiessens (HOD), Rashid Imran (BDG-S, Academics), Tri Afringgasari (SMG-S), dst.
- Script: `capture_leave_approval.py` (login `POST /ais/asd/index.php` {user, pass}, lalu navigate ke `home.php?vmenu=form_leave_teacher`).
- Screenshot ini sudah di-copy ke `features/form-leave-workflow/screenshots/admin-view-all-request.png` sebagai referensi engineer.

**Koreksi lanjutan — modal komentar berhasil di-capture via interaksi:** Script `capture_comment_modal.py` login ulang, navigate ke `home.php?vmenu=form_leave_teacher`, lalu memicu `$('#ModalComments').modal('show')` via JS. Capture sukses:

- File: `ais_legacy/bbs_tools/interact_modal_comments.png` (153KB) — modal "Comments" tampil di atas tabel approval, dengan label "Comments:" + textarea `commentsleave` (placeholder "Type Comment AB Here") + tombol Cancel/Submit.
- File: `ais_legacy/bbs_tools/interact_iframe.html` (1.1MB) — konten iframe `vmenu/form_leave_teacher.php` (halaman daftar pengajuan lengkap) + `interact_shell.png` (shell dashboard).
- Keduanya sudah di-copy ke `features/form-leave-workflow/screenshots/admin-comment-modal.png` + `admin-submission-list.html` sebagai referensi engineer.

**Kesimpulan:** UI approval legacy kini punya referensi visual lengkap: (1) tabel daftar pengajuan + filter (admin-view-all-request.png), (2) modal komentar ASD/Admin `#ModalComments` + `commentsleave` (admin-comment-modal.png). Khusus **komentar Principal** (`comments_principal`, tipe '2') tetap hanya ada sebagai blok di-comment out di `leave_form.html:1233-1246` — belum di-capture visual karena di legacy di-nonaktifkan.

## Keputusan yang Perlu Di-review

| # | Keputusan | Status |
|---|-----------|--------|
| D-06 | Approval workflow sebagai feature terpisah (`form-leave-workflow`), bukan digabung dengan `form-leave` | Disetujui — lihat spec.md dan scope pemisahan fase 1 vs fase 2 |
| D-07 | Scope campus: Admin/Principal hanya bisa ubah status leave guru di campus yang sama | TBD — lihat EC-11 |
| D-08 | State machine: `PENDING → APPROVED_BY_ADMIN → APPROVED_BY_PRINCIPAL` + `PENDING/APPROVED_BY_ADMIN → REJECTED` | TBD — lihat EC-09 |
| D-09 | Reject wajib komentar (EC-10) | TBD |
| D-10 | `adminComment` vs `principalComment` terpisah (mengikuti legacy `commentsleave` vs `comments_principal`) | Disetujui — berdasarkan bukti legacy `leave_form.html:1233-1246` |
| D-11 | Record soft-delete tidak bisa diubah statusnya (EC-12) | TBD |
| D-12 | Penamaan modul diselaraskan dengan implementasi smartbag existing (`/v1/leaves`, entity `Leave`, `employeeId`, `ModulesTypeEnum.LEAVE`, `LeaveStatusEnum`, `attachmentFileId` uuid, `LeaveTypeEnum` numeric). DRAFT spec fase 1/2 memakai nama konseptual `TeacherLeave`; brief fase 2 kini memakai nama aktual smartbag. | Disetujui — hasil audit smartbag (Sep 1) |

## Analisis Varian Form (dari ais_legacy — tambahan untuk fase 2)

| POV | Temuan | Implikasi untuk fase 2 |
|-----|--------|----------------------|
| Admin POV (status) | `leave_form.html:1194-1212` — dropdown status per record, POST `{status, rec, tipe: '1'}` ke `savestatusLeave.php` | Frontend Admin Portal: tambah dropdown status + textarea komentar per record |
| Admin POV (komentar) | `leave_form.html:1214-1231` — `#submitcommentleave` POST `{comment, status, rec, tipe: '1'}` | Komentar Admin disimpan di `adminComment` (tipe '1') |
| Principal POV | `leave_form.html:1233-1246` — di-comment out, `tipe: '2'` untuk `comments_principal` | Komentar Principal disimpan di `principalComment` (tipe '2') — perlu di-uncomment di frontend baru |
| Reviewer context | `leave_form.html:1275-1279` — `form_param` base64 `user_id`, `fullname`, `campus` | `statusChangedBy` diisi dari `req.user.id` — lebih bersih dari base64 |

## Enhancement Ideas (di luar scope fase 2)

- **Notifikasi** ke guru saat status berubah (email/push/in-app) — priority tinggi setelah fase 2.
- **Auto-integrasi ke Attendance** — cuti `APPROVED_BY_PRINCIPAL` otomatis menandai hari absen guru sebagai "Leave" (bukan absent tanpa keterangan).
- **History table** (`leave_status_log`) — riwayat lengkap setiap perubahan status (siapa, dari status apa, ke status apa, kapan, komentar).
- **Bulk action** — approve/reject multiple records sekaligus.
- **Role HOD** — tambahan layer approval di Departemen (setelah Admin, sebelum Principal).
- **Restore soft-deleted record** — Admin bisa mengembalikan record yang dihapus oleh guru.
- **Filter by status** di Admin Portal — filter Submission List berdasarkan status (Pending/Approved/Rejected).

## Referensi Legacy (savestatusLeave.php)

### Payload dari `leave_form.html:1194-1212` (ganti status)

```javascript
$.post('services/savestatusLeave.php', {
  status: selectedStatus,  // nilai dari dropdown (belum diketahui)
  rec: recordId,           // ID record leave
  tipe: '1'                // '1' = Admin commentsleave
});
```

### Payload dari `leave_form.html:1214-1231` (submit komentar + status)

```javascript
$.post('services/savestatusLeave.php', {
  comment: commentText,    // isi textarea commentsleave
  status: selectedStatus,  // nilai dari dropdown
  rec: recordId,           // ID record leave
  tipe: '1'                // '1' = Admin commentsleave
});
```

### Blok Principal (di-comment out) `leave_form.html:1233-1246`

```javascript
// dataMap['field'] = 'comments_principal';
// tipe: '2'
```

### form_param (base64) `leave_form.html:1275-1279`

```html
<form id="form_param">
  <input name="user_id" value="NTg2">  <!-- base64 dari 586 -->
  <input name="fullname" value="">
  <input name="campus" value="">
  <input name="campus_name" value="">
</form>
```

## Referensi Konvensi Codebase

Sama dengan fase 1 (`form-leave/notes.md`):
- Backend: `api_nest` — `src/modules/teacher-leave/` (extend).
- Frontend: `bbs/client` (Admin) + `bbs/client-teacher` (Teacher).
- API: `makeApiRequestThunk` + `fromApi.js` + `useFromApi` — BUKAN axios.
- UI: `bbs-client-common` + `@coreui/react` CDataTable.