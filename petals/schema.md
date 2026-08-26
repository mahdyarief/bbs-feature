# Schema — PETALS Summary Report

> Status: DRAFT — mengikuti konvensi TypeORM 0.3.10 di `api_nest` (PostgreSQL).
> Tabel-tabel ini dipakai bersama oleh modul **Lesson Observation Input** (`features/petals-observation/`) — report PETALS hanya membaca agregasi.

## Entity: `TeacherAppraisal` → tabel `teacher_appraisal`

Menyimpan ringkasan skor PETALS per guru per AY. Diisi otomatis oleh modul observasi saat skor disimpan (agregasi dari skor per item), atau dihitung on-the-fly dari tabel `teacher_observation_score`.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| teacher_id | int FK → employee.id | no | | index |
| campus_id | int FK → campus.id | no | | index |
| academic_year_id | int FK → academic_year.id | no | | |
| observer_id | int FK → employee.id | no | | yang melakukan observasi |
| score_p | int | no | 0 | skor dimensi P (0-12) |
| score_e | int | no | 0 | skor dimensi E (0-12) |
| score_t | int | no | 0 | skor dimensi T (0-20) |
| score_a | int | no | 0 | skor dimensi A (0-24) |
| score_l | int | no | 0 | skor dimensi L (0-8) |
| strength | text | yes | NULL | catatan kekuatan (dari observasi) |
| areas_of_concern | text | yes | NULL | catatan area perbaikan |
| date_submit | timestamp | yes | NULL | tanggal observasi di-submit |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | soft delete |
| created_at / updated_at | timestamp | no | now() | dari `BaseEntityWithDates` |

**Index & constraint:**
- `UNIQUE (teacher_id, campus_id, academic_year_id, observer_id)` — satu guru satu observer per AY (per observasi).
- `INDEX (campus_id, academic_year_id)` — filter utama report.

**Migrations:** `npm run migration:generate --name=create-teacher-appraisal` (di `api_nest`).

## Tabel pendukung (diimplementasikan di modul observasi — lihat `features/petals-observation/schema.md`)

| Tabel | Isi | Catatan |
|-------|-----|---------|
| `petals_observation_item` | 18 item rubrik (P=3, E=3, T=5, A=6, L=2) dengan label + max mark | di-seed dari `PETALS_Form.pdf` |
| `teacher_observation` | header observasi (teacher, observer, campus, AY, strength, areas_of_concern, status) | 1:N ke skor |
| `teacher_observation_score` | mark 0-4 per item per observasi | sumber agregasi ke `teacher_appraisal` |

## Seed Data

- **`seed-data.json`** / **`seed-data.csv`** di folder ini — hasil scrap langsung dari halaman legacy `petals_summary.php` (37 guru PIK-S, skor P/E/T/A/L, average, strength, areas of concern).
- Dipakai sebagai **source data awal** untuk seeding tabel `teacher_appraisal` (via script seed TypeORM atau import manual).
- Field `userId` pada seed = `User ID` legacy yang menunjuk ke `employee.id` — perlu di-map saat seeding agar `teacher_id` sesuai dengan data employee di api_nest.

## Catatan desain

- `averagePct` TIDAK disimpan — dihitung di query: `(score_p + score_e + score_t + score_a + score_l) / 76 * 100`.
- Jika modul observasi belum ada, `teacher_appraisal` bisa diisi langsung via seed data (fase 1), lalu dialihkan ke agregasi otomatis saat observasi aktif (fase 2).
