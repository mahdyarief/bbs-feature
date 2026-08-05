---
feature: GPA Transcript Zero Grades Fix
slug: gpa-transcript-zero-grades
status: draft
author: arielwirawan
date: 2026-08-05
target_release: TBD
---

# GPA Transcript Zero Grades Fix

## Overview

Fix bug where GPA transcript PDF shows '0' for most grades despite the database containing actual grade data. The root cause is that the report generation logic in `secondary-gpa-report.helper.ts` looks up syllabus and threshold records using `academicYearId` as a direct foreign key, but the `syllabus` table stores **all 166 records with `academic_year_id = NULL`** (using `effective_from`/`effective_to` integer year ranges instead), so `Syllabus.findOne({ academicYearId })` always returns null. Since the `calculatePUM` conversion is gated on `syllabus && boundaries` (line 239-243), the conversion is always skipped and grades default to '0'. The fix replaces both direct `findOne` calls with date-range-based lookups: reuse the existing `SyllabusService.findEffectiveForSubjectYear` (syllabus.service.ts:499) and add an equivalent `ThresholdService.findEffectiveForSubjectYear`. The guard is also relaxed to threshold-only: conversion runs whenever `threshold.avgValue` is a non-empty object, so subjects with a valid threshold but no matching syllabus still get converted grades (via `calculatePUM`'s `GRADE_ORDER` fallback) instead of '0'.

## Problem / Motivation

- All GPA transcripts show placeholder '0' values instead of actual student grades
- Academic reports are unusable for grading and assessment
- Students and parents receive incorrect academic information
- The `syllabus` table has 166 records all with `academic_year_id = NULL`, using `effective_from`/`effective_to` instead
- The `threshold` table has 418 records: 332 with `academic_year_id` set (79.4%), only 86 NULL (20.6%) — the threshold lookup alone is NOT the blocker
- The existing `Syllabus.findOne({ where: { academicYearId } })` query at helper line 213 always returns `null` (because ALL syllabi have NULL `academic_year_id`), causing the `syllabus && boundaries` guard to fail, `calculatePUM` to be skipped, and grades to default to '0'

## Scope

### In Scope
- Add `findEffectiveForSubjectYear` method to `ThresholdService` using `effective_from`/`effective_to` date ranges
- Reuse the existing `SyllabusService.findEffectiveForSubjectYear` (syllabus.service.ts:499) for the syllabus lookup
- Update `secondary-gpa-report.helper.ts` to use both services instead of direct `findOne` with `academicYearId`
- Remove dependency on `Syllabus.findOne` with `academicYearId` (both lookups now range-based on `subjectId`)
- Keep passing `syllabus?.gradeSchema` to `calculatePUM` — when the syllabus resolves, the schema filters grade keys (A–C, A*–E); when it does not, `gradeSchema` is `undefined` and `getPassingGradeEntries` falls back to `GRADE_ORDER` filtered by `avgValue` keys
- Relax the guard from `syllabus && boundaries` to `boundaries`-only (non-empty): a valid ACTIVE threshold is sufficient to run `calculatePUM` — the syllabus is optional, contributing only `gradeSchema`

### Out of Scope
- Database migration (no schema changes required)
- Changes to the `calculatePUM` function itself
- Changes to the syllabus lookup for other report types
- Frontend/UI changes

## User Stories

### As a school administrator
I want GPA transcripts to display correct student grades
So that academic reports are accurate and usable for university applications

### As a student
I want my GPA transcript to reflect my actual grades
So that my academic record is correctly represented

## Acceptance Criteria

- [ ] `ThresholdService.findEffectiveForSubjectYear(subjectId, academicYearId)` returns the correct active threshold record by matching `effective_from <= year <= effective_to`
- [ ] `secondary-gpa-report.helper.ts` no longer queries threshold using `academicYearId` as a direct FK
- [ ] GPA transcript PDF shows correct grade values (not '0') for subjects with valid threshold data
- [ ] Subjects with a valid ACTIVE threshold but no matching syllabus still show converted grades (not '0') — `calculatePUM` falls back to `GRADE_ORDER` when `syllabus.gradeSchema` is unavailable
- [ ] No database migration is required
- [ ] Existing threshold creation/activation flows are not affected

## UI / UX Changes

No UI changes. This is a backend-only fix affecting the GPA transcript PDF generation.

### Affected Portals
- [ ] Admin (client/)
- [ ] Student (client-student/)
- [ ] Teacher (client-teacher/)

(No portal changes — PDF output only)

## API Changes

No new endpoints. Internal service method addition only.

| Method | Path | Description |
|--------|------|-------------|
| — | — | No API changes. New internal method `ThresholdService.findEffectiveForSubjectYear` |

## Database Changes

### New Tables
- None

### Modified Tables
- None

### Migrations
- None required. The fix uses existing `effective_from`/`effective_to` columns on the `threshold` table.

## Business Rules / Validation

1. **Effective year extraction**: The academic year string (e.g., `"2022/2023"`) is parsed by taking the first part before `/` to get the integer year (e.g., `2022`). This matches the existing pattern in `threshold.service.ts` line 57: `Number(academicYear.year.split('/')[0])`.
2. **Threshold matching**: A threshold is considered effective for a given academic year if `effectiveFrom <= year <= effectiveTo` AND `statusType = ACTIVE`.
3. **Threshold type filtering**: When the student's `masterLevelId` is 15, 16, or 17, use `ThresholdTypeEnum.AS_PRELIM`; otherwise use `ThresholdTypeEnum.IGCSE_ALEVEL`.
4. **avgValue validation**: With this approach, validation responsibility shifts to the `avg_value` field in threshold records. The `avgValue` (jsonb) must contain valid grade keys (e.g., `{a_star: 90, a: 80, b: 70, ...}`) for `calculatePUM` to correctly map scores to US equivalents.
5. **Syllabus is optional**: The conversion guard is threshold-only (`boundaries` non-empty). The syllabus contributes only `gradeSchema`; when no syllabus matches, `gradeSchema` is `undefined` and `calculatePUM` falls back to `GRADE_ORDER_IGCSE = ['a*', 'a', 'b', 'c', 'd', 'e', 'f', 'g']` or `GRADE_ORDER_AS = ['a', 'b', 'c', 'd', 'e']` filtered by the keys present in `avgValue` (cambridge-grading.ts:53-69).

## Error Handling

| Error | Condition | Behavior |
|-------|-----------|----------|
| No threshold found | `findEffectiveForSubjectYear` returns null | `boundaries` is undefined, `calculatePUM` is skipped, grade defaults to '0' |
| No syllabus found | `syllabusService.findEffectiveForSubjectYear` returns null | Conversion still runs on `threshold.avgValue`; `syllabus?.gradeSchema` is `undefined` → `GRADE_ORDER` fallback (grade range widens to all valid keys in `avgValue`) |
| Invalid academic year format | `academicYear.year` cannot be parsed | `effectiveYear` is NaN, method returns null |
| No academic year record | `academicYearId` doesn't match any record | Falls back to using `academicYearId` as the year integer directly |

## Dependencies

- **Threshold module**: `ThresholdService` in `src/modules/threshold/threshold.service.ts`
- **AcademicYear entity**: `src/modules/academic-year/entities/academic-year.entity.ts` — uses `year` field (string, e.g., `"2022/2023"`)
- **Threshold entity**: `src/modules/threshold/entities/threshold.entity.ts` — `effectiveFrom`, `effectiveTo`, `avgValue`, `subjectId`, `statusType`, `thresholdType`
- **calculatePUM helper**: `src/helpers/grading/cambridge-grading.ts` — handles grade conversion with fallback behavior
- **Secondary GPA report helper**: `src/helpers/reports/secondary-gpa-report.helper.ts` — the file containing the bug
