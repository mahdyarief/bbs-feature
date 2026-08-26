# API Contract — E-PETALS Dashboard & Petal Chart

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.PETALS`.
> **Base path:** global prefix `api` → URL lengkap `/api/v1/epetals`.
> **Dual portal:** Teacher Portal (Principal HQ / Super Admin, scope lintas campus) + Admin Portal (admin dengan `PETALS_MANAGE`, read-only lintas campus).

## Konvensi Response

- Sukses (single): `{ "data": {...} }`
- Sukses (list): `{ "data": [...], "count": number, "meta": PageMetaDto }`
- Error: `HttpException` standar NestJS.

## Endpoint Detail

### 1. GET /v1/epetals/campuses — Daftar campus untuk navigasi shell

**Response 200:**
```json
{
  "data": [
    { "id": 1, "name": "KJ-P", "fullName": "Kebon Jeruk Primary" },
    { "id": 2, "name": "KJ-S", "fullName": "Kebon Jeruk Secondary" },
    { "id": 4, "name": "PIK-S", "fullName": "Panta Indah Kapuk Secondary" },
    ... 11 campus
  ]
}
```
Hanya campus yang aktif + user punya akses (Super Admin/Principal HQ → semua; admin → semua dengan `PETALS_MANAGE`).

### 2. GET /v1/epetals/summary — Ringkasan PETALS per campus (wrapper)

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `campusId` | int | yes | campus target |
| `ay` | int | no | Default AY aktif (27 = 2026/2027) |
| `page`, `pageSize` | int | no | Pagination |

**Permission:** Principal HQ / Super Admin / Admin (`PETALS_MANAGE`). Teacher biasa → 403.

**Response 200:** sama dengan `GET /v1/petals/report` (lihat `features/petals/api-contract.md`) — wrapper yang meneruskan query ke modul petals. Contoh:
```json
{
  "data": [
    {
      "teacherId": 4137,
      "name": "Alex Norel Nalayog",
      "campusId": 4,
      "campusName": "PIK-S",
      "academicYearId": 27,
      "scoreP": 10, "scoreE": 11, "scoreT": 17, "scoreA": 20, "scoreL": 7,
      "averagePct": 85.53,
      "strength": "...", "areasOfConcern": "..."
    }
  ],
  "count": 37
}
```

### 3. GET /v1/epetals/chart — Data agregat avg PETALS per campus untuk chart

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `ay` | int | no | Academic year id. Default AY aktif. |

**Response 200:**
```json
{
  "data": [
    { "campusId": 1, "campusName": "KJ-P", "avgPct": 82.4, "teacherCount": 41 },
    { "campusId": 4, "campusName": "PIK-S", "avgPct": 85.5, "teacherCount": 37 },
    ... hanya campus yang punya data
  ],
  "count": 6
}
```
`avgPct = AVG((score_p + score_e + score_t + score_a + score_l) / 76 * 100)` per campus per AY.

**Behavior:** jika tidak ada data sama sekali untuk AY → `{ "data": [], "count": 0 }` (frontend tampilkan empty state, EC-EP-04).

## Referensi Implementasi

- Reuse: modul `petals` (`features/petals/api-contract.md`) untuk summary; hanya `campuses` + `chart` yang baru.
- Query agregasi: TypeORM `QueryBuilder` dengan `GROUP BY campus_id` + `AVG()`.
- Chart library frontend: cek yang sudah dipakai di bbs (recharts/echarts) — ganti Chart.js v2.7.2 legacy.
- Controller pattern: `src/modules/lesson/lesson.controller.ts`
- Enums: `src/types/enums` — `ModulesTypeEnum.PETALS`
