# Schema — New Appraisal / Staff Database

> Status: DRAFT — mengikuti konvensi TypeORM 0.3.10 di `api_nest` (PostgreSQL).

## Entity: `StaffAppraisal` → tabel `staff_appraisal`

Header appraisal per guru per AY (mirroring `asd_staff_app_new.php` legacy).

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| teacher_id | int FK → employee.id | no | | guru yang di-appraise |
| campus_id | int FK → campus.id | no | | diambil dari teacher |
| academic_year_id | int FK → academic_year.id | no | | AY penilaian |
| reviewer_id | int FK → employee.id | no | | Principal/HOD yang menilai |
| appraisal_type | enum(TEACHER, HOD, PRINCIPAL) | no | TEACHER | varian view — menentukan komponen score |
| score | decimal(10,6) | yes | NULL | score aggregate (mis. 95.520833333333) |
| grade | char(1) | yes | NULL | A/B/C/D — dihitung dari score |
| teacher_score | decimal(10,6) | yes | NULL | komponen teacher (untuk HOD view, 80%) |
| hod_score | decimal(10,6) | yes | NULL | komponen HOD (untuk HOD view, 20%) |
| status | enum(INCOMPLETE, COMPLETED) | no | INCOMPLETE | |
| date_submit | timestamp | yes | NULL | last submit time (format legacy `2026-04-14 08:30:02`) |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | soft delete |
| created_at / updated_at | timestamp | no | now() | |

**Index & constraint:**
- `UNIQUE (teacher_id, academic_year_id)` — satu appraisal per guru per AY (legacy hanya 1 record per guru per AY; perbedaan `appraisal_type` ditangani via `reviewer_id` + record terpisah hanya jika diperlukan).
- `UNIQUE (teacher_id, academic_year_id, appraisal_type, reviewer_id)` — satu reviewer per tipe per guru per AY.
- `INDEX (campus_id, academic_year_id)` — filter list.
- `INDEX (status)` — filter status Completed/Incomplete.

## Entity: `StaffAppraisalScore` → tabel `staff_appraisal_score`

Skor per dimensi penilaian (detail). Konfigurasi dimensi PETALS (18 item) di-seed oleh `features/petals-observation/` — tabel ini hanya menyimpan nilai skor per dimensi untuk appraisal tertentu.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| appraisal_id | int FK → staff_appraisal.id | no | | cascade delete |
| dimension_id | int FK → petals_dimension.id | no | | referensi config dimensi PETALS (18 item) |
| score | int | no | 0 | mark 0-4 sesuai rubrik PETALS |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | soft delete |

**Index & constraint:**
- `UNIQUE (appraisal_id, dimension_id)` — satu skor per dimensi per appraisal.
- `CHECK (score BETWEEN 0 AND 4)` — batas rubrik PETALS (mark 0-4).

## Catatan: Grade & Weighting

- **Grade**: diturunkan dari `score` (0-100). Threshold (lihat `spec.md` Business Rules): A >= 90, B >= 75, C >= 60, D < 60. Nilai disimpan di kolom `grade` agar tidak perlu dihitung ulang di tiap query.
- **HOD weighting 80/20**: `score = teacher_score * 0.8 + hod_score * 0.2`. Kedua komponen disimpan terpisah (`teacher_score`, `hod_score`) agar kolom "Appraisal Teacher(80%)" dan "Appraisal HOD (20%)" bisa ditampilkan langsung tanpa kalkulasi frontend.
- **"Never!"**: kondisi `score = 0` (atau `score IS NULL` + `status = INCOMPLETE` + tidak ada record submit) → frontend menampilkan "Never!" (Principal view) / "0()" (Teacher view). Tidak ada kolom khusus; diturunkan dari nilai score.

## Migrations

- `npm run migration:generate --name=create-staff-appraisal`
- `npm run migration:generate --name=create-staff-appraisal-score`

## Seed Data

- Tidak ada seed khusus di fitur ini.
- Konfigurasi 18 dimensi PETALS di-seed oleh `features/petals-observation/` (tabel `petals_dimension`) — tabel `staff_appraisal_score` mereferensinya via `dimension_id`.

## Relasi

```
staff_appraisal 1 ── * staff_appraisal_score
staff_appraisal * ── 1 employee (teacher)
staff_appraisal * ── 1 employee (reviewer)
staff_appraisal * ── 1 campus
staff_appraisal * ── 1 academic_year
staff_appraisal_score * ── 1 petals_dimension (dari features/petals-observation/)
```
