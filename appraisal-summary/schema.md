# Schema — Appraisal Summary Report

> Status: DRAFT — mengikuti konvensi TypeORM 0.3.10 di `api_nest` (PostgreSQL).
> Fitur ini **bersifat read-only aggregation view** — tidak ada tabel baru yang wajib dibuat. Data bersumber dari tabel skor appraisal yang ada di `features/appraisal-new/` (gateway).

## Sumber Data (Aggregation Source)

Appraisal Summary Report tidak menyimpan data sendiri. Data dibaca dari tabel-tabel berikut (didefinisikan di `features/appraisal-new/`):

| Tabel (perkiraan dari gateway) | Peran | Kolom relevan |
|--------------------------------|-------|---------------|
| `staff_appraisal` atau `appraisal` | Header skor appraisal per guru per AY | `employee_id`, `campus_id`, `academic_year_id`, `type` (TEACHER/HOD) |
| `staff_appraisal_score` atau `appraisal_score` | Skor per dimensi per guru | `appraisal_id`, `dimension_key`, `score` |

> **Catatan:** Nama tabel eksak tergantung implementasi gateway (`features/appraisal-new/`). Jika gateway menggunakan `staff_appraisal` dan `staff_appraisal_score` (mirroring legacy), summary report tinggal membaca dari sana. Jika gateway menggunakan nama lain, sesuaikan.

## Tabel Opsional 1: `appraisal_dimension`

Config dimensi kompetensi (static config, di-seed sekali). Tidak wajib — backend bisa hardcode 17/9 dimensi, tapi tabel ini direkomendasikan agar fleksibel.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| type | enum(TEACHER, HOD) | no | | membedakan set dimensi teacher vs HOD |
| key | varchar(50) | no | | snake_case key (e.g. `professional_knowledge`) |
| label | varchar(200) | no | | label persis dari legacy (e.g. "Professional Knowledge and Practice") |
| sort_order | int | no | | urutan kolom di tabel |
| max_mark | decimal(5,1) | yes | NULL | maks skor untuk dimensi ini (teacher); NULL untuk HOD (skala seragam 0-5) |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | |

**Seed data — 17 dimensi TEACHER:**

| id | type | key | label | sort | max_mark |
|----|------|-----|-------|------|----------|
| 1 | TEACHER | professional_knowledge | Professional Knowledge and Practice | 1 | 14 |
| 2 | TEACHER | delivery_of_lessons | Delivery of lessons | 2 | 14 |
| 3 | TEACHER | classroom_management | Classroom management | 3 | 8 |
| 4 | TEACHER | preparation | Preparation | 4 | 6 |
| 5 | TEACHER | assessment | Assessment | 5 | 6 |
| 6 | TEACHER | co_curricular_activities | Co-curricular activities | 6 | 5 |
| 7 | TEACHER | leadership | Leadership ,contribution to school and community | 7 | 5 |
| 8 | TEACHER | professional_learning | Professional Learning | 8 | 4 |
| 9 | TEACHER | monitoring | Monitoring | 9 | 3 |
| 10 | TEACHER | motivational_skills | Motivational skills | 10 | 3 |
| 11 | TEACHER | conduct | Conduct | 11 | 5 |
| 12 | TEACHER | professionalism | Professionalism | 12 | 5 |
| 13 | TEACHER | responsibility | Responsibility | 13 | 4 |
| 14 | TEACHER | work_attitude | Work attitude | 14 | 4 |
| 15 | TEACHER | initiative | Initiative | 15 | 4 |
| 16 | TEACHER | adaptability | Adaptability to change | 16 | 4 |
| 17 | TEACHER | interpersonal_relationships | Interpersonal relationships | 17 | 4 |

> Maks per dimensi diestimasi dari data probe (nilai tertinggi teramati). Total teoretis: 14+14+8+6+6+5+5+4+3+3+5+5+4+4+4+4+4 = **99**. Total tertinggi teramati: 94.5 (Ulan Hernawan).

**Seed data — 9 dimensi HOD:**

| id | type | key | label | sort | max_mark |
|----|------|-----|-------|------|----------|
| 18 | HOD | leadership_vision | Leadership/Vision | 1 | NULL |
| 19 | HOD | strategic_planning | Strategic Planning & Administration | 2 | NULL |
| 20 | HOD | development_staff | Development & Management of Staff | 3 | NULL |
| 21 | HOD | professional_development | Professional Development | 4 | NULL |
| 22 | HOD | management_processes | Management of Processes | 5 | NULL |
| 23 | HOD | management_resources | Management of Resources | 6 | NULL |
| 24 | HOD | professional_knowledge | Professional Knowledge | 7 | NULL |
| 25 | HOD | professional_practice | Professional Practice | 8 | NULL |
| 26 | HOD | professional_engagement | Professional Engagement | 9 | NULL |

> HOD memakai skala seragam **0-5** — `max_mark` tidak relevan. TOTAL = rata-rata 9 dimensi.

## Tabel Opsional 2: `appraisal_summary_cache`

Materialized summary table untuk caching agregasi (hanya jika performa query jadi masalah). Diperlukan jika tabel skor appraisal sangat besar sehingga agregasi per-call lambat.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| employee_id | int FK → employee.id | no | | |
| campus_id | int FK → campus.id | no | | |
| academic_year_id | int FK → academic_year.id | no | | |
| type | enum(TEACHER, HOD) | no | | |
| dimension_1..17 | decimal(5,1) | yes | NULL | satu kolom per dimensi (opsional, bisa JSON) |
| total | decimal(6,1) | no | | TOTAL = sum atau avg |
| grade | varchar(2) | yes | NULL | enhancement: grade A/B/C/D |
| last_sync_at | timestamp | no | now() | kapan terakhir di-refresh |

**Index unik:** `(employee_id, campus_id, academic_year_id, type)`.

## Query Agregasi (tanpa cache)

Contoh query SQL untuk agregasi teacher summary:

```sql
SELECT
    e.id AS employee_id,
    e.user_id,
    e.name,
    s.dimension_key,
    s.score
FROM employee e
JOIN staff_appraisal a ON a.employee_id = e.id
    AND a.campus_id = :campusId
    AND a.academic_year_id = :academicYearId
    AND a.type = 'TEACHER'
    AND a.active_status = 'ACTIVE'
JOIN staff_appraisal_score s ON s.appraisal_id = a.id
    AND s.active_status = 'ACTIVE'
ORDER BY e.id, s.sort_order;
```

Kemudian di-backend: group by employee, map ke 17 dimensi, hitung total = sum.

## Migrations

- (Opsional) `npm run migration:generate --name=create-appraisal-dimension`
- (Opsional) `npm run migration:generate --name=create-appraisal-summary-cache`

## Relasi

```
employee 1 ── * staff_appraisal
staff_appraisal 1 ── * staff_appraisal_score
appraisal_dimension (static config, tidak ada relasi FK langsung)
```