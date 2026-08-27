# API Contract — Smartbook Integration (Full Manage heyhi.sg)

Backend: `api_nest` (NestJS). Prefix: `/api/v1`.

## Authentication & Authorization

- Semua endpoint butuh `@Auth()` (JWT) + `@CheckPermissions(...)`.
- Permission: `SMARTBOOK_MANAGE` (Admin) atau role `Principal` / `Super Admin` (Teacher Portal, scope lintas campus).
- Teacher biasa: hanya bisa akses `POST /smartbook/sso/url` (SSO) — endpoint manage lain → 403.

## Endpoint

### POST `/smartbook/sso/url`
Generate URL SSO ke heyhi.sg (token HMAC).

**Body:**
```json
{ "roleUser": 1, "username": "teacher202181", "reg": "4" }
```

**Response 200:**
```json
{
  "data": {
    "url": "https://teachers.binabangsaschool.com/sso_heyhi.php?role_user=1&auth=dGVhY2hlcjIwMjE4MQ==&reg=NA==&token=<hmac>"
  }
}
```

**Error:**
| Code | Condition | Message |
|------|-----------|---------|
| 401 | token tidak bisa di-generate | "Invalid SSO token" |
| 403 | role tidak berhak | "You don't have permission to access Smartbook" |

### GET `/smartbook/campuses`
Daftar campus untuk filter viewer.

**Response 200:**
```json
{
  "data": [
    { "id": 2, "shortName": "KJ-S", "name": "Kelapa Jakarta Secondary" },
    { "id": 4, "shortName": "PIK-S", "name": "PIK Secondary" }
  ]
}
```

### GET `/smartbook/cohorts`
Daftar cohort untuk filter viewer.

**Response 200:**
```json
{
  "data": [
    { "id": 7, "name": "Sec1Acc" },
    { "id": 16, "name": "JC1" }
  ]
}
```

### GET `/smartbook/viewer`
Detail per student (dengan filter).

**Query params:** `ayId` (required), `campusId` (optional), `cohortId` (optional), `status` (optional: ENROLLED | PAID | NOT_PAID | NONE), `page`, `limit`

**Response 200:**
```json
{
  "data": {
    "items": [
      {
        "studentId": 22785,
        "studentName": "Siswa Contoh",
        "campus": "PIK-S",
        "class": "JC1",
        "cohortId": 16,
        "subject": "EL",
        "enrollStatus": "PAID",
        "paidAt": "2026-08-20T00:00:00Z"
      }
    ],
    "total": 901,
    "page": 1,
    "limit": 50
  }
}
```

**Error:**
| Code | Condition | Message |
|------|-----------|---------|
| 400 | ayId/status invalid | "Invalid academic year or enroll status" |
| 403 | role tidak berhak | "You don't have permission to access Smartbook viewer" |
| 404 | tidak ada data | "No smartbook data found for the selected filters" |

### GET `/smartbook/summary`
Agregasi enrolled/paid per subject.

**Query params:** `ayId` (required), `campusId` (optional)

**Response 200:**
```json
{
  "data": [
    {
      "subject": "EL",
      "total": 901,
      "enrolled": 828,
      "enrolledPct": 91.9,
      "paid": 729,
      "paidPct": 80.91
    },
    { "subject": "MATH", "total": 1131, "enrolled": 1034, "enrolledPct": 91.42, "paid": 947, "paidPct": 83.73 }
  ]
}
```

### PATCH `/smartbook/payment/:id`
Update status payment satu enrollment.

**Body:**
```json
{ "enrollStatus": "PAID", "version": 0 }
```

**Valid values:** `ENROLLED`, `PAID`, `NOT_PAID`, `NONE`

**Response 200:** `{ "id": 123, "enrollStatus": "PAID", "version": 1, "updatedAt": "..." }`

**Error:**
| Code | Condition | Message |
|------|-----------|---------|
| 400 | status invalid | "Invalid enroll status" |
| 404 | id tidak ditemukan | "Smartbook enrollment not found" |
| 409 | optimistic lock conflict | "Enrollment has been updated by another user" |

### GET `/smartbook/export-paid`
Export laporan paid → PDF (stream).

**Query params:** `ayId` (required), `campusId` (optional), `cohortId` (optional)

**Response:** `200` `application/pdf` (Content-Disposition attachment, `smartbook-paid-report-<ayId>.pdf`)

**Error:**
| Code | Condition | Message |
|------|-----------|---------|
| 404 | tidak ada data | "No smartbook data found for the selected filters" |
| 500 | gagal generate | "Failed to generate paid report" |

### GET `/smartbook/sso-log`
Riwayat percobaan SSO per user per tanggal.

**Query params:** `campusId` (optional), `dateFrom` (optional, default 7 hari lalu), `dateTo` (optional, default hari ini), `page`, `limit`

**Response 200:**
```json
{
  "data": {
    "items": [
      {
        "userId": 586,
        "userName": "teacher202181",
        "campus": "PIK-S",
        "date": "2026-08-26",
        "status": "SUCCESS",
        "attempts": 1
      }
    ],
    "total": 120,
    "page": 1,
    "limit": 50
  }
}
```

**Error:**
| Code | Condition | Message |
|------|-----------|---------|
| 400 | dateFrom > dateTo | "Invalid date range" |
| 403 | role tidak berhak | "You don't have permission to access Smartbook SSO log" |

### GET `/smartbook/tickets`
Validasi ticket/token (tkn + utp) → data halaman Tickets.

**Query params:** `tkn`, `utp` (keduanya required)

**Response 200:** data halaman Tickets (layout + data)

**Error:**
| Code | Condition | Message |
|------|-----------|---------|
| 401 | tkn/utp tidak cocok | "Token Invalid!" (mirror legacy) |
| 403 | role tidak berhak | "You don't have permission" |

### GET `/smartbook/dashboards`
Daftar URL dashboard heyhi.sg (untuk embed/redirect).

**Response 200:**
```json
{
  "data": [
    { "key": "teacher_stats", "name": "Teacher Stats", "url": "https://clientreport.heyhi.sg/public/dashboard/8f589a1a-5693-4d87-926c-a2406b6d351d" },
    { "key": "worksheet_stats", "name": "Worksheet Stats", "url": "https://clientreport.heyhi.sg/public/dashboard/a596cec7-ef8c-4e4e-9a76-77dc468e4419" },
    { "key": "subject_stats", "name": "Subject Stats", "url": "https://clientreport.heyhi.sg/public/dashboard/da204a56-968f-45a0-b326-1f6964a59b81" }
  ]
}
```

### GET `/smartbook/leaps`
Data Leaps per kategori.

**Query params:** `campusId` (optional), `ayId` (optional), `category` (optional: LEADERSHIP | ACHIEVEMENTS | SERVICE), `page`, `limit`

**Response 200:**
```json
{
  "data": {
    "items": [
      {
        "studentId": 22785,
        "studentName": "Siswa Contoh",
        "campus": "PIK-S",
        "class": "JC1",
        "category": "LEADERSHIP",
        "leapsLevel": "L3",
        "updatedAt": "2026-08-20T00:00:00Z"
      }
    ],
    "total": 50,
    "page": 1,
    "limit": 50
  }
}
```

### PATCH `/smartbook/leaps/:id/category`
Update kategori Leaps (mirror `update_leap_cat.php`).

**Body:**
```json
{ "category": "SERVICE" }
```

**Valid values:** `LEADERSHIP`, `ACHIEVEMENTS`, `SERVICE`

**Response 200:** `{ "id": 123, "category": "SERVICE", "updatedAt": "..." }`

**Error:**
| Code | Condition | Message |
|------|-----------|---------|
| 400 | kategori invalid | "Invalid leaps category" |
| 404 | id tidak ditemukan | "Leaps record not found" |

## Catatan Implementasi

- Repositori: modul baru `api_nest/src/modules/smartbook/` — controller, service, entity `SmartbookEnrollment`, `SmartbookSsoLog`, `SmartbookLeaps`, `SmartbookTicket`.
- Reuse modul `leaps-event` & `leaps-type` yang sudah ada di `api_nest` untuk Leaps.
- SSO token: `crypto.createHmac('sha256', secret)` — bukan MD5 plain (security fix). `auth`/`reg` tetap base64.
- PDF: reuse pattern modul `report`/`student-report` (template engine `src/templates`).
