# API Contract — Teacher Card (Kartu Guru)

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions`.
> **Base path:** global prefix `api` → URL lengkap `/api/v1/teacher-card` atau `/api/v1/employees/:id/card`.
> **Dual portal:** Admin Portal (Admin/ASD, lintas campus) + Teacher Portal (Teacher self-view; Principal scope `req.user.campusId`).

## Konvensi Response

- Sukses (single): `{ "data": {...} }`
- Error: `HttpException` standar NestJS.

## Catatan: Dua Opsi Endpoint

Fitur ini dapat diimplementasikan melalui **salah satu** dari dua pendekatan berikut:

| Opsi | Endpoint | Keuntungan |
|------|----------|------------|
| **A (endpoint baru)** | `GET /api/v1/teacher-card?employeeId=&campusId=` | Dedicated, tidak mengganggu endpoint employee existing. |
| **B (reuse + enrich)** | `GET /api/v1/employees/:id` dengan query param `?include=card` | Reuse endpoint yang sudah ada; tidak perlu controller baru. |

Dokumen ini mendokumentasikan **Opsi A** (endpoint baru) sebagai proposal utama. Jika memilih Opsi B, cukup tambahkan join ke `employee-identity` dan `employee-position` di service employee yang sudah ada.

---

## GET /api/v1/teacher-card — Ambil data kartu guru

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `employeeId` | int | yes | ID employee/guru yang akan ditampilkan kartunya. |
| `campusId` | int | no | Digunakan untuk validasi akses (Principal di teacher portal). Default dari `req.user.campusId`. |

### Response 200 (sukses)

```json
{
  "data": {
    "employeeId": 4138,
    "name": "Budi Santoso",
    "nip": "198503142010011012",
    "nik": "3174011403850003",
    "gender": "L",
    "birthPlace": "Jakarta",
    "birthDate": "1985-03-14",
    "photoUrl": "https://storage.bbs.sch.id/photos/employee/4138.jpg",
    "position": "Guru Matematika",
    "positionType": "PNS",
    "campusId": 4,
    "campusName": "PIK-S",
    "email": "budi.santoso@binabangsaschool.com",
    "phone": "081234567890",
    "employeeStatus": "ACTIVE",
    "joinDate": "2010-07-01",
    "cardVersion": "1.0"
  }
}
```

### Response 200 — foto tidak tersedia (placeholder)

```json
{
  "data": {
    "employeeId": 4138,
    "name": "Budi Santoso",
    "nip": "198503142010011012",
    "nik": "3174011403850003",
    "gender": "L",
    "birthPlace": "Jakarta",
    "birthDate": "1985-03-14",
    "photoUrl": null,
    "position": "Guru Matematika",
    "positionType": "PNS",
    "campusId": 4,
    "campusName": "PIK-S",
    "email": "budi.santoso@binabangsaschool.com",
    "phone": "081234567890",
    "employeeStatus": "ACTIVE",
    "joinDate": "2010-07-01",
    "cardVersion": "1.0"
  }
}
```

> **Catatan:** `photoUrl = null` menandakan foto tidak tersedia. Frontend bertanggung jawab menampilkan placeholder (inisial nama atau icon default).

### Response 404 — staff tidak ditemukan

```json
{
  "statusCode": 404,
  "message": "Staff not found",
  "error": "Not Found"
}
```

### Response 403 — tidak punya akses

```json
{
  "statusCode": 403,
  "message": "You don't have permission to view this teacher card",
  "error": "Forbidden"
}
```

### Response 400 — campus mismatch

```json
{
  "statusCode": 400,
  "message": "Staff is not in your campus",
  "error": "Bad Request"
}
```

## Field Descriptions

| Field | Tipe | Sumber Data | Estimasi / Dipastikan | Keterangan |
|-------|------|-------------|-----------------------|------------|
| `employeeId` | int | `employee.id` | Dipastikan | Primary key employee |
| `name` | string | `employee.name` | Dipastikan | Nama lengkap |
| `nip` | string | `employee-identity.nip` | **Estimasi** | NIP (Nomor Induk Pegawai) — field dari employee-identity |
| `nik` | string | `employee-identity.nik` | **Estimasi** | NIK (Nomor Induk Kependudukan) — field dari employee-identity |
| `gender` | string | `employee.gender` | Dipastikan | L / P |
| `birthPlace` | string | `employee.birth_place` | **Estimasi** | Tempat lahir |
| `birthDate` | date | `employee.birth_date` | Dipastikan | Tanggal lahir (ISO 8601) |
| `photoUrl` | string/null | `employee-identity.photo_url` | **Estimasi** | URL foto profil; null jika tidak ada |
| `position` | string | `employee-position.position_name` | **Estimasi** | Nama jabatan/posisi |
| `positionType` | string | `employee-position.position_type` | **Estimasi** | Tipe posisi (PNS, Honorer, Kontrak, dll.) |
| `campusId` | int | `employee.campus_id` | Dipastikan | ID campus penempatan |
| `campusName` | string | `campus.name` | Dipastikan | Nama campus |
| `email` | string | `employee.email` | Dipastikan | Email staff |
| `phone` | string | `employee.phone` | **Estimasi** | Nomor telepon |
| `employeeStatus` | string | `employee.active_status` | Dipastikan | ACTIVE / INACTIVE / RESIGNED |
| `joinDate` | date | `employee.join_date` | **Estimasi** | Tanggal bergabung |
| `cardVersion` | string | — | **Estimasi** | Versi layout kartu untuk caching/format |

> **Semua field bertanda "Estimasi" perlu diverifikasi dengan probe ke legacy `recruitment_new/slick_app.php` atau dengan membaca entity definitions di `api_nest`.**

## Referensi Implementasi

- Controller pattern: `src/modules/employee/employee.controller.ts` (atau controller baru `src/modules/teacher-card/teacher-card.controller.ts` jika Opsi A)
- Entity pattern: `src/modules/employee/entities/employee.entity.ts`, `src/modules/employee-identity/entities/employee-identity.entity.ts`, `src/modules/employee-position/entities/employee-position.entity.ts`
- Pagination: `src/common/dto/page-meta.dto` (jika endpoint list diperlukan)
- Enums: `src/types/enums` — `ModulesTypeEnum`, `StatusTypeEnum`
- Storage: `src/modules/storage/` — untuk foto/upload URL generation