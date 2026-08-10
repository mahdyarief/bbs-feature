# Notes — FTP Evaluation Skip Empty Term

> Data trace dari **database production** (`binabangsa_prod_mig_v01`, db-tools) untuk verifikasi engineer.
> Tanggal pengambilan: 2026-08-10.

## 1. Case Verifikasi (Data Trace)

**Student target:** Joses Elden Budiman (id **103033**) — masuk di term 3, academic year 2025/2026.

### Konteks relasi

| Tabel | Data |
|-------|------|
| `academic_year` | id **26** = `2025/2026` |
| `class_year` | id **100097** (academic_year_id 26, campus 5, classroom 1296, level 5, master_level_id 1) |
| `student_report` | id **3758** (student_id 103033, class_year_id 100097) — satu-satunya student_report di academic year 26 |

> Catatan: student juga punya student_report id 3883 (class_year 100939) tapi itu **academic year 27 (2026/2027)** — tidak relevan untuk case ini.

### `student_ftp_conduct` — 72 rows, HANYA term 3 & 4 (0 rows term 1/2)

- 36 rows per term = 12 `ftp_report_table_id` × 3 `ftp_grade_master_subject_id`
- `report_status` = **2** untuk semua (DONE)
- Distribusi term: `term 3` → 36 rows, `term 4` → 36 rows, `term 1` → **0**, `term 2` → **0**

Contoh aggregasi badges per (table, term):

| ftp_report_table_id | term 3 (sum 3 subject) | term 4 (sum 3 subject) |
|---|---|---|
| 1 | 4 (1+1+2) | 4 (1+2+1) |
| 2 | 3 (1+1+1) | 4 (2+1+1) |
| 3 | 4 (1+2+1) | 4 (1+1+2) |
| 4 | 4 (1+2+1) | 5 (1+2+2) |
| 5 | 4 (2+1+1) | 4 (2+1+1) |
| 6 | 5 (1+2+2) | ... |

### `student_ftp_conduct_evaluation` — 4 rows, ADA untuk SEMUA term

| id | term | score | evaluation_mark | student_report_id | ftp_report_content_id |
|----|------|-------|------------------|-------------------|----------------------|
| 12899 | 1 | **21** | BELOW_EXPECTATIONS | 3758 | 1 |
| 12879 | 2 | **22** | BELOW_EXPECTATIONS | 3758 | 2 |
| 10895 | 3 | 21 | BELOW_EXPECTATIONS | 3758 | 3 |
| 12392 | 4 | 19 | BELOW_EXPECTATIONS | 3758 | 4 |

**Temuan kunci:** score term 1 = 21 dan term 2 = 22 **ADA** padahal tidak ada satupun conduct row term 1/2. Score tersebut dihitung dari data term 3/4 (lihat verifikasi di bawah).

## 2. Setting Evaluasi & Threshold (untuk verifikasi kalkulasi)

### `ftp_evaluation_setting` — 12 rows, tiap term memilih 3 table berbeda

| ftp_report_content_id (term) | ftp_report_table_id |
|------------------------------|---------------------|
| 1 (term 1) | 4, 5, 7 |
| 2 (term 2) | 1, 6, 10 |
| 3 (term 3) | 3, 11, 12 |
| 4 (term 4) | 2, 8, 9 |

### `ftp_grade_setting` — academic_year 26, master_level 1

| term | always | most_of_the_time | sometimes | less_observed |
|------|--------|------------------|-----------|---------------|
| 1 | 5 | 4 | 3 | 0 |
| 2 | 5 | 4 | 3 | 0 |
| 3 | 5 | 4 | 3 | 0 |
| 4 | 5 | 4 | 3 | 0 |

## 3. Verifikasi Manual Kalkulasi (kenapa score term 1 = 21)

Logika `ftpEvaluation` (ftp-evaluation.ts:62-83): filter by `ftpReportTableId` saja, group by table → group by term, tiap group badges di-sum → `getConductPoint` (≥5→4, ≥4→3, ≥3→2, else→1).

Untuk **evaluation term 1** (tables [4,5,7], threshold 5/4/3) dengan conduct yang tersedia (term 3 & 4 saja):

| table | badges (term 3) | point | badges (term 4) | point |
|-------|-----------------|-------|-----------------|-------|
| 4 | 4 | 3 | 5 | 4 |
| 5 | 4 | 3 | 4 | 3 |
| 7 | 4 | 3 | ~4 | 3 |

Total ≈ **3+3+3+4+3+3 = 19-21** → cocok dengan score term 1 = **21**.

**Kesimpulan verifikasi:** score evaluation term X = jumlah point dari conduct **SEMUA term yang ada datanya** (term 3+4) yang table-nya cocok dengan setting term X. Karena tidak ada filter term, term 1/2 tanpa data tetap mendapatkan score dari data term 3/4.

## 4. Query yang Dipakai (db-tools)

```bash
cd binabangsa-db-tools

# Academic year 2025/2026 → id 26
python -m db query "SELECT id, year FROM academic_year WHERE year LIKE '%2025%' OR year LIKE '%2026%' ORDER BY id"

# Student
python -m db query "SELECT id, full_name, nisn FROM student WHERE id = 103033"

# Student report (class_year 100097 = academic year 26)
python -m db query "SELECT id, academic_year_id, campus_id, classroom_id, level FROM class_year WHERE id IN (100097, 100939)"
python -m db query "SELECT id, student_id, class_year_id FROM student_report WHERE student_id = 103033"

# FTP conduct — hanya term 3 & 4
python -m db query "SELECT id, badges, term, report_status, ftp_report_table_id, ftp_grade_master_subject_id, student_report_id FROM student_ftp_conduct WHERE student_report_id = 3758 ORDER BY term, ftp_report_table_id"

# FTP evaluation — ada untuk term 1-4
python -m db query "SELECT id, term, score, evaluation_mark, student_report_id, ftp_report_content_id FROM student_ftp_conduct_evaluation WHERE student_report_id = 3758 ORDER BY term"

# Setting evaluasi per term
python -m db query "SELECT id, ftp_report_content_id, ftp_report_table_id FROM ftp_evaluation_setting ORDER BY ftp_report_content_id, ftp_report_table_id"

# Threshold grade setting
python -m db query "SELECT id, term, always, most_of_the_time, sometimes, less_observed, master_level_id, academic_year_id FROM ftp_grade_setting WHERE academic_year_id = 26 AND master_level_id = 1 ORDER BY term"
```

> ⚠️ Catatan teknis: db-tools `query` memblokir operasi yang mengandung substring "DELETE" — jangan sertakan kolom `deleted_at` di query SELECT, atau gunakan command `data` (tidak ada deteksi write).

## 5. Pertanyaan Terbuka untuk Tim

1. **Sumber "term masuk" student** — apakah akan pakai derive dari data (min term yang punya conduct rows) atau perlu field baru? (Lihat spec.md Scope — field `entry_term` di-out-of-scope sementara.)
2. ~~Cleanup data yang sudah tercemar~~ — **SUDAH DIPUTUSKAN (2026-08-10):** tidak ada script cleanup terpisah. Eval rows term 1/2 yang sudah ada (id 12899, 12879) **tetap di DB; nilainya akan diperbarui (recompute) saat report di-update** setelah implementasi kalkulasi baru. (EC-03, keputusan: opsi C)
3. ~~Definisi "kosong" untuk badges 0~~ — **SUDAH DIPUTUSKAN (2026-08-10):** null (tidak ada row) = diabaikan; row ada dengan badges 0 = **tetap dihitung** point minimum 1 per (table, term). (EC-02, keputusan: opsi B)
4. **Threshold mark untuk partial-year** — apakah normalisasi threshold (≥42/≥25) masuk backlog terpisah? (Out of scope brief ini.)
