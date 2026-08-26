# API Contract — PETALS Lesson Observation Input

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.PETALS`.
> **Base path:** global prefix `api` → URL lengkap `/api/v1/petals/observations`.
> **Dual portal:** Teacher Portal (scope `req.user.id`, Principal/HOD input per campus-nya) + Admin Portal (admin punya `PETALS_MANAGE` untuk input lintas campus).

## Konvensi Response

- Sukses (single): `{ "data": {...} }`
- Sukses (list): `{ "data": [...], "count": number, "meta": PageMetaDto }`
- Error: `HttpException` standar NestJS.

## Item Rubrik (static config)

18 item di-seed dari PETALS Form.pdf. Tidak ada endpoint CRUD untuk item — hanya dibaca.

### GET /v1/petals/observations/items — List semua item rubrik

**Response 200:**
```json
{
  "data": [
    { "id": 1, "dimension": "P", "label": "Teacher communicates learning objectives with the class.", "maxMark": 4, "sortOrder": 1 },
    { "id": 2, "dimension": "P", "label": "Teacher selects appropriate learning strategies, learning activities and resources.", "maxMark": 4, "sortOrder": 2 },
    { "id": 3, "dimension": "P", "label": "Teacher develops a workable/appropriate time schedule.", "maxMark": 4, "sortOrder": 3 },
    { "id": 4, "dimension": "E", "label": "Teacher uses appropriate resources (i.e. ICT, media, etc.) effectively...", "maxMark": 4, "sortOrder": 4 },
    ... 18 items total
  ]
}
```

## Observasi (CRUD)

### GET /v1/petals/observations — List observasi per campus

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `campusId` | int | no | Default dari req.user. Admin bisa kirim campus lain. |
| `ay` | int | no | Default AY aktif. |
| `teacherId` | int | no | Filter guru spesifik |
| `status` | enum(DRAFT, SUBMITTED) | no | Filter status |
| `page`, `pageSize` | int | no | Pagination |

**Response 200:**
```json
{
  "data": [
    {
      "id": 1,
      "teacherId": 4137,
      "teacherName": "Alex Norel Nalayog",
      "campusId": 4,
      "academicYearId": 27,
      "observerId": 21046,
      "observerName": "Herlina Susanti",
      "scoreP": 10, "scoreE": 11, "scoreT": 17, "scoreA": 20, "scoreL": 7,
      "averagePct": 85.53,
      "status": "SUBMITTED",
      "dateSubmit": "2026-08-20T09:00:00.000Z"
    }
  ],
  "count": 37,
  "meta": { "page": 1, "pageSize": 50, "itemCount": 37, "pageCount": 1, ... }
}
```

### GET /v1/petals/observations/:id — Detail satu observasi

**Response 200:**
```json
{
  "data": {
    "id": 1,
    "teacherId": 4137,
    "teacherName": "Alex Norel Nalayog",
    "campusId": 4,
    "academicYearId": 27,
    "observerId": 21046,
    "strength": "Clear explanation and good rapport with students.",
    "areasOfConcern": "Need more varied assessment strategies.",
    "status": "SUBMITTED",
    "dateSubmit": "2026-08-20T09:00:00.000Z",
    "scores": [
      { "itemId": 1, "dimension": "P", "mark": 3 },
      { "itemId": 2, "dimension": "P", "mark": 4 },
      ... 18 items
    ]
  }
}
```

### POST /v1/petals/observations — Buat observasi baru (draft)

**Body (CreateObservationDto):**
```json
{
  "teacherId": 4137,
  "academicYearId": 27
}
```
`campusId` dan `observerId` diambil dari `req.user`. Validasi: teacher belum punya observasi aktif di AY ini.

**Response 201:** `{ "data": { "id": 2, "status": "DRAFT", ... } }`

### PUT /v1/petals/observations/:id/items — Simpan mark per item (bulk)

**Body (UpdateObservationScoresDto):**
```json
{
  "scores": [
    { "itemId": 1, "mark": 3 },
    { "itemId": 2, "mark": 4 },
    ... 18 items
  ]
}
```
Validasi: mark 0-4, itemId harus valid. Hanya bisa diubah jika status DRAFT.

**Response 200:** `{ "data": { "id": 1, "scoreP": 10, ... } }` — mengembalikan skor terbaru.

### PUT /v1/petals/observations/:id — Update strength/areas_of_concern + submit

**Body (UpdateObservationDto):**
```json
{
  "strength": "Clear explanation...",
  "areasOfConcern": "Need more...",
  "status": "SUBMITTED"
}
```
Jika status=SUBMITTED, validasi minimal ada mark yang diisi. Setelah submit, data tidak bisa diubah (kecuali admin dengan `PETALS_MANAGE`).

### DELETE /v1/petals/observations/:id — Soft delete (hanya draft)

Hanya bisa hapus observasi dengan status DRAFT. Set `activeStatus = INACTIVE`.

## Referensi Implementasi

- Controller pattern: `src/modules/lesson/lesson.controller.ts`
- Entity pattern: `src/modules/lesson/entities/lesson.entity.ts`
- Pagination: `src/common/dto/page-meta.dto`
- Enums: `src/types/enums` — `ModulesTypeEnum.PETALS`, `StatusTypeEnum`