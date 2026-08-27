# API Contract — Appraisal Summary Report

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.PETALS` (atau enum appraisal setara — lihat `edgecases.md`).
> **Base path:** global prefix `api` → URL lengkap `/api/v1/appraisal/summary`.
> **Dual portal:** Teacher Portal (scope `req.user.id`, Principal/HOD melihat per campus-nya) + Admin Portal (admin punya permission appraisal untuk akses lintas campus).
> **Bersifat read-only:** fitur ini hanya membaca & mengagregasi skor yang di-input di gateway (`features/appraisal-new/`). Tidak ada endpoint tulis.

## Konvensi Response

- Sukses (single): `{ "data": {...} }`
- Sukses (list): `{ "data": [...], "count": number, "meta": PageMetaDto }`
- Error: `HttpException` standar NestJS.

## GET /v1/appraisal/summary — Ringkasan skor appraisal per campus

Mengagregasi skor appraisal dari tabel skor (lihat `schema.md`) menjadi satu baris per guru, dengan kolom per dimensi (17 untuk TEACHER, 9 untuk HOD) + kolom `total`.

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `campusId` | int | no | Default dari req.user (teacher portal). Admin bisa kirim campus lain. |
| `academicYearId` | int | no | Default AY aktif. |
| `type` | enum(TEACHER, HOD) | no | Default `TEACHER`. `HOD` memakai 9 dimensi skala 0-5. |
| `page`, `pageSize` | int | no | Pagination |

**Response 200 (type=TEACHER):**
```json
{
  "data": [
    {
      "employeeId": 5439,
      "userId": 5439,
      "name": "Muhammad Affan",
      "campusId": 4,
      "academicYearId": 27,
      "dimensions": [
        { "key": "professionalKnowledge", "label": "Professional Knowledge and Practice", "score": 12.5 },
        { "key": "deliveryOfLessons", "label": "Delivery of lessons", "score": 11 },
        { "key": "classroomManagement", "label": "Classroom management", "score": 5 },
        { "key": "preparation", "label": "Preparation", "score": 3.5 },
        { "key": "assessment", "label": "Assessment", "score": 4.5 },
        { "key": "coCurricularActivities", "label": "Co-curricular activities", "score": 5 },
        { "key": "leadership", "label": "Leadership ,contribution to school and community", "score": 3.5 },
        { "key": "professionalLearning", "label": "Professional Learning", "score": 3 },
        { "key": "monitoring", "label": "Monitoring", "score": 3 },
        { "key": "motivationalSkills", "label": "Motivational skills", "score": 1.5 },
        { "key": "conduct", "label": "Conduct", "score": 3.5 },
        { "key": "professionalism", "label": "Professionalism", "score": 3 },
        { "key": "responsibility", "label": "Responsibility", "score": 3 },
        { "key": "workAttitude", "label": "Work attitude", "score": 2.5 },
        { "key": "initiative", "label": "Initiative", "score": 3.5 },
        { "key": "adaptability", "label": "Adaptability to change", "score": 4 },
        { "key": "interpersonalRelationships", "label": "Interpersonal relationships", "score": 4 }
      ],
      "total": 76,
      "grade": null
    }
  ],
  "count": 45,
  "meta": { "page": 1, "pageSize": 50, "itemCount": 45, "pageCount": 1, ... }
}
```

**Response 200 (type=HOD):**
```json
{
  "data": [
    {
      "employeeId": 602,
      "userId": 602,
      "name": "Chandi Wijaya",
      "campusId": 4,
      "academicYearId": 27,
      "dimensions": [
        { "key": "leadershipVision", "label": "Leadership/Vision", "score": 4.6 },
        { "key": "strategicPlanning", "label": "Strategic Planning & Administration", "score": 5 },
        { "key": "developmentStaff", "label": "Development & Management of Staff", "score": 4.71 },
        { "key": "professionalDevelopment", "label": "Professional Development", "score": 5 },
        { "key": "managementProcesses", "label": "Management of Processes", "score": 4.5 },
        { "key": "managementResources", "label": "Management of Resources", "score": 4.25 },
        { "key": "professionalKnowledge", "label": "Professional Knowledge", "score": 4.75 },
        { "key": "professionalPractice", "label": "Professional Practice", "score": 4.38 },
        { "key": "professionalEngagement", "label": "Professional Engagement", "score": 4.25 }
      ],
      "total": 4.61,
      "grade": null
    }
  ],
  "count": 4,
  "meta": { "page": 1, "pageSize": 50, "itemCount": 4, "pageCount": 1, ... }
}
```

> **Catatan skor:** `total` teacher = sum 17 dimensi (skala maks per dimensi berbeda, lihat `schema.md`). `total` HOD = rata-rata 9 dimensi (skala 0-5). Field `dimensions` menggunakan struktur array key-value agar bisa menampung 17 atau 9 kolom dinamis sesuai `type`.

## GET /v1/appraisal/summary/export — Export summary ke file

Menghasilkan file Excel (.xls) berisi header kolom + semua baris dari data yang sama dengan `/summary`.

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `format` | enum(excel) | no | Default `excel`. Hanya `excel` yang didukung. |
| `campusId` | int | no | Default dari req.user. |
| `academicYearId` | int | no | Default AY aktif. |
| `type` | enum(TEACHER, HOD) | no | Default `TEACHER`. |

**Response 200:** file binary (Content-Type: `application/vnd.ms-excel`, `Content-Disposition: attachment; filename="appraisal_summary.xls"`).

> Referensi legacy: `print_appraisal_summary.php` (teacher) & `print_appraisal_summary_hod.php` (HOD) — link "Export to Excel File".

## GET /v1/appraisal/summary/print — Print view summary

Mengembalikan data untuk halaman print-friendly (menggunakan data yang sama dengan `/summary`).

**Query params:** sama dengan `/summary` (`campusId`, `academicYearId`, `type`).

**Response 200:** sama dengan `/summary` — frontend menampilkan print view dan memanggil `window.print()` (mirror `print_div()` legacy yang menyembunyikan elemen `#print` lalu `window.print(); self.close();`).

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| Academic year not found / inactive | 400 | "Academic year not found or inactive" |
| Campus not found | 404 | "Campus not found" |
| Invalid type | 400 | "Type must be TEACHER or HOD" |
| Unauthorized (role tanpa permission) | 403 | "You don't have permission to access appraisal summary" |
| No data for campus+AY | 200 + empty array | `{ "data": [], "count": 0 }` |
| Export format unsupported | 400 | "Format must be excel" |

## Referensi Implementasi

- Controller pattern: `src/modules/lesson/lesson.controller.ts`
- Export pattern: lihat `features/petals/` (export `print_petals_summary.php`) & keputusan D-03 (library `exceljs`) di `features/petals/notes.md`.
- Pagination: `src/common/dto/page-meta.dto`
- Enums: `src/types/enums` — `ModulesTypeEnum.PETALS`, `StatusTypeEnum`
