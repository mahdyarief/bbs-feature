# API Contract — Lesson Plan

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.LESSON_PLAN` (baru, perlu ditambahkan ke `src/types/enums` + ability di `src/modules/casl/casl-ability.factory.ts`).
> **Base path:** global prefix `api` → URL lengkap `/api/v1/lesson-plans`.

## Konvensi Response

- Sukses (single): `{ "data": {...} }`
- Sukses (list): `{ "data": [...], "count": number, "meta": PageMetaDto }` — `PageMetaDto` dari `src/common/dto/page-meta.dto`; query filter extends `PageOptionsDto` (`page`, `pageSize`, `order`, `sortBy`, `relations`, `query`).
- Error: `HttpException` standar NestJS (`src/errors` / `src/filters`).

## Endpoint Detail

### 1. GET /v1/lesson-plans — List lesson plan milik guru (paginated)

**Query params (GetLessonPlansDto):**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `ay` | int | no | Academic year id (contoh: 27 = 2026/2027) |
| `classSubjectId` | int | no | Filter kelas/subject yang diampu |
| `term` | int | no | 1-4 |
| `week` | int | no | 1-40 |
| `page` | int | no | Default 1 |
| `pageSize` | int | no | Default 10 (dari `PageOptionsDto`) |

**Response 200:**
```json
{
  "data": [
    {
      "id": 102260,
      "topic": "Rate of Reactions",
      "term": 1,
      "week": 3,
      "teacherId": 21046,
      "teacherName": "Herlina Susanti",
      "classSubjectId": 38103,
      "className": "Sec 4 Unity",
      "subjectName": "Chemistry",
      "levelName": "Sec 4",
      "academicYearId": 27,
      "createdAt": "2026-08-26T09:00:00.000Z",
      "updatedAt": "2026-08-26T09:00:00.000Z"
    }
  ],
  "count": 12,
  "meta": { "page": 1, "pageSize": 10, "itemCount": 10, "pageCount": 2, "hasPreviousPage": false, "hasNextPage": true }
}
```

### 2. POST /v1/lesson-plans — Buat lesson plan

**Body (CreateLessonPlanDto):**

```json
{
  "classSubjectId": 38103,
  "academicYearId": 27,
  "term": 1,
  "week": 3,
  "topic": "Rate of Reactions",
  "mainObjectives": "Understand factors affecting rate of reaction (from SOW)",
  "higherOrderObjectives": "Evaluate collision theory in real scenarios",
  "pedagogy": ["lecture", "Group Discussion"],
  "materialResources": ["Textbook Ch.7", "Worksheet 7.2", "Video: collision theory"],
  "activities": "2 periods (Monday) / 20 min intro / 40 min experiment demo",
  "assessmentBefore": ["Questioning"],
  "assessmentDuring": ["Observation", "Questioning", "Discussion"],
  "assessmentAfter": ["Short quiz", "Discussion", "Peer/self assessment"],
  "assignment": "Worksheet 7.3 due next week",
  "reflection": ""
}
```

**Catatan:** `teacherId` TIDAK boleh ada di body — diambil dari `req.user.id`. Validasi: kombinasi unik `(teacherId, classSubjectId, academicYearId, term, week)`; `classSubjectId` harus ∈ kelas yang diampu guru; `week` harus valid untuk `term` pada AY tersebut. `mainObjectives` diisi **manual** oleh guru (mengacu dokumen SOW) — modul SOW (`/v1/sow`) hanya menyediakan link dokumen referensi, bukan data objectives terstruktur (lihat `notes.md` NQ-02).

**Response 201:** `{ "data": { "id": 102260, ... } }` (LessonPlanDto lengkap + detail).

### 3. GET /v1/lesson-plans/:id — Detail lesson plan

**Response 200:**
```json
{
  "data": {
    "id": 102260,
    "topic": "Rate of Reactions",
    "term": 1,
    "week": 3,
    "teacherId": 21046,
    "teacherName": "Herlina Susanti",
    "classSubjectId": 38103,
    "className": "Sec 4 Unity",
    "subjectName": "Chemistry",
    "levelName": "Sec 4",
    "academicYearId": 27,
    "detail": {
      "mainObjectives": "...",
      "higherOrderObjectives": "...",
      "pedagogy": ["lecture", "Group Discussion"],
      "materialResources": ["Textbook Ch.7"],
      "activities": "...",
      "assessmentBefore": ["Questioning"],
      "assessmentDuring": ["Observation"],
      "assessmentAfter": ["Short quiz"],
      "assignment": "...",
      "reflection": "..."
    },
    "comments": [
      {
        "id": 5,
        "commentType": "HOD",
        "comment": "Good structure, please add differentiation",
        "commenterId": 40,
        "commenterName": "Ibu HOD",
        "createdAt": "2026-08-26T10:00:00.000Z"
      }
    ],
    "sourceLessonPlanId": null,
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

### 4. PUT /v1/lesson-plans/:id — Update penuh (header + detail)

**Body (UpdateLessonPlanDto):** sama dengan CreateLessonPlanDto, semua opsional (PartialType). Hanya owner yang bisa (service check `req.user.id == lessonPlan.teacherId`).

### 5. PATCH /v1/lesson-plans/:id — Update status / header-only

**Body:** subset `{ topic?, term?, week?, classSubjectId?, activeStatus? }`. Hanya owner.

### 6. DELETE /v1/lesson-plans/:id — Soft delete

Hanya owner. Set `activeStatus = INACTIVE`.

### 7. POST /v1/lesson-plans/:id/copy — Copy lesson plan

**Body (CopyLessonPlanDto):**
```json
{ "academicYearCopyId": 26, "classSubjectCopyId": 38105 }
```

**Behavior:** salin header + detail (tanpa comments); `sourceLessonPlanId` = id sumber. Jika kombinasi target sudah ada → 409.

### 8. GET /v1/lesson-plans/library — Library viewer (semua guru di campus)

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `ay` | int | yes | Academic year id |
| `campusId` | int | yes | Diambil dari `req.user` (campus teacher) jika tidak dikirim |
| `classSubjectId` | int | no | Filter kelas/subject (dari dropdown Subject) |
| `term` | int | no | 1-4 |
| `week` | int | no | 1-40 |
| `page`, `pageSize` | int | no | Pagination |

**Response 200:** sama seperti list, tapi kolom tambahan `teacherName` wajib (karena menampilkan guru lain).

### 9. GET /v1/lesson-plans/library/classrooms — Daftar kelas untuk filter

**Query:** `ay` (required).
**Response:** `{ "data": [{ "id": 38103, "name": "Sec 4 Unity", "levelName": "Sec 4" }] }`

### 10. GET /v1/lesson-plans/library/subjects — Daftar subject untuk filter

**Query:** `ay` (required), `classroomId` (optional — filter subject per kelas).
**Response:** `{ "data": [{ "id": 270, "name": "CHEM" }] }`

### 11. GET /v1/lesson-plans/no-submission — Guru yang belum submit

**Query:** `ay` (required), `term` (required), `week` (required).
**Permission:** `@CheckPermissions([{ action: READ, subject: LESSON_PLAN }])` + role check HOD/Principal.
**Response:**
```json
{
  "data": [
    { "teacherId": 100, "teacherName": "Budi", "classSubjectId": 38103, "className": "Sec 4 Unity", "subjectName": "Chemistry", "submitted": false }
  ],
  "count": 3
}
```

### 12. POST /v1/lesson-plans/:id/comments — Simpan comment

**Body (CreateLessonPlanCommentDto):**
```json
{ "comment": "Good structure, please add differentiation", "commentType": "HOD" }
```
**Permission:** `commentType=HOD` → user harus HOD; `commentType=PRINCIPAL` → user harus Principal. Role mismatch → 403.
**Response 201:** `{ "data": { "id": 5, "commentType": "HOD", "comment": "...", "commenterId": 40, "createdAt": "..." } }`

### 13. GET /v1/lesson-plans/:id/comments — List comments

**Response 200:** `{ "data": [ { "id": 5, "commentType": "HOD", "comment": "...", "commenterId": 40, "commenterName": "...", "createdAt": "..." } ] }`

## Referensi Implementasi (contoh pola dari modul existing)

- Controller pattern: `src/modules/lesson/lesson.controller.ts` — `@Controller({ version: '1', path: 'lessons' })`, `@CheckPermissions`, `@Req() req: Request`, `ParseIntPipe`, `{ data }` wrapper.
- Entity pattern: `src/modules/lesson/entities/lesson.entity.ts` — `BaseEntityWithDates`, `@Index()` FK, `@ManyToOne` + `@JoinColumn`, `Relation<T>`, `StatusTypeEnum`.
- Pagination: `src/common/dto/page-meta.dto` (`PageMetaDto`).
- Enums: `src/types/enums` — `ACLTypeEnum`, `ModulesTypeEnum` (tambah `LESSON_PLAN`), `StatusTypeEnum`.
