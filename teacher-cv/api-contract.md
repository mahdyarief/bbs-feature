# API Contract — Teacher CV

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject modul Employee/HR (mis. `EMPLOYEE_READ` / permission HR setara — konfirmasi enum di `src/types/enums`).
> **Base path:** global prefix `api` → URL lengkap `/api/v1/teacher-cv`.
> **Dual portal:** Admin Portal (Admin/ASD, lintas campus) + Teacher Portal (guru hanya CV sendiri — `req.user.employeeId`; Principal scope `req.user.campusId`).
>
> **Catatan field:** seluruh nama field pada dokumen ini adalah **PROPOSAL** — legacy `staff/asd_staff_cv.php` tidak memiliki HTML dump/probe sehingga field aslinya tidak terkonfirmasi (lihat `notes.md`).

## Konvensi Response

- Sukses (single): `{ "data": {...} }`
- Sukses (list): `{ "data": [...], "count": number, "meta": PageMetaDto }`
- Error: `HttpException` standar NestJS.

## Endpoint

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/teacher-cv?employeeId=&campusId=` | Composed CV payload — `personal` + `education[]` + `experience[]` + `qualifications[]` + `training[]` (read-only) |

Tidak ada endpoint tulis (POST/PUT/PATCH/DELETE) — fitur ini read-only. Edit data CV adalah tanggung jawab modul HR terpisah.

## GET /v1/teacher-cv — Composed CV payload

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `employeeId` | int | yes* | id employee (staff/guru) yang CV-nya diminta. Untuk role guru di teacher portal, param ini **diabaikan/di-override** dengan `req.user.employeeId`. |
| `campusId` | int | no | Default dari `req.user.campusId`. Admin bisa kirim campus lain (validasi scope, lihat EC-05). |

*`employeeId` wajib untuk Admin Portal; untuk guru di Teacher Portal nilai diambil dari token (jika dikirim param yang berbeda dari diri sendiri → 403).

**Response 200:**
```json
{
  "data": {
    "personal": {
      "employeeId": 4137,
      "name": "Devie Lana",
      "nip": "197504152001122001",
      "nik": "3174065504750002",
      "dateOfBirth": "1975-04-15",
      "gender": "FEMALE",
      "photoUrl": "https://cdn.bbs.sch.id/photos/4137.jpg",
      "email": "devie.lana@binabangsaschool.com",
      "phone": "+62 812-3456-7890",
      "address": "Jl. Raya PIK Blok A No. 12, Jakarta Utara",
      "campusId": 4,
      "campusName": "PIK-S",
      "position": "Kepala Program SD"
    },
    "education": [
      {
        "id": 1,
        "level": "S1",
        "institution": "Universitas Negeri Jakarta",
        "major": "Pendidikan Bahasa Inggris",
        "graduationYear": 1998
      }
    ],
    "experience": [
      {
        "id": 1,
        "company": "SMA Bina Bangsa",
        "position": "Guru Bahasa Inggris",
        "startDate": "2003-07-01",
        "endDate": null,
        "description": "Mengajar Bahasa Inggris kelas X-XII"
      }
    ],
    "qualifications": [
      {
        "id": 1,
        "name": "Sertifikasi Pendidik",
        "issuer": "Kemendikbud",
        "issuedDate": "2010-06-15",
        "expiryDate": null
      }
    ],
    "training": [
      {
        "id": 1,
        "name": "Pelatihan Kurikulum Merdeka",
        "provider": "Kemendikbud",
        "startDate": "2023-01-09",
        "endDate": "2023-01-13",
        "certificateNumber": "TRN-2023-00412"
      }
    ]
  }
}
```

**Catatan field (semua PROPOSAL, perlu konfirmasi):**
- `personal.*` — dipetakan dari `employee` + `employee-identity`. `nip`/`nik` bisa kosong (lihat EC-04).
- `education[]` — dari tabel pendidikan existing di api_nest ATAU tabel opsional `teacher_education` (lihat `schema.md`). Jika sumber belum ada → array kosong.
- `experience[]` — dari `employee-position` (riwayat jabatan) ATAU tabel opsional `teacher_experience`. Jika belum ada → array kosong.
- `qualifications[]` — sertifikasi/kualifikasi; sumber data perlu konfirmasi modul HR. Jika belum ada → array kosong.
- `training[]` — riwayat pelatihan dari modul `training-staff` (perlu konfirmasi integrasi). Jika belum ada → array kosong.
- `endDate`/`expiryDate` `null` = masih berjalan / berlaku → ditampilkan "Sekarang" / "Present".
- Seksi kosong **tetap** dikembalikan sebagai `[]` (bukan dihilangkan) — frontend memutuskan placeholder (EC-01).

**Error:**

| HTTP Code | Kondisi |
|-----------|---------|
| 404 | Employee tidak ditemukan — "Employee not found" |
| 400 | `employeeId`/`campusId` tidak valid atau tidak terkirim — "Invalid employee or campus id" |
| 403 | Role tanpa permission / guru mengakses employeeId orang lain / staff di luar campus scope — "You don't have permission to view teacher CV" |
| 401 | Token JWT tidak ada/expired |

## Alternatif (tidak direkomendasikan untuk fitur ini)

Alternatif desain: **reuse endpoint existing** `GET /api/v1/employees/:id` + sub-resources terpisah (`/employee-identity`, `/employee-position`, dst.) lalu frontend menyusun sendiri payload CV.

**Kenapa dipilih endpoint composed `GET /v1/teacher-cv`:**
- Menjaga single source of truth di backend — logika agregasi seksi CV tidak terduplikasi di dua frontend (Admin Portal + Teacher Portal).
- Menyederhanakan render dokumen & print: frontend cukup menerima satu object CV utuh.
- Jika suatu saat ada kebutuhan export PDF server-side, endpoint composed sudah siap dijadikan sumber data.

Alternatif ini tetap valid bila tim memilih dekomposisi — namun konsekuensinya logika compose/urutan seksi harus diimplementasi dua kali (dual portal).

## Referensi Implementasi

- Controller pattern: `src/modules/lesson/lesson.controller.ts`
- Entity pattern: `src/modules/lesson/entities/lesson.entity.ts`
- Pagination: `src/common/dto/page-meta.dto`
- Enums: `src/types/enums` — `StatusTypeEnum` (ACTIVE/INACTIVE), `GenderTypeEnum` bila ada
- Modul sumber: `employee`, `employee-identity`, `employee-position` (baca relasi via TypeORM `relations` / `find`)
