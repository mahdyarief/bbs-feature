# Bug Report — External Attendance Integration (Bulk)

**Feature:** `external-attendance-integration`
**Endpoint:** `POST /api/v1/attendances/integration/bulk`
**Date Found:** 2026-08-14
**Severity:** Critical
**Status:** Open

---

## Summary

Investigasi production data tanggal 14 Agustus 2026 menemukan beberapa bug di implementasi bulk attendance integration. Bug paling kritis adalah **race condition** yang menyebabkan duplicate attendance records dan `is_first_inserted = true` ganda untuk student yang sama.

---

## Bug #1 — Race Condition: Duplicate `is_first_inserted` Records

**Severity:** 🔴 Critical
**File:** `src/modules/attendance/attendance.service.ts`
**Lines:** 1497-1498, 1925-1979

### Deskripsi

`createAttendanceIntegrationBulk()` memproses semua record dalam satu chunk secara **concurrent** menggunakan `Promise.all`:

```typescript
// Line 1497-1498
const chunkResults = await Promise.all(
  chunk.map((record) => this.processIntegrationRecord(record)),
);
```

Method `createOrUseExistingDailyAttendance()` (line 1961-1979) dipanggil **di luar** advisory lock dan tidak punya mekanisme locking sendiri. Ketika dua concurrent request untuk student yang sama masuk (gate double-tap), keduanya bisa:

1. `findOneBy(DailyAttendance)` → keduanya dapat `null` (belum commit)
2. Keduanya create `DailyAttendance` baru dengan ID berbeda
3. `pg_advisory_xact_lock(studentId, dailyAttendanceId)` → lock key berbeda → **tidak ada serialisasi**
4. Keduanya set `isFirstInserted = true`

### Race Scenario

```
Time  | Request A (student X)              | Request B (student X)
------|-------------------------------------|------------------------------------
T1    | findOneBy(dailyAttendance) → null   |
T2    |                                     | findOneBy(dailyAttendance) → null
T3    | save → dailyAttendance.id = 100     |
T4    |                                     | save → dailyAttendance.id = 101
T5    | advisory_lock(X, 100) → acquired    |
T6    |                                     | advisory_lock(X, 101) → acquired
T7    | isFirstInserted = true → save       |
T8    |                                     | isFirstInserted = true → save
```

### Bukti Production (14 Aug 2026)

- **User 24629**: 2 attendance records, keduanya `is_first_inserted = true`
- **24 user lain**: multiple check-in records dari gate double-tap

### Impact

- Data absensi ganda — laporan kehadiran tidak akurat
- Classroom/class-in-year yang filter by `isFirstInserted = true` menampilkan duplikat
- Student report attendance bisa salah hitung

### Rekomendasi Fix

**Option A** — Lock di level `studentId + dateHash` sebelum `DailyAttendance` lookup, bukan `studentId + dailyAttendanceId`.

**Option B** — Group records by student sebelum processing. Records untuk student yang sama diproses sequential, beda student bisa parallel.

```typescript
const grouped = groupBy(records, r => r.userID ?? r.cardnumber);
await Promise.all(
  Object.values(grouped).map(async (studentRecords) => {
    for (const record of studentRecords) {
      results.push(await this.processIntegrationRecord(record));
    }
  })
);
```

### Acceptance Criteria

- [ ] Tidak ada duplicate `is_first_inserted = true` untuk student + date yang sama
- [ ] Tidak ada duplicate DailyAttendance records untuk classYear + date yang sama
- [ ] Gate double-tap (2+ records student yang sama dalam 1 batch) hanya menghasilkan 1 attendance record
- [ ] Concurrent requests dari external system untuk student yang sama tetap serialized
- [ ] Performance tidak menurun signifikan (bulk 100 records tetap < 5 detik)

### Test Cases

```
TC-1.1: Gate Double-Tap — Same Student, Same Batch
  Given: 2 records dengan userID yang sama, statusGate "1", day yang sama
  When: POST /api/v1/attendances/integration/bulk
  Then: Hanya 1 attendance record tercipta dengan is_first_inserted = true
        Response: total_success = 1, total_failed = 1 (duplicate rejected)

TC-1.2: Concurrent Requests — Same Student, Different HTTP Requests
  Given: 2 HTTP requests bersamaan untuk student yang sama, check-in
  When: Kedua request diproses bersamaan
  Then: Hanya 1 attendance record dengan is_first_inserted = true
        Request kedua: update existing record (bukan create baru)

TC-1.3: Concurrent Requests — Different Students, Same Batch
  Given: 50 records untuk 50 student berbeda dalam 1 batch
  When: POST /api/v1/attendances/integration/bulk
  Then: Semua 50 records berhasil (tidak ada lock contention)
        Performance < 5 detik

TC-1.4: DailyAttendance Deduplication
  Given: 2 concurrent requests untuk student di classYear yang sama, date yang sama
  When: Keduanya memanggil createOrUseExistingDailyAttendance
  Then: Hanya 1 DailyAttendance record tercipta
```

### Deviation dari Spec

**Spec (`edgecases.md`) bilang:**
> Duplicate Check-in (Same Student, Same Day, status_gate=1) → **(A) Reject** → Return error "Already has check-in record"

**Implementasi saat ini:**
- Tidak ada pengecekan duplicate check-in sebelum insert
- `Promise.all` memproses semua records bersamaan, tidak ada mekanisme reject duplicate
- Justru spec bilang reject, tapi implementasinya malah allow — ini yang menyebabkan double-tap problem

---

## Bug #2 — Student Tanpa `current_class_year_id` Gagal Permanen

**Severity:** 🟡 High
**File:** `src/modules/attendance/attendance.service.ts`
**Lines:** 1614-1623

### Deskripsi

```typescript
const classYear = await ClassYear.findOne({
  where: { id: Number(student.currentClassYearId) },  // NULL → NaN
});
if (!classYear) {
  return { status: 'failed', error: 'Class year not found' };
}
```

Student dengan `current_class_year_id = NULL` (siswa baru belum di-assign, atau naik kelas tapi field belum update) **selalu gagal** absen lewat gate. Tidak ada fallback atau error message yang informatif ke admin.

### Bukti Production

- **User 100683**: `current_class_year_id = NULL` → error "Missing student, class year, or daily attendance for save"

### Rekomendasi Fix

Tambahkan validasi di awal `processIntegrationRecord`:

```typescript
if (!student.currentClassYearId) {
  return {
    ...baseResult,
    status: 'failed',
    error: `Student ${student.id} has no active class year assignment`,
  };
}
```

Tambahkan alerting/notifikasi ke admin.

### Acceptance Criteria

- [ ] Error message jelas: "Student {id} has no active class year assignment"
- [ ] Admin mendapat notifikasi/alert ketika student gagal karena alasan ini
- [ ] Response API menyertakan `studentId` dan `cardnumber` di result agar admin bisa trace
- [ ] Student yang sudah di-assign class year bisa langsung absen tanpa restart/retry

### Test Cases

```
TC-2.1: Student Without Class Year
  Given: Student dengan current_class_year_id = NULL
  When: POST /api/v1/attendances/integration/bulk dengan record student tersebut
  Then: Response status = "failed"
        Error message = "Student {id} has no active class year assignment"
        Result menyertakan studentId dan cardnumber

TC-2.2: Student With Class Year (Positive)
  Given: Student dengan current_class_year_id = 5
  When: POST /api/v1/attendances/integration/bulk
  Then: Attendance berhasil tersimpan normal
```

---

## Bug #3 — Check-out Flow Tidak Menggunakan Advisory Lock

**Severity:** 🟡 Medium
**File:** `src/modules/attendance/attendance.service.ts`
**Lines:** 1669-1691

### Deskripsi

Check-out flow memanggil `checkIsFirstAttendance()` dan langsung `save()` **tanpa advisory lock**:

```typescript
let targetAttendance = await this.checkIsFirstAttendance(
  student.id, classYear.id, dailyAttendance.id,
);
if (!targetAttendance) {
  targetAttendance = await this.findCheckInForDay(student.id, record.day);
}
targetAttendance.checkOutDate = combinedDateTime;
await targetAttendance.save();  // ← No lock!
```

Berbeda dengan check-in yang lewat `saveIntegrationAttendanceExclusive()` (pakai advisory lock), check-out **tidak di-lock sama sekali**.

### Impact

- Jika check-out dan check-in terjadi bersamaan (race), check-out bisa update row yang salah
- `checkOutDate` bisa masuk ke attendance record yang bukan `is_first_inserted`

### Rekomendasi Fix

Wrap check-out flow dalam advisory lock yang sama seperti check-in.

### Acceptance Criteria

- [ ] Check-out flow menggunakan `pg_advisory_xact_lock` dengan key yang sama seperti check-in
- [ ] Check-out selalu update attendance record yang `is_first_inserted = true`
- [ ] Tidak ada race antara check-in dan check-out untuk student yang sama

### Test Cases

```
TC-3.1: Concurrent Check-in and Check-out
  Given: Student sudah check-in, lalu check-out dan check-in baru datang bersamaan
  When: Kedua request diproses bersamaan
  Then: Check-out update record yang benar (is_first_inserted = true)
        Check-in baru ditolak atau update record yang tepat

TC-3.2: Check-out Updates Correct Record
  Given: Student punya 2 attendance records (akibat Bug #1 yang sudah ada)
  When: Check-out datang
  Then: checkOutDate di-set ke record dengan is_first_inserted = true
        Bukan record duplikat
```

---

## Bug #4 — `findCheckInForDay` Non-deterministic (No ORDER BY)

**Severity:** 🟡 Medium
**File:** `src/modules/attendance/attendance.service.ts`
**Lines:** 1805-1819

### Deskripsi

```typescript
return await Attendance.findOne({
  where: {
    studentId,
    date: Between(startOfDay, endOfDay),
    checkInDate: Not(IsNull()),
  },
  // ← No ORDER BY — non-deterministic!
});
```

Ketika ada multiple check-in records (akibat Bug #1), `findOne` tanpa `ORDER BY` mengembalikan row yang **non-deterministic**. Check-out bisa update row yang salah.

### Rekomendasi Fix

Tambahkan `order: { checkInDate: 'ASC' }` di `findCheckInForDay`.

### Acceptance Criteria

- [ ] `findCheckInForDay` selalu return record dengan `checkInDate` paling awal (ASC)
- [ ] Deterministic: query yang sama selalu return row yang sama
- [ ] Check-out selalu update record check-in pertama

### Test Cases

```
TC-4.1: Multiple Check-in Records — Deterministic Selection
  Given: Student punya 2 check-in records (07:00 dan 07:01) akibat double-tap
  When: findCheckInForDay dipanggil
  Then: Selalu return record dengan checkInDate = 07:00 (yang pertama)

TC-4.2: Check-out After Duplicate Check-in
  Given: Student punya 2 check-in records
  When: Check-out datang
  Then: checkOutDate di-set ke record check-in pertama (07:00)
```

---

## Bug #5 — Check-out Tanpa Check-in Ditolak Tanpa Fallback

**Severity:** 🟡 Medium
**File:** `src/modules/attendance/attendance.service.ts`
**Lines:** 1626-1637

### Deskripsi

Ketika `statusGate = '2'` (check-out) dikirim tapi student belum punya check-in record hari itu, record langsung ditolak. Mayoritas failure di production disebabkan oleh ini.

**Skenario yang mungkin:**
1. Student check-in lewat teacher portal, lalu check-out lewat gate
2. Gate hanya kirim check-out karena check-in sebelumnya error/timeout
3. Student tap check-in tapi tidak ter-record (hardware issue)

### Bukti Production (14 Aug 2026)

- 10 dari 14 failed records adalah error "has no check-in record"

### Rekomendasi Fix

Tambahkan graceful fallback: jika check-out datang tanpa check-in, auto-create check-in record dengan timestamp = batas waktu check-in (misal 07:00) lalu langsung set check-out.

### Acceptance Criteria

- [ ] Check-out tanpa check-in: auto-create check-in dengan timestamp = lateTimeSetting.lateSchedule atau 07:00
- [ ] Response menunjukkan status = "success" dengan note bahwa check-in di-auto-create
- [ ] Attendance record tercipta dengan checkInDate (auto) dan checkOutDate (dari gate)
- [ ] attendanceStatus = "PRESENT" (bukan "LATE" karena check-in waktu tidak diketahui)

### Test Cases

```
TC-5.1: Check-out Without Check-in — Auto-create
  Given: Student belum punya check-in hari ini
  When: POST check-out (statusGate = "2")
  Then: Attendance record tercipta dengan:
        - checkInDate = batas waktu check-in (dari lateTimeSetting)
        - checkOutDate = waktu dari gate
        - attendanceStatus = "PRESENT"
        - isFirstInserted = true
  Response: status = "success", note = "auto check-in created"

TC-5.2: Check-out Without Check-in — Student Not Found
  Given: Student tidak ada di database
  When: POST check-out
  Then: Response status = "failed", error = "Student not found"
```

---

## Production Data Evidence (14 August 2026)

| Metric | Count |
|--------|-------|
| Total GATE_SCAN logs (PARTIAL status) | 9 requests |
| Total failed records | 14 |
| Failed: "no check-in record" | 10 |
| Failed: "Missing student/class year" | 1 (userID 100683) |
| Failed: "Student not found" | 1 (cardnumber "b") |
| Failed: "Gate not found" | 2 |
| Users with duplicate check-in records | 24 |
| Users with duplicate `is_first_inserted=true` | 1 (userID 24629) |

---

## Priority Matrix

| Priority | Bug | Effort |
|----------|-----|--------|
| **P0** — Fix now | #1: Race condition duplicate `is_first_inserted` | Medium |
| **P1** — This sprint | #2: Missing class year handling | Low |
| **P1** — This sprint | #3: Check-out without advisory lock | Low |
| **P2** — Next sprint | #4: Non-deterministic `findCheckInForDay` | Low |
| **P2** — Next sprint | #5: Check-out without check-in fallback | Medium |

---

## Affected Files

| File | Lines | Description |
|------|-------|-------------|
| `attendance.service.ts` | 1479-1516 | `createAttendanceIntegrationBulk` — Promise.all concurrency |
| `attendance.service.ts` | 1518-1794 | `processIntegrationRecord` — main processing logic |
| `attendance.service.ts` | 1925-1958 | `saveIntegrationAttendanceExclusive` — advisory lock (ineffective) |
| `attendance.service.ts` | 1961-1979 | `createOrUseExistingDailyAttendance` — race condition source |
| `attendance.service.ts` | 1805-1819 | `findCheckInForDay` — non-deterministic findOne |
| `attendance.service.ts` | 1669-1691 | Check-out flow — no lock |
| `attendance-log.interceptor.ts` | 1-237 | Log interceptor (works correctly, no bug) |
| `create-attendance-integration-bulk.dto.ts` | 1-67 | DTO definitions (no bug) |

---

## Data Migration & Cleanup Plan

Setelah bug fix di-deploy, data corrupt yang sudah ada di production perlu dibersihkan.

### Step 1: Identify Corrupt Data

```sql
-- Find all duplicate is_first_inserted records
SELECT student_id, date, COUNT(*) as cnt
FROM attendance
WHERE is_first_inserted = true
  AND check_in_date IS NOT NULL
GROUP BY student_id, date
HAVING COUNT(*) > 1;

-- Find all duplicate check-in records (multiple check-in same day)
SELECT student_id, date, COUNT(*) as cnt
FROM attendance
WHERE check_in_date IS NOT NULL
GROUP BY student_id, date
HAVING COUNT(*) > 1;
```

### Step 2: Deduplicate `is_first_inserted`

```sql
-- Keep the earliest check-in record as the "real" first inserted
-- Set is_first_inserted = false for duplicates
UPDATE attendance a
SET is_first_inserted = false
WHERE a.is_first_inserted = true
  AND a.id NOT IN (
    SELECT MIN(id)
    FROM attendance
    WHERE is_first_inserted = true
      AND check_in_date IS NOT NULL
    GROUP BY student_id, date
  );
```

### Step 3: Merge Duplicate Check-in Records

```sql
-- For duplicate check-in records, merge check_out_date to the earliest record
-- then delete the duplicates
-- (Review manually before executing — some duplicates may have check_out_date)

-- Step 3a: Move check_out_date from duplicates to the keeper record
UPDATE attendance keeper
SET check_out_date = dup.check_out_date
FROM attendance dup
WHERE dup.student_id = keeper.student_id
  AND dup.date = keeper.date
  AND dup.check_in_date IS NOT NULL
  AND keeper.check_in_date IS NOT NULL
  AND dup.id != keeper.id
  AND keeper.id = (
    SELECT MIN(id) FROM attendance
    WHERE student_id = dup.student_id AND date = dup.date
      AND check_in_date IS NOT NULL
  )
  AND dup.check_out_date IS NOT NULL
  AND keeper.check_out_date IS NULL;

-- Step 3b: Delete duplicate check-in records
DELETE FROM attendance
WHERE id NOT IN (
  SELECT MIN(id)
  FROM attendance
  WHERE check_in_date IS NOT NULL
  GROUP BY student_id, date
)
AND check_in_date IS NOT NULL
AND student_id IN (
  SELECT student_id FROM attendance
  WHERE check_in_date IS NOT NULL
  GROUP BY student_id, date
  HAVING COUNT(*) > 1
);
```

### Step 4: Deduplicate DailyAttendance

```sql
-- Find duplicate DailyAttendance records
SELECT class_year_id, date, COUNT(*) as cnt
FROM daily_attendance
GROUP BY class_year_id, date
HAVING COUNT(*) > 1;

-- Reassign attendance records to the keeper DailyAttendance, then delete duplicates
-- (Review carefully — attendance.daily_attendance_id must be updated)
```

### Step 5: Verify

```sql
-- After cleanup, verify no more duplicates
SELECT 'duplicate is_first_inserted' as check_name, COUNT(*) as issues
FROM (
  SELECT student_id, date FROM attendance
  WHERE is_first_inserted = true AND check_in_date IS NOT NULL
  GROUP BY student_id, date HAVING COUNT(*) > 1
) t
UNION ALL
SELECT 'duplicate check-in', COUNT(*)
FROM (
  SELECT student_id, date FROM attendance
  WHERE check_in_date IS NOT NULL
  GROUP BY student_id, date HAVING COUNT(*) > 1
) t;
```

### ⚠️ Important Notes

- **Backup database** sebelum menjalankan migration
- Jalankan migration di **maintenance window** — bisa lock table
- Review hasil query SELECT di Step 1 sebelum execute UPDATE/DELETE
- Step 3b (DELETE) harus dijalankan setelah Step 3a (merge check_out_date)
- Setelah cleanup, jalankan Step 5 (Verify) untuk konfirmasi

---

## Monitoring & Alerting

### 1. Duplicate Detection Query (Run Daily)

```sql
-- Cron: setiap hari jam 23:00
-- Alert jika count > 0
SELECT COUNT(*) as duplicate_count
FROM (
  SELECT student_id, date
  FROM attendance
  WHERE is_first_inserted = true
    AND check_in_date IS NOT NULL
  GROUP BY student_id, date
  HAVING COUNT(*) > 1
) t;
```

### 2. Attendance Log Monitoring

```sql
-- Monitor PARTIAL/FAILED GATE_SCAN logs per hari
-- Alert jika failure rate > 10%
SELECT
  DATE(created_at) as log_date,
  COUNT(*) as total_logs,
  SUM(CASE WHEN status = 'PARTIAL' THEN 1 ELSE 0 END) as partial_count,
  SUM(CASE WHEN status = 'FAILED' THEN 1 ELSE 0 END) as failed_count
FROM attendance_log
WHERE source_type = 'GATE_SCAN'
  AND created_at >= NOW() - INTERVAL '1 day'
GROUP BY DATE(created_at);
```

### 3. Students Without Class Year

```sql
-- Alert jika ada student aktif tanpa class year yang gagal absen
-- Cek attendance_log response untuk error "Class year not found"
SELECT created_at, request, response
FROM attendance_log
WHERE source_type = 'GATE_SCAN'
  AND status = 'FAILED'
  AND response::text LIKE '%Class year not found%'
  AND created_at >= NOW() - INTERVAL '1 hour';
```

### 4. Application-Level Logging

Tambahkan structured log di `processIntegrationRecord`:

```typescript
this.logger.warn('Duplicate check-in detected and rejected', {
  studentId: student.id,
  date: record.day,
  existingAttendanceId: existing.id,
  duplicateRecordId: record.userID,
});
```

### 5. Dashboard Metrics (Optional)

| Metric | Threshold | Action |
|--------|-----------|--------|
| Duplicate `is_first_inserted` per day | > 0 | 🔴 Immediate investigation |
| GATE_SCAN failure rate | > 10% | 🟡 Investigate root cause |
| Students failed: "no class year" | > 0 | 🟡 Notify admin untuk assign class |
| GATE_SCAN check-out without check-in | > 5/hour | 🟡 Investigate gate hardware |
