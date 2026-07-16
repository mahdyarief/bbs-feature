# Schema Changes — External Attendance Integration

## No Schema Changes Required

The existing `attendance` table already supports all required fields:

### Existing Fields (Reuse)

| Field | Type | Usage |
|---|---|---|
| `id` | INT PK | Auto-generated attendance ID |
| `student_id` | INT FK | Student reference (find by `cardNumber`) |
| `class_year_id` | INT FK | Student's current class year |
| `classroom_name` | VARCHAR | Classroom name (from class year) |
| `date` | DATE | Attendance date |
| `check_in_date` | DATETIME | Check-in timestamp |
| `check_out_date` | DATETIME | Check-out timestamp |
| `attendance_status` | ENUM | `PRESENT` or `LATE` |
| `campus_gate_id` | INT FK | Gate reference (from `campusID` + `gateID`) |
| `is_first_inserted` | BOOLEAN | `true` if first record of the day |
| `daily_attendance_id` | INT FK | Daily attendance reference |
| `created_at` | DATETIME | Record creation timestamp |
| `updated_at` | DATETIME | Record update timestamp |

### Existing Student Entity (Reuse)

| Field | Type | Usage |
|---|---|---|
| `id` | INT PK | Student ID |
| `cardNumber` | VARCHAR (unique) | Card number (find student by this) |
| `studentStatus` | ENUM | Must be `CURRENT` to process attendance |
| `currentClassYearId` | INT FK | Student's current class year |

### Existing CampusGate Entity (Reuse)

| Field | Type | Usage |
|---|---|---|
| `id` | INT PK | Gate ID |
| `campusId` | INT FK | Campus reference |
| `name` | VARCHAR | Gate name |

## Data Flow

1. **Input:** `cardnumber`, `day`, `timelog`, `status_gate`, `campusID`, `gateID`
2. **Find Student:** `SELECT * FROM student WHERE cardNumber = {cardnumber}`
3. **Validate Student:** Check `studentStatus = CURRENT`
4. **Get Class Year:** `SELECT * FROM class_year WHERE id = student.currentClassYearId`
5. **Get Late Time Setting:** `SELECT * FROM late_time_setting WHERE masterLevelId = classYear.masterLevelId AND programmeId = classYear.programmeId`
6. **Combine Date + Time:** `day + timelog` → `2025-07-16 08:15:00` (GMT+7) → convert to UTC
7. **Check First Insert:** `SELECT * FROM attendance WHERE student_id = {id} AND daily_attendance_id = {id} AND isFirstInserted = true`
8. **Determine Status:** Compare `timelog` with `lateTimeSetting.lateSchedule`
9. **Find Campus Gate:** `SELECT * FROM campus_gate WHERE id = {gateID} AND campusId = {campusID}`
10. **Create Attendance:** Insert new record with all fields
11. **Sync Daily Attendance:** Update daily attendance counts

## Notes

- No new tables or columns needed
- Reuse existing entities and relationships
- `campusID` and `gateID` map to existing `campus_gate.id` and `campus_gate.campusId`
- Multiple attendance records per day allowed (check-in + check-out)
- `isFirstInserted` flag tracks first record of the day
