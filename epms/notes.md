# Notes — EPMS Work Review (Employee Performance Management System)

## Ringkasan Analisis

| Portal legacy | Temuan |
|---------------|--------|
| Zone ASD (portal admin) | Menu Appraisal Summary Report (322), Appraisal Data Analysis (325), Appraisal Raw Data (326), campus charts (3231-3234), New Appraisal Teachers (3218), HOD (3219), Principal (32110) |
| Teacher portal | Menu id 341 (Appraisal Summary Report), 342 (HOD Appraisal Summary), **391 (EPMS → `principal_teacher_review.php`)**, 392 (Appraisal Lock/Unlock) |
| Principals portal | Menu sidebar "Appraisal/PETALS/Audit Per Group" — menggabungkan semua appraisal dalam satu group |

## Alur Legacy (reverse-engineer)

1. Principal login → menu EPMS id 391 → `staff/principal_teacher_review.php`
2. Halaman load → `js/principal_staff_review.js` (3987 bytes) → POST `services/principal_get_staff_review.php` (body: `branch=4&active=1&name=&ay=19`)
3. Daftar guru muncul (kolom: Name, Branch, Job Desc, Interschool, tombol Review)
4. Klik Review → form Work Review 7 section dengan kolom Semester 1 & 2
5. Isi skor → Submit → form name="form1" → textarea `s72` (RO Comments Sem 1) & `s74` (RO Comments Sem 2)

## Struktur Form EPMS (7 Section)

Detail lengkap di `reference/work_review_form.html` (hasil probe dari teacher web).

| Section | Nama | Jenis input |
|---------|------|-------------|
| Header | Job Desc, Interschool | 2 field teks |
| 1 | Key Performance Areas (KRA) | 5 sub-group, 9 item x 2 semester |
| 2 | Teaching Competencies | 7 item x 2 semester |
| 3 | Co-Curricular Activities | 2 item x 2 semester |
| 4 | Leadership Potentials and Professional Development | 2 item x 2 semester |
| 5 | Professional Qualities of A Teacher | 7 item x 2 semester |
| 6 | Training and Development Plans | textarea (colspan 2) |
| 7 | Review and Comments | Teacher's Comments + RO Comments (Sem 1 & 2) |

## Hubungan dengan PETALS

**EPMS ≠ PETALS.** Keduanya adalah fitur terpisah dalam modul Appraisal & Performance:

| Aspek | EPMS | PETALS |
|-------|------|--------|
| Nama | Employee Performance Management System | Lesson Observation framework |
| File legacy | `principal_teacher_review.php` | `petals_summary.php`, `asd_observation.php` |
| Sifat | Work Review tahunan (7 section, per semester) | Observasi kelas (18 item, per sesi) |
| Skor | ~100, grade A/B/C/D | 76 = 100% |
| Input | Principal (Reporting Officer) | Principal/HOD |

## NQ-01: Perbedaan menu "New Appraisal" vs "EPMS"

- **New Appraisal Teachers** (id 3218/329): `asd_staff_app_new.php` — daftar guru + tombol Appraisal (hijau/merah), kolom Score/Grade. Ini adalah **gerbang masuk** ke form PETALS (`asd_observation.php`) dan form Appraisal (`asd_appraisal_new.php`).
- **EPMS** (id 391): `principal_teacher_review.php` — Work Review tahunan 7 section, langsung form review tanpa perantara.

## NQ-02: Tingkatan appraisal

- **Teacher**: `asd_staff_app_new.php` → `asd_observation.php` (PETALS) + `asd_appraisal_new.php` (18 dimensi)
- **HOD**: `asd_staff_app_hod_new.php` (id 3219) — varian untuk HOD
- **Principal**: `asd_staff_app_principal_new.php` (id 32110) — varian untuk Principal
- **EPMS (semua)**: `principal_teacher_review.php` — Principal mereview semua guru

## NQ-03: Konvensi Penamaan

| Folder slug | Deskripsi | Scope |
|-------------|-----------|-------|
| `epms` | EPMS Work Review — fitur ini | Principal mereview kinerja guru tahunan |
| `petals` | PETALS Summary Report (report view) | Melihat hasil observasi PETALS |
| `petals-observation` | PETALS Lesson Observation Input | Input skor observasi 18 item |
| `epetals-dashboard` | E-PETALS Dashboard & Petal Chart | Multi-campus dashboard + chart |
