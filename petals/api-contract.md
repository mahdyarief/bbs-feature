# API Contract — PETALS Summary Report

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.PETALS` (baru, perlu ditambahkan ke `src/types/enums` + ability di `src/modules/casl/casl-ability.factory.ts`).
> **Base path:** global prefix `api` → URL lengkap `/api/v1/petals`.
> **Dual portal:** endpoint yang sama dikonsumsi dua portal — Teacher Portal (scope campus `req.user`, pengguna utama Principal/HOD) dan Admin Portal (admin punya permission `PETALS_MANAGE` untuk perbantuan lintas campus).
> **Data source:** skor PETALS di-entry di modul terpisah (Lesson Observation — lihat `features/petals-observation/`). Endpoint di sini hanya **membaca** agregasi dari tabel `teacher_appraisal` (+ `teacher_observation` jika observasi per item dipakai).

## Konvensi Response

- Sukses (single): `{ "data": {...} }`
- Sukses (list): `{ "data": [...], "count": number, "meta": PageMetaDto }` — `PageMetaDto` dari `src/common/dto/page-meta.dto`; query filter extends `PageOptionsDto` (`page`, `pageSize`, `order`, `sortBy`, `relations`, `query`).
- Error: `HttpException` standar NestJS (`src/errors` / `src/filters`).

## Endpoint Detail

### 1. GET /v1/petals/report — Ringkasan skor PETALS per campus

**Query params (GetPetalsReportDto):**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `campusId` | int | no | Default dari `req.user` campusId. Admin Portal bisa kirim campus lain. |
| `ay` | int | no | Academic year id (contoh: 27 = 2026/2027). Default AY aktif. |
| `page` | int | no | Default 1 |
| `pageSize` | int | no | Default 50 (report view, bukan entry form) |
| `order` | enum(ASC, DESC) | no | Default ASC |
| `sortBy` | enum(teacherId, name, averagePct) | no | Default name |

**Permission:** `@CheckPermissions([{ action: READ, subject: PETALS }])` — Principal/HOD/Staff. Teacher biasa → 403.

**Response 200:**
```json
{
  "data": [
    {
      "teacherId": 4137,
      "name": "Alex Norel Nalayog",
      "email": "alex@binabangsaschool.com",
      "campusId": 4,
      "campusName": "PIK-S",
      "academicYearId": 27,
      "scoreP": 10,
      "scoreE": 11,
      "scoreT": 17,
      "scoreA": 20,
      "scoreL": 7,
      "averagePct": 85.53,
      "strength": "Clear explanation and good rapport with students.",
      "areasOfConcern": "Need more varied assessment strategies.",
      "observerId": 21046,
      "observerName": "Herlina Susanti",
      "dateSubmit": "2026-08-20T09:00:00.000Z"
    }
  ],
  "count": 37,
  "meta": { "page": 1, "pageSize": 50, "itemCount": 37, "pageCount": 1, "hasPreviousPage": false, "hasNextPage": false }
}
```

**Catatan:** `averagePct = (scoreP + scoreE + scoreT + scoreA + scoreL) / 76 * 100` — dihitung backend (bukan disimpan). Filter campus + AY wajib efektif; jika `campusId` tidak dikirim dan `req.user` tidak punya campusId → 400 (lihat EC-03).

### 2. GET /v1/petals/export — Export ke Excel (download)

**Query params:** sama dengan report (`campusId`, `ay`) — `page`/`pageSize` diabaikan (ambil semua data).

**Permission:** `@CheckPermissions([{ action: READ, subject: PETALS }])` + `PETALS_MANAGE` untuk admin lintas campus.

**Response 200:** `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` (XLSX via `exceljs`), header kolom: `#`, `User ID`, `Name`, `P`, `E`, `T`, `A`, `L`, `Average (%)`, `Strength`, `Areas of Concern`. Content-Disposition: `attachment; filename="petals-summary-{campusName}-{ay}.xlsx"`.

**Behavior:** jika tidak ada data → file kosong dengan header saja (lihat EC-04).

### 3. GET /v1/petals/campuses — Daftar campus untuk filter (opsional, AC-2)

**Response 200:**
```json
{
  "data": [
    { "id": 4, "name": "PIK-S", "fullName": "Panta Indah Kapuk Secondary" }
  ]
}
```
Hanya campus yang punya akses user (Principal/HOD → campus miliknya; admin → semua).

## Referensi Implementasi (contoh pola dari modul existing)

- Controller pattern: `src/modules/lesson/lesson.controller.ts` — `@Controller({ version: '1', path: 'lessons' })`, `@CheckPermissions`, `@Req() req: Request`, `ParseIntPipe`, `{ data }` wrapper.
- Entity pattern: `src/modules/lesson/entities/lesson.entity.ts` — `BaseEntityWithDates`, `@Index()` FK, `@ManyToOne` + `@JoinColumn`, `Relation<T>`, `StatusTypeEnum`.
- Pagination: `src/common/dto/page-meta.dto` (`PageMetaDto`).
- Export Excel: `exceljs` (cek library existing di api_nest; fallback `xlsx`).
- Enums: `src/types/enums` — `ACLTypeEnum`, `ModulesTypeEnum` (tambah `PETALS`), `StatusTypeEnum`.
