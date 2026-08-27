# API Contract — Training Staff

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.EMPLOYEE` (atau enum modul Staff & HR setara, mis. `TRAINING_STAFF` — lihat `edgecases.md`).
> **Base path:** global prefix `api` → URL lengkap `/api/v1/training-staff`.
> **Dual portal:** Admin Portal (Admin/ASD, lintas campus) + Teacher Portal (Principal scope `req.user.campusId`; guru biasa akses riwayat sendiri).

## Konvensi Response

- Sukses (single): `{ "data": {...} }`
- Sukses (create): `{ "data": {...} }` — HTTP 201
- Sukses (list): `{ "data": [...], "count": number, "meta": PageMetaDto }`
- Error: `HttpException` standar NestJS.

## Pemetaan Field Legacy → API

| Field form legacy | ID HTML legacy | API Property (DTO) | Tipe API | Keterangan |
|-------------------|----------------|--------------------|----------|------------|
| Judul pelatihan | `#title` | `title` | string | wajib |
| Bentuk partisipasi | `#participation` | `participation` | string | opsional |
| Tanggal mulai | `#datefrom` | `dateFrom` | date (`yyyy-mm-dd`) | wajib, datepicker `datepickerformat2` |
| Tanggal selesai | `#dateto` | `dateTo` | date (`yyyy-mm-dd`) | wajib, `dateFrom <= dateTo` |
| Tempat | `#venue` | `venue` | string | opsional |
| Penyelenggara | `#conducted_by` | `conductedBy` | string | opsional |
| Kota | `#city` | `city` | string | opsional |
| Id negara | `#country_id` | `countryId` | int (FK → country.id) | opsional, default jika kosong |
| Rincian | `#details` | `details` | text | opsional |
| Komentar | `#comments` | `comments` | text | opsional |
| Id staff (multi) | `#form-field-tags` | `staffIds[]` | int[] (FK → employee.id) | minimal 1 |
| Record id | attribute tombol `recid` | `id` (path param) | int | 0 = create, n = update |

## Endpoints

### GET /v1/training-staff — List pelatihan

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `campusId` | int | no | Default dari req.user. Admin bisa kirim campus lain. 0 = All Campus. |
| `name` | string | no | Filter judul/title pelatihan |
| `page`, `pageSize` | int | no | Pagination |

**Response 200:**
```json
{
  "data": [
    {
      "id": 101,
      "title": "Workshop Kurikulum Merdeka",
      "participation": "Peserta",
      "dateFrom": "2026-07-10",
      "dateTo": "2026-07-12",
      "venue": "Hotel Grand Tropic",
      "conductedBy": "Kemendikbud",
      "city": "Jakarta",
      "countryId": 102,
      "countryName": "Indonesia",
      "details": "Pelatihan implementasi kurikulum merdeka untuk guru SD.",
      "comments": "Sertifikat diterima via email.",
      "staff": [
        { "id": 4137, "name": "Devie Lana" },
        { "id": 4138, "name": "Budi Santoso" }
      ]
    }
  ],
  "count": 1,
  "meta": { "page": 1, "pageSize": 50, "itemCount": 1, "pageCount": 1, ... }
}
```

Catatan: filter `campusId` diterapkan berdasarkan campus staff ter-assign (derived) atau kolom `campus_id` (lihat `schema.md` — keputusan campus_id). `countryName` berasal dari join `country`.

### GET /v1/training-staff/:id — Detail pelatihan + staff ter-assign

**Path param:** `id` (int) — id record pelatihan.

**Response 200:**
```json
{
  "data": {
    "id": 101,
    "title": "Workshop Kurikulum Merdeka",
    "participation": "Peserta",
    "dateFrom": "2026-07-10",
    "dateTo": "2026-07-12",
    "venue": "Hotel Grand Tropic",
    "conductedBy": "Kemendikbud",
    "city": "Jakarta",
    "countryId": 102,
    "countryName": "Indonesia",
    "details": "Pelatihan implementasi kurikulum merdeka untuk guru SD.",
    "comments": "Sertifikat diterima via email.",
    "staff": [
      { "id": 4137, "name": "Devie Lana", "campusId": 4, "nip": "..." },
      { "id": 4138, "name": "Budi Santoso", "campusId": 4, "nip": "..." }
    ],
    "activeStatus": "ACTIVE",
    "createdAt": "2026-07-01T02:00:00.000Z",
    "updatedAt": "2026-07-01T02:00:00.000Z"
  }
}
```

Jika record tidak ada atau `active_status = INACTIVE` → **404 "Training record not found"**.

### POST /v1/training-staff — Create pelatihan baru

**Body (CreateTrainingStaffDto):**
```json
{
  "title": "Workshop Kurikulum Merdeka",
  "participation": "Peserta",
  "dateFrom": "2026-07-10",
  "dateTo": "2026-07-12",
  "venue": "Hotel Grand Tropic",
  "conductedBy": "Kemendikbud",
  "city": "Jakarta",
  "countryId": 102,
  "details": "Pelatihan implementasi kurikulum merdeka untuk guru SD.",
  "comments": "Sertifikat diterima via email.",
  "staffIds": [4137, 4138]
}
```

Validasi:
- `title` wajib (string, non-empty) → 400 "Validation failed: title is required".
- `dateFrom` dan `dateTo` wajib, format `yyyy-mm-dd`; `dateFrom <= dateTo` → 400 "Date from must be before or equal to date to".
- `staffIds` wajib, minimal 1 elemen → 400 "At least one staff must be assigned".
- `countryId` opsional; jika kosong → default (mis. `country_id` default sistem); jika terisi harus valid → 400 "Country not found".
- Semua `staffIds` harus employee aktif → 404/400 "Staff not found" / "Staff is not active".
- Duplicate check: kombinasi title + date range yang sama → 409 (lihat EC-05).
- Insert ke `staff_training` + insert relasi `staff_training_staff`.

**Response 201:**
```json
{
  "data": {
    "id": 101,
    "title": "Workshop Kurikulum Merdeka",
    "participation": "Peserta",
    "dateFrom": "2026-07-10",
    "dateTo": "2026-07-12",
    "venue": "Hotel Grand Tropic",
    "conductedBy": "Kemendikbud",
    "city": "Jakarta",
    "countryId": 102,
    "countryName": "Indonesia",
    "details": "Pelatihan implementasi kurikulum merdeka untuk guru SD.",
    "comments": "Sertifikat diterima via email.",
    "staff": [
      { "id": 4137, "name": "Devie Lana" },
      { "id": 4138, "name": "Budi Santoso" }
    ],
    "activeStatus": "ACTIVE"
  }
}
```

### PATCH /v1/training-staff/:id — Update pelatihan (partial)

**Body (UpdateTrainingStaffDto):** sama dengan Create, semua field opsional (partial). Jika `staffIds` disertakan → **upsert assignment**: hapus relasi lama di `staff_training_staff`, insert relasi baru. Endpoint yang sama dipakai untuk **inline update** satu field (mekanisme legacy `update_subjectais.php`) — hanya field yang dikirim yang di-update.

**Body contoh (full update):**
```json
{
  "title": "Workshop Kurikulum Merdeka 2026",
  "dateFrom": "2026-07-10",
  "dateTo": "2026-07-13",
  "staffIds": [4137]
}
```

**Response 200:**
```json
{
  "data": {
    "id": 101,
    "title": "Workshop Kurikulum Merdeka 2026",
    "participation": "Peserta",
    "dateFrom": "2026-07-10",
    "dateTo": "2026-07-13",
    "venue": "Hotel Grand Tropic",
    "conductedBy": "Kemendikbud",
    "city": "Jakarta",
    "countryId": 102,
    "countryName": "Indonesia",
    "details": "Pelatihan implementasi kurikulum merdeka untuk guru SD.",
    "comments": "Sertifikat diterima via email.",
    "staff": [
      { "id": 4137, "name": "Devie Lana" }
    ],
    "activeStatus": "ACTIVE"
  }
}
```

Validasi sama dengan create (field yang dikirim saja). Record tidak ada / INACTIVE → **404**.

### DELETE /v1/training-staff/:id — Soft delete

Soft delete via `active_status = INACTIVE`; baris relasi `staff_training_staff` tetap (histori assignment tidak dihapus — penting untuk CV guru).

**Response 200:**
```json
{
  "data": {
    "id": 101,
    "activeStatus": "INACTIVE",
    "deletedAt": "2026-08-27T03:00:00.000Z"
  }
}
```

### GET /v1/countries — Dropdown `country_id`

**Catatan:** endpoint ini **existing atau to-create** — perlu dicek apakah modul `country` sudah ada di `api_nest`. Jika belum, dibuat sebagai modul sederhana (entity `Country` + seeder data negara). Dipakai untuk dropdown `country_id` pada form.

**Response 200:**
```json
{
  "data": [
    { "id": 102, "name": "Indonesia" },
    { "id": 100, "name": "Singapore" },
    { "id": 76, "name": "Malaysia" }
  ],
  "count": 3,
  "meta": { "page": 1, "pageSize": 50, "itemCount": 3, "pageCount": 1, ... }
}
```

## Referensi Implementasi

- Controller pattern: `src/modules/lesson/lesson.controller.ts`
- Entity pattern: `src/modules/employee/entities/employee.entity.ts`
- Pagination: `src/common/dto/page-meta.dto`
- Enums: `src/types/enums` — `ModulesTypeEnum`, `StatusTypeEnum`
- Soft delete: `active_status` enum(ACTIVE, INACTIVE) sesuai konvensi modul lain.
