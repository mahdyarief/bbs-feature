# Schema — EPMS Work Review

> Status: DRAFT — mengikuti konvensi TypeORM 0.3.10 di `api_nest` (PostgreSQL).

## Entity: `EpmsReviewItem` → tabel `epms_review_item`

Item kompetensi EPMS (static config, di-seed sekali dari form Work Review legacy).

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| section | int | no | | nomor section 1-7 |
| section_name | varchar(100) | no | | nama section (e.g. "Key Performance Areas (KRA)") |
| group_id | int | no | | sub-group dalam section |
| group_label | varchar(255) | yes | NULL | label sub-group (e.g. "1. Holistic Development of Students through") |
| label | varchar(500) | no | | teks kompetensi (e.g. "Quality Learning of Students") |
| sort_order | int | no | | urutan dalam section |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | |

**Seed data (26 rows, Section 1-5 — kompetensi yang di-skor per semester):**

> Section 6 (Training and Development Plans) dan Section 7 (Review and Comments) **bukan** item di tabel ini — keduanya berupa field teks di `teacher_review` (`training_plan`, `teacher_comments_sem1`, `reporting_officer_comments_sem1`, `teacher_comments_sem2`, `reporting_officer_comments_sem2`).

| id | section | section_name | group_id | group_label | label | sort |
|----|---------|-------------|----------|-------------|-------|------|
| 1 | 1 | Key Performance Areas (KRA) | 1 | 1. Holistic Development of Students through | Quality Learning of Students | 1 |
| 2 | 1 | Key Performance Areas (KRA) | 1 | 1. Holistic Development of Students through | Pastoral Care and Well-being of students | 2 |
| 3 | 1 | Key Performance Areas (KRA) | 1 | 1. Holistic Development of Students through | Co-Curricular Activities | 3 |
| 4 | 1 | Key Performance Areas (KRA) | 2 | 2. Contribution to School | Committees | 4 |
| 5 | 1 | Key Performance Areas (KRA) | 2 | 2. Contribution to School | Others | 5 |
| 6 | 1 | Key Performance Areas (KRA) | 3 | 3. Collaborations with Parents | (empty) | 6 |
| 7 | 1 | Key Performance Areas (KRA) | 4 | 4. Professional Development | (empty) | 7 |
| 8 | 1 | Key Performance Areas (KRA) | 5 | 5. Others | (empty) | 8 |
| 9 | 2 | Teaching Competencies | 1 | null | 1. Professional Knowledge and Practice | 1 |
| 10 | 2 | Teaching Competencies | 2 | null | 2. Delivery of Lessons | 2 |
| 11 | 2 | Teaching Competencies | 3 | null | 3. Classroom Management | 3 |
| 12 | 2 | Teaching Competencies | 4 | null | 4. Lesson Preparations | 4 |
| 13 | 2 | Teaching Competencies | 5 | null | 5. Assessment | 5 |
| 14 | 2 | Teaching Competencies | 6 | null | 6. Monitoring | 6 |
| 15 | 2 | Teaching Competencies | 7 | null | 7. Motivational Skills | 7 |
| 16 | 3 | Co-Curricular Activities | 1 | null | 1. Managing CCA | 1 |
| 17 | 3 | Co-Curricular Activities | 2 | null | 2. New Initiatives | 2 |
| 18 | 4 | Leadership Potentials and Professional Development | 1 | null | 1. Leadership Contribution to School and Community | 1 |
| 19 | 4 | Leadership Potentials and Professional Development | 2 | null | 2. Professional Learning | 2 |
| 20 | 5 | Professional Qualities of A Teacher | 1 | Personal Qualities | 1. Conduct | 1 |
| 21 | 5 | Professional Qualities of A Teacher | 2 | null | 2. Professionalism | 2 |
| 22 | 5 | Professional Qualities of A Teacher | 3 | null | 3. Responsibility | 3 |
| 23 | 5 | Professional Qualities of A Teacher | 4 | null | 4. Work Attitude | 4 |
| 24 | 5 | Professional Qualities of A Teacher | 5 | null | 5. Initiative | 5 |
| 25 | 5 | Professional Qualities of A Teacher | 6 | null | 6. Adaptability to Change | 6 |
| 26 | 5 | Professional Qualities of A Teacher | 7 | null | 7. Interpersonal relationship | 7 |

## Entity: `TeacherReview` → tabel `teacher_review`

Header Work Review per guru per AY.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| teacher_id | int FK → employee.id | no | | guru yang di-review |
| campus_id | int FK → campus.id | no | | diambil dari teacher |
| academic_year_id | int FK → academic_year.id | no | | |
| reviewer_id | int FK → employee.id | no | | Principal yang mereview |
| job_desc | varchar(255) | yes | NULL | Job Desc dari header form |
| interschool | varchar(255) | yes | NULL | Interschool dari header form |
| status | enum(DRAFT, SUBMITTED) | no | DRAFT | |
| date_submit | timestamp | yes | NULL | saat submit |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | soft delete |
| created_at / updated_at | timestamp | no | now() | |

**Index & constraint:**
- `UNIQUE (teacher_id, academic_year_id, reviewer_id)` — satu reviewer untuk satu guru per AY.
- `INDEX (campus_id, academic_year_id)` — filter list.

## Entity: `TeacherReviewScore` → tabel `teacher_review_score`

Skor per kompetensi per semester.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| review_id | int FK → teacher_review.id | no | | cascade delete |
| item_id | int FK → epms_review_item.id | no | | |
| semester | int | no | | 1 = Semester 1, 2 = Semester 2 |
| score | int | no | 0 | nilai sesuai skala item (default 0-100) |

**Index & constraint:**
- `UNIQUE (review_id, item_id, semester)` — satu skor per item per semester per review.

## Entity: `TeacherReviewComment` → tabel `teacher_review_comment` (opsional)

Komentar per review (bisa disimpan di kolom header `teacher_review` jika prefer single table).

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| review_id | int FK → teacher_review.id | no | | cascade delete |
| type | enum(TRAINING_PLAN, TEACHER_COMMENT_S1, RO_COMMENT_S1, TEACHER_COMMENT_S2, RO_COMMENT_S2) | no | | |
| content | text | yes | NULL | isi komentar |

## Migrations

- `npm run migration:generate --name=create-epms-review-items` (seed data)
- `npm run migration:generate --name=create-teacher-review`
- `npm run migration:generate --name=create-teacher-review-score`
- `npm run migration:generate --name=create-teacher-review-comment`

## Relasi

```
teacher_review 1 ── * teacher_review_score
teacher_review 1 ── * teacher_review_comment
epms_review_item 1 ── * teacher_review_score
teacher_review * ── 1 employee (teacher)
teacher_review * ── 1 employee (reviewer)
teacher_review * ── 1 campus
teacher_review * ── 1 academic_year
```