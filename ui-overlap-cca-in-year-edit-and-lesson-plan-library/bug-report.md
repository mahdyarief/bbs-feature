---
title: UI Overlap — dropdown select menimpa elemen di bawahnya (Edit CCA In Year admin & Teacher Lesson Plan Library)
status: open
severity: minor
product: BBS LMS
portal: Admin, Teacher
author: System Analyst
date: 2026-09-21
jam: ~
---

# UI Overlap — Dropdown select menimpa elemen di bawahnya

## Summary

Terdapat dua lokasi dengan gejala UI overlap yang sama: **menu dropdown (pop-up list) dari komponen select terpotong / menimpa (overlap) elemen lain** yang berada di bawahnya, sehingga opsi dropdown tertutup atau bertumpuk dengan konten lain dan menyulitkan interaksi user:

1. **Admin portal — Edit CCA In Year** (`/cca-in-year/:id/edit`): dropdown **Level(s)** dan dropdown **Coordinator** saling menimpa / tertimpa elemen form di sekitarnya saat dibuka.
2. **Teacher portal — Lesson Plan Library** (`/lesson-plan/library`): dropdown **Academic Year** menimpa konten di bawahnya (baris filter lain / tabel), dan **pagination** (`CPagination`) tampil menempel/overlap dengan area tabel ketika dropdown dibuka di dekat daerah tersebut.

**Ekspektasi:** dropdown select harus render di atas semua konten (z-index yang benar / portal rendering) dan tidak boleh tertutup atau menimpa elemen lain secara tidak terbaca. Pagination harus berada pada layout flow normal, tidak overlap dengan tabel atau filter bar.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | System Analyst |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | — |
| Portal URL | https://admin.smartbag.binabangsaschool.com/cca-in-year/:id/edit dan https://teacher.smartbag.binabangsaschool.com/lesson-plan/library |
| Environment API | admin/teacher.smartbag.binabangsaschool.com |
| Data konteks (class ID / daId / tanggal) | — |
| Browser / OS | — |

---

## Steps to Reproduce

### Kasus 1 — Edit CCA In Year (Admin)

1. Login ke Admin portal, buka daftar CCA In Year, pilih salah satu record → `/cca-in-year/:id/edit`.
2. Scroll ke field **Level(s)**, klik untuk membuka dropdown multi-select.
3. Scroll ke section **Coordinators**, klik dropdown coordinator.

**Actual Result:** Menu dropdown tampil tetapi menimpa / tertimpa elemen form lain di sekitarnya (field tetangga, section Coordinator), opsi sulit dibaca atau tidak bisa diklik.

**Expected Result:** Dropdown tampil di atas seluruh konten, semua opsi terlihat dan bisa diklik.

### Kasus 2 — Lesson Plan Library (Teacher)

1. Login ke Teacher portal, buka menu **Lesson Plan Library**.
2. Klik dropdown **Academic Year** pada filter bar.
3. Amati posisi dropdown terhadap baris filter lain (Subject, Term, Week), tabel, dan area pagination di bawah tabel.

**Actual Result:** Dropdown Academic Year menimpa elemen di bawahnya; area pagination juga tampak overlap/menempel dengan tabel sehingga terlihat tidak rapi atau tertutup.

**Expected Result:** Dropdown tampil normal di atas konten; pagination berada pada layout normal di bawah tabel dengan spacing yang benar.

---

## Affected Files / Root Cause Analysis

### Kasus 1 — Edit CCA In Year (Admin)

- `bbs/client/src/views/ccaInYears/CCAInYearFormUpdate.jsx`
  - Dropdown **Level(s)**: `BBSResourceSelect` `name="masterLevelIds"` (line ~194-211), render dalam `CCol md={6}` bersebelahan dengan `CCAsSelector` (CCA) di kolom lain.
  - Dropdown **Coordinator**: `CCACoordinatorsField` (line ~212-219) → `bbs/client/src/views/ccaInYears/components/CCACoordinatorsField.jsx`, masing-masing baris koordinator berisi select teacher.
- Root cause yang diduga: menu dropdown dari select component (berbasis react-select / custom) dirender **inline dalam container form** tanpa portal, sehingga `z-index` menu kalah dengan elemen lain (atau menu terpotong oleh sibling dengan stacking context sendiri). Pada layout 2 kolom (`CCol md={6}`), menu dropdown Level(s) dari kolom kiri dapat tumpang tindih dengan select di kolom kanan.

### Kasus 2 — Lesson Plan Library (Teacher)

- `bbs/client-teacher/src/views/lessonPlan/LessonPlanLibrary.jsx`
  - Dropdown **Academic Year**: `BBSResourceSelect` `name="ay"` (line ~146-160) dalam filter bar `CRow className="am-filter-bar"`.
  - Pagination: `CPagination` (line ~241-245) langsung setelah `CDataTable` tanpa wrapper/spacing.
- Root cause yang diduga: sama seperti kasus 1 — menu dropdown tidak menggunakan portal / z-index menu lebih rendah dari konten di bawahnya; ditambah `CPagination` yang tidak diberi margin (`mt-2` dsb.) sehingga menempel dengan tabel. Perlu dicek juga `FILTER_SELECT_MENU_PROPS` di `bbs/client-teacher/src/views/lessonPlan/lessonPlanUtils.js` — props `menuPortalTarget` / `styles.menu` kemungkinan tidak diset untuk semua select (hanya sebagian).

> Catatan: kedua kasus menggunakan komponen select shared (`BBSResourceSelect` di `client/src/components/BBSResourceSelect.jsx` dan `BBSControlledSelect` dari `bbs-client-common`). Jika akar masalahnya di komponen shared (menu style/z-index), perbaikan sekali di sana akan menyembuhkan kedua halaman.

---

## Impact

- User tidak bisa melihat/memilih opsi dropdown dengan jelas → potensi salah pilih Level/Coordinator/Academic Year.
- Data yang tersimpan bisa salah (coordinator atau level yang tidak diinginkan).
- Pagination yang overlap membuat navigasi halaman sulit diklik.

---

## Saran Solusi

1. **Perbaiki di komponen select shared** (preferred): pastikan menu dropdown dirender dengan `menuPortalTarget={document.body}` (react-select `MenuPortal`) + `zIndex: 1050+` pada `styles.menuPortal`, atau gunakan portal/render di body untuk semua varian select (`BBSResourceSelect`, `BBSControlledSelect`, `CCAsSelector`, select di `CCACoordinatorsField`).
2. Audit `FILTER_SELECT_MENU_PROPS` (lessonPlanUtils) dan pastikan dipakai konsisten di semua select filter Lesson Plan Library, termasuk Academic Year.
3. Beri spacing pada pagination (`<CPagination className="mt-2" ...>` atau wrapper) di `LessonPlanLibrary.jsx`.
4. Regression check halaman lain yang memakai select component yang sama (CCA In Year add/edit, filter halaman lain) agar tidak muncul overlap baru.

---

## Acceptance Criteria

- [ ] Dropdown Level(s) & Coordinator di `/cca-in-year/:id/edit` tampil penuh di atas semua elemen dan semua opsi bisa diklik.
- [ ] Dropdown Academic Year (dan Subject/Term/Week) di Lesson Plan Library tampil di atas tabel/filter lain.
- [ ] Pagination Lesson Plan Library tidak overlap dengan tabel dan mudah diklik.
- [ ] Tidak ada regresi overlap di halaman lain yang memakai komponen select yang sama.
