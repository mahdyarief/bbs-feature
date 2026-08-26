# Notes — Form Leave (Teacher Leave)

> Status: DRAFT — open questions, decisions, discussion notes.

## Open Questions

### NQ-01: Nama modul — `teacher-leave` vs `leave` vs `form-leave`?
Folder feature memakai slug `form-leave` (nama halaman di teacher web). Untuk modul backend api_nest, rekomendasi `teacher-leave` (table `teacher_leave`) karena `leave` terlalu generik dan bisa bentrok dengan konsep leave lain (student leave di dashboard). **Perlu konfirmasi** saat implementasi.

### NQ-02: Endpoint penyimpanan legacy — **TERJAWAB (endpoint `savestatusLeave.php` ditemukan)**
Analisis awal (teacher POV) menyimpulkan semua `/services/*.php` 404 dan endpoint tidak terpapar. **Koreksi:** analisis aplikasi legacy menangkap **`POST /ais/asd/services/savestatusLeave.php`** — endpoint simpan status cuti di varian Admin (`home.php?vmenu=form_leave_teacher`). Jadi endpoint **ada**; hanya saja halaman teacher POV tidak memuat handler JS saat diakses langsung. **Implikasi:** fase 1 tetap mendesain endpoint baru (`POST /v1/teacher-leaves`) mengikuti konvensi api_nest, tapi `savestatusLeave.php` bisa dipakai sebagai **referensi payload legacy** (fields `user_id`, `fullname`, `campus`, `HOD`, `as_code`, `commentsleave`).

### NQ-03: Apakah perlu approval workflow (HOD/Principal)? — **PERLU KONFIRMASI**
Nama endpoint `savestatusLeave` (save **status** leave) + field `HOD`/`as_code` di varian Admin mengindikasikan ada **konsep status/approval** di sisi Principal/Admin — bertentangan dengan asumsi awal "tidak ada approval flow". **Rekomendasi:** fase 1 tetap tanpa approval (create + list + soft delete) untuk guru, TAPI konfirmasi ke stakeholder dulu apakah status (pending/approved/rejected) harus disertakan di fase 1 mengingat endpoint legacy menyiratkan hal tersebut.

### NQ-04: Permission module `TEACHER_LEAVE` di `ModulesTypeEnum`?
Modul baru butuh entri baru di `src/types/enums` (`ModulesTypeEnum.TEACHER_LEAVE`) + ACL entry di database (modul `casl`). Sama dengan alur registrasi lesson-plan (`LESSON_PLAN`) — lihat NQ-04 di `lesson-plan/notes.md`.

## Analisis Varian Form (Admin & Principal POV)

| POV | Temuan | Implikasi untuk brief |
|-----|--------|----------------------|
| Admin POV | `home.php?vmenu=form_leave_teacher` (97KB) — form varian admin dengan fields `user_id`, `fullname`, `campus`, `campus_name`, `campus_code`, `user_type`, `HOD`, `as_code` + textarea `commentsleave` | Admin POV melihat & mengelola leave semua guru — cocok dengan mirroring Admin Portal |
| Admin POV | `POST /ais/asd/services/savestatusLeave.php` — endpoint simpan status | Referensi payload legacy; nama endpoint menyiratkan ada status |
| Principal POV | `ais/principals/teacher_file_leave_form.php` (22096 bytes) — versi principal dari form | Principal punya form sendiri di path `ais/principals/` |
| Principal POV | `home.php?vmenu=form_leave_teacher&vname=Form Leave Teacher` (4116 bytes) — "View All Request" | Principal bisa melihat SEMUA pengajuan — dasar fitur lintas guru di fase lanjutan |

## Keputusan yang Perlu Di-review

| # | Keputusan | Status |
|---|-----------|--------|
| D-01 | `leaveType` sebagai string enum (`SICK_L`, dst), bukan int legacy `leavetype_id` | Disetujui (lihat schema.md catatan #3) |
| D-02 | Soft delete via `activeStatus` (bukan `deletedAt`) | Mengikuti pola modul `lesson` |
| D-03 | Upload PDF memakai modul `file` existing (`ATTACHMENT_FILE`), bukan upload terpisah | Disetujui — `file-entity-type.ts:12` sudah punya `ATTACHMENT_FILE` |
| D-04 | Tidak ada cek overlap di fase 1 | TBD — lihat EC-02 (rekomendasi tetap tolak 409) |
| D-05 | **Dual portal**: implementasi di Teacher Portal (`client-teacher`, pengguna utama) + Admin Portal (`client/`, mirroring — admin bantu pengajuan/perubahan atas nama guru, permission `TEACHER_LEAVE_MANAGE`) | Disetujui — lihat spec.md "Dual Portal (Mirroring)" |

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
