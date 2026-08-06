---
title: Attendance Teacher Portal — "Mark All Present" tidak berefek dan check-in time menjadi 00:00:00
status: open
severity: major
product: BBS LMS
portal: Teacher Portal
author: System Analyst
date: 2026-08-06
jam: https://jam.dev/c/39d9ee11-ed31-4f57-b78e-0f5eafa35514
---

# Attendance "Mark All Present" — Button Tidak Berefek & Check-In Time 00:00:00

## Summary

**Jam recording:** https://jam.dev/c/39d9ee11-ed31-4f57-b78e-0f5eafa35514 (video 27s, reporter: Mahdy Arief)

Di Teacher Portal, tombol **Mark All Present** pada halaman Classroom Details → Attendance memiliki 2 masalah:

1. **Klik tidak menimbulkan perubahan yang terlihat** — request sukses (HTTP 201) tapi tabel tidak berubah; pada kondisi semua siswa sudah punya record attendance, aksi menjadi no-op yang membingungkan.
2. **Check-in time semua siswa menjadi `00:00:00`** — siswa yang di-mark via bulk mendapat `checkInDate` = tengah malam (start of day), bukan waktu klik aktual. Siswa yang sudah punya scan gate asli (mis. `07:11:57`) tidak terdampak.

**Ekspektasi:** "Mark All Present" harus menandai siswa dengan **waktu aktual saat klik** (atau membiarkan check-in kosong sampai diisi scan gate), dan harus memberikan feedback yang jelas (jumlah siswa yang baru di-mark) — bukan toast sukses tanpa perubahan data.

---

## Test Identity / Akun Akses

Data berikut diambil dari Jam recording (author + console log) untuk mempermudah recreate & testing:

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Mahdy Arief |
| Email (Jam account) | mahdy.arief.torche.indonesia@gmail.com |
| Jam author ID | `7ebe5b87-970c-4f3d-95f5-c4f39b901a03` |
| Teacher ID (`selfUser` dari console log `[useSecondaryTeacher]`) | `4781` |
| Portal URL | `https://teacher.smartbag.binabangsaschool.com` |
| Environment API | `api.binabangsaschool.dev` (staging/dev) |
| Class ID (Classroom) | `100940` |
| Daily Attendance ID (daId) | `29819` (Wed, Aug 5, 2026) |
| Browser / OS | Brave 150.0.0.0 (Chromium) / Windows 11 |

---

## Steps to Reproduce

1. Login sebagai **Teacher** (`selfUser` ID `4781` — akun Mahdy Arief) ke Teacher Portal (`teacher.smartbag.binabangsaschool.com`)
2. Buka **Classroom Details** (classroom 100940) → tab **Attendance** (daId=29819, Wed Aug 5, 2026)
3. Pastikan keadaan awal: `Total Student: 20`, `Present: 0`, `Unmarked: 20 (100%)`
4. Klik **Mark All Present** (`button.mx-2.btn.btn-success`) → modal konfirmasi muncul → klik **Yes** (`button.w-100.btn.btn-primary`)
5. Amati tabel dan toast

**Actual Result:**
- Toast "Successfully marked all students as present!" muncul
- Kolom Check In semua siswa = `00:00:00` (kecuali siswa yang sudah scan gate, mis. "Aaron Calvinson Sumarli" tetap `07:11:57`)
- Jika klik lagi pada hari yang sama: toast sukses lagi tapi **tidak ada perubahan data** (semua siswa sudah punya record)

**Expected Result:**
- Check-in time = waktu aktual saat tombol diklik (atau kosong/null jika belum ada scan)
- Klik kedua saat semua siswa sudah ter-mark: tombol dinonaktifkan atau feedback menunjukkan 0 siswa baru di-mark

---

## Root Cause Analysis

### Bug #2 — Check-in time `00:00:00` — `attendance.service.ts` (line 606-629)

Method `createBulk()` di `AttendanceService` mengirim `checkInDate` = **start of day** (tengah malam), bukan waktu klik:

```typescript
// api_nest/src/modules/attendance/attendance.service.ts:606-622
const normalizedDate = moment(options.date)
  .tz('Asia/Jakarta')
  .startOf('day')
  .toDate();
// ...
const attendance = await this.create(userId, {
  classYearId: options.classYearId,
  studentId: studentId.toString(),
  date: normalizedDate,
  checkInDate: normalizedDate,   // ← startOf('day') = 00:00:00
  attendanceStatus: AttendanceStatusTypeEnum[AttendanceStatusTypeEnum.PRESENT],
});
```

Faktor pendukung:
- `CreateAttendanceBulkDto` (`create-attendance-bulk.dto.ts`) hanya menerima `classYearId` + `date` — **tidak ada field checkInDate** dari frontend, sehingga nilai default dari entity yang berlaku.
- `Attendance` entity (`attendance.entity.ts:65-66`): kolom `checkInDate` punya default `CURRENT_DATE` — PostgreSQL `CURRENT_DATE` = tengah malam (00:00:00). Jika field dihilangkan, hasilnya sama.
- Siswa yang sudah punya record attendance hari itu di-*exclude* (`filteredStudents`, line 602-604) → scan gate asli (07:11:57) tidak ketimpa, hanya siswa baru yang dapat 00:00:00.

### Bug #1 — "Tidak terjadi apa-apa" — `attendance.service.ts:602-604` + `ClassroomDetailsAttendance.jsx:405-421`

Jika **semua 20 siswa sudah punya record** attendance hari itu:

```typescript
const filteredStudents = studentIdsInClassYear.filter(
  (s) => !excludedStudentIds.includes(s),
);  // → [] jika semua sudah ter-mark
```

- `Promise.all([])` → server membalas **201 dengan array kosong**
- Frontend tetap menampilkan toast sukses (`bbsToaster.success`) tanpa perubahan data → user melihat "tidak terjadi apa-apa"

Frontend (`ClassroomDetailsAttendance.jsx:531-540`) hanya men-disable tombol saat `dailyAttendanceStatus === "COMPLETE"` — **tidak** men-disable saat `unmarkedCount === 0`:

```javascript
<CButton
  className="mx-2"
  color="success"
  onClick={handleSubmitBulkAttendanceChange}
  disabled={dailyAttendance?.dailyAttendanceStatus === "COMPLETE"}
>
  Mark All Present
</CButton>
```

---

## Bukti dari Jam (https://jam.dev/c/39d9ee11-ed31-4f57-b78e-0f5eafa35514)

| Sumber | Temuan |
|--------|--------|
| **Video (t=24.7s)** | Hanya "Aaron Calvinson Sumarli" yang check-in `07:11:57` (scan gate asli); **semua siswa lain `00:00:00`** |
| **Network** | 2× `POST /api/v1/attendances/bulk` → **201 (sukses)**; tidak ada 4xx/5xx; GET `/api/v1/attendances?pageSize=0&dailyAttendanceId=29819` → 200 |
| **Console** | Tidak ada error/warning — hanya log info (`useSecondaryTeacher` dll) |
| **User events** | "Mark All Present" diklik 2×, "Yes" diklik 2× — klik selalu ter-trigger, modal selalu muncul |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/attendance/attendance.service.ts` | `createBulk()` (line 606-629) set `checkInDate` = start of day; `filteredStudents` (line 602-604) bisa kosong tanpa validasi |
| Backend DTO | `api_nest/src/modules/attendance/dto/create-attendance-bulk.dto.ts` | Hanya menerima `classYearId` + `date`, tidak ada field waktu |
| Backend Entity | `api_nest/src/modules/attendance/entities/attendance.entity.ts` | `checkInDate` default `CURRENT_DATE` (00:00:00) |
| Frontend Teacher | `bbs/client-teacher/src/views/classrooms/ClassroomDetailsAttendance.jsx` | `handleSubmitBulkAttendanceChange` (line 405-421) selalu tampilkan toast sukses; button hanya disabled saat status COMPLETE (line 534-537) |

---

## Proposed Solution Options

### Option A: Gunakan Waktu Aktual (Recommended)

Backend `attendance.service.ts:createBulk()` — set `checkInDate` ke waktu saat klik:

```typescript
const checkInTime = moment().tz('Asia/Jakarta').toDate();
// ...
checkInDate: checkInTime,
```

**Catatan:** pertimbangkan apakah "Mark All Present" memang harus mencatat waktu manual, atau lebih tepat membiarkan `checkInDate` kosong/null agar waktu check-in tetap dari scan gate (integrasi). Perlu konfirmasi business rules dengan pihak sekolah.

### Option B: Validasi & Feedback Bermakna

- Backend: tolak bulk (`400`) atau kembalikan `createdCount` jika `filteredStudents` kosong
- Frontend: disable tombol saat `unmarkedCount === 0`; tampilkan feedback berupa jumlah siswa yang baru di-mark

### Option C: Idempotensi Eksplisit

Buat aksi bulk bersifat idempotent: klik berulang tidak menciptakan double-record (sudah aman via `filteredStudents`), namun feedback harus menyatakan "0 siswa baru di-mark" supaya tidak membingungkan user.

---

## Notes

- Jam recording: https://jam.dev/c/39d9ee11-ed31-4f57-b78e-0f5eafa35514 (video 27s, Mahdy Arief, Brave 150 / Windows 11)
- Browser: Brave 150.0.0.0 (Chromium), OS Windows 11
- Environment: `api.binabangsaschool.dev` (staging)
- Perlu koordinasi dengan BE Engineer (waktu check-in) dan FE Engineer (disable state + feedback)
- Setelah keputusan final, buat spec implementasi di folder fitur ini (ikuti `_templates/spec-template.md`)
