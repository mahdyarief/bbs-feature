# Notes — E-PETALS Dashboard & Petal Chart

> Status: DRAFT — open questions, decisions, discussion notes.

## Open Questions

### OQ-EP-01: Siapa yang bisa akses E-PETALS lintas campus?

Di legacy, `epetals_summary_asd.php` memakai `userid=586` (Super Admin/HQ). Di sistem baru, perlu klarifikasi role mana yang berhak: Super Admin, Principal HQ, atau semua Principal? Rekomendasi: batasi ke Super Admin + Principal HQ + Admin (`PETALS_MANAGE`). Principal campus biasa pakai PETALS single-campus.

### OQ-EP-02: Apakah chart perlu menampilkan trend antar AY?

Legacy `epetal_chart.php` hanya menampilkan satu AY (hidden `ay=27`). Enhancement: dropdown AY + perbandingan multi-AY. Fase 1 cukup satu AY + filter.

### OQ-EP-03: Chart library mana yang dipakai di frontend baru?

Legacy memakai Chart.js v2.7.2. Sistem baru (`bbs`) perlu cek library yang sudah dipakai (recharts/echarts). Ini keputusan implementasi frontend — perlu dicek di `bbs/client-*/package.json`.

### OQ-EP-04: Apakah E-PETALS perlu dipecah dari report PETALS?

Iya — ini brief terpisah (`features/epetals-dashboard/`) karena scope-nya berbeda: multi-campus + chart, bukan single-campus report. Tapi backend reuse modul `petals` (agregasi sama).

## Keputusan yang Perlu Di-review

| # | Keputusan | Status |
|---|-----------|--------|
| D-EP-01 | E-PETALS adalah modul terpisah dari PETALS report (scope lintas campus + chart) | TBD |
| D-EP-02 | Chart library frontend: recharts/echarts (ganti Chart.js legacy) | TBD — cek package.json bbs |
| D-EP-03 | Backend: modul `epetals` dengan 3 endpoint (campuses, summary wrapper, chart) | TBD |
| D-EP-04 | Hanya Super Admin/Principal HQ/Admin (`PETALS_MANAGE`) yang akses lintas campus | TBD |

## Referensi Analisis Proxy

- Shell: `staff/epetals_summary_asd.php` — proxy: `../../staff/epetals_summary_asd.php`
- Konten per campus: `staff/epetals_summary.php?camp_id=X` — proxy: `../../staff/epetals_summary.php`
- Chart: `ais/asd/epetal_chart.php` — proxy: `../../ais/asd/epetal_chart.php`
- JS chart: `js/epetals_chart.js` (di halaman chart, hidden `ay=27`, `current_ay=27`)
- Data source: sama dengan `features/petals/` (tabel `teacher_appraisal`)

## Catatan Akses (role)

- Halaman `epetals_summary_asd.php` dan `epetal_chart.php` muncul di menu home dashboard (varian Admin).
- Di POV Principal, halaman E-PETALS ini tidak muncul — yang ada hanya `asd_staff_app_new.php?campus=0` (New Appraisal). Jadi E-PETALS kemungkinan khusus diakses oleh role Super Admin/Admin (bukan Principal campus).

## Konvensi Penamaan

| Prefix | Lokasi | Arti |
|--------|--------|------|
| `petals` | `features/petals/` | PETALS framework itu sendiri (Lesson Observation) — report summary, schema, seed |
| `petals-observation` | `features/petals-observation/` | Modul input skor observasi (18 item rubrik, mark 0-4) |
| `epetals-dashboard` | `features/epetals-dashboard/` | Varian multi-campus dashboard + chart (prefiks "E-" dari legacy "E-PETALS") — brief ini |

**Aturan:** `petals` = base framework, `petals-observation` = input, `epetals-dashboard` = multi-campus shell/chart. Tidak perlu rename folder — perbedaan ini sengaja dipertahankan karena scope-nya berbeda.
