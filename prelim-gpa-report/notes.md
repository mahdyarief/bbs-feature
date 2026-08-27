---
feature: Prelim Report & GPA June/Nov
slug: prelim-gpa-report
---

# Notes — Prelim Report & GPA June/Nov

## Sumber Data

Feature brief ini didasarkan pada:

1. **Legacy Report Endpoint Inventory** — hasil parsing `class_in_year_ay25.html` dan `class_in_year_content.html` dari menu Class in Year portal admin (`ais_new`). Menemukan 31 endpoint script unik dari 23 class di 2 AY, termasuk Prelim Report Card (IGCSE/AS/A Level) dan GPA (Mid/June/Nov).
2. **Smartbag FE** (`bbs/`) — `client/src/` (admin portal), `client-teacher/src/` (teacher portal), `api/src/` (Express backend). Navigation, component, dan API call mapping.
3. **Smartbag BE** (`api_nest/`) — `src/helpers/reports/` (report generators), `src/modules/` (threshold, grade-moderation, student-cambridge-grade, secondary-gpa), `src/templates/student-reports/` (.hbs templates), `src/database/migrations/` (PrelimsMigration).

## Key Findings

### 1. Konsolidasi Report Type di Smartbag

Legacy punya **6 endpoint terpisah** untuk Prelim + GPA, masing-masing dengan parameter `classid` (base64):

| Endpoint | Level | Term |
|----------|-------|------|
| `prelimreport_igcse.php` | Sec 4, JC1 | — |
| `prelimreport_aslevel.php` | JC2 | — |
| `prelimreport_alevel.php` | JC2 | — |
| `mid_gpa.php` | Semua | — |
| `gpa.php` (GPA June) | JC2 saja | — |
| `gpa_nov.php` (GPA Nov) | JC2 saja | — |

Smartbag mengkonsolidasi semua ke **1 controller** (`POST /api/v1/reports`) dengan parameter `reportType` + `term`:

| reportType | term | Level |
|------------|------|-------|
| `CAMBRIDGE_IGCSE` | 4 | Sec 4, JC1 |
| `CAMBRIDGE_ASLEVEL` | 2 | JC2 |
| `CAMBRIDGE_ALEVEL` | 4 | JC2 |
| `GPA` | 1-4 | Semua (term 2/4 untuk JC2) |

### 2. Timing Logic (FE)

`bbs/client/src/utils/cambridgeReportTypesForTerm.js`:
- Term 2, JC2 → ASLEVEL
- Term 4, JC2 → ALEVEL
- Term 4, Sec4/JC1 → IGCSE
- Selain itu → empty (tidak ada prelim)

### 3. Dual Backend Architecture

Smartbag menggunakan **dua backend** terpisah:
- **`bbs/api` (Express.js)** — CRUD dasar: `POST /reports`, `POST /reports/update`, `POST /studentReports`, `GET /studentReports`
- **`api_nest` (NestJS)** — Semua logika lanjutan: threshold, grade-moderation, student-cambridge-grade, syllabus, gradeSchema, reportSettings, gpaGradingScales, gpaSubjectUnits

### 4. PDF Generation

- **Legacy:** Server-side PHP (`html2pdf`), download langsung
- **Smartbag:** BE render HTML (Handlebars + Puppeteer) → simpan di DB → FE render PDF final (pdfmake) di browser

### 5. Alur Legacy (Moderate → Lock → Generate)

Dari `home.php.html` (teacher portal ASD):
- **Moderate Component Prelim** (`moderate_component_prelim_new.php`) — input nilai prelim per paper
- **Lock Prelim** (`view_lock_prelim.php?dev=asd`) — lock/unlock exam type: **AS Prelim, IGCSE, A Level** (term `lock_as`, `lock_ig`, `lock_a2`)
- Generate report hanya bisa setelah nilai di-lock
- **Status di smartbag:** Lock Prelim belum teridentifikasi setara

## Asumsi yang Perlu Diverifikasi

| # | Asumsi | Dampak jika salah |
|---|--------|-------------------|
| A-1 | GPA Mid/June/Nov dikonsolidasi jadi 1 reportType + term sudah cukup | Sekolah butuh template berbeda per periode |
| A-2 | Lock Prelim tidak diperlukan di smartbag (moderation cukup) | Nilai prelim bisa di-generate sebelum final |
| A-3 | Pembatasan level JC2-saja untuk GPA June/Nov dipertahankan di smartbag | GPA di term 2/4 muncul untuk semua level Secondary |
| A-4 | `cambridgeReportTypesForTerm.js` sudah mencakup semua skenario | Ada level/term lain yang butuh prelim |
| A-5 | Prelim A Level composite (AS overlay) sudah benar diimplementasi | PUM A Level tidak akurat |

## Dependensi & Cross-Reference

- **FEATURE_COMPARISON.md** — gap Grading & Report
- **FEATURE_IMPROVEMENT_RECOMMENDATIONS.md** — rekomendasi fase 1
- **National Report** (`features/national-report/`) — fitur terkait, menggunakan data GP Value × Unit yang sama
- **GPA Calculator** (`gpa_calculation_view.php`) — sumber data GP Value, terhubung ke `secondary-gpa-grading-scale` dan `secondary-gpa-subject-unit`
- **Moderation** (`moderate_component_prelim_new.php`) — terkait dengan data PUM/IGCSE/A Level
