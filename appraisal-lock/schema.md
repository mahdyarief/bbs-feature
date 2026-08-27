# Schema — Appraisal Lock/Unlock

> Status: DRAFT — mengikuti konvensi TypeORM 0.3.10 di `api_nest` (PostgreSQL).

## Entity: `AppraisalLock` → tabel `appraisal_lock`

Status lock per guru per AY per tab.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| academic_year_id | int FK → academic_year.id | no | | |
| campus_id | int FK → campus.id | no | | campus saat lock dibuat |
| teacher_id | int FK → employee.id | no | | guru yang di-lock/unlock |
| tab | enum(ACADEMIC, CCA, REMARKS, PRELIM, APPRAISAL) | no | ACADEMIC | sesuai tab `view_lock.php` |
| is_locked | boolean | no | false | true = LOCKED (tidak bisa diedit) |
| locked_by | int FK → employee.id | yes | NULL | admin/principal yang mengunci terakhir |
| locked_at | timestamp | yes | NULL | waktu lock terakhir |
| unlocked_by | int FK → employee.id | yes | NULL | admin/principal yang membuka terakhir |
| unlocked_at | timestamp | yes | NULL | waktu unlock terakhir |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | soft delete |
| created_at / updated_at | timestamp | no | now() | |

**Index & constraint:**
- `UNIQUE (teacher_id, academic_year_id, tab)` — satu baris lock per guru per AY per tab.
- `INDEX (campus_id, academic_year_id, tab)` — filter lock view.
- `INDEX (academic_year_id, tab)` — query list.

**Catatan desain:**
- Baris dibuat via **upsert** saat admin pertama kali lock/unlock. Guru tanpa baris dianggap **UNLOCKED** (default) — tidak perlu seeding.
- `locked_by/locked_at` terisi saat `is_locked = true`; `unlocked_by/unlocked_at` terisi saat `is_locked = false`. Riwayat lengkap (perubahan berulang) tetap tersedia di `appraisal_lock_audit`.

## Entity: `AppraisalLockAudit` → tabel `appraisal_lock_audit` (opsional, disarankan)

Audit trail append-only dari semua aksi lock/unlock.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| academic_year_id | int FK → academic_year.id | no | | |
| campus_id | int FK → campus.id | no | | |
| teacher_id | int FK → employee.id | no | | |
| tab | enum(ACADEMIC, CCA, REMARKS, PRELIM, APPRAISAL) | no | | |
| action | enum(LOCK, UNLOCK) | no | | LOCK = dikunci, UNLOCK = dibuka |
| from_status | boolean | no | | status sebelum aksi |
| to_status | boolean | no | | status setelah aksi |
| actor_id | int FK → employee.id | no | | admin/principal yang melakukan aksi |
| created_at | timestamp | no | now() | append-only |

**Constraint:**
- `INDEX (academic_year_id, campus_id, teacher_id, tab)` — filter riwayat.
- Tidak ada update/delete — baris hanya di-insert (append-only).

## Migrations

- `npm run migration:generate --name=create-appraisal-lock` (tabel `appraisal_lock`)
- `npm run migration:generate --name=create-appraisal-lock-audit` (tabel `appraisal_lock_audit`)

## Seed Data

- **Tidak ada seed data.** Baris `appraisal_lock` dibuat via upsert saat admin pertama kali lock/unlock. Guru tanpa baris dianggap UNLOCKED. Audit dimulai kosong.

## Relasi

```
appraisal_lock * ── 1 employee (teacher)
appraisal_lock * ── 1 employee (locked_by / unlocked_by)
appraisal_lock * ── 1 campus
appraisal_lock * ── 1 academic_year
appraisal_lock_audit * ── 1 employee (teacher)
appraisal_lock_audit * ── 1 employee (actor)
appraisal_lock_audit * ── 1 campus
appraisal_lock_audit * ── 1 academic_year
```

## Integrasi Enforce

Tabel ini **dibaca** oleh endpoint edit skor appraisal (EPMS `teacher_review_score`, PETALS, New Appraisal). Sebelum menulis skor/remark untuk (teacher_id, academic_year_id, tab):
- jika ada baris `appraisal_lock` dengan `is_locked = true` dan `active_status = ACTIVE` → tolak dengan 409 "Appraisal entry is locked. Ask admin to unlock to edit."
- konsisten dengan `EC-EP-07` (EPMS): status SUBMITTED juga menolak edit 409; unlock pada fitur ini = override eksplisit yang membuka kembali.
