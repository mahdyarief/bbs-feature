# API Contract — External Attendance Integration

## Endpoint

### POST /api/v1/attendances/integration/bulk

**Authentication:** API Key (hardcoded token, reuse existing mechanism)

**Request:**
```json
{
  "token": "e5N7Zq4Szsw3TbFp5o8rNh",
  "records": [
    {
      "cardnumber": "28102019",
      "day": "2025-07-16",
      "timelog": "08:15:00",
      "status_gate": "1",
      "campusID": 1,
      "gateID": 2
    },
    {
      "cardnumber": "28102019",
      "day": "2025-07-16",
      "timelog": "15:30:00",
      "status_gate": "2",
      "campusID": 1,
      "gateID": 2
    },
    {
      "cardnumber": "99999999",
      "day": "2025-07-16",
      "timelog": "08:00:00",
      "status_gate": "1",
      "campusID": 1,
      "gateID": 2
    }
  ]
}
```

**Response:**
```json
{
  "data": {
    "total_success": 2,
    "total_failed": 1,
    "results": [
      {
        "cardnumber": "28102019",
        "day": "2025-07-16",
        "timelog": "08:15:00",
        "status_gate": "1",
        "status": "success",
        "attendance_id": 12345,
        "attendance_status": "LATE",
        "is_first_inserted": true
      },
      {
        "cardnumber": "28102019",
        "day": "2025-07-16",
        "timelog": "15:30:00",
        "status_gate": "2",
        "status": "success",
        "attendance_id": 12346,
        "is_first_inserted": false
      },
      {
        "cardnumber": "99999999",
        "day": "2025-07-16",
        "timelog": "08:00:00",
        "status_gate": "1",
        "status": "failed",
        "error": "Student not found"
      }
    ]
  }
}
```

## Payload Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `token` | string | Yes | API key (hardcoded for now) |
| `records` | array | Yes | Array of attendance records |
| `records[].cardnumber` | string | Yes | Student card number (find student by this) |
| `records[].day` | string | Yes | Date in `YYYY-MM-DD` format (GMT+7) |
| `records[].timelog` | string | Yes | Time in `HH:mm:ss` format (GMT+7) |
| `records[].status_gate` | string | Yes | `1` = IN (check-in), `2` = OUT (check-out) |
| `records[].campusID` | number | Yes | Campus ID (must exist in database) |
| `records[].gateID` | number | Yes | Gate ID (must exist in database) |

## Response Fields

| Field | Type | Description |
|---|---|---|
| `data.total_success` | number | Count of successfully processed records |
| `data.total_failed` | number | Count of failed records |
| `data.results` | array | Per-record results |
| `results[].status` | string | `success` or `failed` |
| `results[].attendance_id` | number | Attendance record ID (if success) |
| `results[].attendance_status` | string | `PRESENT` or `LATE` (if success) |
| `results[].is_first_inserted` | boolean | `true` if first record for the day |
| `results[].error` | string | Error message (if failed) |

## Error Cases

| Error | HTTP Code | Message |
|---|---|---|
| Invalid token | 401 | `Invalid API key` |
| Empty records array | 400 | `Records array cannot be empty` |
| Student not found | - | `Student not found` (in results) |
| Student not CURRENT | - | `Student is not active` (in results) |
| Campus not found | - | `Campus not found` (in results) |
| Gate not found | - | `Gate not found` (in results) |
| Check-out without check-in | - | `cardNumber {cardnumber} has no check-in record on {day}` (in results) |
| Late time setting not found | - | `Late time setting not found` (in results) |
| Class year not found | - | `Class year not found` (in results) |

## Notes

- Reuse existing `POST /api/v1/attendances/integration` endpoint logic
- Add new bulk endpoint or modify existing to accept array
- Process each record independently (partial success)
- Return HTTP 200 even if some records fail (check `total_failed` in response)
- Timezone: convert GMT+7 to UTC for DB storage (follow existing convention)
