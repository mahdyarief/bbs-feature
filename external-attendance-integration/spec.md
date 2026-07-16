---
feature: External Attendance Integration with API Key
slug: external-attendance-integration
status: draft
author: BBS Team
date: 2025-07-16
target_release: TBD
---

# External Attendance Integration with API Key

## Overview

Enable external/third-party systems (e.g., card reader devices, mobile apps) to push student attendance data via API with API key authentication. Support bulk attendance records with partial success handling, late detection based on actual tap time, and proper check-in/check-out validation.

## Problem / Motivation

Current attendance system only supports manual entry via admin panel or single-record integration endpoint. Need to:
1. Support bulk attendance records from external devices (card readers, mobile apps)
2. Proper late detection based on actual tap time (`timelog`) vs scheduled late time
3. Handle multiple attendance records per day (check-in + check-out)
4. Validate check-out requires prior check-in
5. Return detailed success/failure summary for bulk operations
6. Sync with campus and gate information for reporting

## Scope

### In Scope
- Modify existing integration endpoint to support bulk operations
- New payload format with `cardnumber`, `timelog`, `day`, `status_gate`, `campusID`, `gateID`
- Late detection: compare `timelog` with `lateTimeSetting.lateSchedule`
- Multiple attendance records per day (use existing `isFirstInserted` flag)
- Partial success: return per-record results (success/failed)
- Response summary: total success, total failed, list of errors
- Validation: reject check-out if no check-in exists for that day
- Timezone handling: payload GMT+7, storage follows existing DB convention
- Student attendance only (no employee)

### Out of Scope
- Employee attendance integration
- API key management UI (use hardcoded token for now, reuse existing mechanism)
- Changes to existing manual attendance entry
- Changes to attendance reporting

## User Stories

### As a System Administrator
I want external devices to push attendance data in bulk
So that student attendance is automatically recorded without manual entry

### As a System Administrator
I want detailed error reports for failed attendance records
So that I can troubleshoot integration issues (wrong card number, missing check-in, etc.)

## Acceptance Criteria

- [ ] Endpoint accepts bulk attendance records (array of records)
- [ ] Each record contains: `cardnumber`, `timelog`, `day`, `status_gate`, `campusID`, `gateID`
- [ ] API key authentication (reuse existing hardcoded token mechanism)
- [ ] Late detection uses `timelog` (actual tap time) compared to `lateTimeSetting.lateSchedule`
- [ ] Multiple attendance records per day allowed (check-in + check-out)
- [ ] `isFirstInserted = true` for first record of the day, `false` for subsequent
- [ ] Check-out (`status_gate = 2`) rejected if no check-in exists for that card on that day
- [ ] Partial success: process all records, return summary with success/failed counts and error details
- [ ] Timezone: payload `timelog` and `day` are GMT+7, convert to DB storage format
- [ ] Response includes: `total_success`, `total_failed`, `results` array with per-record status
- [ ] Student must be found by `cardnumber` and have `studentStatus = CURRENT`
- [ ] Campus and gate must exist in database

## UI / UX Changes

### Affected Portals
- [ ] Admin (client/)
- [ ] Student (client-student/)
- [ ] Teacher (client-teacher/)

No UI changes. Backend API only.

## Business Rules / Validation

### 1. Late Detection Logic

```
IF timelog > lateTimeSetting.lateSchedule THEN
  attendanceStatus = LATE
ELSE
  attendanceStatus = PRESENT
END IF
```

Example:
- `lateTimeSetting.lateSchedule = 08:00:00`
- Student taps at `08:15:00` → `attendanceStatus = LATE`
- Student taps at `07:45:00` → `attendanceStatus = PRESENT`

### 2. Check-in / Check-out Validation

| status_gate | Meaning | Validation |
|---|---|---|
| `1` | IN (check-in) | Create new attendance record with `checkInDate = day + timelog` |
| `2` | OUT (check-out) | **Reject** if no check-in exists for this `cardnumber` on this `day` |

Error message for missing check-in:
```
"cardNumber {cardnumber} has no check-in record on {day}"
```

### 3. Multiple Records Per Day

- First record of the day: `isFirstInserted = true`
- Subsequent records (check-out): `isFirstInserted = false`
- Use existing `checkIsFirstAttendance()` method

### 4. Partial Success Handling

Process all records independently. Return summary:

```json
{
  "total_success": 8,
  "total_failed": 2,
  "results": [
    { "cardnumber": "28102019", "status": "success", "attendance_id": 123 },
    { "cardnumber": "99999999", "status": "failed", "error": "Student not found" },
    ...
  ]
}
```

### 5. Timezone Conversion

Payload:
- `day = "2025-07-16"` (GMT+7)
- `timelog = "08:15:00"` (GMT+7)

Convert to DB storage:
- Combine: `2025-07-16 08:15:00` (GMT+7)
- Store as: `2025-07-16T01:15:00.000Z` (UTC) or follow existing DB convention

### 6. Student Status Check

Only process attendance for students with `studentStatus = CURRENT`. Skip others (return in failed results).


## Dependencies

- `student` module (find by `cardNumber`)
- `campus-gate` module (validate `campusID` + `gateID`)
- `late-time-setting` module (get late schedule for student's level/programme)
- `daily-attendance` module (create or use existing daily attendance)
- `class-year` module (get student's current class year)
- Existing `attendance` module (reuse logic)
