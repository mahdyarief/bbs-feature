# Schema — Teacher Card (Kartu Guru)

> Status: DRAFT — mengikuti konvensi TypeORM 0.3.10 di `api_nest` (PostgreSQL).
> **Tidak ada tabel baru.** Fitur ini hanya membaca dari tabel/entity yang sudah ada.

## Entity yang Digunakan

Fitur Teacher Card membaca data dari tiga entity utama:

### 1. Entity: `Employee` → tabel `employee`

Data dasar staff/guru. Sumber: `src/modules/employee/entities/employee.entity.ts`.

| Column | Type | Null | Keterangan |
|--------|------|------|------------|
| `id` | int PK | no | Employee ID |
| `name` | varchar | no | Nama lengkap |
| `gender` | varchar(1) | yes | L / P |
| `birth_place` | varchar | yes | Tempat lahir (estimasi) |
| `birth_date` | date | yes | Tanggal lahir |
| `email` | varchar | yes | Email staff |
| `phone` | varchar | yes | Nomor telepon (estimasi) |
| `campus_id` | int FK → campus.id | no | Campus penempatan |
| `active_status` | enum(ACTIVE, INACTIVE, RESIGNED) | no | Status kepegawaian |
| `join_date` | date | yes | Tanggal bergabung (estimasi) |
| `created_at` / `updated_at` | timestamp | no | |

**Catatan:** Kolom bertanda "estimasi" perlu diverifikasi dari entity definition di `api_nest`.

### 2. Entity: `EmployeeIdentity` → tabel `employee_identity`

Data identitas kependudukan staff. Sumber: `src/modules/employee-identity/`.

| Column | Type | Null | Keterangan |
|--------|------|------|------------|
| `id` | int PK | no | |
| `employee_id` | int FK → employee.id | no | Relasi ke employee |
| `nip` | varchar | yes | NIP (Nomor Induk Pegawai) — estimasi |
| `nik` | varchar | yes | NIK (Nomor Induk Kependudukan) — estimasi |
| `photo_url` | varchar | yes | URL foto profil — estimasi |
| `active_status` | enum(ACTIVE, INACTIVE) | no | |

**Catatan:** Nama kolom di atas adalah **estimasi berdasarkan konvensi penamaan module**. Perlu diverifikasi dari entity definition yang sebenarnya di `api_nest/src/modules/employee-identity/`.

### 3. Entity: `EmployeePosition` → tabel `employee_position`

Data jabatan/posisi staff. Sumber: `src/modules/employee-position/`.

| Column | Type | Null | Keterangan |
|--------|------|------|------------|
| `id` | int PK | no | |
| `employee_id` | int FK → employee.id | no | Relasi ke employee |
| `position_name` | varchar | no | Nama jabatan (estimasi) |
| `position_type` | varchar | yes | Tipe posisi: PNS/Honorer/Kontrak (estimasi) |
| `is_primary` | boolean | yes | Apakah posisi utama (untuk multiple positions) — estimasi |
| `active_status` | enum(ACTIVE, INACTIVE) | no | |

### 4. Entity: `Campus` → tabel `campus`

Data campus. Sumber: `src/modules/campus/`.

| Column | Type | Null | Keterangan |
|--------|------|------|------------|
| `id` | int PK | no | |
| `name` | varchar | no | Nama campus |

## Relasi

```
employee 1 ── * employee_identity
employee 1 ── * employee_position
employee * ── 1 campus
```

## Query Logic

Untuk menyusun kartu guru, backend melakukan query:

```sql
SELECT
  e.id AS employeeId,
  e.name,
  e.gender,
  e.birth_place AS birthPlace,
  e.birth_date AS birthDate,
  e.email,
  e.phone,
  e.campus_id AS campusId,
  e.active_status AS employeeStatus,
  e.join_date AS joinDate,
  ei.nip,
  ei.nik,
  ei.photo_url AS photoUrl,
  ep.position_name AS position,
  ep.position_type AS positionType,
  c.name AS campusName
FROM employee e
LEFT JOIN employee_identity ei ON ei.employee_id = e.id AND ei.active_status = 'ACTIVE'
LEFT JOIN employee_position ep ON ep.employee_id = e.id AND ep.is_primary = true AND ep.active_status = 'ACTIVE'
LEFT JOIN campus c ON c.id = e.campus_id
WHERE e.id = :employeeId
  AND e.active_status IN ('ACTIVE', 'INACTIVE', 'RESIGNED');
```

> **Catatan:** Query di atas adalah **proposal**. Kolom eksak dan join conditions perlu disesuaikan dengan skema aktual di `api_nest`.

## Caching (Opsional)

Karena data kartu guru jarang berubah, disarankan caching:

- **Cache key:** `teacher-card:{employeeId}`
- **TTL:** 1 jam (3600 detik)
- **Invalidasi:** saat employee, employee-identity, atau employee-position di-update.
- **Storage:** Redis (jika tersedia) atau in-memory cache.

## Migrations

- **Tidak ada migration baru.** Tidak ada perubahan skema database.

## Seed Data

- **Tidak ada seed data baru.** Data kartu dibaca langsung dari data employee yang sudah ada.