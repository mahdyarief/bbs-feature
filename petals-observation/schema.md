# Schema — PETALS Lesson Observation Input

> Status: DRAFT — mengikuti konvensi TypeORM 0.3.10 di `api_nest` (PostgreSQL).
> Tabel-tabel ini adalah sumber data untuk Report PETALS (`features/petals/`).

## Entity: `PetalsObservationItem` → tabel `petals_observation_item`

Rubrik 18 item observasi (static config, di-seed sekali).

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| dimension | enum(P, E, T, A, L) | no | | dimensi PETALS |
| label | varchar(500) | no | | teks rubrik |
| max_mark | int | no | 4 | nilai maksimum (fixed 4) |
| sort_order | int | no | | urutan dalam dimensi |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | |

**Seed data (18 rows):**

| id | dim | label | sort |
|----|-----|-------|------|
| 1 | P | Teacher communicates learning objectives with the class. | 1 |
| 2 | P | Teacher selects appropriate learning strategies, learning activities and resources. | 2 |
| 3 | P | Teacher develops a workable/appropriate time schedule. | 3 |
| 4 | E | Teacher uses appropriate resources (ICT, media, etc.) effectively. | 1 |
| 5 | E | Teacher provides opportunities for real-life application. | 2 |
| 6 | E | Teacher makes relevant connections to students' previous experiences. | 3 |
| 7 | T | Teacher is responsive to feedback and includes students' ideas. | 1 |
| 8 | T | Teacher arranges collaborative opportunities and manages them. | 2 |
| 9 | T | Teacher sets and enforces rules/routines effectively. | 3 |
| 10 | T | Teacher establishes a supportive learning environment. | 4 |
| 11 | T | Teacher uses voice and language appropriately. | 5 |
| 12 | A | Teacher allows flexibility and choice in tasks. | 1 |
| 13 | A | Students are able to ask well-thought questions. | 2 |
| 14 | A | Teacher encourages constructive criticism. | 3 |
| 15 | A | Teacher provides enough wait time for students to respond. | 4 |
| 16 | A | Teacher provides timely, specific and meaningful feedback. | 5 |
| 17 | L | The content of the lesson is relevant to requirements. | 1 |
| 18 | L | Students are able to comprehend the lesson content. | 2 |

## Entity: `TeacherObservation` → tabel `teacher_observation`

Header satu sesi observasi.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| teacher_id | int FK → employee.id | no | | guru yang diobservasi |
| campus_id | int FK → campus.id | no | | diambil dari teacher |
| academic_year_id | int FK → academic_year.id | no | | |
| observer_id | int FK → employee.id | no | | Principal/HOD yang mengobservasi |
| strength | text | yes | NULL | catatan kekuatan |
| areas_of_concern | text | yes | NULL | catatan area perbaikan |
| status | enum(DRAFT, SUBMITTED) | no | DRAFT | |
| date_submit | timestamp | yes | NULL | saat submit |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | soft delete |
| created_at / updated_at | timestamp | no | now() | |

**Index & constraint:**
- `UNIQUE (teacher_id, academic_year_id, observer_id)` — satu observer untuk satu guru per AY.
- `INDEX (campus_id, academic_year_id)` — filter list.

## Entity: `TeacherObservationScore` → tabel `teacher_observation_score`

Mark per item per observasi.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| observation_id | int FK → teacher_observation.id | no | | cascade delete |
| item_id | int FK → petals_observation_item.id | no | | |
| mark | int | no | 0 | nilai 0-4 |

**Index & constraint:**
- `UNIQUE (observation_id, item_id)` — satu mark per item per observasi.

## Migrations

- `npm run migration:generate --name=create-petals-observation-items` (seed data)
- `npm run migration:generate --name=create-teacher-observation`
- `npm run migration:generate --name=create-teacher-observation-score`

## Relasi

```
teacher_observation 1 ── * teacher_observation_score
petals_observation_item 1 ── * teacher_observation_score
teacher_observation * ── 1 employee (teacher)
teacher_observation * ── 1 employee (observer)
teacher_observation * ── 1 campus
teacher_observation * ── 1 academic_year
```

## Agregasi ke PETALS Report

Skor per dimensi dihitung via query:
- `scoreP = SUM(mark) WHERE dimension='P'` → max 12 (3 item x 4)
- `scoreE = SUM(mark) WHERE dimension='E'` → max 12
- `scoreT = SUM(mark) WHERE dimension='T'` → max 20
- `scoreA = SUM(mark) WHERE dimension='A'` → max 24
- `scoreL = SUM(mark) WHERE dimension='L'` → max 8

Tabel `teacher_appraisal` (di modul report) bisa diisi via trigger/scheduler atau dihitung on-the-fly.