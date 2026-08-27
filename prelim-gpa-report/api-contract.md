---
status: DRAFT
feature: Prelim Report & GPA June/Nov
slug: prelim-gpa-report
---

# API Contract — Prelim Report & GPA June/Nov

## Konvensi

- **Base path:** `/api/v1`
- **Format response:** `{ data: { ... } }` untuk sukses, `{ errors: [...] }` untuk error
- **Auth:** Semua endpoint memerlukan session/authentication (JWT/session)

---

## 1. Generate Report (Prelim / GPA)

### POST /api/v1/reports

Generate satu report per siswa. Ini endpoint inti untuk Prelim Report (IGCSE/AS/A Level) dan GPA — dibedakan oleh `reportType` + `term`.

**Request Body (CreateReportDto):**

| Field | Tipe | Required | Deskripsi |
|-------|------|----------|-----------|
| `studentReportId` | string | Yes | ID StudentReport |
| `term` | integer | Yes | Term 1-4 (`VALID_TERMS = [1,2,3,4]`) |
| `reportType` | enum | Yes | `CAMBRIDGE_IGCSE` \| `CAMBRIDGE_ASLEVEL` \| `CAMBRIDGE_ALEVEL` \| `GPA` \| dll |
| `reportUrl` | string | No | URL report |
| `html` | string | No | HTML yang disimpan (output generator) |
| `gradeSnapshot` | object | No | Snapshot nilai |
| `reportStatus` | enum | No | Status report |
| `weightedAverage` | number | No | Rata-rata tertimbang |

**Contoh request — Prelim IGCSE:**
```json
{
  "studentReportId": "uuid-1234",
  "term": 4,
  "reportType": "CAMBRIDGE_IGCSE"
}
```

**Contoh request — GPA:**
```json
{
  "studentReportId": "uuid-1234",
  "term": 2,
  "reportType": "GPA"
}
```

**Response 201:**
```json
{
  "data": {
    "id": "report-uuid",
    "studentReportId": "uuid-1234",
    "term": 4,
    "reportType": "CAMBRIDGE_IGCSE",
    "html": "<div>...rendered report html...</div>",
    "reportStatus": "COMPILED"
  }
}
```

**Error umum:**
| Kode | Kasus |
|------|-------|
| 400 | `term` di luar 1-4, atau `reportType` tidak valid |
| 404 | `studentReportId` tidak ditemukan |
| 422 | Tidak ada data nilai untuk reportType tersebut |

---

### POST /api/v1/reports/update

Regenerate ulang report yang sudah ada (retry saat kompilasi gagal). Body sama dengan `create`.

---

### GET /api/v1/reports

Daftar report. Query: `studentReportId`, `reportType`, `term`, `page`, `pageSize`.

### GET /api/v1/reports/:id

Detail satu report (termasuk `html`).

### PUT /api/v1/reports/:id

Update report (misal ganti `html`/`reportStatus`).

### POST /api/v1/reports/disable-undisable-view-report

Toggle visibilitas report ke student portal.

---

## 2. Konfigurasi Prelim

### GET /api/v1/threshold/subject/:subjectId?thresholdType=AS_PRELIM

Threshold (grade boundaries) per subject untuk prelim. `thresholdType`: `IGCSE_ALEVEL` | `AS_PRELIM`.

### POST /api/v1/threshold, PATCH /api/v1/threshold/:id, DELETE /api/v1/threshold/:id

CRUD threshold.

### GET/POST/PATCH /api/v1/syllabus*, /api/v1/gradeSchema*

Konfigurasi syllabus components + grade schema (A*–E untuk IGCSE/A Level, a–u untuk AS Prelim).

### GET /api/v1/grade-moderation/student-grades, POST /api/v1/grade-moderation/bulk-update

Input moderation nilai prelim per paper.

---

## 3. Konfigurasi GPA

### GET/POST /api/v1/gpaGradingScales

Grading scale GPA — `config` (jsonb) berisi `gpValue` per grade.

**Response 200 (contoh):**
```json
{
  "data": [
    {
      "id": 1,
      "config": {
        "A*": 4.0,
        "A": 4.0,
        "B": 3.5,
        "C": 3.0,
        "D": 2.5,
        "E": 2.0,
        "F": 1.0
      }
    }
  ]
}
```

### GET/POST /api/v1/gpaSubjectUnits

Subject unit per level — `unit` (float) per `masterLevelId` + `subjectId` + `effectiveFromAcademicYearId`.

---

## 4. Mapping Legacy → Smartbag

| Legacy | Smartbag |
|--------|----------|
| `prelimreport_igcse.php?classid=<b64>` | `POST /api/v1/reports` `reportType=CAMBRIDGE_IGCSE` |
| `prelimreport_aslevel.php?classid=<b64>` | `POST /api/v1/reports` `reportType=CAMBRIDGE_ASLEVEL` |
| `prelimreport_alevel.php?classid=<b64>` | `POST /api/v1/reports` `reportType=CAMBRIDGE_ALEVEL` |
| `*_copy-of-report.php` | varian "Copy of Report" (FE) |
| `mid_gpa.php?classid=<b64>` | `reportType=GPA` + term |
| `gpa.php?classid=<b64>` (June) | `reportType=GPA` + term 2 |
| `gpa_nov.php?classid=<b64>` (Nov) | `reportType=GPA` + term 4 |

> Catatan: legacy memakai `classid` (base64 class id) sebagai param; smartbag memakai `studentReportId` (per siswa). Level class (JC2/Sec4/JC1) menentukan `reportType` yang tersedia per term — lihat `cambridgeReportTypesForTerm.js` di FE.
