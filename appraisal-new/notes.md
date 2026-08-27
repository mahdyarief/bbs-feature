# Notes — New Appraisal / Staff Database (Gateway Appraisal)

## Ringkasan Analisis

| Portal legacy | Temuan |
|---------------|--------|
| Portal ASD | Menu **New Appraisal Teachers (3218)** → `asd_staff_app_new.php` (title "BBS Staff Database", ~640 guru); **HOD (3219)** → `asd_staff_app_hod_new.php` (~69 guru); **Principal (32110)** → `asd_staff_app_principal_new.php` (~15 guru) |
| Teacher portal | Menu id 341 (Appraisal Summary Report), 342 (HOD Appraisal Summary), 392 (Appraisal Lock/Unlock) |
| Principals portal | Menu sidebar "Appraisal/PETALS/Audit Per Group" — menggabungkan semua appraisal dalam satu group |

## Alur Legacy (reverse-engineer)

1. Principal/HOD login → menu New Appraisal (id 3218/3219/32110) → `staff/asd_staff_app_new.php` (atau varian HOD/Principal)
2. Halaman load → JS `loadData()` → POST `services/get_staff_app.php` (body: `branch=4&active=1&name=`)
3. Daftar guru muncul dengan kolom: #, User ID, Name, Campus, Appraisal (tombol warna), Score (Grade), Date (last submit)
4. Warna tombol Appraisal: **RED** = Incomplete, **GREEN** = Completed (legend tercetak di halaman)
5. Score tampil sebagai `95.520833333333(A)` / `80.364583333333(B)` / `0()` jika belum diisi; format tanggal `2026-04-14 08:30:02`
6. Klik tombol Appraisal → masuk form PETALS (`asd_observation.php`) atau form Appraisal 18 dimensi (`asd_appraisal_new.php`)
7. Tombol PDF Report hanya aktif jika Completed; Blank Form selalu tersedia (cetak form kosong)

## Hubungan dengan PETALS / EPMS

**Fitur ini adalah GATEWAY, bukan instrumen penilaian.** Halaman hanya menampilkan status + score; skor itu sendiri dihasilkan oleh instrumen:

| Aspek | Appraisal-new (fitur ini) | PETALS | EPMS |
|-------|---------------------------|--------|------|
| Peran | Gateway / Staff Database | Instrumen Lesson Observation | Instrumen Work Review tahunan |
| File legacy | `asd_staff_app_new.php` (+ HOD/Principal varian) | `asd_observation.php`, `petals_summary.php` | `principal_teacher_review.php` |
| Sifat | Daftar guru + status + score (monitoring) | Observasi kelas (18 item, mark 0-4, 76 = 100%) | Work Review 7 section per semester |
| Data | Membaca score dari `staff_appraisal_score` | Menghasilkan score | Menghasilkan score sendiri (tidak terkait) |

**Arah aliran data:** PETALS (input) → `staff_appraisal_score` → halaman ini (tampil score+grade). Fitur ini TIDAK menulis skor PETALS, hanya menampilkan agregasi.

## NQ-01: Konvensi Penamaan

| Folder slug | Deskripsi | Scope |
|-------------|-----------|-------|
| `appraisal-new` | New Appraisal / Staff Database — fitur ini | Gateway: daftar guru + status + score/grade, gerbang ke PETALS/EPMS |
| `petals` | PETALS Summary Report (report view) | Melihat hasil observasi PETALS |
| `petals-observation` | PETALS Lesson Observation Input | Input skor observasi 18 item |
| `epetals-dashboard` | E-PETALS Dashboard & Petal Chart | Multi-campus dashboard + chart |
| `epms` | EPMS Work Review | Principal mereview kinerja guru tahunan |

## NQ-02: Varian Halaman

- **Teacher view** (`asd_staff_app_new.php`, id 3218): daftar semua guru (~640) dengan score tunggal.
- **HOD view** (`asd_staff_app_hod_new.php`, id 3219): daftar guru yang di-appraise HOD (~69) dengan kolom "Appraisal Teacher(80%)", "Appraisal HOD (20%)", "Score (Grade)" — menunjukkan weighting 80/20.
- **Principal view** (`asd_staff_app_principal_new.php`, id 32110): daftar guru yang di-appraise Principal (~15) dengan score tunggal; menampilkan "Never!" jika skor 0.
