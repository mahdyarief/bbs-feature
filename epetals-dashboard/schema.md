# Schema — E-PETALS Dashboard & Petal Chart

> Status: DRAFT — mengikuti konvensi TypeORM 0.3.10 di `api_nest` (PostgreSQL).

## Tidak ada tabel baru

E-PETALS Dashboard dan Petal Chart adalah **view agregasi** dari tabel yang sudah ada di modul PETALS report (`teacher_appraisal`).

| Tabel sumber | Dipakai untuk |
|--------------|---------------|
| `teacher_appraisal` (lihat `features/petals/schema.md`) | Agregasi avg per campus, ringkasan per campus |
| `teacher_observation` / `teacher_observation_score` (lihat `features/petals-observation/schema.md`) | Alternatif jika agregasi dihitung on-the-fly dari skor per item |

## Query agregasi chart (AVG Per Campus)

```sql
SELECT
  ta.campus_id,
  c.name AS campus_name,
  AVG((ta.score_p + ta.score_e + ta.score_t + ta.score_a + ta.score_l) / 76.0 * 100) AS avg_pct,
  COUNT(DISTINCT ta.teacher_id) AS teacher_count
FROM teacher_appraisal ta
JOIN campus c ON c.id = ta.campus_id
WHERE ta.academic_year_id = :ay
  AND ta.active_status = 'ACTIVE'
GROUP BY ta.campus_id, c.name
ORDER BY c.name ASC;
```

- `avg_pct` dalam persen (0-100).
- Campus tanpa data TIDAK muncul di hasil (frontend bisa skip atau tampilkan 0 — lihat `edgecases.md` EC-EP-03).

## Migration

- Tidak ada migration baru untuk tabel. Jika mau, bisa dibuat **view**:
  `npm run migration:generate --name=create-view-epetals-avg-per-campus` (opsional — cukup dihitung di query).

## Relasi

Tidak ada entity baru. Hanya `ManyToOne` dari `teacher_appraisal` ke `campus` dan `academic_year` yang reuse dari modul petals.
