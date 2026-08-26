# Notes — Form Leave (Teacher Leave)

> Status: DRAFT — open questions, decisions, discussion notes.

## Open Questions

### NQ-01: Nama modul — `teacher-leave` vs `leave` vs `form-leave`?
Folder feature memakai slug `form-leave` (nama halaman di teacher web). Untuk modul backend api_nest, rekomendasi `teacher-leave` (table `teacher_leave`) karena `leave` terlalu generik dan bisa bentrok dengan konsep leave lain (student leave di dashboard). **Perlu konfirmasi** saat implementasi.

### NQ-02: Endpoint penyimpanan legacy tidak ditemukan — harus desain baru?
Deep analysis: halaman `teacher_file_leave_form.php` (200 OK) merender form lengkap, tapi handler JS untuk tombol `#saveLeaveform` **tidak terikat** saat halaman diakses langsung (bukan lewat shell home.php), dan semua kandidat `/services/*.php` (save_leave, submit_leave, dll) mengembalikan 404. Ini menandakan endpoint save terikat di JS yang hanya dimuat saat halaman dirender di dalam shell home.php (yang belum tertangkap), ATAU fitur ini memang belum berfungsi penuh di akun demo. **Implikasi:** fase 1 mendesain endpoint baru (`POST /v1/teacher-leaves`) mengikuti konvensi api_nest — tidak ada kontrak legacy yang harus dipertahankan.

### NQ-03: Apakah perlu approval workflow (HOD/Principal)?
Teacher web hanya menyimpan pengajuan (tidak ada status approve/reject di form yang tertangkap). **Rekomendasi:** fase 1 tanpa approval — hanya create + list + soft delete. Approval flow masuk enhancement (lihat notes di bawah).

### NQ-04: Permission module `TEACHER_LEAVE` di `ModulesTypeEnum`?
Modul baru butuh entri baru di `src/types/enums` (`ModulesTypeEnum.TEACHER_LEAVE`) + ACL entry di database (modul `casl`). Sama dengan alur registrasi lesson-plan (`LESSON_PLAN`) — lihat NQ-04 di `lesson-plan/notes.md`.

## Keputusan yang Perlu Di-review

| # | Keputusan | Status |
|---|-----------|--------|
| D-01 | `leaveType` sebagai string enum (`SICK_L`, dst), bukan int legacy `leavetype_id` | Disetujui (lihat schema.md catatan #3) |
| D-02 | Soft delete via `activeStatus` (bukan `deletedAt`) | Mengikuti pola modul `lesson` |
| D-03 | Upload PDF memakai modul `file` existing (`ATTACHMENT_FILE`), bukan upload terpisah | Disetujui — `file-entity-type.ts:12` sudah punya `ATTACHMENT_FILE` |
| D-04 | Tidak ada cek overlap di fase 1 | TBD — lihat EC-02 (rekomendasi tetap tolak 409) |

## Enhancement Ideas (di luar scope fase 1)

- Approval workflow HOD/Principal (status PENDING → APPROVED/REJECTED) + notifikasi.
- Auto-integrasi ke Attendance: cuti yang disetujui otomatis menandai hari absen guru.
- Role Principal/HOD melihat semua pengajuan leave per campus (dengan filter department).
- Mapping `department` dan `position` dari data Employee/EmployeePosition otomatis (sekarang input manual, mengikuti teacher web).
- History log / audit trail perubahan status.

## Referensi Konvensi Codebase

### Backend (`api_nest`)
- Base entity: `src/common/base.entity` (`BaseEntityWithDates` — id, createdAt, updatedAt, deletedAt).
- Pagination: `src/common/dto/page-meta.dto` (`PageMetaDto`), `PageOptionsDto`.
- Decorator permission: `src/decorators/permission.decorator.ts` (`@CheckPermissions`), `src/decorators/auth.decorator.ts` (`@Auth`).
- File upload: `src/modules/file/file.controller.ts` — `FileInterceptor('file')` + `ParseFilePipe` + `MIME_TYPES`; `FileEntityTypeEnum.ATTACHMENT_FILE` di `src/types/enums/file-entity-type.ts:12`.
- Enums: `src/types/enums` — `ACLTypeEnum`, `ModulesTypeEnum` (tambah `TEACHER_LEAVE`), `StatusTypeEnum`.
- Modul referensi: `src/modules/lesson/` (pola CRUD + soft delete), `src/modules/employee/` (relasi teacher).
- Migration: `src/database/migrations/` + `migration-source.ts`; command `npm run migration:generate --name=...`.

### Frontend (`bbs/client-teacher`)
- JavaScript (bukan TypeScript) — `jsconfig.json`, file `.jsx`.
- Routing: React Router v5, config-array routing di `src/routes.js` — `React.lazy(() => import("./views/..."))`.
- **API pattern: `makeApiRequestThunk` (`src/actions/makeApiRequest.js`) + endpoint registry `src/actions/fromApi.js` + hook `useFromApi` + JSON:API Redux state — BUKAN axios.**
- Redux: `src/actions/`, `src/reducers/` (util `createSimpleReducer.js`), `src/store/`.
- Auth: token localStorage `bbs-teacher-web-token` / `bbs-web-refresh-token` (`src/global.js`); user di Redux `state.selfUser` / `state.authUser`.
- Shared UI: `bbs-client-common` (`lib/index.js` — BBSHeader, BBSSelect, BBSControlledSelect, BBSTextField, BBSTextArea, BBSButton, bbsConfirm, bbsToaster, BBSSpinner, BBSNoItemCard, BBSTag, BBSBadge); tabel dari `@coreui/react` (CDataTable).
- Referensi view: `src/views/form-class/leaps/` (Leaps.jsx list/DataTable, LeapsForm.jsx create/edit), `src/views/lessonBuilder/`.
