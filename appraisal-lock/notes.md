# Notes — Appraisal Lock/Unlock

## Ringkasan Analisis

| Portal legacy | Temuan |
|---------------|--------|
| Zone ASD (portal admin) | `view_lock.php` (80KB) — lock summary page, tab Academic/CCA/Remarks (dan prelim), satu baris per guru + toggle lock. Varian `view_lock_cca.php`, `view_lock_remarks.php`. |
| Teacher portal | Menu id **392** — "Appraisal Lock/Unlock" (`appraisal_lockunlock.php`); probe `/ais/teachers/appraisal_lockunlock.php` → **404** → halaman aktual di bawah `/staff/` zone portal ASD → fungsi ini bersifat ASD/principal-side admin |
| Principals portal | Menampilkan halaman lock view (tab Academic/CCA/Remarks) untuk mengunci skor appraisal guru per campus |
| View pendukung | `teacher_score.php` — teacher score view (skor guru per item, direferensikan untuk konteks skor yang di-lock) |

## Alur Legacy (reverse-engineer)

1. Menu "Appraisal Lock/Unlock" id 392 di portal teacher → `appraisal_lockunlock.php` → 404 di path teacher → sebenarnya hidup di `/staff/` (zone portal ASD).
2. Halaman lock view utama = `view_lock.php` (80KB) — tab **Academic / CCA / Remarks (dan prelim)**, satu baris per guru dengan toggle lock/unlock.
3. Admin memilih AY + campus → memilih tab → tabel guru + status lock → toggle per guru → save.
4. Status tersimpan: `LOCKED` (tidak bisa diedit) / `UNLOCKED` (terbuka untuk koreksi).
5. Varian per tab: `view_lock_cca.php` (CCA), `view_lock_remarks.php` (Remarks).

## Hubungan dengan appraisal-new & EPMS (lock layer di atas submitted status)

**Appraisal Lock/Unlock adalah lapisan workflow control DI ATAS status submit skor.** Konsisten dengan `EC-EP-07` (brief `features/epms/`):

| Aspek | EPMS / New Appraisal (skor) | Appraisal Lock/Unlock (fitur ini) |
|-------|-----------------------------|-----------------------------------|
| Sifat | Menyimpan/men-submit skor appraisal | Mengunci/membuka entri skor per guru |
| Status | `DRAFT` / `SUBMITTED` (review) | `LOCKED` / `UNLOCKED` (lock flag) |
| Edit SUBMITTED | Ditolak 409 (EC-EP-07) | Lock flag menolak edit 409 juga |
| Override | Admin `PETALS_MANAGE` bisa override | Unlock = override eksplisit yang tercatat di audit |
| Tabel | `teacher_review`, `teacher_review_score`, dll (EPMS) / skor appraisal-new | `appraisal_lock` + `appraisal_lock_audit` |

- Lock/unlock tidak menggantikan status DRAFT/SUBMITTED — keduanya berjalan paralel. Unlock hanya membuka akses edit; submit ulang yang mengembalikan status SUBMITTED.
- Endpoint edit skor (EPMS `PUT /scores`, PETALS, New Appraisal) wajib mengecek `appraisal_lock` sebelum menulis → jika `is_locked = true` untuk (teacher, AY, tab), tolak 409.
- Referensi silang tanpa duplikasi konten: `features/epms/` (EC-EP-07), `features/appraisal-new/` (gateway skor), `features/appraisal-summary/` (report yang distabilkan oleh lock), `features/petals/` & `features/petals-observation/` & `features/epetals-dashboard/` (instrumen lain).

## Konvensi Penamaan

| Folder slug | Deskripsi | Scope |
|-------------|-----------|-------|
| `appraisal-lock` | Appraisal Lock/Unlock — fitur ini | Workflow control: kunci/buka entri skor per guru per AY per tab |
| `appraisal-new` | New Appraisal / Staff Database gateway | Gerbang input appraisal (PETALS + appraisal form) |
| `appraisal-summary` | Appraisal Summary Report | Report hasil appraisal (dilindungi oleh lock) |
| `epms` | EPMS Work Review | Work Review tahunan 7 section |
| `petals` / `petals-observation` | PETALS Summary / Lesson Observation Input | Observasi kelas 18 item |
| `epetals-dashboard` | E-PETALS Dashboard & Petal Chart | Multi-campus dashboard + chart |

## Catatan Desain

- **Dual portal:** Admin Portal (`client/`) = pengguna utama (ASD/principal lintas campus); Teacher Portal (`client-teacher/`) = mirroring (Principal per campus-nya; guru read-only).
- **Permission:** admin/principal/super admin → boleh lock/unlock; teacher biasa → 403.
- **Default UNLOCKED:** guru tanpa baris `appraisal_lock` dianggap terbuka — tidak perlu seeding.
- **Audit trail:** `appraisal_lock_audit` append-only mencatat actor, timestamp, tab, from_status → to_status.
- **Race condition:** optimistic locking via `updated_at` untuk mencegah konflik 2 admin (EC-AL-07).
- **Cross-tab:** tab independen; tab `APPRAISAL` bersifat master yang menolak edit semua tab (EC-AL-03).
