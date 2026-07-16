# API Contract — GPA Threshold Config Fix

## Endpoint Changes

| Method | Path | Description | Status |
|--------|------|-------------|--------|
| GET | `/api/v1/thresholds` | List thresholds — now filtered by `academic_year_id` | Modified |
| POST | `/api/v1/thresholds` | Create threshold — now requires `academic_year_id` | Modified |
| POST | `/api/v1/thresholds/copy` | **New** — Auto-copy thresholds from previous AY | New |
| PUT | `/api/v1/thresholds/:id` | Update threshold — `academic_year_id` in body/response | Modified |
| GET | `/api/v1/thresholds/academic-years` | **New** — List AYs with threshold config status | New |

## Request / Response Shapes

### POST /api/v1/thresholds

**Request:**
```json
{
  "academic_year_id": 25,
  "effective_from": 2022,
  "effective_to": 2022,
  "status": 1,
  "avg_value": 2.50,
  "papers": [
    { "paper_name": "Paper 1", "weight": 0.6, "total_marks": 100 }
  ]
}
```

**Response:**
```json
{
  "data": {
    "id": 1,
    "academic_year_id": 25,
    "effective_from": 2022,
    "effective_to": 2022,
    "status": 1,
    "avg_value": 2.50,
    "created_at": "2025-07-15T..."
  }
}
```

### POST /api/v1/thresholds/copy

**Request:**
```json
{
  "target_academic_year_id": 26,
  "source_academic_year_id": 25
}
```

**Response:**
```json
{
  "data": {
    "copied_count": 2,
    "entries": [
      { "source_id": 2, "new_id": 4, "effective_from": 2023 },
      { "source_id": 3, "new_id": 5, "effective_from": 2024 }
    ]
  }
}
```

### GET /api/v1/thresholds?academic_year_id=25

**Query params:**
- `academic_year_id` (required) — Filter by AY
- `page`, `limit` — Pagination

**Response:**
```json
{
  "data": [
    { "id": 1, "effective_from": 2022, "effective_to": 2022, "avg_value": 2.50, "status": 1 },
    { "id": 2, "effective_from": 2023, "effective_to": 2023, "avg_value": 2.55, "status": 1 },
    { "id": 3, "effective_from": 2024, "effective_to": 2024, "avg_value": 2.60, "status": 1 }
  ],
  "meta": { "total": 3 }
}
```

## Notes

- `effective_from` / `effective_to` tetap ada tapi sekarang single year (bukan range)
- Backward compatible: endpoint lama tetap jalan, hanya tambah filter `academic_year_id`
- Copy endpoint bisa dipanggil dari UI (auto) atau manual (admin tool)
