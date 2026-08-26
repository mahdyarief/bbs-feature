# Notes — PETALS Summary Report

> Status: DRAFT — open questions, decisions, discussion notes.

## Open Questions

### NQ-01: Apa kepanjangan PETALS? — **TERJAWAB (via PETALS Form.pdf + form input)**
PETALS adalah akronim **Lesson Observation** framework, bukan appraisal EPMS. Dari `PETALS_Form.pdf` (2 halaman) dan form input `asd_observation.php`:
- **P** = Pedagogy
- **E** = Experiences of Learning
- **T** = Tone of Environment
- **A** = Assessment for Learning
- **L** = Learning Content

### NQ-02: Apakah PETALS hanya untuk Principal/HOD, atau teacher juga bisa lihat?
Teacher web: menu id 343 ada di getmenulist.json tapi tidak muncul di sidebar teacher (home_direct.html). Akses via rmenu menampilkan shell fallback (iframe kosong). Ini mengindikasikan PETALS adalah **staff/principal-only feature**. Tapi perlu konfirmasi apakah teacher bisa melihat skor sendiri (read-only) di fase 2.

### NQ-03: Di mana data PETALS di-entry? — **TERJAWAB (bukan EPMS!)**
Data PETALS TIDAK di-entry dari EPMS (`teacher_review.php`). EPMS adalah **Work Review** terpisah (Section 1-7: KRA, Teaching Competencies, Co-Curricular, Leadership, Professional Qualities, Training, Review/Comments — skor per semester, sistem skor 100, grade A/B/C/D). PETALS adalah **Lesson Observation** dengan rubrik 18 item yang di-entry via **`asd_observation.php?userid=<id>&tname=<nama>&tipe=1`**.

**Alur input PETALS (hasil reverse-engineer):**
1. `asd_staff_app_new.php` (menu "New Apprisal Teachers" id 329) — daftar semua guru dengan tombol "Appraisal" per guru (btn hijau=Completed, merah=Incomplete) + kolom Score (Grade) + link Blank Form/PDF Report.
2. Klik tombol Appraisal → JS `asd_staff_app_asd_new.js` baris 141-145 → `window.location.replace('asd_appraisal_new.php?userid=<id>&tname=<nama>&tipe=1')` (form appraisal 18 dimensi).
3. Form observasi PETALS ada di **`asd_observation.php`** (dibuka via handler `ob_` di JS yang sama, baris 152-156: `window.open('asd_observation.php?userid=<id>&tname=<nama>&tipe=1')`).
4. `asd_observation.php` menampilkan "PETALs Form of <nama>" — 18 item observasi (dropdown mark 0-4 per item, `did` = id item di DB, nilai dari `services/app_get_observation.php`), 5 kartu ringkasan skor P/E/T/A/L + Average, dan 2 textarea `str_<staffid>` (Strength) & `ar_<staffid>` (Areas of Concern).
5. Simpan skor via service `services/update_appraisal_new.php` (POST `value&recid&staffid&user_update&state`).
6. Data skor PETALS di-agregasi ke halaman `petals_summary.php` (report view).

**Perbandingan skema:**
- PETALS: P(0-12), E(0-12), T(0-20), A(0-24), L(0-8), total 76 = 100%. Dari dropdown mark 0-4 × jumlah item (P=3 item, E=3, T=5, A=6, L=2 → max 12/12/20/24/8).
- EPMS/Appraisal: 18 dimensi, total ~100, grade A/B/C/D (mis. 86.04(B)).

### NQ-04: Permission module `PETALS` di `ModulesTypeEnum`?
Modul baru butuh entri baru di `src/types/enums` (`ModulesTypeEnum.PETALS` atau `ModulesTypeEnum.TEACHER_APPRAISAL`) + ACL entry di database (modul `casl`). Hanya untuk role Principal/HOD/Staff.

## Keputusan yang Perlu Di-review

| # | Keputusan | Status |
|---|-----------|--------|
| D-01 | Nama modul backend: `teacher-appraisal` (bukan `petals`) — lebih deskriptif | TBD |
| D-02 | Report hanya untuk Principal/HOD (bukan teacher biasa) | TBD |
| D-03 | Export Excel menggunakan `exceljs` library | TBD — cek library existing di api_nest |
| D-04 | Filter AY + Campus dropdown di report | TBD — lihat EC-03 dan EC-06 |
| D-05 | **Dual portal**: implementasi di Teacher Portal (`client-teacher`, pengguna utama Principal/HOD) + Admin Portal (`client/`, mirroring — admin bantu pengelolaan appraisal/export atas nama Principal/HOD, permission `PETALS_MANAGE`) | Disetujui — lihat spec.md "Dual Portal (Mirroring)" |

## Konvensi Penamaan

| Prefix | Lokasi | Arti |
|--------|--------|------|
| `petals` | `features/petals/` | PETALS framework itu sendiri (Lesson Observation) — report summary, schema, seed |
| `petals-observation` | `features/petals-observation/` | Modul input skor observasi (18 item rubrik, mark 0-4) |
| `epetals-dashboard` | `features/epetals-dashboard/` | Varian multi-campus dashboard + chart (prefiks "E-" dari legacy "E-PETALS") |

**Aturan:** `petals` = base framework, `petals-observation` = input, `epetals-dashboard` = multi-campus shell/chart. Tidak perlu rename folder — perbedaan ini sengaja dipertahankan karena scope-nya berbeda.

## Enhancement Ideas (di luar scope fase 1)

- Dril-down: klik nama guru → lihat detail appraisal per observer.
- Teacher view: guru bisa lihat skor PETALS sendiri (read-only).
- Perbandingan antar campus untuk Super Admin.
- Chart visual: bar chart distribusi skor per dimensi.
- History trend: bandingkan skor antar AY untuk guru yang sama.

## Referensi Analisis Proxy

PETALS tidak bisa diakses langsung via `staff/petals_summary.php` (404). Harus melalui proxy `link_preschool_dashboard.php` dengan parameter `links=<base64 path>`. Pola:

- `base64("../../staff/petals_summary.php")` = `Li4vLi4vc3RhZmYvcGV0YWxzX3N1bW1hcnkucGhw`
- URL lengkap: `https://zone.binabangsaschool.com/ais/teachers/link_preschool_dashboard.php?avsdjasgdjahdhas=MjEwNDY=&links=Li4vLi4vc3RhZmYvcGV0YWxzX3N1bW1hcnkucGhw`
- Export: `print_petals_summary.php` — trigger download file.

## Analisis Ekosistem Legacy (varian POV) — untuk sinkronisasi

Ekosistem PETALS ternyata **lebih luas** dari `petals_summary.php` (versi teacher POV, single-campus PIK-S). Berikut temuan dari POV lain:

### POV Principal
- Akses **langsung** via `zone.binabangsaschool.com/staff/...` (tanpa proxy!) dengan param `?campus=0` (semua campus):
  - `staff/asd_staff_app_new.php?campus=0` — 409KB, daftar semua guru (98 staff, lintas campus)
  - `staff/asd_staff_app_hod_new.php?campus=0` — 71KB
- Tidak ada halaman PETALS summary tersendiri di POV principal — yang ada adalah **New Appraisal** (input) dan aksesnya via `campus=0`.

### POV Admin/BBS
**Staff Database & Appraisal:**
- Service data: `staff_get_staff_dbais_new.php` (466KB, teachers), `staff_get_staff_hod_new.php` (66KB), `staff_get_staff_principal_new.php` (14KB) — **ada 3 tingkatan: Teacher / HOD / Principal**
- Halaman: `asd_staff_app_new.php` (Teachers), `asd_staff_app_hod_new.php` (HOD), **`asd_staff_app_principal_new.php` (Principal)** — kandidat tambahan di spec kita
- Report: **`appraisal_summary_asd.php?campus=1&cname=KJP`** — versi `_asd` dengan param campus+cname (berbeda dari `appraisal_summary.php` di spec kita)
- **`appdata_index.php`** (Appraisal Data Analysis) dan **`library/html2excel/demo/appinfo.php`** (Raw data Excel)

**E-PETALS & Petal Chart (BELUM tercakup di spec kita):**
- **`staff/epetals_summary_asd.php`** (1.9KB, title "E-PETALS", userid=586) — shell multi-campus: menu KJ-P, KJ-S, PIK-P, PIK-S, BDG-P, BDG-S, SMG-P, SMG-S, MLG-P, MLG-S, BPN-P → masing-masing membuka `epetals_summary.php?camp_id=X` di iframe
- **`epetals_summary.php?camp_id=X`** — data E-PETALS per campus (konten iframe)
- **`ais/asd/epetal_chart.php`** (6.3KB) — Chart.js v2.7.2 "AVG Per Campus" bar chart (`canvas#perCampusChart`, `js/epetals_chart.js`, hidden `ay=27`)

### Implikasi untuk spec kita
1. Spec saat ini hanya meniru `petals_summary.php` (teacher POV, single campus). **E-PETALS** (`epetals_summary_asd.php` + `epetals_summary.php?camp_id=X`) adalah versi multi-campus yang lebih baru/komplit — perlu dipertimbangkan sebagai referensi tambahan untuk AC-2 (filter campus) dan enhancement chart.
2. Ada **tiga tingkatan appraisal** (Teacher/HOD/Principal) — spec kita hanya menyebut Principal/HOD sebagai pengguna; perlu klarifikasi apakah report PETALS mencakup ketiganya.
3. `appraisal_summary_asd.php` (versi dengan param campus) lebih relevan untuk mirroring admin lintas campus daripada `appraisal_summary.php` di spec kita.