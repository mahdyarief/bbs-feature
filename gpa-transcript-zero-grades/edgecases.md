# Edge Cases — GPA Transcript Zero Grades Fix

> **Status: DECIDED** — semua keputusan dipilih 2026-08-05, konsisten dengan implementation.patch.

## EC-01: Multiple Active Thresholds for Same Subject/Year
**Scenario:** Database contains more than one ACTIVE threshold record for the same `subjectId` where `effectiveFrom <= year <= effectiveTo`.

| Opsi | Behavior |
|------|----------|
| (A) Return first match | `getOne()` returns whichever record the DB returns first (non-deterministic) |
| (B) Return latest created | Add `ORDER BY createdAt DESC` to get the most recently created threshold |
| (C) Throw error | Log a warning and return the first match, or throw a duplicate data error |

**Decision:** **(B) Return latest created** — tambah `ORDER BY createdAt DESC` di `ThresholdService.findEffectiveForSubjectYear` (sudah masuk implementation.patch File 1). Pilihan deterministik (threshold aktif terbaru menang), tidak mengubah alur aktivasi yang ada.

---

## EC-02: Academic Year Format is Unexpected
**Scenario:** `AcademicYear.year` is not in `"YYYY/YYYY"` format (e.g., `"2022"`, `"2022-2023"`, or empty string).

| Opsi | Behavior |
|------|----------|
| (A) Return null | `parseInt` returns NaN, method returns null, grade defaults to '0' |
| (B) Fallback to academicYearId | Use the raw `academicYearId` integer as the year for comparison |
| (C) Throw error | Raise an exception to make the data issue visible |

**Decision:** **(B) Fallback to academicYearId** — sudah diimplementasikan di method (baris `const effectiveYear = academicYear?.year ? parseInt(academicYear.year.split('/')[0], 10) : academicYearId;`). Jika format tidak sesuai, gunakan `academicYearId` sebagai tahun integer langsung.

---

## EC-03: No Threshold Record Matches the Date Range
**Scenario:** No threshold exists where `effectiveFrom <= year <= effectiveTo` for the given subject (gap in threshold coverage).

| Opsi | Behavior |
|------|----------|
| (A) Return null silently | Grade defaults to '0' in the transcript (current behavior for missing data) |
| (B) Fallback to nearest threshold | Find the closest threshold by `effectiveFrom` or `effectiveTo` |
| (C) Log warning + return null | Log a warning for admin visibility, return null, grade defaults to '0' |

**Decision:** **(A) Return null silently** — behavior parity dengan kode existing. Jika tidak ada threshold yang match, grade default ke '0'. Tidak ada logging tambahan untuk menghindari noise di log production.

---

## EC-04: avgValue Contains Invalid or Missing Grade Keys
**Scenario:** `threshold.avgValue` (jsonb) has keys that don't match `GRADE_ORDER_IGCSE` or `GRADE_ORDER_AS` (e.g., `{pass: 50, fail: 0}` instead of `{a_star: 90, a: 80, ...}`).

| Opsi | Behavior |
|------|----------|
| (A) calculatePUM handles it | `calculatePUM` filters `GRADE_ORDER` by keys present in boundaries — non-matching keys are ignored, may result in all grades mapping to same value |
| (B) Validate on threshold save | Add validation when threshold is created/updated to ensure avgValue contains valid grade keys |
| (C) Both A and B | Rely on runtime fallback + add save-time validation to prevent bad data |

**Decision:** **(A) calculatePUM handles it** — function sudah ada filter `VALID_GRADE_KEYS` (cambridge-grading.ts:9). Key yang tidak valid diabaikan, hanya key yang match dengan `GRADE_ORDER_IGCSE`/`GRADE_ORDER_AS` yang dipakai. Tidak perlu validasi tambahan di threshold save.

---

## EC-05: Threshold Exists but avgValue is Empty Object
**Scenario:** A matching ACTIVE threshold is found but `avgValue = {}` (empty jsonb object).

| Opsi | Behavior |
|------|----------|
| (A) Skip conversion | `boundaries` is `{}` (truthy), but `calculatePUM` finds no matching grade entries — returns undefined |
| (B) Treat as null | Check `Object.keys(boundaries).length > 0` before passing to `calculatePUM` |
| (C) Both A and B | Same runtime behavior as A, but explicit check makes intent clearer |

**Decision:** **(B) Treat as null** — sudah diimplementasikan di implementation.patch File 2c: `Object.keys(boundaries).length > 0` di guard. Meskipun `{}` truthy, check eksplisit membuat intent jelas dan mencegah `calculatePUM` dipanggil dengan boundaries kosong.

---

## EC-06: academicYearId is NULL on Threshold Record
**Scenario:** The threshold record has `academicYearId = NULL` but valid `effectiveFrom`/`effectiveTo` values.

| Opsi | Behavior |
|------|----------|
| (A) Works correctly | The new query uses `effectiveFrom`/`effectiveTo` only, doesn't depend on `academicYearId` — this is the expected scenario |
| (B) N/A | This is the primary case the fix addresses, not an edge case |

**Decision:** **(A) Works correctly** — ini adalah skenario utama yang di-fix, bukan edge case. Query baru menggunakan `effectiveFrom`/`effectiveTo` saja, tidak depend on `academicYearId`. Record dengan `academic_year_id = NULL` (86 record) akan ter-match dengan benar.

---

## EC-07: Student's ClassYear Has No AcademicYear Relation Loaded
**Scenario:** `classYear.academicYear` is undefined because the relation wasn't eager-loaded or joined in the query.

| Opsi | Behavior |
|------|----------|
| (A) Use academicYearId directly | Pass `academicYearId` to `findEffectiveForSubjectYear`, which fetches AcademicYear internally |
| (B) Ensure relation is loaded | Add `relations: { academicYear: true }` to the classYear query upstream |
| (C) Both A and B | Method fetches AcademicYear internally (self-contained), but also ensure upstream loads the relation for other uses |

**Decision:** **(A) Use academicYearId directly** — `findEffectiveForSubjectYear` fetch AcademicYear sendiri via `AcademicYear.getRepository().findOne({ where: { id: academicYearId } })`. Method self-contained, tidak butuh relation loaded di upstream.

---

## EC-08: Threshold Type Mismatch
**Scenario:** The correct threshold exists but has a different `thresholdType` than expected (e.g., IGCSE threshold exists but student needs AS_PRELIM).

| Opsi | Behavior |
|------|----------|
| (A) Return null | No match found for the specified type, grade defaults to '0' |
| (B) Fallback to any type | If no match for the specific type, try without type filter |
| (C) Log + return null | Log the mismatch for debugging, return null |

**Decision:** **(A) Return null** — jika tidak ada threshold dengan type yang sesuai, return null, grade default ke '0'. Tidak ada fallback ke type lain karena AS_PRELIM dan IGCSE_ALEVEL memiliki grade schema berbeda dan tidak interchangeable.

---

## EC-09: Threshold Exists but No Syllabus Matches
**Scenario:** A valid ACTIVE threshold with non-empty `avgValue` is found for the subject/year, but `syllabusService.findEffectiveForSubjectYear` returns null (no syllabus record covers the effective range).

| Opsi | Behavior |
|------|----------|
| (A) Skip conversion | Keep old guard `syllabus && boundaries` — grade defaults to '0' even though `avgValue` is valid (the bug we want to fix) |
| (B) Convert on threshold alone | Relax guard to `boundaries`-only (non-empty). `calculatePUM` receives `gradeSchema = undefined` → `getPassingGradeEntries` falls back to `GRADE_ORDER_IGCSE`/`GRADE_ORDER_AS` filtered by keys present in `avgValue` (cambridge-grading.ts:53-69). Grade shows a converted value, not '0' |
| (C) Log + skip | Log a warning and skip conversion — keeps the '0' but adds visibility |

**Decision:** **(B) Convert on threshold alone** — diimplementasikan di implementation.patch File 2c: guard menjadi `boundaries && Object.keys(boundaries).length > 0`, `syllabus` hanya optional. Saat syllabus null, `syllabus?.gradeSchema` jadi `undefined` dan `calculatePUM` fallback ke `GRADE_ORDER` (range grade melebar ke semua valid keys di `avgValue`). Konversi tetap jalan selama `avgValue` non-empty; tidak ada grade yang jatuh ke '0' hanya karena syllabus tidak match.
