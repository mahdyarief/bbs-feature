---
title: GPA Report JC2 — Penilaian memakai AS bukan A Level
status: open
severity: major
product: BBS LMS
portal: Admin
author: System Analyst
date: 2026-10-19
jam: (tidak tersedia — laporan berbasis investigasi DB + code review)
---

# GPA Report JC2 — Penilaian memakai AS bukan A Level

## Summary

GPA Report untuk **JC 2** menghitung `finalMark` mapel Cambridge memakai konteks threshold **AS_PRELIM** (`ThresholdTypeEnum.AS_PRELIM`), padahal JC2 dinilai dengan standar **A Level** (`ThresholdTypeEnum.IGCSE_ALEVEL`). Akibatnya `calculatePUM()` memetakan skor mentah ke skala AS (band 80→90), bukan skala A Level (band 90→100 untuk a*, lalu 80→90) — nilai akhir dan GPA JC2 terdistorsi.

Kasus pembanding: student **21416**, class **JC 2 Mendel** (`class_year_id = 100392`, AY 26). 6 mapel akademik Cambridge (Chemistry, Computer Science, English General Paper, Mathematics, Physics, Chinese Level JC2 Special) saat ini memakai jalur prelim (`predicted_pum` / `as_predicted_pum`) atau — bila lewat jalur threshold — selalu `AS_PRELIM` untuk `masterLevelId` 15/16/17. Dihitung ulang dengan standar A Level (skor SA mentah term 2, tanpa PUM prelim), GPA akademik turun dari **~3.44 → ~3.29**.

**Ekspektasi:** GPA Report JC2 (master level JC2) memakai konteks **A Level** — baik untuk pemetaan threshold `calculatePUM()` maupun sumber nilai prelim yang dipakai sebagai `finalMark`.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | System Analyst (investigasi code + DB) |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | — |
| Portal URL | Report GPA — class JC 2 Mendel |
| Environment API | DB `binabangsa_prod_mig_v01` via `db-access-tools` |
| Data konteks (class ID / daId / tanggal) | classYearId=100392 (JC 2 Mendel, academic_year 26, master_level 17); student_id=21416; term=2 |
| Browser / OS | — |

---

## Steps to Reproduce

1. Generate GPA Report untuk class **100392** (JC 2 Mendel), student **21416**, term 2.
2. Perhatikan `finalMark` 6 mapel akademik Cambridge — nilainya berasal dari `student_grade.predicted_pum` / `as_predicted_pum` (jalur prelim), atau bila lewat jalur threshold dipetakan dengan `AS_PRELIM`.
3. Bandingkan dengan skor SA mentah term 2 (`grade.exam_type = 11/SA`, `term = 2`) — selisih menunjukkan distorsi skala AS vs A Level (contoh: Chemistry SA 62 vs prelim 79; Physics SA 69 vs prelim 66).
4. Hitung ulang GPA dengan standar A Level (tabel di bagian Bukti) — hasil berbeda signifikan (~3.44 vs ~3.29).

**Actual Result:**
- `secondary-gpa-report.helper.ts` baris 249-253 memaksa `AS_PRELIM` untuk semua `masterLevelId` 15/16/17 (termasuk JC2).
- Mapel prelim Cambridge (`cambridgeProgramme !== NONE`) memakai `predictedPum`/`asPredictedPum` langsung sebagai `finalMark`, tanpa pembanding `Math.max(skor, PUM)`.

**Expected Result:**
- JC2 memakai konteks **A Level** (`ThresholdTypeEnum.IGCSE_ALEVEL`) di `calculatePUM()`.
- Sumber nilai prelim yang dipakai untuk JC2 adalah prediksi **A Level** (bukan AS), atau — bila desain menghendaki skor mentah — `finalMark = Math.max(skor SA/FYA, PUM A Level)` konsisten dengan mapel non-prelim.

---

## Root Cause Analysis

### Bug #1 — JC2 dipaksa ke `AS_PRELIM` — `secondary-gpa-report.helper.ts` (line 249-253)

File: `api_nest/src/helpers/reports/secondary-gpa-report.helper.ts`

```ts
// path:249-253 — JC (15/16/17) selalu AS_PRELIM, Sec memakai IGCSE_ALEVEL
? ThresholdTypeEnum.AS_PRELIM // JC
: ThresholdTypeEnum.IGCSE_ALEVEL, // Sec
```

Dengan kondisi penuh:

```ts
const usEquivalent = boundaries
  ? calculatePUM(
      subjectAssessmentGrade.score,
      boundaries,
      [15, 16, 17].includes(
        subjectYearBanding.subjectYear.classYear.masterLevelId,
      )
        ? ThresholdTypeEnum.AS_PRELIM // JC
        : ThresholdTypeEnum.IGCSE_ALEVEL, // Sec
      syllabus?.gradeSchema,
    )
  : null;
```

`calculatePUM()` di `api_nest/src/helpers/grading/cambridge-grading.ts:72-124` memetakan berbeda untuk dua konteks ini: band AS 80→90 vs band A Level 90→100 (a*) lalu 80→90. Jadi satu skor mentah yang sama menghasilkan PUM berbeda — JC2 yang seharusnya A Level justru dinilai dengan kurva AS.

### Bug #2 — Mapel prelim bypass pembanding skor-vs-PUM — `secondary-gpa-report.helper.ts` (line 184-201, 258-262)

```ts
// path:184-201 — prelim langsung pakai predicted PUM
subjectFinalMark =
  term == 2
    ? studentGrade?.asPredictedPum
    : studentGrade?.predictedPum;
```

Sedangkan mapel non-prelim memakai pembanding:

```ts
// path:258-262
subjectFinalMark = Math.max(
  subjectAssessmentGrade.score || 0,
  usEquivalent || 0,
  0,
);
```

Tidak ada fungsi pembanding FYE-vs-PUM di helper ini (FYE = `GradeExamTypeEnum.FYE = 4` tidak dipakai sama sekali di file ini; pembanding yang ada adalah **FYA/SA vs PUM**). FYE hanya dipakai sebagai display `fyeScore` di `secondary-semester-report.helper.ts:234-236,309-311`. Untuk JC2, `finalMark` prelim = kolom prediksi apa adanya — bila kolom itu berisi prediksi AS, maka seluruh GPA JC2 berbasis AS.

---

## Bukti dari Database

Diambil dari `binabangsa_prod_mig_v01` (class 100392, student 21416, term 2):

| Mapel (Cambridge prelim) | SA mentah (exam_type=11, term 2) | predicted_pum | as_predicted_pum → finalMark saat ini |
|---|---|---|---|
| Chemistry | 62 | 79 | 79 |
| Computer Science | 68 | 68 | 68 |
| English General Paper | 65 | 46 | 46 |
| Mathematics | 72 | 72 | 72 |
| Physics | 69 | 66 | 66 |
| Chinese Level JC2 Special | 75 | 75 | 75 |

Perbandingan GPA akademik (6 mapel, bobot unit = 1, skala grading AY26):

| Jalur | Sumber finalMark | Rata-rata finalMark | GPA |
|---|---|---|---|
| Saat ini (prelim/AS) | `as_predicted_pum` | 67.67 | **~3.44** |
| A Level (skor mentah) | SA term 2 mentah | 68.50 | **~3.29** |

Catatan: threshold AY26 untuk silabus terkait (9701/9618/8021/9709/9702) **tidak ditemukan** (hanya AY23/AY25 yang aktif), sehingga pada jalur A Level non-prelim `usEquivalent = null` dan `finalMark = skor mentah`. Perbedaan GPA di atas murni dari perbedaan sumber nilai (prediksi AS vs skor mentah), sebelum efek kurva PUM A Level dihitung.

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend report helper | `api_nest/src/helpers/reports/secondary-gpa-report.helper.ts:249-253` | JC2 memakai kurva AS di `calculatePUM()` |
| Backend report helper | `api_nest/src/helpers/reports/secondary-gpa-report.helper.ts:184-201` | Prelim JC2 memakai prediksi AS sebagai finalMark |
| Grading util | `api_nest/src/helpers/grading/cambridge-grading.ts:72-124` | Perilaku `calculatePUM` per `ThresholdTypeEnum` (tidak perlu diubah, hanya konteks pemanggil) |
| Data pendukung | `threshold` (AY26 untuk silabus JC2), `student_grade.predicted_pum` | Perlu dipastikan tersedia versi A Level bila jalur prelim dipertahankan |

---

## Proposed Solution Options

### Option A: Petakan JC2 ke `IGCSE_ALEVEL` + sumber prelim A Level (Recommended)

1. Ubah kondisi baris 249-253 agar hanya JC1 (atau level yang memang AS) memakai `AS_PRELIM`, sedangkan JC2 (`masterLevelId = 17`) memakai `IGCSE_ALEVEL`:

```ts
[15, 16].includes(masterLevelId)
  ? ThresholdTypeEnum.AS_PRELIM
  : ThresholdTypeEnum.IGCSE_ALEVEL, // JC2 + Sec
```

(Alternatif setara: `masterLevelId === 17 ? IGCSE_ALEVEL : AS_PRELIM` untuk seluruh JC bila JC1 juga dinilai A Level — perlu konfirmasi kurikulum.)

2. Pastikan `student_grade.predicted_pum` / `as_predicted_pum` yang dibaca untuk JC2 berisi prediksi **A Level**; bila kolom tersebut memang prediksi AS, ganti sumbernya (atau samakan perlakuan dengan non-prelim: `Math.max(skor, PUM A Level)`).
3. Pastikan `threshold` AY berjalan tersedia untuk silabus JC2 (9701, 9618, 8021, 9709, 9702, 8681) dengan `statusType = ACTIVE`, agar `usEquivalent` tidak null.

### Option B: Samakan prelim dengan non-prelim (`Math.max`)

Hapus jalur khusus prelim (baris 184-201) sehingga semua mapel akademik memakai `Math.max(skor FYA/SA, PUM)` dengan konteks threshold yang benar per level. Lebih konsisten, tetapi mengubah perilaku lebih luas (berdampak ke JC1/Sec juga) — perlu regresi penuh.

---

## Notes

- Enum: `GradeExamTypeEnum` (`api_nest/src/types/enums/grade-exam-type.ts`) — TT=1, BMT=2, MYE=3, FYE=4, DCA=5, VT=6, IC=7, PU=8, CA=9, CAT=10, SA=11, FYA=12, MEDAL=13, AA=14. FYE tidak dipakai di helper GPA.
- Enum: `ThresholdTypeEnum` — `AS_PRELIM` vs `IGCSE_ALEVEL`; `IThresholdStatusType` (`threshold-status-type.ts`) — DRAFT=0, ACTIVE=1. Query threshold wajib filter `statusType = ACTIVE` (sudah ada di kode baris 238).
- `CambridgeProgrammeTypeEnum` (`cambridge-programme-type.ts`) — IGCSE=1, ASLEVEL, ALEVEL, NONE.
- Perlu konfirmasi kurikulum: apakah JC1 tetap AS (maka Option A dengan `[15,16]` tepat) atau seluruh JC memakai A Level.
- Query verifikasi:

```sql
-- skor SA mentah term 2, student 21416, class 100392
SELECT s.name, g.term, g.exam_type::text AS exam, g.score
FROM student_grade sg
JOIN subject_year sy ON sg.subject_year_id = sy.id
JOIN subject s ON sy.subject_id = s.id
LEFT JOIN grade g ON g.student_grade_id = sg.id
WHERE sy.class_year_id = 100392 AND sg.student_id = 21416
  AND g.term = 2 AND g.exam_type::text = '11'
ORDER BY s.name;

-- prediksi prelim
SELECT s.name, sg.predicted_pum, sg.as_predicted_pum
FROM student_grade sg
JOIN subject_year sy ON sg.subject_year_id = sy.id
JOIN subject s ON sy.subject_id = s.id
WHERE sy.class_year_id = 100392 AND sg.student_id = 21416
ORDER BY s.name;

-- threshold aktif silabus JC2 per AY
SELECT t.academic_year_id, s.name, s.syllabus_code, t.status_type::text
FROM threshold t JOIN subject s ON t.subject_id = s.id
WHERE s.syllabus_code IN ('9701','9618','8021','9709','9702','8681')
ORDER BY s.syllabus_code, t.academic_year_id DESC;
```
