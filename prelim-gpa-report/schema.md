---
feature: Prelim Report & GPA June/Nov
slug: prelim-gpa-report
---

# Schema — Prelim Report & GPA June/Nov

## Ringkasan

Fitur ini sudah terimplementasi di smartbag — data dibaca dari tabel yang sudah ada di `api_nest`. Tidak ada tabel baru yang diperlukan.

---

## 1. Prelim Report — Data Model

### student_cambridge_grade

Tabel utama untuk nilai prelim (Cambridge IGCSE/AS/A Level).

| Column | Tipe | Keterangan |
|--------|------|------------|
| id | integer (PK) | |
| student_id | integer (FK) | Relasi ke student |
| subject_year_id | integer (FK) | Relasi ke subject_year |
| syllabus_component_id | integer (FK) | Relasi ke syllabus component |
| component_name | varchar | Nama komponen paper |
| cambridge_programme | enum | IGCSE / ASLEVEL / ALEVEL |
| raw_mark | float | Nilai mentah |
| max_mark_snapshot | float | Nilai maksimum |
| grade_type | enum | **PRELIMINARY** / ACTUAL / PREDICTED |

Sumber: `src/modules/student-cambridge-grade/entities/student-cambridge-grade.entity.ts`

### student_grade

Menyimpan PUM (Pure Unit Mark) dan predicted grade.

| Column | Tipe | Keterangan |
|--------|------|------------|
| id | integer (PK) | |
| student_id | integer (FK) | |
| subject_year_id | integer (FK) | |
| **pum** | int | PUM aktual |
| **predicted_pum** | int | Predicted PUM (untuk GPA term != 2) |
| **as_pum** | int | AS PUM |
| **as_predicted_pum** | int | AS Predicted PUM (untuk GPA term = 2) |
| grade_mark | varchar | Grade mark (contoh: "A", "B") |
| predicted_grade_mark | varchar | Predicted grade mark |
| predicted_final_grade | float | Predicted final grade |

Sumber: `src/modules/student-grade/entities/student-grade.entity.ts`

### grade_moderation

Menyimpan hasil moderation nilai prelim per subject.

| Column | Tipe | Keterangan |
|--------|------|------------|
| id | integer (PK) | |
| academic_year_id | integer (FK) | |
| subject_id | integer (FK) | |
| moderation_value | float | Nilai moderation |
| actual_moderation_value | float | Nilai moderation aktual |
| max_raw_mark | float | Nilai maksimum mentah |
| applied_at | timestamp | Waktu diterapkan |
| applied_by_id | integer (FK) | User yang menerapkan |

Sumber: `src/modules/grade-moderation/entities/grade-moderation.entity.ts`

### Tabel Pendukung Prelim (dibuat oleh PrelimsMigration)

| Tabel | Keterangan |
|-------|------------|
| `grade_schema` | Skema grade (jsonb `schema`), enum cambridge_programme (IGCSE/ASLEVEL/ALEVEL) |
| `syllabus` | Syllabus per subject-year, linked ke grade_schema |
| `syllabus_component` | Komponen paper (key enum: Components/Weight/Total Marks/a*-g) |
| `threshold` | Grade boundaries (jsonb `avg_value`), effective_from/to |
| `threshold_config` | Mapping threshold → syllabus |

---

## 2. GPA Report — Data Model

### gpa_grading_scale

| Column | Tipe | Keterangan |
|--------|------|------------|
| id | integer (PK) | |
| config | jsonb | GP value per grade: `{"A*": 4.0, "A": 4.0, "B": 3.5, ...}` |

Sumber: `src/modules/secondary-gpa-grading-scale/entities/gpa-grading-scale.entity.ts`

### gpa_subject_unit

| Column | Tipe | Keterangan |
|--------|------|------------|
| id | integer (PK) | |
| master_level_id | integer (FK) | Level pendidikan |
| subject_id | integer (FK) | Subject |
| unit | float | Bobot unit (contoh: 1.0, 0.5) |
| effective_from_academic_year_id | integer (FK) | AY mulai berlaku |

Sumber: `src/modules/secondary-gpa-subject-unit/entities/gpa-subject-unit.entity.ts`

---

## 3. Report Generation — Entity

### report (student_report)

| Column | Tipe | Keterangan |
|--------|------|------------|
| id | uuid (PK) | |
| student_report_id | string (FK) | Relasi ke StudentReport |
| term | integer | 1-4 |
| report_type | enum | CAMBRIDGE_IGCSE / ASLEVEL / ALEVEL / GPA / dll |
| html | text | HTML hasil render template |
| report_url | varchar | URL PDF (jika diupload) |
| report_status | enum | Status kompilasi |
| grade_snapshot | jsonb | Snapshot nilai saat generate |
| weighted_average | float | Rata-rata tertimbang |

Sumber: `src/modules/report/`

---

## 4. Kalkulasi

### Prelim Report — PUM & Grade

```
StudentCambridgeGrade.getBySubjectYear(gradeType=PRELIMINARY)
  → scaledTotal → gradeMark (dari threshold boundaries)
  → PUM = calculatePUM(scaledTotal, thresholdType, gradeSchema)

Dua sumber PUM:
  - pumAfterModeration: dari GradeModeration + paperModeration
  - pumGradebook: dari getBySubjectYear (teacher gradebook)
Report pilih pumAfterModeration jika ada, fallback ke pumGradebook
```

### GPA Report — GP Value × Unit

```
per subject dalam class-year history (slice 0-4):
  gpaNumerator += gpValue * (unit || 1.0)
  gpaDenominator += unit || 1.0

GPA = gpaNumerator / gpaDenominator

PUM source:
  term == 2 → asPredictedPum (AS predicted PUM)
  term != 2 → predictedPum
```

---

## 5. Key Enums

| Enum | Values | Lokasi |
|------|--------|--------|
| `ReportTypeEnum` | `CAMBRIDGE_IGCSE=10`, `CAMBRIDGE_ASLEVEL`, `CAMBRIDGE_ALEVEL`, `GPA=13` | `src/types/enums/index.ts` |
| `ThresholdTypeEnum` | `IGCSE_ALEVEL`, `AS_PRELIM` | `src/types/enums/threshold-type.ts` |
| `CambridgeProgrammeTypeEnum` | `IGCSE`, `ASLEVEL`, `ALEVEL` | `src/types/enums/index.ts` |
| `StudentCambridgeGradeGradeTypeEnum` | `ACTUAL`, `PREDICTED`, `PRELIMINARY` | `src/types/enums/student-cambridge-grade-grade-type.ts` |
| `SveExamTypeEnum` | `MYE`, `FYE`, `AS_PRELIM`, `IGCSE_A2_PRELIM`, `BMT1`, `BMT2` | `src/types/enums/sve-exam-type.ts` |