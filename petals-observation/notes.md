# Notes — PETALS Lesson Observation Input

> Status: DRAFT — open questions, decisions, discussion notes.

## Open Questions

### OQ-OBS-01: Apakah list guru observasi bisa digabung dengan modul employee?

List guru di `asd_staff_app_new.php` legacy menampilkan semua guru di campus. Di sistem baru, list guru untuk observasi bisa menggunakan data dari modul `Employee` yang sudah ada. Perlu filter: hanya guru aktif di campus user, dengan status observasi (Completed/Incomplete) dari tabel `teacher_observation`.

### OQ-OBS-02: Bagaimana dengan status "Completed" vs "Incomplete"?

Di legacy (`asd_staff_app_new.php`), tombol Appraisal berwarna merah (Incomplete) atau hijau (Completed). Status ini perlu dipertahankan: DRAFT = merah (belum submit), SUBMITTED = hijau (sudah submit). Tapi bagaimana menentukan "completed"? Apakah semua 18 item harus di-mark? Atau minimal observer bisa menandai sebagai submitted meskipun beberapa item belum di-mark?

### OQ-OBS-03: Apakah rubrik 18 item bisa dikonfigurasi?

Di legacy, rubrik 18 item tampaknya hardcoded (di-seed di DB). Apakah sistem baru perlu menyediakan UI untuk mengelola item rubrik? Untuk fase 1, cukup di-seed dari PETALS Form.pdf. Enhancement di fase 2.

### OQ-OBS-04: Bagaimana relasi dengan modul Appraisal (asd_appraisal_new.php)?

`asd_appraisal_new.php` adalah form **appraisal 18 dimensi** (skor 100, grade A-D) — sistem berbeda dari PETALS. Keduanya diakses dari halaman yang sama (`asd_staff_app_new.php`) tapi merupakan modul terpisah. Appraisal 18 dimensi tidak termasuk dalam scope PETALS.

## Keputusan yang Perlu Di-review

| # | Keputusan | Status |
|---|-----------|--------|
| D-OBS-01 | Nama modul: `petals-observation` (backend: `teacher-observation`) | TBD |
| D-OBS-02 | Rubrik 18 item di-seed (dari `PETALS_Form.pdf`), bukan konfigurabel di fase 1 | TBD |
| D-OBS-03 | Status: DRAFT / SUBMITTED (mirip Completed/Incomplete legacy) | TBD |
| D-OBS-04 | Agregasi ke `teacher_appraisal` via query on-the-fly (bukan trigger) | TBD |
| D-OBS-05 | `strength` & `areasOfConcern` tetap **free-text** (level observasi, TANPA relasi/tagging ke dimensi PETALS) — hanya informasi tambahan untuk pembaca report | Disetujui (keputusan stakeholder) |

## Konvensi Penamaan

| Prefix | Lokasi | Arti |
|--------|--------|------|
| `petals` | `features/petals/` | PETALS framework itu sendiri (Lesson Observation) — report summary, schema, seed |
| `petals-observation` | `features/petals-observation/` | Modul input skor observasi (18 item rubrik, mark 0-4) — brief ini |
| `epetals-dashboard` | `features/epetals-dashboard/` | Varian multi-campus dashboard + chart (prefiks "E-" dari legacy "E-PETALS") |

**Aturan:** `petals` = base framework, `petals-observation` = input, `epetals-dashboard` = multi-campus shell/chart. Tidak perlu rename folder — perbedaan ini sengaja dipertahankan karena scope-nya berbeda.

## Referensi Analisis Proxy

- Menu: `staff/asd_staff_app_new.php` (id 329) — proxy: `../../staff/asd_staff_app_new.php`
- Form observasi: `staff/asd_observation.php?userid=<id>&tname=<nama>&tipe=1` — proxy: `../../staff/asd_observation.php`
- JS handler: `staff/js/asd_staff_app_asd_new.js` (baris 152-156: `window.open('asd_observation.php?userid=<id>&tname=<nama>&tipe=1')`)
- Service load: `staff/services/app_get_observation.php` (POST `staffid=<id>`)
- Service save: `staff/services/update_appraisal_new.php` (POST `value&recid&staffid&user_update&state`)
- JS form: `staff/js/asd_observation.js`