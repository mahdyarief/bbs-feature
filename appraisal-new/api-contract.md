# API Contract — New Appraisal / Staff Database

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.APPRAISAL` (atau enum appraisal setara — lihat `edgecases.md`).
> **Base path:** global prefix `api` → URL lengkap `/api/v1/appraisal/reviews`.
> **Dual portal:** Teacher Portal (scope `req.user` campusId, Principal/HOD melihat data per campus-nya) + Admin Portal (admin punya permission appraisal untuk melihat lintas campus).

## Konvensi Response

- Sukses (single): `{ "data": {...} }`
- Sukses (list): `{ "data": [...], "count": number, "meta": PageMetaDto }`
- Error: `HttpException` standar NestJS.

## Konfigurasi Dimensi (static config)

Dimensi penilaian (18 item PETALS / appraisal) di-seed dan dimiliki oleh `features/petals-observation/` — brief ini TIDAK membuat CRUD dimensi, hanya membaca untuk agregasi score.

## List Guru + Status Appraisal

### GET /v1/appraisal/reviews — List guru + status appraisal per campus

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `campusId` | int | no | Default dari req.user. Admin bisa kirim campus lain. Filter "All Campus" = tanpa parameter ini. |
| `ay` | int | no | Default AY aktif. |
| `name` | string | no | Filter nama guru (text input) |
| `active` | enum(YES, NO) | no | Filter status aktif guru |
| `type` | enum(TEACHER, HOD, PRINCIPAL) | no | Varian view — menentukan komponen score yang ditampilkan |
| `page`, `pageSize` | int | no | Pagination |

**Response 200 (Teacher view / `type=TEACHER`):**
```json
{
  "data": [
    {
      "userId": "BBS000123",
      "teacherId": 4137,
      "teacherName": "Devie Lana",
      "campusId": 4,
      "campusName": "PIK-S",
      "active": "YES",
      "appraisalStatus": "COMPLETED",
      "score": 95.520833333333,
      "grade": "A",
      "dateSubmit": "2026-04-14 08:30:02",
      "hasPdfReport": true
    },
    {
      "userId": "BBS000456",
      "teacherId": 4138,
      "teacherName": "Rina Kartika",
      "campusId": 4,
      "campusName": "PIK-S",
      "active": "YES",
      "appraisalStatus": "INCOMPLETE",
      "score": 0,
      "grade": null,
      "dateSubmit": null,
      "hasPdfReport": false
    }
  ],
  "count": 640,
  "meta": { "page": 1, "pageSize": 50, "itemCount": 640, "pageCount": 13, ... }
}
```

**Response 200 (HOD view / `type=HOD`):**
```json
{
  "data": [
    {
      "userId": "BBS000123",
      "teacherId": 4137,
      "teacherName": "Devie Lana",
      "campusId": 4,
      "campusName": "PIK-S",
      "teacherScore": 91.979,
      "teacherGrade": "A",
      "hodScore": 90.625,
      "hodGrade": "A",
      "combinedScore": 91.708,
      "combinedGrade": "A",
      "dateSubmit": "2026-04-14 08:30:02"
    }
  ],
  "count": 69,
  "meta": { "page": 1, "pageSize": 50, "itemCount": 69, "pageCount": 2, ... }
}
```

**Response 200 (Principal view / `type=PRINCIPAL`):**
```json
{
  "data": [
    {
      "userId": "BBS000123",
      "teacherId": 4137,
      "teacherName": "Devie Lana",
      "campusId": 4,
      "campusName": "PIK-S",
      "appraisalStatus": "COMPLETED",
      "score": 95.520833333333,
      "grade": "A",
      "neverAppraised": false,
      "dateSubmit": "2026-04-14 08:30:02"
    },
    {
      "userId": "BBS000789",
      "teacherId": 4201,
      "teacherName": "Budi Santoso",
      "campusId": 4,
      "campusName": "PIK-S",
      "appraisalStatus": "INCOMPLETE",
      "score": 0,
      "grade": null,
      "neverAppraised": true,
      "dateSubmit": null
    }
  ],
  "count": 15,
  "meta": { "page": 1, "pageSize": 50, "itemCount": 15, "pageCount": 1, ... }
}
```

**Catatan tampilan:**
- Teacher view: `score` + `grade` ditampilkan sebagai `"95.520833333333(A)"` — jika `score = 0` tampil `"0()"`.
- HOD view: `combinedScore = teacherScore * 0.8 + hodScore * 0.2`.
- Principal view: jika `neverAppraised = true`, kolom score menampilkan `"Never!"`.

### GET /v1/appraisal/reviews/:teacherId — Detail appraisal untuk satu guru

**Response 200:**
```json
{
  "data": {
    "userId": "BBS000123",
    "teacherId": 4137,
    "teacherName": "Devie Lana",
    "campusId": 4,
    "campusName": "PIK-S",
    "academicYearId": 27,
    "reviewerId": 21046,
    "reviewerName": "Herlina Susanti",
    "appraisalType": "TEACHER",
    "status": "COMPLETED",
    "score": 95.520833333333,
    "grade": "A",
    "dateSubmit": "2026-04-14 08:30:02",
    "dimensions": [
      { "dimensionId": 1, "dimensionName": "P", "label": "Pedagogy", "score": 4 },
      { "dimensionId": 2, "dimensionName": "E", "label": "Engagement", "score": 4 },
      ... 18 dimensi PETALS
    ]
  }
}
```

Jika guru belum pernah di-appraise, return 404 dengan `code: "NO_APPRAISAL"` (lihat Error Handling).

### GET /v1/appraisal/reviews/:teacherId/score — Score & grade breakdown

Digunakan untuk HOD view — mengembalikan komponen teacher dan HOD secara terpisah.

**Query params:** `type` (enum TEACHER/HOD/PRINCIPAL) — default TEACHER.

**Response 200 (type=HOD):**
```json
{
  "data": {
    "teacherId": 4137,
    "teacherScore": 91.979,
    "teacherGrade": "A",
    "hodScore": 90.625,
    "hodGrade": "A",
    "combinedScore": 91.708,
    "combinedGrade": "A"
  }
}
```

### PUT /v1/appraisal/reviews/:teacherId/status — Update status appraisal

Menandai appraisal sebagai COMPLETED (submit) atau kembali INCOMPLETE (buka).

**Body (UpdateAppraisalStatusDto):**
```json
{
  "status": "COMPLETED"
}
```

Validasi: hanya bisa dipanggil oleh reviewer berhak (Principal/HOD untuk guru di campus-nya) atau admin. Saat status = COMPLETED, `date_submit` diisi otomatis.

**Response 200:** `{ "data": { "teacherId": 4137, "status": "COMPLETED", "dateSubmit": "2026-04-14 08:30:02" } }`

## PDF Report & Blank Form

### GET /v1/appraisal/reviews/:teacherId/report — Download PDF Report

Hanya tersedia jika status = `COMPLETED`. Mengembalikan file PDF (Content-Type `application/pdf`).

**Error:** 400 "Appraisal is not completed yet" jika status bukan COMPLETED.

### GET /v1/appraisal/reviews/:teacherId/blank-form — Download Blank Form

Selalu tersedia — form appraisal kosong (PDF) untuk diisi manual/print.

## Referensi Implementasi

- Controller pattern: `src/modules/lesson/lesson.controller.ts`
- Entity pattern: `src/modules/lesson/entities/lesson.entity.ts`
- Pagination: `src/common/dto/page-meta.dto`
- Enums: `src/types/enums` — `ModulesTypeEnum.APPRAISAL`, `StatusTypeEnum`