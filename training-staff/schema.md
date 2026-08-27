# Schema — Training Staff

> Status: DRAFT — mengikuti konvensi TypeORM 0.3.10 di `api_nest` (PostgreSQL).

## Entity: `StaffTraining` → tabel `staff_training`

Master record pelatihan (riwayat training per staff).

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| title | varchar(255) | no | | judul pelatihan — wajib |
| participation | varchar(255) | yes | NULL | bentuk partisipasi (peserta/narasumber, dst.) |
| date_from | date | no | | tanggal mulai (`datefrom` legacy, `yyyy-mm-dd`) |
| date_to | date | no | | tanggal selesai (`dateto` legacy) |
| venue | varchar(255) | yes | NULL | tempat pelatihan |
| conducted_by | varchar(255) | yes | NULL | penyelenggara |
| city | varchar(100) | yes | NULL | kota |
| country_id | int FK → country.id | yes | NULL | id negara; default sistem jika kosong (lihat EC-03) |
| details | text | yes | NULL | rincian |
| comments | text | yes | NULL | komentar |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | soft delete |
| created_at / updated_at | timestamp | no | now() | |

**Index & constraint:**
- `INDEX (date_from, date_to)` — filter rentang tanggal.
- `INDEX (title)` — pencarian/duplicate check title.
- `INDEX (country_id)` — FK lookup.
- **Campus index — keputusan desain:** legacy memfilter per campus, namun `staff_training` tidak menyimpan campus secara langsung. Keputusan yang diambil pada brief ini: **menyimpan `campus_id` eksplisit** (kolom `campus_id` int FK → campus.id) yang diisi dari **campus staff pertama (primary) pada saat create**, dan diperbarui saat assignment berubah. Alasan: (1) filter list per campus harus cepat tanpa join ulang ke join table; (2) record training tunggal dapat melibatkan staff lintas campus (EC-04) sehingga butuh satu campus "utama" untuk grouping/filter; (3) skema legacy menyediakan campus selector sebagai filter utama halaman. Alternatif derived-only (tanpa kolom) didokumentasikan di `notes.md` dan `edgecases.md` EC-04 untuk finalisasi.
  - `INDEX (campus_id)` — filter list per campus.

**Catatan desain:**
- `date_from <= date_to` di-enforce di aplikasi (validasi DTO), bukan constraint DB (PostgreSQL check constraint opsional).
- Soft delete: `active_status = INACTIVE` — data histori tetap tersimpan untuk CV guru.

## Entity: `StaffTrainingStaff` → tabel `staff_training_staff`

Join table multi-staff (N:M). Satu training bisa punya banyak staff; satu staff bisa punya banyak training.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| training_id | int FK → staff_training.id | no | | PK bagian 1 |
| employee_id | int FK → employee.id | no | | PK bagian 2 — staff/guru |
| created_at | timestamp | no | now() | |

**Constraint:**
- `PRIMARY KEY (training_id, employee_id)` — kombinasi unik.
- `INDEX (employee_id)` — list training per staff (untuk CV guru).
- `INDEX (training_id)` — list staff per training.

**Catatan desain:**
- Tidak ada `active_status` di join table — assignment dihapus/insert ulang saat update (upsert assignment). Histori hard-deleted assignment tidak dipertahankan (record master tetap ada).
- Relasi dibangun via `@ManyToMany` atau dua `@OneToMany` + entity join eksplisit. Rekomendasi: entity join eksplisit agar mudah query per-staff (CV).

## Entity: `Country` → tabel `country` (existing atau to-create)

Dipakai untuk `country_id` di form. **Perlu dicek** apakah modul `country` sudah ada di `api_nest`. Jika belum ada: entity minimal `id` (int PK), `name` (varchar), `iso_code` (varchar opsional) + seeder data negara. Tidak ada perubahan kolom jika sudah ada.

## Relasi

```
staff_training 1 ── * staff_training_staff * ── 1 employee
staff_training * ── 1 country
staff_training * ── 1 campus        (kolom campus_id — keputusan desain)
```

```
staff_training 1 ── * staff_training_staff
staff_training_staff * ── 1 employee
staff_training * ── 1 country
```

## Migrations

- `npm run migration:generate --name=create-staff-training` (tabel `staff_training`)
- `npm run migration:generate --name=create-staff-training-staff` (tabel `staff_training_staff`)
- Jika modul `country` belum ada: `npm run migration:generate --name=create-country`

## Seed Data

- **Country:** jika modul `country` sudah ada → tidak perlu seed ulang. Jika belum ada → seeder data negara (minimal Indonesia, Singapore, Malaysia, dst. sesuai kebutuhan dropdown form).
- **Training:** tidak ada seed data — record dibuat via UI (Add Training).
