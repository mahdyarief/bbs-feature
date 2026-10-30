---
title: Teacher Secondary/JC — Employment & Branch Kosong (57 baris)
status: open
severity: major
product: BBS LMS
portal: Admin
author: System Analyst
date: 2026-10-13
jam:
---

# Teacher Secondary/JC — Employment (employee_types) & Branch kosong

## Summary

Ditemukan **57 employee** yang mengajar (punya jadwal) di level **Secondary** dan/atau **Junior College** namun **employment-nya kosong** (`employee.employee_types` NULL / `{}` / `''`) **dan** **branch-nya kosong** (tidak ada baris di `employee_branches_branch`).

Kondisi ini berpotensi menyebabkan data kepegawaian tidak lengkap: employee terdaftar mengajar subject year di level Sec/JC tetapi tidak punya tipe employment maupun afiliasi branch, sehingga rawan bermasalah pada reporting HR, filtering guru per branch, dan fitur yang bergantung pada `employee_types` / branch.

**Ekspektasi:** Setiap employee yang aktif mengajar di Sec/JC memiliki `employee_types` yang valid dan memiliki minimal 1 baris branch di `employee_branches_branch`.

**Catatan:** 8 baris berstatus `active_status = 2` (non-aktif), termasuk **3 baris artifak migrasi** (`Unknown Teacher 781/22170/24678` — email `migrated.unknown.*`) yang kemungkinan sisa proses migrasi dan perlu diverifikasi apakah boleh diabaikan.

---

## Definisi Istilah (disepakati berdasarkan inspeksi schema live)

| Istilah | Definisi Operasional |
|---------|----------------------|
| **employment kosong** | `employee.employee_types IS NULL` OR `employee.employee_types::text IN ('{}','')` (array PostgreSQL kosong / string kosong). Tidak ada kolom literal `employment` di DB — konsep employment dipetakan ke `employee_types`. |
| **branch kosong** | Tidak ada baris relasi di `employee_branches_branch` untuk `employee.id` tersebut. |
| **mengajar Secondary/JC** | Employee terhubung ke `subject_year` (jadwal mengajar) → `class_year` → `programme` → `master_programme` dengan `master_programme_id IN (3,4)` (3 = Secondary, 4 = Junior College). Rantai via tabel relasi `subject_year_teachers_employee`. |

### Peta Relasi

```
master_programme (3=Secondary, 4=Junior College)
      ▲
      │ master_programme_id
   programme
      ▲
      │ programme_id
   class_year ──deleted_at IS NULL──┐
      ▲                             │
      │ class_year_id               │
   subject_year                      │
      ▲                             │
      │ subject_year_id             │
subject_year_teachers_employee       │
      ▲                             │
      │ employee_id                 │
   employee ────────────────────────┘
      │
      ├── employee_types (employment)   → kosong
      └── employee_branches_branch      → tidak ada baris (branch kosong)
```

> Catatan: tabel `employee_master_programmes_master_programme` **kosong (0 baris)**, sehingga jalur langsung employee→master_programme tidak dapat dipakai. Definisi "mengajar Sec/JC" dipakai via rantai jadwal mengajar di atas.

---

## Metode / Query yang Dipakai

Dijalankan read-only via `db.connection.connect(readonly=True)` (menghindari false-positive write-guard pada `db/data.py:58` yang mem-body substring `DELETE` di dalam `deleted_at`).

```sql
SELECT DISTINCT e.id, e.staff_id, e.full_name, e.email_bbs, e.active_status,
  COALESCE(e.employee_types::text,'NULL') AS emp_types,
  string_agg(DISTINCT mp.name, ', ' ORDER BY mp.name) AS programmes
FROM employee e
JOIN subject_year_teachers_employee syte ON syte.employee_id = e.id
JOIN subject_year sy  ON sy.id = syte.subject_year_id
JOIN class_year cy    ON cy.id = sy.class_year_id
JOIN programme p      ON p.id  = cy.programme_id
JOIN master_programme mp ON mp.id = p.master_programme_id
WHERE (e.employee_types IS NULL OR e.employee_types::text IN ('',''))
  AND NOT EXISTS (SELECT 1 FROM employee_branches_branch ebb WHERE ebb.employee_id = e.id)
  AND cy.deleted_at IS NULL
  AND p.master_programme_id IN (3,4)
GROUP BY e.id, e.staff_id, e.full_name, e.email_bbs, e.active_status, e.employee_types
ORDER BY e.full_name
```

**Distribusi nilai `employee_types` (keseluruhan tabel employee):** `{3}`:2088, `{2}`:250, `{}`:77, `NULL`:61, `{2,3}`:57, `{1}`:5, `{1,2,3}`:4, `{1,2}`:3.

---

## Hasil — 57 Employee (employment & branch kosong, mengajar Secondary/JC)

`active_status`: `1` = aktif, `2` = non-aktif.
`programmes`: level mengajar (Secondary / Junior College).

| No | employee_id | staff_id | Nama | Email | active_status | employee_types | Programmes |
|----|-------------|----------|------|-------|---------------|----------------|------------|
| 1 | 28382 | EMP28382 | Abigael Montemayor Manzano | abigael.piks@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 2 | 27630 | EMP27630 | Aloysius Gonzaga Adi Suryo Nugroho | aloysiusgonzaga.piks@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 3 | 27614 | EMP27614 | Andri Setyo Subagyo | andrisetyo.mlgs@binabangsaschool.com | 2 | NULL | Secondary |
| 4 | 27618 | EMP27618 | Casielyn Miraflores Capistrano | casielyn.mlgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 5 | 27177 | EMP27177 | Chandra Seto Agus | chandra.smgs@binabangsaschool.com | 1 | NULL | Secondary |
| 6 | 27631 | EMP27631 | Christine Joy Jaurigue David | christinejoy.piks@binabangsaschool.com | 1 | NULL | Secondary |
| 7 | 28757 | EMP28757 | Dameita Sumbayak | dameitasumbayak.piks@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 8 | 27632 | EMP27632 | Damien Martin Joseph Hutchman | damien.piks@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 9 | 27615 | EMP27615 | Daniel David Widiya | daniel.mlgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 10 | 27633 | EMP27633 | Desy Carolina | desy.piks@binabangsaschool.com | 1 | NULL | Secondary |
| 11 | 28771 | EMP28771 | Dusmant Bastarayi | dusmantbastarayi.mlgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 12 | 29130 | EMP29130 | Dwi Kurniati Halim | dwikurniati.bdg@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 13 | 28850 | EMP28850 | Edna Mamawag Saga | edna.mlgs@binabangsaschool.com | 1 | NULL | Secondary |
| 14 | 28756 | EMP28756 | Erick Heriyanto | erickheriyanto.piks@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 15 | 27693 | EMP27693 | Evert Octavianus Amanupunjo | evert.bdgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 16 | 29036 | EMP29036 | Feby Dharmastuti | feby.bdg@binabangsaschool.com | 1 | NULL | Secondary |
| 17 | 28484 | EMP28484 | Fransisca Agustin Suryaningtyas | fransisca.smgs@binabangsaschool.com | 1 | NULL | Secondary |
| 18 | 27698 | EMP27698 | Gary Lee Turnmire | gary.kjs@binabangsaschool.com | 2 | NULL | Secondary |
| 19 | 28759 | EMP28759 | Georgina Bailey Mills | georgina.piks@binabangsaschool.com | 1 | NULL | Secondary |
| 20 | 28760 | EMP28760 | Godlas Harianja | godlas.piks@binabangsaschool.com | 1 | NULL | Secondary |
| 21 | 28770 | EMP28770 | Godofredo Dilla Degsi Jr. | godofredodegsi.mlgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 22 | 27620 | EMP27620 | Guindo Renegades Nada Dewa Min Syah | Guindo@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 23 | 26925 | EMP26925 | Haikal | Haikal.bdgs.migdup@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 24 | 27023 | EMP27023 | Heriawan | heriawan.bdgs@binabangsaschool.com | 1 | NULL | Secondary |
| 25 | 28776 | EMP28776 | Huang Jingran | huangjingran.smgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 26 | 28767 | EMP28767 | Hu Jingqiu | hujingqiu.bdgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 27 | 27634 | EMP27634 | Jeffrey Johan | jeffrey.piks@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 28 | 28735 | EMP28735 | Jellie Ann Moraza Tiu | jellieann.smgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 29 | 27635 | EMP27635 | Jia Xiaoyu | jiaxiaoyu.piks@binabangsaschool.com | 2 | NULL | Junior College, Secondary |
| 30 | 26983 | EMP26983 | John Joseph Estillore | john.mlgs@binabangsaschool.com | 2 | NULL | Secondary |
| 31 | 28407 | EMP28407 | Joko Puji Widodo | joko.smgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 32 | 27636 | EMP27636 | Joseph Roy Gordon | josephroy.piks@binabangsaschool.com | 2 | NULL | Secondary |
| 33 | 27495 | EMP27495 | Julie Ann Basconcillo | julieann.pik@binabangsaschool.com | 1 | NULL | Junior College |
| 34 | 28758 | EMP28758 | Laxman Singh Rawat | laxman.piks@binabangsaschool.com | 1 | NULL | Secondary |
| 35 | 28180 | EMP28180 | Li Xin Ran | lixinran.bpns@binabangsaschool.com | 1 | NULL | Secondary |
| 36 | 27637 | EMP27637 | Louis Wirja | louis.piks@binabangsaschool.com | 2 | NULL | Secondary |
| 37 | 28772 | EMP28772 | Mark Dave Gagal Rarugal | markdave.mlgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 38 | 27638 | EMP27638 | Mark Joseph Canumay Diaz | markjoseph.piks@binabangsaschool.com | 1 | NULL | Secondary |
| 39 | 27304 | EMP27304 | Orlene Abrazado Gagui | orlene.mlgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 40 | 28775 | EMP28775 | Qin Beibei | qinbeibei.smgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 41 | 27639 | EMP27639 | Qin Zhiting | qinzhiting.piks@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 42 | 28060 | EMP28060 | Richard Herawaty | richard.migdup@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 43 | 27640 | EMP27640 | Rivoldy Christian Taroreh | rivoldy.piks@binabangsaschool.com | 1 | NULL | Secondary |
| 44 | 27641 | EMP27641 | Simran Kishore Ramchandani | simran.piks@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 45 | 27642 | EMP27642 | Song Sisi | songsisi.piks@binabangsaschool.com | 2 | NULL | Junior College, Secondary |
| 46 | 28745 | EMP28745 | Spencer James McGill | spencerjamesmcgill.kjs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 47 | 28747 | EMP28747 | Tetyana Dyedkova | tetyanadyedkova.kjs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 48 | 22170 | EMP22170 | Unknown Teacher 22170 | migrated.unknown.22170@binabangsaschool.com | 2 | NULL | Junior College, Secondary |
| 49 | 24678 | EMP24678 | Unknown Teacher 24678 | migrated.unknown.24678@binabangsaschool.com | 2 | NULL | Junior College, Secondary |
| 50 | 781 | EMP781 | Unknown Teacher 781 | migrated.unknown.781@binabangsaschool.com | 2 | NULL | Junior College, Secondary |
| 51 | 29171 | EMP29171 | Vania Jolie | vania_jolie.music.piks@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 52 | 27083 | EMP27083 | Vita Fradiantika | vita.bdgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 53 | 28744 | EMP28744 | Wayan Jana Eko Putra | wayanjanaekoputra.kjs@binabangsaschool.com | 1 | NULL | Secondary |
| 54 | 27643 | EMP27643 | Xu Shuo | xushuo.piks@binabangsaschool.com | 2 | NULL | Junior College, Secondary |
| 55 | 27703 | EMP27703 | Yogeeta Rajesh Bhambani | yogeeta.bdgs@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 56 | 27644 | EMP27644 | Yu Li Jin | yulijin.piks@binabangsaschool.com | 1 | NULL | Junior College, Secondary |
| 57 | 28794 | EMP28794 | Zhang Yaning | zhangyaning.piks@binabangsaschool.com | 1 | NULL | Secondary |

### Ringkasan Distribusi

| Kategori | Jumlah |
|----------|--------|
| Total baris | 57 |
| `active_status = 1` (aktif) | 49 |
| `active_status = 2` (non-aktif) | 8 |
| Artifak migrasi (`Unknown Teacher ... migrated.unknown.*`) | 3 |
| Mengajar **Junior College + Secondary** | 38 |
| Mengajar **Secondary saja** | 18 |
| Mengajar **Junior College saja** | 1 (Julie Ann Basconcillo, id 27495) |

---

## Affected Components

| Layer | Item | Impact |
|-------|------|--------|
| DB — employment | `employee.employee_types` (NULL / `{}` / `''`) | Tipe employment kosong untuk 57 employee yang mengajar Sec/JC |
| DB — branch | `employee_branches_branch` (tidak ada baris) | Afiliasi branch kosong → tidak muncul di filter per branch |
| DB — jadwal | `subject_year_teachers_employee` → `subject_year` → `class_year` → `programme` → `master_programme` | Jalur penentuan level mengajar Sec/JC |
| Data quality | Email `migrated.unknown.*` | 3 baris artifak migrasi perlu verifikasi |

---

## Proposed Solution Options

### Option A: Backfill `employee_types` (Recommended — data fix)
Isi `employee.employee_types` untuk 49 employee aktif dengan tipe yang benar (mis. `{2,3}` untuk teaching staff). Butuh konfirmasi mapping tipe employment resmi dari tim HR.

### Option B: Backfill branch
Untuk setiap employee yang belum punya branch, tambahkan baris di `employee_branches_branch` sesuai home branch (dapat diturunkan dari `employee_position.branch_id` bila tersedia di masa lalu).

### Option C: Bersihkan artifak migrasi
Verifikasi & tangani 3 baris `Unknown Teacher` (id 781, 22170, 24678) — kemungkinan data legacy hasil migrasi yang sudah tidak dipakai.

---

## Notes

- Tidak ada kolom literal bernama `employment` di database — dipetakan ke `employee.employee_types` berdasarkan inspeksi schema live.
- Jalur `employee_master_programmes_master_programme` kosong (0 baris) sehingga tidak bisa dipakai; definisi Sec/JC memakai rantai jadwal mengajar.
- Rantai via `subject_year_teachers_teacher` (3.238 baris) menghasilkan **0** baris untuk kriteria ini; rantai via `subject_year_teachers_employee` (19.805 baris) menghasilkan 57. Dipakai yang terakhir karena mewakili penugasan aktual.
- Untuk daftar **aktif saja** (49 baris): tambahkan filter `AND e.active_status = 1`.
- Query read-only; tidak ada perubahan data. Untuk ekspor CSV, gabungkan query di atas dengan `--export` pada CLI atau jalankan via koneksi readonly langsung.
- **Data diambil:** 2026-10-13. Jumlah/isi dapat berubah bila ada update kepegawaian.
