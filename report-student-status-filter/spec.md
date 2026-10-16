---
title: # Report — Filter roster murid: AY berjalan hanya murid active, AY lampau tanpa filter status
status: open
severity: major
product: BBS LMS
portal: Admin
author: arielwirawan
date: 2026-10-16
jam:
---

# Report — Roster murid dibedakan AY berjalan vs AY lampau

## Summary

Task update perilaku pemilihan murid pada Report Nasional, GPA transcript, remark report, dan roster FE report viewer.

**Aturan baru:**

- **AY berjalan:** hanya murid dengan status active (`activeStatus: ACTIVE`) yang diikutkan.
- **AY sebelumnya (lampau):** tanpa memandang status active/inactive — selama murid tersebut pernah berada di kelas tersebut (relasi `classYears`) dan punya nilai, laporan tetap bisa di-generate.

Saat ini semua alur memfilter `activeStatus: ACTIVE` tanpa syarat (maupun tanpa filter sama sekali di jalur GPA), sehingga murid non-active di AY lampau tidak muncul / tidak bisa di-generate.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | |
| Email (Jam account) | |
| Jam author ID | |
| User ID (dari console log, mis. `selfUser`) | |
| Portal URL | |
| Environment API | |
| Data konteks (class ID / AY / tanggal) | |
| Browser / OS | |

---

## Perilaku Saat Ini vs Ekspektasi

| Skenario | Saat ini | Ekspektasi |
|----------|----------|------------|
| Report untuk AY berjalan | Filter `activeStatus: ACTIVE` (BE+FE) | ✅ Sama — hanya murid active |
| Report untuk AY lampau — murid sudah non-active | Tidak muncul / tidak bisa di-generate (terhalang filter `ACTIVE` di BE & FE) | ✅ Tetap muncul & di-generate, selama pernah di kelas itu dan punya nilai |
| Report untuk AY lampau — guard `ClassYear` INACTIVE | Error `NoClassYearFoundError` | ✅ Report AY lampau tetap jalan |

---

## Titik-Titik Kode yang Terlibat (perlu diubah)

### 1. National Report — filter `ACTIVE` tanpa syarat — `national-report.service.ts:33-43`

```typescript
// L33-34 — guard ini memblokir AY lampau jika ClassYear sudah INACTIVE
if (!classYear || classYear.activeStatus !== StatusTypeEnum.ACTIVE) {
  NoClassYearFoundError();
}

// L37-43 — filter ACTIVE tanpa membedakan AY
const students = await Student.find({
  where: {
    classYears: { id: classYearId },
    activeStatus: StatusTypeEnum.ACTIVE,
  },
  order: { fullName: 'ASC' },
});
```

Tidak ada branching AY berjalan vs lampau di `src/modules/national-report/` maupun `src/modules/student-report/` (grep-confirmed). Relasi membership: `class-year.entity.ts:83-87` (`@ManyToMany Student students + @JoinTable()`), invers di `student.entity.ts:154-157` (`classYears`).

Catatan: filter `activeStatus` pada `employee` di `getNationalReportContext` (`national-report.helper.ts:218-230`) tetap dipertahankan — itu untuk kepala sekolah/karyawan, bukan murid.

### 2. Remark report — filter in-memory tanpa syarat — `remark-report.helper.ts:17-19`

```typescript
const students = classYear?.students?.filter(
  (s) => s.activeStatus === StatusTypeEnum.ACTIVE,
);
```

### 3. GPA report — roster tanpa filter status — `student-report.service.ts:45-46`

Generator `createSecondaryStudentGpaReport(studentReport, term)` (`secondary-gpa-report.helper.ts:28-31`) per-student, nol hit status. Roster dari upstream:

```typescript
// src/modules/student-report/student-report.service.ts:45-46
const classYearStudentIds = classYear.students.map(({ id }) => id);
if (!classYearStudentIds.includes(student.id)) StudentNotInClassYearError();
```

Tanpa filter status — jadi untuk AY lampau jalur ini sudah benar (ikut relasi `classYear.students`). Yang perlu dipastikan: untuk AY berjalan roster-nya juga hanya murid active, dan referensi `secondary-gpa-report.helper.ts:338` (`1/${classYear.students.length}`) konsisten dengan aturan AY yang sama. Call site tunggal: `src/helpers/create-report.helper.ts:94`.

### 4. FE report viewers — hardcode `activeStatus: "ACTIVE"` — `ClassReportViewer.jsx`

Roster murid semua report dari `GET /students` via `getStudentsV2` dengan `activeStatus: "ACTIVE"` hardcoded:
- Admin: `client/src/views/classReports/ClassReportViewer.jsx:275`
- Teacher: `client-teacher/src/views/classrooms/viewer/ClassReportViewer.jsx:233`

Payload National Report (`GET /nationalReports/:classYearId`) dan GPA (`GET /secondary-gpa-reports/report-terms/...`) tidak membawa param status — roster sepenuhnya dari call `/students` ini. Tidak ada dropdown status di flow report; `STUDENT_STATUS_ENUM` (`client/src/enums.js:240`) hanya dipakai di student management.

---

## Mekanisme AY berjalan

- Admin: `academicYearId` query param wajib + Select AY di `ClassReports.jsx` (baris 109, 136-148, 251-266)
- Teacher: AY diturunkan implisit dari route param `classYearId`
- Tidak ada kolom `isCurrent` di `AcademicYear`. Helper: `src/helpers/get-current-academic-year-helpers.ts` — `getCurrentAcademicYear()` (rentang tanggal `term1_start_date <= now <= term4_end_date` + fallback), bandingkan dengan `classYear.academicYearId`.
- Seorang murid "active" untuk AY berjalan = `activeStatus: StatusTypeEnum.ACTIVE` (1). Untuk AY lampau: abaikan `activeStatus` / `studentStatus` sepenuhnya — cukup keanggotaan kelas + punya nilai.

---

## Arah Perbaikan

1. **BE National Report** — jadikan filter `national-report.service.ts:40` kondisional: hanya terapkan `activeStatus: ACTIVE` jika `classYear.academicYearId == getCurrentAcademicYear().id`; untuk AY lampau tanpa filter status. Evaluasi guard `activeStatus` di L33-34 agar report AY lampau tidak gagal total.
2. **BE remark report** — `remark-report.helper.ts:17-19`: branching sama — AY berjalan filter ACTIVE, AY lampau tanpa filter.
3. **BE GPA** — pastikan roster `classYear.students` (`student-report.service.ts:45-46`) dan `classYear.students.length` (`secondary-gpa-report.helper.ts:338`) mengikuti aturan AY yang sama.
4. **FE** — ganti hardcode `activeStatus: "ACTIVE"` di `ClassReportViewer.jsx:275` (admin) dan `:233` (teacher) dengan parameter yang mencerminkan AY yang dipilih (AY berjalan → ACTIVE, AY lampau → tanpa filter status).

---

## Referensi enum & entity

- `src/types/enums/status-type.ts:1-4` — `StatusTypeEnum`: ACTIVE = 1, INACTIVE
- `src/types/enums/student-status-type.ts:1-8` — `StudentStatusTypeEnum`: CURRENT = 1, ALUMNI, TRANSFERRED, WITHDRAWN, PROSPECT, ADMISSION (tidak dipakai sebagai kriteria filter pada task ini)
- `src/modules/student/entities/student.entity.ts:106-114` — `activeStatus: StatusTypeEnum` (nullable), `studentStatus: StudentStatusTypeEnum` (default PROSPECT)
