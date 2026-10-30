---
feature: Homeroom Daily Attendance Status Inconsistency (100% vs Incomplete)
slug: homeroom-daily-attendance-status-inconsistent
status: draft
author: BBS Team
date: 2026-10-30
target_release: TBD
---

## Overview
Saat guru homeroom mengecek absensi harian untuk sebuah kelas, header menampilkan "100%" (status COMPLETE), tapi saat di-scroll ke daftar per-siswa di bawah, beberapa/semua siswa masih tertulis INCOMPLETE. Ketika detail siswa dibuka, datanya sudah terisi lengkap. Insiden dilaporkan oleh Albert Jerome Lupisan (Homeroom Teacher Primary 6 Humility, Bandung Primary).

## Problem / Motivation
- Header agregat (`daily_attendance.daily_attendance_status`) menunjukkan COMPLETE/100% padahal daftar per-siswa masih INCOMPLETE.
- Data absensi per siswa sebenarnya sudah tersimpan lengkap di DB (`attendance` table); inkonsistensi hanya pada agregat status harian dan logika tampilan list.
- Membingungkan guru apakah absensi hari itu benar-benar sudah 100%.

## Root Cause (diagnosis read-only, tanpa perubahan kode)
1. `daily_attendance_status` di tabel `daily_attendance` bisa stuck di COMPLETE dari sinkronisasi sebelumnya. Backend `synchronizeDailyAttendanceStatus()` hanya mem-promosi ke COMPLETE bila `totalAttendanceStudents >= studentCount`; tidak ada branch yang mengembalikan ke INCOMPLETE saat jumlah terisi menurun di bawah jumlah siswa.
2. Frontend list (`ClassroomDetailsAttendance.jsx`) memfilter `attendances.filter((a) => !a.bySubjectTeacher)` — hanya absensi catatan homeroom teacher yang ditampilkan. Absensi yang dicatat oleh subject teacher ada di DB (lihat di detail) tapi disembunyikan di list homeroom, sehingga baris terlihat kosong/incomplete.
3. Gate completeness `checkCompleteness: unmarkedAmount < 2` artinya bila 2+ siswa belum diberi status, backend tidak diminta menghitung ulang status harian → status jadi stale.

## Scope
### In Scope
- Backend: `api_nest/src/modules/daily-attendance/daily-attendance.service.ts` — `synchronizeDailyAttendanceStatus`
- Frontend: `bbs/client-teacher/src/views/classrooms/ClassroomDetailsAttendance.jsx` — rendering list + gate completeness
- DB: tabel `daily_attendance` (`daily_attendance_status`) dan `attendance`

### Out of Scope
- Mengubah data absensi itu sendiri (sudah tersimpan benar)
- Redesign workflow absensi subject teacher

## User Stories
### As a Homeroom Teacher
I want the daily attendance header percentage to match the per-student list status so I know whether attendance is truly complete.

## Acceptance Criteria
- [ ] Header "100%"/Complete hanya muncul ketika setiap siswa terdaftar memiliki catatan absensi valid untuk tanggal tersebut
- [ ] Baris per-siswa hanya INCOMPLETE bila status absensinya benar-benar INCOMPLETE atau belum ada catatan
- [ ] Membuka detail siswa menunjukkan data konsisten dengan list
- [ ] Status harian dihitung ulang (termasuk kembali ke INCOMPLETE) saat jumlah terisi turun di bawah jumlah siswa

## UI / UX Changes
- Portal yang terpengaruh: Teacher (`client-teacher/`)
- List baris dan ringkasan header di `ClassroomDetailsAttendance`

## API Changes
Tidak ada endpoint baru; perubahan behavior pada logika `synchronizeDailyAttendanceStatus` yang ada.

## Database Changes
Tidak ada perubahan skema; data sudah konsisten pada level record.

## Business Rules / Validation
1. `daily_attendance_status` = COMPLETE iff setiap siswa terdaftar memiliki catatan absensi valid untuk tanggal tersebut
2. Sebaliknya = INCOMPLETE
3. Dihitung ulang pada setiap create/update/delete absensi

## Error Handling
Tidak ada error baru; penanganan 4xx/5xx yang ada tetap berlaku.

## Dependencies
- Modul `daily-attendance` di `api_nest`
- View classroom attendance di `bbs/client-teacher`
