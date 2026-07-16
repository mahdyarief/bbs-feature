# Schema Changes — GPA Threshold Config Fix

## Table: `threshold`

### BEFORE (Current)

| Column | Type | Note |
|--------|------|------|
| id | INT PK | Auto increment |
| effective_from | YEAR | Tahun awal berlaku (range) |
| effective_to | YEAR | Tahun akhir berlaku (range) |
| status | INT | 1 = active, 0 = inactive |
| avg_value | DECIMAL | Nilai avg threshold |
| ... | ... | Kolom lainnya |

**No `academic_year_id`.** Threshold only linked to year range.

### AFTER (New)

| Column | Type | Note |
|--------|------|------|
| id | INT PK | Auto increment |
| **academic_year_id** | **INT FK** | **NEW** — FK ke `academic_year.id`, nullable |
| effective_from | YEAR | Single year parameter (contoh: 2022) |
| effective_to | YEAR | Single year parameter (contoh: 2022) |
| status | INT | 1 = active, 0 = inactive |
| avg_value | DECIMAL | Nilai avg threshold |
| ... | ... | Kolom lainnya |

## Migration

### Step 1: Add Column (nullable)

```sql
ALTER TABLE threshold
ADD COLUMN academic_year_id INT NULL;

ALTER TABLE threshold
ADD CONSTRAINT fk_threshold_academic_year
FOREIGN KEY (academic_year_id) REFERENCES academic_year(id);
```

### Step 2: Backfill (Phase 2, not mandatory)

```sql
-- Mapping needs to be documented and reviewed before execution
-- Each existing threshold row must be assigned to the correct AY
UPDATE threshold SET academic_year_id = <AY_ID> WHERE id = <ID>;
```

### Step 3: Make NOT NULL (after backfill complete)

```sql
ALTER TABLE threshold
ALTER COLUMN academic_year_id SET NOT NULL;
```

## Data Comparison

| Aspect | BEFORE | AFTER |
|--------|--------|-------|
| Relation | `effective_from` + `effective_to` (range) | `academic_year_id` (FK) + single year |
| Scope | Lintas tahun (1 row = multi tahun) | Per AY (1 row = 1 tahun di 1 AY) |
| Active | `status` 1/0 per range | `status` 1/0 per row per AY |

## Sample Data

### AY ID 25 (2025/2026):

| id | academic_year_id | effective_from | effective_to | status | avg_value |
|----|------------------|----------------|--------------|--------|-----------|
| 1 | 25 | 2022 | 2022 | 1 | 2.50 |
| 2 | 25 | 2023 | 2023 | 1 | 2.55 |
| 3 | 25 | 2024 | 2024 | 1 | 2.60 |

### AY ID 26 (2026/2027) — after copy:

| id | academic_year_id | effective_from | effective_to | status | avg_value | Note |
|----|------------------|----------------|--------------|--------|-----------|------|
| 4 | 26 | 2023 | 2023 | 1 | 2.55 | Copy dari AY25 id=2 (entry baru) |
| 5 | 26 | 2024 | 2024 | 1 | 2.60 | Copy dari AY25 id=3 (entry baru) |
| 6 | 26 | 2025 | 2025 | 1 | 0.00 | Isi sendiri (tahun baru) |

## Notes

- FK is nullable initially for backward compatibility
- Existing rows without `academic_year_id` need backfill in separate phase
- `effective_from` / `effective_to` columns retained but semantics changed (single year)
