# API Contract — EPMS Work Review

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.PETALS` (atau enum appraisal setara — lihat `edgecases.md`).
> **Base path:** global prefix `api` → URL lengkap `/api/v1/epms/reviews`.
> **Dual portal:** Teacher Portal (scope `req.user.id`, Principal input per campus-nya) + Admin Portal (admin punya permission appraisal untuk input lintas campus).

## Konvensi Response

- Sukses (single): `{ "data": {...} }`
- Sukses (list): `{ "data": [...], "count": number, "meta": PageMetaDto }`
- Error: `HttpException` standar NestJS.

## Item Kompetensi (static config)

Item kompetensi EPMS di-seed dari form Work Review legacy (`reference/work_review_form.html`). Tidak ada endpoint CRUD untuk item — hanya dibaca.

### GET /v1/epms/items — List semua item kompetensi

**Response 200:**
```json
{
  "data": [
    { "id": 1, "section": 1, "sectionName": "Key Performance Areas (KRA)", "groupId": 1, "groupLabel": "1. Holistic Development of Students through", "label": "Quality Learning of Students", "sortOrder": 1 },
    { "id": 2, "section": 1, "sectionName": "Key Performance Areas (KRA)", "groupId": 1, "groupLabel": "1. Holistic Development of Students through", "label": "Pastoral Care and Well-being of students", "sortOrder": 2 },
    ... 20+ items total (7 section)
  ]
}
```

Struktur item mengikuti tabel di `schema.md`.

## Review (CRUD)

### GET /v1/epms/reviews — List guru + status review per campus

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `campusId` | int | no | Default dari req.user. Admin bisa kirim campus lain. |
| `ay` | int | no | Default AY aktif. |
| `name` | string | no | Filter nama guru |
| `status` | enum(DRAFT, SUBMITTED) | no | Filter status |
| `page`, `pageSize` | int | no | Pagination |

**Response 200:**
```json
{
  "data": [
    {
      "id": 1,
      "teacherId": 4137,
      "teacherName": "Devie Lana",
      "campusId": 4,
      "academicYearId": 27,
      "reviewerId": 21046,
      "reviewerName": "Herlina Susanti",
      "jobDesc": "Secondary English Teacher",
      "interschool": "PIK-S",
      "status": "SUBMITTED",
      "dateSubmit": "2026-08-20T09:00:00.000Z"
    }
  ],
  "count": 37,
  "meta": { "page": 1, "pageSize": 50, "itemCount": 37, "pageCount": 1, ... }
}
```

### GET /v1/epms/reviews/:id — Detail satu review

**Response 200:**
```json
{
  "data": {
    "id": 1,
    "teacherId": 4137,
    "teacherName": "Devie Lana",
    "campusId": 4,
    "academicYearId": 27,
    "reviewerId": 21046,
    "jobDesc": "Secondary English Teacher",
    "interschool": "PIK-S",
    "status": "DRAFT",
    "scores": [
      { "itemId": 1, "semester": 1, "score": 4 },
      { "itemId": 1, "semester": 2, "score": 4 },
      { "itemId": 2, "semester": 1, "score": 3 },
      ... semua item x 2 semester
    ],
    "trainingPlan": "Need CELTA certification...",
    "teacherCommentsSem1": "...",
    "reportingOfficerCommentsSem1": "...",
    "teacherCommentsSem2": "...",
    "reportingOfficerCommentsSem2": "..."
  }
}
```

### POST /v1/epms/reviews — Buat review baru (draft)

**Body (CreateReviewDto):**
```json
{
  "teacherId": 4137,
  "academicYearId": 27
}
```
`campusId` dan `reviewerId` diambil dari `req.user`. Validasi: teacher belum punya review aktif di AY ini.

**Response 201:** `{ "data": { "id": 2, "status": "DRAFT", ... } }`

### PUT /v1/epms/reviews/:id/scores — Simpan skor per kompetensi per semester (bulk)

**Body (UpdateReviewScoresDto):**
```json
{
  "scores": [
    { "itemId": 1, "semester": 1, "score": 4 },
    { "itemId": 1, "semester": 2, "score": 4 },
    { "itemId": 2, "semester": 1, "score": 3 },
    ... semua item x 2 semester
  ]
}
```
Validasi: score sesuai skala rubrik item (0-100 default), itemId harus valid, semester ∈ {1, 2}. Hanya bisa diubah jika status DRAFT.

**Response 200:** `{ "data": { "id": 1, "status": "DRAFT", ... } }`

### PUT /v1/epms/reviews/:id/comments — Simpan komentar + training plan

**Body (UpdateReviewCommentsDto):**
```json
{
  "trainingPlan": "Need CELTA certification...",
  "teacherCommentsSem1": "...",
  "reportingOfficerCommentsSem1": "...",
  "teacherCommentsSem2": "...",
  "reportingOfficerCommentsSem2": "..."
}
```
Validasi: max 1000 karakter per field (opsional).

**Response 200:** `{ "data": { "id": 1, ... } }`

### PUT /v1/epms/reviews/:id — Update header + submit

**Body (UpdateReviewDto):**
```json
{
  "jobDesc": "Secondary English Teacher",
  "interschool": "PIK-S",
  "status": "SUBMITTED"
}
```
Jika status=SUBMITTED, validasi minimal Section 1-7 terisi. Setelah submit, data tidak bisa diubah (kecuali admin dengan permission appraisal).

### DELETE /v1/epms/reviews/:id — Soft delete (hanya draft)

Hanya bisa hapus review dengan status DRAFT. Set `activeStatus = INACTIVE`.

## Referensi Implementasi

- Controller pattern: `src/modules/lesson/lesson.controller.ts`
- Entity pattern: `src/modules/lesson/entities/lesson.entity.ts`
- Pagination: `src/common/dto/page-meta.dto`
- Enums: `src/types/enums` — `ModulesTypeEnum.PETALS`, `StatusTypeEnum`
