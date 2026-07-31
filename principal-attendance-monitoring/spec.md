---
feature: Principal Attendance Monitoring
slug: principal-attendance-monitoring
status: draft
author: BBS Team
date: 2026-07-31
target_release: TBD
---

# Principal Attendance Monitoring

## Overview

Halaman baru di **teacher portal** (`client-teacher/`) yang memungkinkan **Principal** untuk memonitor daily attendance per class secara **real-time**. Menampilkan daftar class, status complete/incomplete attendance, dan drill-down ke detail per-student. Menggunakan **endpoint API yang sudah ada** — tidak ada perubahan backend signifikan.

## Sebelum Mulai Implementasi

Fitur ini **100% reuse** dari halaman yang sudah ada di admin portal. Sebelum coding, Engineer WAJIB mempelajari implementasi existing berikut untuk memahami pola yang sudah ada:

### 1. Admin Daily Attendance (`bbs/client/src/views/attendance/dailyAttendance/`)
- **File:** `DailyAttendance.jsx`
- Pelajari: struktur tabel (BBSDataTable + kolom), filter (programme, academic year), data source (`fromApi.getDailyAttendancesOnClassYear`)
- Ini adalah **pattern utama** untuk halaman `/attendance-monitoring`

### 2. Admin Attendance Overview (`bbs/client/src/views/attendance/generalOverview/attendanceOverview/`)
- **File:** `AttendanceOverview.jsx`
- Pelajari: bagaimana bottom section menampilkan per-class breakdown, integrasi dengan date picker dan class selection
- Ini akan di-**reuse langsung** di bottom section attendance page

### 3. Admin Discipline Overview (`bbs/client/src/views/attendance/generalOverview/disciplineOverview/`)
- **File:** `DisciplineOverview.jsx` — layout table kiri + side panel kanan, filter programme, term buttons
- **File:** `DetailDiciplineStudent.jsx` — side panel detail student (late dates, absent dates)
- Pelajari: data flow dari klik student → side panel, filter term, classroom name search
- Pattern ini akan di-reuse untuk halaman `/attendance-monitoring/discipline`

### 4. Route Guard (Teacher Portal)
- **File:** `bbs/client-teacher/src/hooks/usePrincipalOrHod.js`
- **File:** `bbs/client-teacher/src/containers/TheContent.jsx`
- Pelajari: bagaimana `requirePrincipalOrHod` guard bekerja, bagaimana membedakan Principal vs HOD
- **Penting:** halaman ini hanya untuk Principal, bukan HOD

### 5. Backend Service Patterns
- **File:** `api_nest/src/modules/daily-attendance/daily-attendance.service.ts` — method `findAllOnClassYear`
- **File:** `api_nest/src/modules/student-attendance-report/student-attendance-report.service.ts` — method `findDiscipline`
- Pelajari: query pattern, filter existing (programmeId, academicYearId), sebagai referensi untuk menambah filter `masterLevelId`

## Problem / Motivation

Principal saat ini tidak memiliki visibility terhadap apakah guru-guru sudah melakukan check attendance setiap hari. Satu-satunya cara adalah masuk ke admin portal atau bertanya manual ke masing-masing guru. Principal membutuhkan dashboard yang bisa menjawab:

1. **Apakah semua kelas sudah check attendance hari ini?**
2. **Kelas mana yang belum/incomplete?**
3. **Siapa homeroom teacher yang bertanggung jawab?**
4. **Bagaimana per-bandigan attendance per class?**

Dengan menempatkan fitur ini di **teacher portal** (yang sudah memiliki Principal/HOD guard), Principal bisa akses tanpa perlu diberikan akses ke admin portal. Academic Year menggunakan **active academic year** secara default (reuse function `getCurrentAcademicYear`).

## Scope

### In Scope

**Page 1: Daily Attendance Monitoring**
- Halaman baru `/attendance-monitoring` di teacher portal
- Table: daftar class year + homeroom teacher + total days + complete count + incomplete count
- Filter: Programme, Level (Master Level), Date
- Bottom section: per-class attendance summary by date (reuse `AttendanceOverview` component)
- Guard: `requirePrincipalOrHod: true` — Principal only (guard name tetap `requirePrincipalOrHod` untuk reuse, tapi akses diberikan hanya untuk Principal/VP)
- Academic Year: **default active academic year** — tidak ada dropdown AY (reuse helper `getCurrentAcademicYear`)
- Navigasi ke detail attendance per class (reuse `/class-in-year/:id/attendance`)
- Filter Level pada BE DTO: tambah `masterLevelId` ke `GetDailyAttendanceOnClassYearDto`

**Page 2: Discipline Monitoring**
- Halaman baru `/attendance-monitoring/discipline` di teacher portal
- Table: daftar student + classroom + late count + absent count + absent without excuse count per term
- Side panel: detail student discipline (late dates, absent dates, counts)
- Filter: Programme, Level (Master Level), Class (classroom name search)
- Term buttons: Term 1-4 (reuse admin behavior, default current term, selectable)
- Reuse `DisciplineOverview` layout dari admin (table kiri + side panel kanan)
- Reuse `DetailDiciplineStudent` component untuk side panel
- Reuse `GET /api/v1/studentAttendanceReports/discipline` endpoint (existing)
- Tambah `masterLevelId` ke `GetStudentAttendanceReportsDto` untuk filter Level

### Out of Scope
- Perubahan UI pada admin portal
- Export / download report
- Notifikasi / alert jika ada class incomplete
- Employee attendance (hanya student attendance)
- Perubahan logic check-in/check-out attendance
- Create tardiness report (reuse admin page, tidak perlu buat baru)

## User Stories

### As a Principal
I want to access the attendance monitoring page
So that I can monitor whether teachers are checking attendance daily

## Acceptance Criteria

### Daily Attendance Monitoring
- [ ] Teacher portal memiliki route `/attendance-monitoring` yang hanya bisa diakses oleh Principal/VP
- [ ] Halaman menampilkan tabel daftar class dengan kolom: Class, Homeroom Teacher, Programme, Total Days, Complete, Incomplete
- [ ] Filter Programme, Level (Master Level), dan Date berfungsi
- [ ] Academic Year default ke active academic year (reuse existing helper)
- [ ] Data complete/incomplete bersumber dari `GET /api/v1/dailyAttendances/onClassYear`
- [ ] Klik "View Detail" pada class → navigate ke `/class-in-year/:id/attendance` di tab baru
- [ ] Bottom section menampilkan attendance summary per-student untuk class dan date yang dipilih (reuse `AttendanceOverview`)
- [ ] Date picker max: today (tidak bisa pilih future date)
- [ ] Empty state muncul jika tidak ada data untuk filter yang dipilih
- [ ] Loading state (skeleton) saat data sedang di-fetch

### Discipline Monitoring
- [ ] Teacher portal memiliki route `/attendance-monitoring/discipline` yang hanya bisa diakses oleh Principal/VP
- [ ] Halaman menampilkan tabel daftar student dengan kolom: Student Name, Classroom, Late, Absent (W/ Excuse), Absent (W/o Excuse)
- [ ] Filter Programme, Level (Master Level), dan Class (classroom name) berfungsi
- [ ] Term buttons (1-4) berfungsi — default ke current term, bisa dipilih manual
- [ ] Academic Year default ke active academic year
- [ ] Data bersumber dari `GET /api/v1/studentAttendanceReports/discipline`
- [ ] Klik nama student → side panel menampilkan detail discipline (late dates, absent dates, counts)
- [ ] Side panel reuse `DetailDiciplineStudent` component
- [ ] Loading state (skeleton) saat data sedang di-fetch

## UI / UX Changes

### Affected Portals
- [ ] Admin (client/)
- [ ] Student (client-student/)
- [x] Teacher (client-teacher/) — **target**

### Halaman `/attendance-monitoring`

**Layout:**

```
┌─────────────────────────────────────────────────────────────┐
│  Attendance Monitoring                                       │
│                                                              │
│  [Programme ▼] [Level ▼] [Date: 31/07/26] │
│                                                              │
│  ┌────────────── Daily Attendance Overview ─────────────────┐│
│  │ Class │ Homeroom │ Programme │ Total │ ✓Complete │ ✗Incomp ││
│  │───────│──────────│───────────│───────│───────────│─────────││
│  │ P1A   │ Ms. Sarah│ Primary   │ 22    │ 20        │ 2       ││
│  │ P1B   │ Mr. John │ Primary   │ 22    │ 22        │ 0       ││
│  │ ...   │ ...      │ ...       │ ...   │ ...       │ ...     ││
│  └──────────────────────────────────────────────────────────┘│
│                                                              │
│  ┌─────────── Class Detail: Primary 1A ─────────────────────┐│
│  │ Date: 31 Jul 2026  |  Total: 25  |  ✓Present: 20  │      ││
│  │                        ⏰Late: 2   |  ✗Absent: 3   │      ││
│  │                                                       │      ││
│  │ No | Student Name | Status | Check-in Time            │      ││
│  │ 1  | Alice       | Present | 07:30                    │      ││
│  │ 2  | Bob         | Late    | 08:15                    │      ││
│  │ 3  | Charlie     | Absent  | -                        │      ││
│  └──────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### Halaman `/attendance-monitoring/discipline`

**Layout:** Reuse admin `DisciplineOverview` layout — table kiri + side panel kanan.
**Filter bar:** [Programme ▼] [Level ▼] [Class: Search...] [Term 1] [Term 2] [Term 3] [Term 4]

```
┌─────────────────────────────────────────────────────────────┐
│  Discipline Overview                                        │
│                                                             │
│  [Programme ▼] [Level ▼] [Class: Search...]                 │
│                                        [T1] [T2] [T3] [T4]  │
│                                                             │
│  ┌────────── Students ───────────────────┐ ┌─ Detail ─────┐│
│  │ Student │ Class │ Late │ Absent W/ │  │ │ Term 1       ││
│  │         │       │      │ Absent W/o│  │ │ John Doe     ││
│  │─────────│───────│──────│───────────│  │ │              ││
│  │ Alice   │ P1A   │ 2    │ 0  │ 1    │  │ │ Late (2)     ││
│  │ Bob     │ P1B   │ 0    │ 1  │ 0    │  │ │ 15 Jul 2026  ││
│  │ ...     │ ...   │ ...  │ ...│ ...   │  │ │ 20 Jul 2026  ││
│  └────────────────────────────────────────┘ │              ││
│                                              │ Absent (1)   ││
│                                              │ 10 Jul 2026  ││
│                                              └──────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### Komponen yang Di-reuse

| Komponen | Dari | Kegunaan |
|----------|------|----------|
| `BBSDataTable` | Admin FE | Tabel daily attendance overview |
| `BBSDatePicker` | Admin FE | Date filter |
| `BBSResourceSelect` | Admin FE | Dropdown Programme, Level |
| `Badge` (complete/incomplete) | Admin FE | Status badge |
| `AttendanceOverview` | Admin FE | Bottom section per-class summary |
| `usePrincipalOrHod` | Teacher FE | Route guard |
| `DisciplineOverview` | Admin FE | Layout + logic discipline page (reuse pattern) |
| `DetailDiciplineStudent` | Admin FE | Side panel detail student discipline |
| `BBSResourceSelect` (classroom) | Admin FE | Filter Class (classroom name search) |

## API Changes

### Endpoint Existing — No New Endpoint

| Method | Path | Description | Reused By |
|--------|------|-------------|-----------|
| GET | `/api/v1/dailyAttendances/onClassYear` | Daftar class year + daily attendance status | Top section table (attendance) |
| GET | `/api/v1/attendances/overview` | Per-class attendance breakdown per date | Bottom section (attendance) |
| GET | `/api/v1/studentAttendanceReports/discipline` | Daftar student + late/absent counts per term | Table discipline page |
| GET | `/api/v1/masterLevels` | Daftar Master Level untuk filter dropdown | Filter Level (both pages) |
| GET | `/api/v1/programmes` | Daftar Programme | Filter Programme (both pages) |

### Minor DTO Changes

**File 1:** `api_nest/src/modules/daily-attendance/dto/get-daily-attendance-on-class-year.dto.ts`

Tambah field `masterLevelId`:

```ts
@ApiPropertyOptional()
@Type(() => Number)
@IsNumber()
@IsOptional()
masterLevelId?: number;
```

**File 2:** `api_nest/src/modules/student-attendance-report/dto/get-student-attendance-reports.dto.ts`

Tambah field `masterLevelId`:

```ts
@ApiPropertyOptional()
@Type(() => Number)
@IsNumber()
@IsOptional()
masterLevelId?: number;
```

### Service Changes

**Service 1:** `api_nest/src/modules/daily-attendance/daily-attendance.service.ts`

Tambah filter `masterLevelId` di `findAllOnClassYear`:

```ts
if (options.masterLevelId) {
  cYearQ.andWhere('cy.master_level_id = :masterLevelId', {
    masterLevelId: options.masterLevelId,
  });
}
```

**Service 2:** `api_nest/src/modules/student-attendance-report/student-attendance-report.service.ts`

Tambah filter `masterLevelId` di `findDiscipline` (setelah filter `programmeId`):

```ts
if (options.masterLevelId) {
  sarQuery.andWhere('cy.master_level_id = :masterLevelId', {
    masterLevelId: options.masterLevelId,
  });
}
```

## Database Changes

**No new tables or columns.** Data sudah tersedia:

- `daily_attendance` table — sudah punya `class_year_id`, `date`, `daily_attendance_status`, `present_count`, `late_count`, `absent_count`
- `class_year` table — punya relasi ke `master_level_id`, `programme_id`, `academic_year_id`, `homeroom_teacher_id`
- `master_level` table — level name (Preschool, Kindergarten, Primary 1, etc.)

## Business Rules / Validation

1. **Principal scope:** Principal dapat melihat **semua class** di seluruh programme
2. **Daily attendance status:** `1` = incomplete, `2` = complete
3. **Discipline scope:** Principal dapat melihat discipline data untuk semua student di programme yang di-assign
4. **Term filter:** Default ke current term, bisa diganti ke Term 1-4 (reuse admin `getCurrentTerm`)
5. **Term behavior:** Filter term mempengaruhi data `studentAttendanceReports/discipline` dan side panel detail
6. **Date range (attendance):** Tidak boleh pilih future date (max: today)
7. **Academic Year:** **Default active academic year** — tidak ada dropdown, reuse function `getCurrentAcademicYear`
8. **Level filter:** Opsional — jika tidak dipilih, tampilkan semua level
9. **View Detail:** Buka `/class-in-year/:id/attendance` di tab baru (reuse admin behavior)

## Error Handling

| Error | HTTP Code | Message | Behavior |
|-------|-----------|---------|----------|
| No active academic year | 404 | "No active academic year found" | Tampilkan error banner |
| No data for filter | 200 (empty) | — | Empty state: "No attendance data found" |
| Invalid date range | 400 | "Date cannot be in the future" | Date picker disabled |
| Class not found | 404 | "Class year not found" | Error toast |
| API loading | — | — | Skeleton loading di tabel |
| Permission denied | 403 | "Access denied" | Redirect ke landing (existing guard) |

## Dependencies

- **Existing module:** `daily-attendance` (BE) — `findAllOnClassYear` method
- **Existing module:** `attendance` (BE) — `findOverview` method
- **Existing module:** `student-attendance-report` (BE) — `findDiscipline` method
- **Existing hook:** `usePrincipalOrHod` (FE teacher) — route guard (Principal/VP only)
- **Existing component:** `BBSDataTable`, `BBSDatePicker`, `BBSResourceSelect`, `AttendanceOverview`
- **Existing component:** `DisciplineOverview` (FE admin) — layout + table discipline
- **Existing component:** `DetailDiciplineStudent` (FE admin) — side panel detail student discipline
- **Existing route:** `/class-in-year/:id/attendance` — detail attendance per class
- **Navigation:** Tambah nav items di `_nav.jsx` teacher portal — `Attendance Monitoring` + `Discipline`

---

## Catatan untuk Engineer

Setelah selesai implementasi, commit & push perubahan dari folder `features/`:

```bash
cd features/
git add principal-attendance-monitoring/
git commit -m "add principal-attendance-monitoring: <deskripsi perubahan>"
git pull --rebase
git push
```

> **Note:** `features/` adalah git repo terpisah — jangan commit dari root project.