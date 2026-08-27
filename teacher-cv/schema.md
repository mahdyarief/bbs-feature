# Schema — Teacher CV

> Status: DRAFT — mengikuti konvensi TypeORM 0.3.10 di `api_nest` (PostgreSQL).
> Fitur Teacher CV adalah **read-only view** — tidak ada tabel baru yang **WAJIB** untuk menampilkan CV. Semua data dapat di-compose dari entitas existing. Dua tabel di bawah ini hanya diperlukan **JIKA** data pendidikan & pengalaman kerja belum tersedia di api_nest.

## Prinsip

1. **Tidak ada tabel baru yang WAJIB.** CV di-compose dari entitas existing yang dibaca saja (lihat "Entitas Existing yang Dibaca").
2. **Tabel `teacher_education` dan `teacher_experience` bersifat OPSIONAL** — hanya dibuat bila data pendidikan/pengalaman tidak terstruktur di modul employee existing.
3. **Perlu konfirmasi dengan modul HR** sebelum membuat tabel baru — jangan dibuat tanpa keputusan modul HR (lihat `notes.md`).
4. Jika modul HR sudah memiliki struktur pendidikan/pengalaman sendiri, fitur ini **membaca dari sana** — tidak menduplikasi data.

## Entitas Existing yang Dibaca (tidak dimodifikasi)

| Entitas / Modul | Kolom yang dipakai (proposal) | Dipakai untuk seksi |
|------------------|-------------------------------|---------------------|
| `employee` | `id`, `name`, `nip`, `active_status`, `campus_id` | Header CV, Data Pribadi |
| `employee-identity` (EmployeeIdentity) | `nik`, `date_of_birth`, `gender`, `phone`, `email`, `address`, `photo_url` | Data Pribadi |
| `employee-position` (EmployeePosition) | `position_name`, `start_date`, `end_date`, `active_status` | Pengalaman Kerja, Kualifikasi |
| `teacher` / `user-profile` / `employee-otp` / `employee-device` | — (pendukung; umumnya tidak dibaca untuk CV) | — |
| modul `training-staff` | riwayat pelatihan (nama, provider, tanggal, nomor sertifikat) — **perlu konfirmasi integrasi** | Pelatihan |

Catatan: seluruh nama kolom entitas existing di atas adalah **proposal pemetaan** — nama kolom aktual mengikuti entity di api_nest.

## Entity Opsional 1: `TeacherEducation` → tabel `teacher_education` (OPSIONAL — perlu konfirmasi modul HR)

Riwayat pendidikan staff. Hanya dibuat jika data pendidikan belum tersedia terstruktur di api_nest.

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| employee_id | int FK → employee.id | no | | staff pemilik riwayat pendidikan |
| level | varchar(50) | no | | proposal: jenjang, mis. `S1`, `S2`, `D3`, `SMA` |
| institution | varchar(255) | no | | nama institusi/perguruan tinggi (EC-06) |
| major | varchar(255) | yes | NULL | jurusan/program studi |
| graduation_year | int | yes | NULL | tahun lulus |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | soft delete |
| created_at / updated_at | timestamp | no | now() | |

**Index & constraint:**
- `INDEX (employee_id)` — filter riwayat pendidikan per staff.
- `UNIQUE (employee_id, level, institution, graduation_year)` — cegah duplikat entri yang sama (proposal).

## Entity Opsional 2: `TeacherExperience` → tabel `teacher_experience` (OPSIONAL — perlu konfirmasi modul HR)

Riwayat pengalaman kerja staff. Hanya dibuat jika data pengalaman belum tersedia terstruktur di api_nest (mis. tidak cukup di-cover `employee-position`).

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | |
| employee_id | int FK → employee.id | no | | staff pemilik riwayat pengalaman |
| company | varchar(255) | no | | nama perusahaan/instansi |
| position | varchar(255) | no | | posisi/jabatan saat bekerja |
| start_date | date | no | | tanggal mulai |
| end_date | date | yes | NULL | tanggal selesai; NULL = masih berjalan ("Sekarang") |
| description | text | yes | NULL | deskripsi pekerjaan (opsional) |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | soft delete |
| created_at / updated_at | timestamp | no | now() | |

**Index & constraint:**
- `INDEX (employee_id)` — filter riwayat pengalaman per staff.

## Migrations (hanya jika tabel opsional disetujui)

- `npm run migration:generate --name=create-teacher-education` (tabel `teacher_education`)
- `npm run migration:generate --name=create-teacher-experience` (tabel `teacher_experience`)

## Seed Data

- **Tidak ada seed data.** CV membaca data kepegawaian existing; tabel opsional diisi oleh modul HR (di luar scope fitur ini).

## Relasi

```
teacher_education  * ── 1 employee
teacher_experience * ── 1 employee
```

Kedua tabel opsional ini **hanya dibaca** oleh fitur Teacher CV — tidak ada endpoint tulis dari fitur ini.

## Integrasi Baca (Compose CV)

Saat `GET /api/v1/teacher-cv` dipanggil, backend menyusun:
1. `personal` ← `employee` + `employee-identity` (+ `campusName`).
2. `education[]` ← tabel pendidikan existing ATAU `teacher_education` (jika dibuat).
3. `experience[]` ← `employee-position` (riwayat jabatan) ATAU `teacher_experience` (jika dibuat).
4. `qualifications[]` ← sumber sertifikasi/kualifikasi (perlu konfirmasi modul HR).
5. `training[]` ← modul `training-staff` (perlu konfirmasi integrasi).

Seksi yang sumber datanya belum ada → dikembalikan sebagai array kosong (`[]`) → frontend menampilkan placeholder "—" (lihat `edgecases.md` EC-01).
