# API Contract — Form Leave (Teacher Leave)

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.TEACHER_LEAVE` (baru, perlu ditambahkan ke `src/types/enums` + ability di `src/modules/casl/casl-ability.factory.ts`).
> **Base path:** global prefix `api` → URL lengkap `/api/v1/teacher-leaves`.

## Konvensi Response

- Sukses (single): `{ "data": {...} }`
- Sukses (list): `{ "data": [...], "count": number, "meta": PageMetaDto }` — `PageMetaDto` dari `src/common/dto/page-meta.dto`; query filter extends `PageOptionsDto` (`page`, `pageSize`, `order`, `sortBy`, `relations`, `query`).
- Error: `HttpException` standar NestJS (`src/errors` / `src/filters`).

## Enum Leave Type (mapping dari legacy)

| Legacy `leavetype_id` | Enum value | Label (EN) | Label (CN) |
|----------------------|------------|------------|------------|
| 1 | `SICK_L` | Sick L | 病假 |
| 3 | `MATERNITY_L` | Maternity L | 产假 |
| 4 | `PATERNITY_L` | Paternity L | 陪产假 |
| 5 | `UNPAID_L` | Unpaid L | 无薪假 |
| 6 | `OTHER` | Other | 其他 |

## Endpoint Detail

### 1. GET /v1/teacher-leaves — List leave milik guru (paginated)

**Query params (GetTeacherLeavesDto):**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `year` | int | no | Filter berdasarkan tahun pengajuan (opsional) |
| `leaveType` | enum | no | Filter jenis cuti |
| `page` | int | no | Default 1 |
| `pageSize` | int | no | Default 10 (dari `PageOptionsDto`) |

**Response 200:**
```json
{
  "data": [
    {
      "id": 101,
      "dateFrom": "2026-09-01",
      "dateTo": "2026-09-02",
      "position": "Teacher",
      "department": "Science",
      "leaveType": "SICK_L",
      "reason": "Flu, need rest",
      "attachmentFileId": 5001,
      "attachment": { "id": 5001, "url": "/api/v1/files/download?id=5001" },
      "teacherId": 21046,
      "teacherName": "Herlina Susanti",
      "campusId": 4,
      "activeStatus": "ACTIVE",
      "createdAt": "2026-08-26T10:00:00.000Z",
      "updatedAt": "2026-08-26T10:00:00.000Z"
    }
  ],
  "count": 3,
  "meta": { "page": 1, "pageSize": 10, "itemCount": 3, "pageCount": 1, "hasPreviousPage": false, "hasNextPage": false }
}
```

### 2. POST /v1/teacher-leaves — Buat pengajuan leave

**Body (CreateTeacherLeaveDto):**

```json
{
  "dateFrom": "2026-09-01",
  "dateTo": "2026-09-02",
  "position": "Teacher",
  "department": "Science",
  "leaveType": "SICK_L",
  "reason": "Flu, need rest",
  "attachmentFileId": 5001
}
```

**Catatan:** `teacherId` dan `campusId` TIDAK boleh ada di body — keduanya diambil dari `req.user` (id + campus teacher). Validasi: `dateTo >= dateFrom`; `leaveType` ∈ enum; `reason` non-empty; `attachmentFileId` (jika ada) harus milik file bertipe PDF yang valid.

**Response 201:** `{ "data": { "id": 101, ... } }` (TeacherLeaveDto lengkap).

### 3. GET /v1/teacher-leaves/:id — Detail leave

**Response 200:**
```json
{
  "data": {
    "id": 101,
    "dateFrom": "2026-09-01",
    "dateTo": "2026-09-02",
    "position": "Teacher",
    "department": "Science",
    "leaveType": "SICK_L",
    "reason": "Flu, need rest",
    "attachmentFileId": 5001,
    "attachment": { "id": 5001, "url": "/api/v1/files/download?id=5001" },
    "teacherId": 21046,
    "teacherName": "Herlina Susanti",
    "campusId": 4,
    "activeStatus": "ACTIVE",
    "createdAt": "2026-08-26T10:00:00.000Z",
    "updatedAt": "2026-08-26T10:00:00.000Z"
  }
}
```

**Permission:** hanya owner (`req.user.id == teacher_leave.teacherId`); selain itu → 403.

### 4. PATCH /v1/teacher-leaves/:id — Update (extensibility)

**Body:** subset opsional `{ dateFrom?, dateTo?, position?, department?, leaveType?, reason?, attachmentFileId?, activeStatus? }` (PartialType dari CreateTeacherLeaveDto). Hanya owner.

### 5. DELETE /v1/teacher-leaves/:id — Soft delete

Hanya owner. Set `activeStatus = INACTIVE`. Response 200 `{ "data": { "id": 101, "activeStatus": "INACTIVE" } }`.

### 6. POST /v1/files — Upload PDF attachment (modul file existing)

Mengikuti pola `FileController` (`src/modules/file/file.controller.ts:67`):

- Method: `POST /api/v1/files` — `@Public()`, `FileInterceptor('file')`, `ParseFilePipe` dengan `MaxFileSizeValidator` (100MB) + `FileTypeValidator`.
- Untuk membatasi PDF-only, gunakan endpoint khusus atau set `entityType` + validasi MIME `application/pdf` di service.
- **Response 201:** `{ "data": { "id": 5001, "url": "/api/v1/files/download?id=5001", ... } }` — `id` inilah yang disimpan sebagai `attachmentFileId`.

## Referensi Implementasi (contoh pola dari modul existing)

- Controller pattern: `src/modules/lesson/lesson.controller.ts` — `@Controller({ version: '1', path: 'lessons' })`, `@CheckPermissions`, `@Req() req: Request`, `ParseIntPipe`, `{ data }` wrapper.
- Entity pattern: `src/modules/lesson/entities/lesson.entity.ts` — `BaseEntityWithDates`, `@Index()` FK, `@ManyToOne` + `@JoinColumn`, `Relation<T>`, `StatusTypeEnum`.
- File upload: `src/modules/file/file.controller.ts` — `FileInterceptor('file')` + `ParseFilePipe` + `MIME_TYPES` (`src/types/constants/mime-type`).
- Pagination: `src/common/dto/page-meta.dto` (`PageMetaDto`).
- Enums: `src/types/enums` — `ACLTypeEnum`, `ModulesTypeEnum` (tambah `TEACHER_LEAVE`), `StatusTypeEnum`, `FileEntityTypeEnum` (`ATTACHMENT_FILE` sudah ada di `file-entity-type.ts:12`).
