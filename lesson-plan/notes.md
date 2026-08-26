# Notes — Lesson Plan

> Status: DRAFT — open questions, decisions, discussion notes.

## Open Questions

### NQ-01: Entity mana yang dipakai untuk `class_subject_id`?
Teacher web memakai `class_list_new.php` / `classsub` (id 38103 = SAMPLE-CHE). Di `api_nest` ada beberapa kandidat: `classroom`, `homeroom-subject-teacher`, `subject-year`, `master-subjects`. **Perlu konfirmasi** entity mana yang merepresentasikan "class-subject assignment guru per AY" — kemungkinan `homeroom-subject-teacher` (menghubungkan teacher + classroom + subject + AY). Lihat `src/modules/homeroom-subject-teacher/`.

### NQ-02: Sumber Main Objectives dari SOW — endpoint mana? ✅ RESOLVED (2026-08-26)
**Keputusan: copy manual.** Audit kode smartbag (`bbs` + `api_nest`) menunjukkan modul SOW di `api_nest/src/modules/sow/` HANYA registry dokumen: entity `SOW` (sow.entity.ts) hanya punya kolom `sowUrl` (link dokumen eksternal), `subjectId`/`subjectName`, `masterLevelId`, `yearFrom`/`yearTo`, `sowType`, `activeStatus`, `createdBy`/`uploadedBy`/`updatedBy` — **TIDAK ada** kolom `term`, `week`, atau `mainObjectives`/`learningObjectives`. Controller `/api/v1/sow` hanya CRUD dokumen (GET/POST/PUT/DELETE). Frontend `Sow.jsx` menampilkan daftar dokumen (No | AY | Level | Subject) → link ke detail URL (`SowDetails.jsx`). Teacher web legacy mengambil objectives dari viewer PHP (`choose_cohort_viewer_teacher.php`) dan guru menyalin manual ke form lesson plan.
**Implikasi implementasi:** `mainObjectives` adalah field teks yang diisi manual oleh guru; UI boleh menampilkan link dokumen SOW terkait (via `/v1/sow`) sebagai referensi di form/detail lesson plan. Auto-fetch objectives per `(classSubjectId, term, week)` membutuhkan model SOW detail baru — out of scope fase 1 (sudah tercatat di Enhancement Ideas).

### NQ-03: Nilai `week` dari mana di dropdown?
`AcademicYearWeek` entity (`src/modules/academic-year/entities/academic-year-week.entity.ts`) kemungkinan menyimpan mapping term→week. **Perlu cek** apakah entity ini berisi `term` + `week` per AY, atau hanya daftar tanggal. Jika tidak ada mapping term→week, perlu sumber lain (misal konstanta 10 minggu/term).

### NQ-04: Permission module `LESSON_PLAN` di `ModulesTypeEnum`?
Modul baru butuh entri baru di `src/types/enums` (`ModulesTypeEnum.LESSON_PLAN`) + ACL entry di database (modul `casl`). **Perlu konfirmasi** alur registrasi module permission yang benar (apakah cukup enum + seed, atau perlu entry di tabel permission).

## Keputusan yang Perlu Di-review

| # | Keputusan | Status |
|---|-----------|--------|
| D-01 | JSON array (pedagogy, materialResources, assessment*) disimpan sebagai TEXT serialized, bukan jsonb | Belum di-review — lihat `schema.md` catatan desain #1 |
| D-02 | Copy dari library memakai `teacherId = req.user.id` (EC-02 opsi A) | TBD |
| D-03 | Soft delete via `activeStatus` (bukan `deletedAt`) | Mengikuti pola modul `lesson` |
| D-04 | Nama modul `lesson-plan` (hyphen) untuk membedakan dari `lesson` (LESSON_BUILDER) | Disetujui |
| D-05 | **Dual portal**: implementasi di Teacher Portal (`client-teacher`, pengguna utama) + Admin Portal (`client/`, mirroring — admin bantu perubahan atas nama guru, permission `LESSON_PLAN_MANAGE`) | Disetujui — lihat spec.md "Dual Portal (Mirroring)" |

## Enhancement Ideas (di luar scope fase 1)

- Edit/hapus komentar oleh penulis (EC-07 opsi B).
- Attach HBL resources ke lesson plan (teacher web punya `list_HBLresources.php` — belum dieksplorasi detail).
- Export lesson plan ke PDF (teacher web memakai html2pdf untuk report, bisa diterapkan di sini).
- Auto-fetch Main Objectives dari SOW berdasarkan `(classSubjectId, term, week)`.

## Referensi Konvensi Codebase

### Backend (`api_nest`)
- Base entity: `src/common/base.entity` (`BaseEntityWithDates` — id, createdAt, updatedAt, deletedAt).
- Pagination: `src/common/dto/page-meta.dto` (`PageMetaDto`).
- Decorator permission: `src/decorators/permission.decorator.ts` (`@CheckPermissions`), `src/decorators/auth.decorator.ts` (`@Auth`).
- Enums: `src/types/enums` — `ACLTypeEnum`, `ModulesTypeEnum`, `StatusTypeEnum`.
- Modul referensi: `src/modules/lesson/` (pola CRUD + duplicate — mirip copy), `src/modules/sow/`, `src/modules/academic-year/`.
- Migration: `src/database/migrations/` + `migration-source.ts`; command `npm run migration:generate --name=...`.

### Frontend (`bbs/client-teacher`)
- JavaScript (bukan TypeScript) — `jsconfig.json`, file `.jsx`.
- Routing: React Router v5, config-array routing di `src/routes.js` — `React.lazy(() => import("./views/..."))`.
- **API pattern: `makeApiRequestThunk` (`src/actions/makeApiRequest.js`) + endpoint registry `src/actions/fromApi.js` + hook `useFromApi` + JSON:API Redux state — BUKAN axios.**
- Redux: `src/actions/`, `src/reducers/` (util `createSimpleReducer.js`), `src/store/`.
- Auth: token localStorage `bbs-teacher-web-token` / `bbs-web-refresh-token` (`src/global.js`); user di Redux `state.selfUser` / `state.authUser`.
- Role hooks: `usePrincipalOrHod` / `useHomeroomTeacher` (`src/hooks/`).
- Shared UI: `bbs-client-common` (`lib/index.js` — BBSHeader, BBSSelect, BBSControlledSelect, BBSResourceSelect, BBSTextField, bbsConfirm, bbsToaster); tabel dari `@coreui/react`.
- Referensi view: `src/views/form-class/leaps/` (Leaps.jsx list/DataTable, LeapsForm.jsx create/edit, LeapsDetail.jsx detail+modals), `src/views/sow/` (viewer), `src/views/lessonBuilder/`.
