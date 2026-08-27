# API Contract — Appraisal Lock/Unlock

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.PETALS` (atau enum appraisal setara, mis. `APPRAISAL_LOCK` — lihat `edgecases.md`).
> **Base path:** global prefix `api` → URL lengkap `/api/v1/appraisal/locks`.
> **Dual portal:** Admin Portal (Admin/ASD, lintas campus) + Teacher Portal (Principal scope `req.user.campusId`; guru biasa read-only).

## Konvensi Response

- Sukses (single): `{ "data": {...} }`
- Sukses (list): `{ "data": [...], "count": number, "meta": PageMetaDto }`
- Error: `HttpException` standar NestJS.

## Enum Tab

| Value | Keterangan |
|-------|-----------|
| `ACADEMIC` | Skor appraisal akademik — tab Academic di `view_lock.php` |
| `CCA` | Skor Co-Curricular Activities — `view_lock_cca.php` |
| `REMARKS` | Remarks/komentar appraisal — `view_lock_remarks.php` |
| `PRELIM` | Prelim (penilaian awal) — tab prelim |
| `APPRAISAL` | Menyeluruh — mengunci semua tab sekaligus (lihat EC-AL-03) |

## Lock Status (list + set)

### GET /v1/appraisal/locks — List guru + status lock per tab

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `academicYearId` | int | no | Default AY aktif. |
| `campusId` | int | no | Default dari req.user. Admin bisa kirim campus lain. |
| `tab` | enum(ACADEMIC, CCA, REMARKS, PRELIM, APPRAISAL) | no | Default `ACADEMIC`. |
| `name` | string | no | Filter nama guru |
| `page`, `pageSize` | int | no | Pagination |

**Response 200:**
```json
{
  "data": [
    {
      "teacherId": 4137,
      "teacherName": "Devie Lana",
      "campusId": 4,
      "academicYearId": 27,
      "tab": "ACADEMIC",
      "isLocked": true,
      "lockedBy": 21046,
      "lockedByName": "Herlina Susanti",
      "lockedAt": "2026-08-20T09:00:00.000Z",
      "unlockedBy": null,
      "unlockedAt": null
    },
    {
      "teacherId": 4138,
      "teacherName": "Budi Santoso",
      "campusId": 4,
      "academicYearId": 27,
      "tab": "ACADEMIC",
      "isLocked": false,
      "lockedBy": null,
      "lockedByName": null,
      "lockedAt": null,
      "unlockedBy": null,
      "unlockedAt": null
    }
  ],
  "count": 2,
  "meta": { "page": 1, "pageSize": 50, "itemCount": 2, "pageCount": 1, ... }
}
```

Catatan: guru tanpa baris di `appraisal_lock` tetap muncul dengan `isLocked = false` (default UNLOCKED) — list dibangun dari daftar guru (employee) untuk campus + AY terpilih, di-LEFT JOIN ke `appraisal_lock`.

### PUT /v1/appraisal/locks/:teacherId — Set lock/unlock per tab (upsert)

**Path param:** `teacherId` (int) — id employee guru.

**Body (UpdateAppraisalLockDto):**
```json
{
  "tab": "ACADEMIC",
  "isLocked": true
}
```

Validasi:
- `tab` ∈ enum di atas — selain itu 400 "Invalid lock tab".
- `academicYearId` dan `campusId` diambil dari query/context: AY default aktif + campus dari `req.user` (admin dapat kirim campus lain). Jika `teacherId` bukan guru aktif di campus tersebut → 400 "Teacher is not in your campus".
- AY harus `activeStatus = ACTIVE` → jika tidak, 400 "Academic year not found or inactive".
- Bersifat **upsert**: create baris baru jika (teacher_id, academic_year_id, tab) belum ada; update jika sudah ada.
- Setiap sukses menulis 1 baris ke `appraisal_lock_audit` (actor, timestamp, tab, dari status → ke status).
- **Conflict detection:** jika `updated_at` baris berubah sejak frontend memuat list (2 admin bersamaan), return 409 "Lock status was changed by another admin. Reload and try again." (lihat EC-AL-07). Body opsional dapat membawa `expectedUpdatedAt` untuk optimistic concurrency.

**Response 200:**
```json
{
  "data": {
    "teacherId": 4137,
    "tab": "ACADEMIC",
    "academicYearId": 27,
    "isLocked": true,
    "lockedBy": 21046,
    "lockedAt": "2026-08-20T09:05:00.000Z",
    "unlockedBy": null,
    "unlockedAt": null
  }
}
```

## Audit Trail

### GET /v1/appraisal/locks/audit — Riwayat audit lock/unlock

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `academicYearId` | int | no | Filter AY |
| `campusId` | int | no | Filter campus |
| `teacherId` | int | no | Filter per guru |
| `tab` | enum(...) | no | Filter per tab |
| `page`, `pageSize` | int | no | Pagination |

**Response 200:**
```json
{
  "data": [
    {
      "id": 1,
      "academicYearId": 27,
      "campusId": 4,
      "teacherId": 4137,
      "teacherName": "Devie Lana",
      "tab": "ACADEMIC",
      "action": "LOCK",
      "fromStatus": false,
      "toStatus": true,
      "actorId": 21046,
      "actorName": "Herlina Susanti",
      "createdAt": "2026-08-20T09:05:00.000Z"
    }
  ],
  "count": 1,
  "meta": { "page": 1, "pageSize": 50, "itemCount": 1, "pageCount": 1, ... }
}
```

`action` ∈ `LOCK | UNLOCK`. Data append-only — tidak ada endpoint update/delete.

## Referensi Implementasi

- Controller pattern: `src/modules/lesson/lesson.controller.ts`
- Entity pattern: `src/modules/lesson/entities/lesson.entity.ts`
- Pagination: `src/common/dto/page-meta.dto`
- Enums: `src/types/enums` — `ModulesTypeEnum.PETALS`, `StatusTypeEnum`
- Enforce saat edit skor: endpoint skor appraisal (EPMS/PETALS/New Appraisal) mengecek `appraisal_lock` sebelum menulis — jika `is_locked = true` untuk (teacher, AY, tab), return 409 (konsisten `EC-EP-07`).
