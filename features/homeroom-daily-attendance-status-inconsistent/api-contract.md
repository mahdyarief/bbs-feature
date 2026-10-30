# API Contract — Homeroom Daily Attendance Status Inconsistency

Tidak ada endpoint baru. Perubahan behavior ada pada logika internal `synchronizeDailyAttendanceStatus` di backend.

## Endpoint terkait (tidak berubah signatur)

| Method | Path | Deskripsi |
|--------|------|-----------|
| GET | `/daily-attendance` | Mengambil daftar daily attendance per class year (header status COMPLETE/INCOMPLETE) |
| POST | `/attendance` | Create/update satu absensi siswa; memicu `syncDailyAttendance` |
| POST | `/attendance/bulk` | Mark all present untuk kelas/date |
| PATCH | `/daily-attendance/:id/complete` | Toggle status harian ke COMPLETE |
| PATCH | `/daily-attendance/:id/incomplete` | Toggle status harian ke INCOMPLETE |

## Perilaku yang diinginkan (setelah fix)
- `synchronizeDailyAttendanceStatus(dailyAttendanceId)` harus menetapkan `INCOMPLETE` bila `totalAttendanceStudents < studentCount`, bukan hanya mem-promosi ke `COMPLETE`.
- Sinkronisasi dipanggil pada setiap create/update/delete absensi tanpa gate `checkCompleteness: unmarkedAmount < 2`.

## Respons example (tidak berubah struktur)
```json
{
  "id": 123,
  "date": "2026-10-30",
  "class_year_id": 456,
  "daily_attendance_status": "COMPLETE",
  "created_at": "...",
  "updated_at": "..."
}
```
