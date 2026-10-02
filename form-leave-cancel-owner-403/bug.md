---
title: Form Leave — Teacher tidak dapat membatalkan request leave milik sendiri (403 silently swallowed)
status: open
severity: major
product: BBS LMS
portal: Teacher
author: System Analyst
date: 2026-10-02
jam: https://jam.dev/c/48dfc13f-d1c0-4996-87ce-77c6a1d288b4
---

# Form Leave — Tombol **Cancel** pada leave request sendiri selalu gagal (403) dan tidak menampilkan error apa pun

## Summary

Dari rekaman Jam (2026-10-02, durasi 1m 29s): user teacher membuka
`https://teacher.smartbag.binabangsaschool.com/leave-form`, menekan tombol
**Cancel** pada leave request miliknya sendiri (leave id 3 dan 4, status
PENDING), mengisi comment, lalu **Save**. Hasilnya **HTTP 403** sebanyak 3 kali
— tidak ada satupun cancel yang berhasil, dan **UI tidak menampilkan error
sama sekali** (modal tetap terbuka, list tetap PENDING, 0 console error dari
425 event log).

Dua bug menyebabkan ini:

1. **Backend — permission model mismatch.** Frontend mengirim
   `PUT /api/v1/leaves/:id/status {leaveStatus:"CANCELED"}`, tetapi service
   backend membatasi endpoint status-update **hanya untuk Principal / Vice
   Principal / Super Admin** (`assertReviewer`). Owner teacher yang membatalkan
   request-nya sendiri ditolak dengan `403 "Only Principal or Vice Principal
   can update leave status."` — padahal endpoint `DELETE /api/v1/leaves/:id`
   (jalur `remove()`) **sudah benar** mengizinkan owner membatalkan leave
   PENDING, hanya saja frontend tidak memanggilnya lagi (kode dipanggil
   komentar).
2. **Frontend — error ditelan.** `LeaveCommentModal.onSubmit` memakai
   `.catch(() => {})`, sehingga 403 hilang tanpa jejak: tidak ada toast, tidak
   ada pesan, user mengira sedang proses / bingung kenapa status tidak berubah.

**Ekspektasi:** sesuai spec `features/form-leave/spec.md` (User Story "I want
to cancel/delete a leave request I created", BR-7, AC-6), owner dapat
membatalkan leave request miliknya sendiri (status PENDING) dari Teacher
Portal — dengan mekanisme **soft-cancel: data TIDAK hilang/dihapus, baris
tetap tampil di Submission List dengan status berubah menjadi CANCELED** —
disertai feedback sukses/gagal yang jelas ke user.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Ariel Wirawan Saputra (Jam author) |
| Email (Jam account) | arielwirawansaputra@gmail.com |
| User ID (dari console log, `selfUser`) | teacher pengguna leave-form (impersonate via Admin Portal) |
| Portal URL | `https://teacher.smartbag.binabangsaschool.com/leave-form` |
| Environment API | `https://api.binabangsaschool.dev` (production) |
| Data konteks | leave id **3** dan **4** (status PENDING), comment `"test"` |
| Browser / OS | Chrome 154.0.8037.58 / macOS (arm) 26.3.0, 1600×890 |
| Jam | `48dfc13f-d1c0-4996-87ce-77c6a1d288b4` (video, 89480 ms, dibuat 2026-10-02T02:21:42.706Z) |

---

## Steps to Reproduce

1. Login ke Teacher Portal (`teacher.smartbag.binabangsaschool.com`),
   impersonate user teacher biasa (bukan Principal/VP).
2. Buat / pastikan ada leave request berstatus **PENDING** (mis. leave id 3).
3. Buka menu **Leave Form** → kolom **Submission List**.
4. Tekan tombol **Cancel** pada baris leave PENDING milik sendiri.
5. Modal "Update Leave Status" terbuka → isi **Comment** (wajib untuk status
   CANCELED) → tekan **Save**.

**Actual Result:**
- Browser mengirim `PUT https://api.binabangsaschool.dev/api/v1/leaves/3/status`
  dengan body `{"data":{"attributes":{"leaveStatus":"CANCELED","comment":"test"}}}`
  → **HTTP 403**,
  response `{"statusCode":"...","code":"...","message":"Only Principal or Vice Principal can update leave status."}`
  (dari `LeaveStatusForbiddenError`).
- Terulang 3× dalam rekaman (leave 3 sekali, leave 4 dua kali).
- **UI tidak menampilkan error apa pun** — modal tetap terbuka, tidak ada
  toast, list tetap PENDING, console bersih (0 error/warn dari 425 log).
  User tidak tahu bahwa aksi gagal.

**Expected Result:**
- Owner teacher dapat membatalkan leave request miliknya sendiri yang berstatus
  PENDING → status berubah **CANCELED** (soft-cancel, baris tetap ada),
  list ter-refresh, toast sukses.
- Bila memang gagal, error harus tampil ke user (toast/modal message), bukan
  ditelan diam-diam.
- Sesuai `form-leave/spec.md`: BR-7 "Delete hanya oleh owner → 403 jika bukan
  owner" (owner JUSTRU harus boleh), AC-6 cancel memakai `bbsConfirm`.

---

## Root Cause Analysis

### Bug #1 — Backend menolak owner: `updateStatus` wajib reviewer — `leave.service.ts:334` + `leave.service.ts:434`

```typescript
// api_nest/src/modules/teacher-leave/leave.service.ts:329-343
async updateStatus(id, callerUserId, options): Promise<LeaveDto> {
  await this.assertReviewer(callerUserId);          // ← baris 334: SELALU dipanggil, tanpa pengecualian owner

  const leave = await Leave.findOne({ where: { id, activeStatus: StatusTypeEnum.ACTIVE }, ... });
  if (!leave) NoActiveLeaveFoundError();

  const campusIds = await getEmployeeScopedCampusIds(this.req.user);
  if (!campusIds.includes(leave.campusId)) CampusPermissionError();
```

```typescript
// api_nest/src/modules/teacher-leave/leave.service.ts:422-435
private async isReviewer(callerUserId: number): Promise<boolean> {
  if (this.isAdminCaller()) return false;
  // Principal, Vice Principal or Super Admin (same rule as the Teacher
  // Portal's usePrincipalOrHod).
  return await checkIfPrincipal(callerUserId, ...);
}

private async assertReviewer(callerUserId: number) {
  if (!(await this.isReviewer(callerUserId))) LeaveStatusForbiddenError();   // ← baris 434
}
```

```typescript
// api_nest/src/errors/ResourceError.ts:5045-5051
export const LeaveStatusForbiddenError = () => {
  throw new ApiError(
    HttpStatus.FORBIDDEN,
    'Forbidden',
    'Only Principal or Vice Principal can update leave status.',
  );
};
```

Route-nya sendiri (`leave.controller.ts:91-109`, `PUT /:id/status`) hanya
menuntut permission `UPDATE LEAVE` (CASL) — yang dimiliki owner — tetapi
guard tambahan di service-lah yang menolaknya.

**Ironinya, jalur owner yang benar sudah ada tapi tidak dipakai:**

```typescript
// api_nest/src/modules/teacher-leave/leave.service.ts:308-327 — DELETE /api/v1/leaves/:id
async remove(id, callerUserId) {
  if (leave.employeeId !== callerUserId) LeaveForbiddenError();  // owner only ✓
  if (leave.activeStatus !== StatusTypeEnum.ACTIVE) NoActiveLeaveFoundError();
  if (leave.leaveStatus !== LeaveStatusEnum.PENDING) LeaveNotPendingError();
  leave.leaveStatus = LeaveStatusEnum.CANCELED;                  // soft-cancel, baris tetap ACTIVE
  ...
}
```

Sedangkan frontend justru memanggil endpoint reviewer — panggilan DELETE lama
sudah dikomentari:

```javascript
// bbs/client-teacher/src/views/form-leave/LeaveForm.jsx:184-196
const handleCancelTeacherLeave = async (leave) => {
  setReview({ leave, leaveStatus: "CANCELED" });   // → modal → updateLeaveStatus (PUT status)

  // bbsConfirm({
  //   message: "Are you sure to cancel this leave request?",
  //   onConfirm: async () => {
  //     return await dispatch(fromApi.removeLeave(id)).then(() => {   // ← DELETE lama, dikomentari
  //       teacherLeavesApi?.refresh();
  //       bbsToaster.success("Successfully cancel the leave request");
  //     });
  //   }
  // });
};
```

Tombol Cancel dirender untuk owner pada baris PENDING
(`LeaveForm.jsx:372-386`) → memicu modal yang sama dengan halaman reviewer
`LeaveRequests.jsx`. Jadi UI menawarkan aksi yang di backend-nya pasti 403
untuk role teacher biasa.

### Bug #2 — Frontend menelan error 403 — `LeaveCommentModal.jsx:65-83`

```javascript
// bbs/client-teacher/src/views/form-leave/components/LeaveCommentModal.jsx:65-83
const onSubmit = async ({ comment }) => {
  setIsSubmitting(true);

  await dispatch(
    fromApi.updateLeaveStatus(leave?.id, {
      leaveStatus,
      comment: comment?.trim() || undefined
    })
  )
    .then(() => {
      bbsToaster.success("Successfully update the leave status");
      onSaved?.();
      onClose();
    })
    .catch(() => {})                          // ← baris 79: 403 dibuang, user tidak dapat feedback apa pun
    .finally(() => {
      setIsSubmitting(false);
    });
};
```

Konsekuensinya modal tidak pernah tertutup, tidak ada toast error, list tidak
refresh — user hanya melihat "tidak terjadi apa-apa". Kombinasi Bug #1 + #2
menghasilkan pengalaman: *tekan Save → diam saja → coba lagi → tetap diam*
(persis yang terekam: 3 percobaan, 3×403, 0 pesan).

### Konteks — konflik dengan spec

`features/form-leave/spec.md` mensyaratkan owner dapat membatalkan request:
User Story *"I want to cancel/delete a leave request I created"*, BR-7
(delete hanya oleh owner), AC-6 (cancel via `bbsConfirm` → soft delete).
Implementasi bermigrasi ke `PUT /leaves/:id/status` (status workflow) tetapi
guard reviewer-nya tidak membedakan "owner membatalkan sendiri" vs
"reviewer mengubah status" → implementasi menyimpang dari spec.

---

## Bukti dari Jam

| Sumber | Temuan |
|--------|--------|
| **Video** | Durasi **89.480 ms** (~1m 29s), direkam via extension Chrome, dibuat 2026-10-02T02:21:42.706Z, host `teacher.smartbag.binabangsaschool.com/leave-form`. Guide Jam: *"3 failed API requests (HTTP 4xx)"*; insight HIGH: *"network errors present but no console errors logged — app may be silently swallowing errors."* |
| **Network — PUT #1** | `2026-10-02 02:20:37.570` (elapsed 27.650 ms) — `PUT https://api.binabangsaschool.dev/api/v1/leaves/3/status` → **403** (85,04 ms). Request `{"data":{"attributes":{"leaveStatus":"CANCELED","comment":"test"}}}`; response `{"statusCode":"...","code":"...","message":"Only Principal or Vice Principal can update leave status."}` |
| **Network — PUT #2** | `02:21:20.162` (elapsed 70.242 ms) — `PUT .../api/v1/leaves/4/status` → **403** (84,05 ms), request/response sama |
| **Network — PUT #3** | `02:21:34.676` (elapsed 84.756 ms) — `PUT .../api/v1/leaves/4/status` → **403** (76,30 ms), request/response sama (user mencoba ulang) |
| **Network — ringkasan** | Total 104 request: `httpStatus {200:67, 201:3, 204:27, **403:3**}`, `httpMethod {GET:66, OPTIONS:27, **PUT:3**, POST:8}` — **semua 3 PUT = 3 error 403**; tidak ada error lain |
| **Console** | 425 event, **semua level `log`** — 0 error, 0 warning, meski 3 request gagal → membuktikan Bug #2 (error ditelan frontend) |
| **User events** | 51 interaksi (44 klik, 7 keyboard) — pola: buka form → cancel → isi comment → save → ulang (3×), tanpa ada feedback error |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/teacher-leave/leave.service.ts:329-372` (`updateStatus`), `:334`, `:433-435` (`assertReviewer`) | Owner teacher selalu ditolak 403 saat membatalkan request sendiri |
| Backend Error | `api_nest/src/errors/ResourceError.ts:5045-5051` (`LeaveStatusForbiddenError`) | Pesan 403 yang muncul di Jam |
| Backend Controller | `api_nest/src/modules/teacher-leave/leave.controller.ts:91-109` (`PUT /:id/status`), `:111-121` (`DELETE /:id`) | Route status dipakai owner; route DELETE (jalur benar) tidak dipanggil frontend |
| Backend DTO | `api_nest/src/modules/teacher-leave/dto/update-status.dto.ts` (`UpdateTeacherLeaveStatusDto`) | Menerima `leaveStatus` + `comment` — tidak ada batasan siapa boleh CANCELED |
| Backend Entity | `api_nest/src/modules/teacher-leave/entities/leave.entity.ts` (`leaveStatus`, `statusChangedBy`, `statusChangedAt`) | Field tujuan update status |
| Frontend View | `bbs/client-teacher/src/views/form-leave/LeaveForm.jsx:184-196, 372-386` | Tombol Cancel owner → modal status; DELETE lama dikomentari |
| Frontend Modal | `bbs/client-teacher/src/views/form-leave/components/LeaveCommentModal.jsx:65-83` (khususnya `:79`) | Mengirim PUT status & `.catch(() => {})` menelan 403 |
| Frontend Action | `bbs/client-teacher/src/actions/fromApi.js:350` (`updateLeaveStatus`), `fromApi.removeLeave` | Aksi terkait pemanggilan endpoint |
| Spec | `bbs-feature/form-leave/spec.md` (User Story, BR-7, AC-6) | Implementasi menyimpang dari perilaku yang disyaratkan |

---

## Proposed Solution Options

### Option A: Izinkan owner self-cancel di `updateStatus` + tampilkan error di modal (Recommended)

**Backend** — di `leave.service.ts updateStatus()`, sebelum `assertReviewer`,
tambahkan cabang owner:

```typescript
const leave = await Leave.findOne({ where: { id, activeStatus: StatusTypeEnum.ACTIVE }, relations: { employee: true, campus: true } });
if (!leave) NoActiveLeaveFoundError();

const isOwner = leave.employeeId === callerUserId;
const isSelfCancel = isOwner && options.leaveStatus === LeaveStatusEnum.CANCELED;

if (!isSelfCancel) {
  await this.assertReviewer(callerUserId);                       // reviewer tetap untuk approve/decline/dst.
  const campusIds = await getEmployeeScopedCampusIds(this.req.user);
  if (!campusIds.includes(leave.campusId)) CampusPermissionError();
}

if (isSelfCancel && leave.leaveStatus !== LeaveStatusEnum.PENDING) {
  LeaveNotPendingError();                                        // owner hanya boleh batalkan yang masih PENDING
}
// aturan comment wajib untuk CANCELED tetap berlaku (baris 347-354)
```

Catatan: `assertReviewer` dipindah setelah `findOne` supaya bisa membandingkan
`leave.employeeId` dengan `callerUserId` (urutan baris 334 vs 336 berubah).

**Frontend** — perbaiki Bug #2 di `LeaveCommentModal.jsx:79`:

```javascript
.catch((err) => {
  bbsToaster.error(err?.response?.data?.message || "Failed to update the leave status");
})
```

Kelebihan: satu endpoint untuk semua transisi status, alur UI tetap sama
(modal + comment, konsisten dengan reviewer), spec terpenuhi (owner bisa
cancel), dan Bug #2 tetap diperbaiki untuk semua pemakai (reviewer termasuk).

### Option B: Kembalikan jalur `DELETE` sesuai spec (owner cancel via endpoint sendiri)

**Frontend** — di `LeaveForm.jsx:184-196`: uncomment kembali `bbsConfirm` +
`fromApi.removeLeave(id)` (DELETE `/api/v1/leaves/:id` → `remove()` yang sudah
benar: owner-only, PENDING-only, soft-cancel), dan hapus/menyembunyikan
pemanggilan `setReview` untuk tombol Cancel di `LeaveForm.jsx:372-386`
(cancel owner tidak lagi lewat modal status).

**Backend** — tidak wajib berubah (`remove()` sudah benar), tapi `DELETE`
sebaiknya tetap memakai `bbsConfirm` di UI sesuai AC-6 spec.

**Frontend tetap** memperbaiki Bug #2 (`LeaveCommentModal.jsx:79`) —
dipakai reviewer di `LeaveRequests.jsx`.

Kelebihan: backend tidak berubah sama sekali, jalur sudah teruji & sesuai
spec persis (AC-6 `bbsConfirm` + soft delete). Kekurangan: dua jalur cancel
berbeda (DELETE untuk owner, PUT status untuk reviewer) — harus konsisten
menentukan mana yang dipakai README/spec ke depan.

### Option C: Minimal — hanya perbaiki visibilitas error (Bug #2 saja)

Ganti `.catch(() => {})` dengan toast error di `LeaveCommentModal.jsx`.
Murah, tapi **tidak menyelesaikan masalah utama** — teacher tetap tidak bisa
cancel request sendiri; hanya errornya yang kini terlihat. Hanya layak sebagai
perbaikan sementara/interim.

---

## Notes

- Rekaman Jam: https://jam.dev/c/48dfc13f-d1c0-4996-87ce-77c6a1d288b4 —
  semua bukti (3× PUT 403, 0 console error) diambil dari metadata + network
  log Jam, bukan observasi manual.
- Severity **major**: fitur inti (cancel request sendiri) tidak berfungsi
  sama sekali untuk semua teacher, dan kegagalannya **silent** (user tidak
  tahu aksinya gagal) — membingungkan dan bisa menyebabkan leave ganda /
  duplikasi request.
- Bug #2 berdampak lebih luas dari bug ini: `.catch(() => {})` juga menelan
  error untuk reviewer (approve/decline gagal) di `LeaveRequests.jsx` —
  kandidat untuk ticket terpisah bila perlu.
- Jalur `DELETE /api/v1/leaves/:id` sudah teruji konsepnya benar (owner-only,
  PENDING-only, soft-cancel ke status CANCELED, `statusChangedBy` terisi) —
  baik Option A maupun B pada dasarnya tinggal menyambungkan ulang frontend
  ke logika yang sudah ada.
- Tidak ada perubahan kode di `api_nest` / `bbs` pada pembuatan tiket ini
  (dokumentasi saja, sesuai `agent.md`).
