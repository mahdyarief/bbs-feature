# Schema — Smartbook Integration (Full Manage heyhi.sg)

## New Tables

### `smartbook_enrollment`
Status enrollment & payment per student per subject per academic year.

| Column | Type | Null | Default | Keterangan |
|--------|------|------|---------|------------|
| id | BIGINT PK AUTO_INCREMENT | no | - | |
| academic_year_id | INT FK → academic_year.id | no | - | AY (27 = 2026/2027) |
| campus_id | INT FK → campus.id | no | - | 2 KJ-S, 4 PIK-S, 6 BDG-S, 8 SMG-S, 10 MLG-S, 14 BPN-S |
| student_id | INT FK → student.id | no | - | |
| cohort_id | INT FK → cohort/master_level.id | no | - | 7 Sec1Acc s/d 17 JC2 |
| subject_id | INT FK → master_subjects.id | no | - | EL, MATH, SCI, PHY, ... |
| enroll_status | ENUM('ENROLLED','PAID','NOT_PAID','NONE') | no | 'NONE' | Status enrollment/payment |
| paid_at | DATETIME | yes | NULL | Waktu pembayaran |
| enrolled_at | DATETIME | yes | NULL | Waktu enroll |
| version | INT | no | 0 | Optimistic lock (EC-04) |
| created_at | TIMESTAMP | no | CURRENT_TIMESTAMP | |
| updated_at | TIMESTAMP | no | CURRENT_TIMESTAMP ON UPDATE | |

**Unique constraint:** `UNIQUE KEY uq_enroll (academic_year_id, student_id, subject_id)` — mencegah duplikat sinkronisasi (EC-03/EC-10).

**Indexes:**
- `idx_enroll_ay_campus (academic_year_id, campus_id)`
- `idx_enroll_cohort (cohort_id)`
- `idx_enroll_status (enroll_status)`

### `smartbook_sso_log`
Riwayat percobaan SSO ke platform Smartbook (heyhi.sg).

| Column | Type | Null | Default | Keterangan |
|--------|------|------|---------|------------|
| id | BIGINT PK AUTO_INCREMENT | no | - | |
| campus_id | INT FK → campus.id | yes | NULL | |
| user_id | INT FK → employee/user.id | no | - | |
| username | VARCHAR(100) | no | - | auth (base64 username) |
| reg | VARCHAR(10) | yes | NULL | region (base64, mis. "4") |
| token | VARCHAR(64) | yes | NULL | token HMAC (dulu MD5) |
| status | ENUM('SUCCESS','FAILED','INVALID_TOKEN','DENIED') | no | 'FAILED' | Hasil percobaan |
| ip_address | VARCHAR(45) | yes | NULL | |
| user_agent | VARCHAR(255) | yes | NULL | |
| created_at | TIMESTAMP | no | CURRENT_TIMESTAMP | |

**Indexes:**
- `idx_sso_user_date (user_id, created_at)`
- `idx_sso_campus (campus_id)`

### `smartbook_leaps`
Data learning engagement per student (kategori Leadership/Achievements/Service).

| Column | Type | Null | Default | Keterangan |
|--------|------|------|---------|------------|
| id | BIGINT PK AUTO_INCREMENT | no | - | |
| academic_year_id | INT FK → academic_year.id | no | - | |
| campus_id | INT FK → campus.id | no | - | |
| student_id | INT FK → student.id | no | - | |
| classroom_id | INT FK → classroom.id | no | - | |
| category | ENUM('LEADERSHIP','ACHIEVEMENTS','SERVICE') | no | - | Kategori leaps |
| leaps_level | VARCHAR(20) | yes | NULL | Level (L1, L2, L3, ...) |
| remark | VARCHAR(255) | yes | NULL | Catatan |
| created_at | TIMESTAMP | no | CURRENT_TIMESTAMP | |
| updated_at | TIMESTAMP | no | CURRENT_TIMESTAMP ON UPDATE | |

**Indexes:**
- `idx_leaps_student_cat (student_id, category)`
- `idx_leaps_ay_campus (academic_year_id, campus_id)`

### `smartbook_ticket`
Ticket/token valid (untuk gettoken_nya).

| Column | Type | Null | Default | Keterangan |
|--------|------|------|---------|------------|
| id | BIGINT PK AUTO_INCREMENT | no | - | |
| tkn | VARCHAR(64) | no | - | token (unique) |
| utp | VARCHAR(64) | no | - | user token hash |
| user_id | INT FK → user.id | yes | NULL | |
| status | ENUM('ACTIVE','USED','EXPIRED','REVOKED') | no | 'ACTIVE' | Status ticket |
| expires_at | DATETIME | yes | NULL | Waktu kadaluarsa |
| created_at | TIMESTAMP | no | CURRENT_TIMESTAMP | |

**Unique constraint:** `UNIQUE KEY uq_tkn (tkn)`

## Catatan Migrasi

- Migration: `CreateSmartbookTables` di `api_nest/src/database/migrations/`.
- Seeder opsional: `smartbook_enrollment` seed dari data legacy crawl (subject EL/MATH/SCI per campus) untuk keperluan dev/testing.
