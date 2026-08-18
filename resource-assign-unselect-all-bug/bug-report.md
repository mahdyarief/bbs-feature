---
title: Resource Assign — Unselect all students per class tidak berfungsi (harus klik 1 per 1)
status: open
severity: minor
product: BBS LMS
portal: Teacher
author: System Analyst
date: 2026-08-18
jam: https://jam.dev/c/9aed76ec-56b1-4abe-931b-25c922f36d17
---

# Resource Assign — Unselect all students per class tidak berfungsi (harus klik 1 per 1)

## Summary

Di halaman Teacher portal `/resources/assign?chapterId=1681` (Assign Resources to Students), ketika guru ingin **uncheck checkbox class** untuk deselect semua student dalam class tersebut, checkbox tidak update dengan benar. Guru dipaksa untuk klik setiap student checkbox satu per satu, bukan melalui class-level "unselect all". Ini terjadi karena ada race condition antara manual state update dan automatic `useEffect` sync pada `selectedClassIds`.

**Ekspektasi:** Ketika guru uncheck checkbox class, semua student dalam class tersebut langsung ter-deselect dan checkbox class menjadi unchecked — tanpa perlu klik 1 per 1.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Mahdy Arief |
| Email (Jam account) | mahdy.arief.torche.indonesia@gmail.com |
| Jam author ID | 7ebe5b87-970c-4f3d-95f5-c4f39b901a03 |
| User ID (dari console log, mis. `selfUser`) | — (tidak ada di Jam) |
| Portal URL | https://teacher.smartbag.binabangsaschool.com/resources/assign?chapterId=1681 |
| Environment API | teacher.smartbag.binabangsaschool.com |
| Data konteks (class ID / daId / tanggal) | chapterId=1681 |
| Browser / OS | Brave 150.0.0.0 / Windows 11 |

---

## Steps to Reproduce

1. Login ke Teacher portal, buka halaman `/resources/assign?chapterId=1681`.
2. Expand salah satu class (klik nama class untuk melihat daftar student).
3. **Check** checkbox class → semua student dalam class tersebut terpilih (checkbox class checked, semua student checkbox checked).
4. **Uncheck** checkbox class yang sama.

**Actual Result:** Checkbox class tidak update secara visual — tetap terlihat checked atau flicker. Student checkbox juga tidak ter-deselect semua. Guru harus klik setiap student checkbox 1 per 1 untuk deselect.

**Expected Result:** Semua student dalam class langsung ter-deselect dan checkbox class menjadi unchecked secara instan.

---

## Root Cause Analysis

### Bug #1 — Race condition antara manual `setSelectedClassIds` dan `useEffect` sync — `AssignTables.jsx` (line 225-235)

`handleToggleCheckClass` melakukan **dua hal sekaligus**: (1) update `selectedStudentIds` via `handleToggleCheckStudents`, dan (2) manual update `selectedClassIds` via `setSelectedClassIds`. Padahal ada `useEffect` di line 194-203 yang **otomatis** recalculate `selectedClassIds` setiap kali `selectedStudentIds` berubah. Ini menyebabkan manual update dan automatic update saling override (race condition).

```javascript
// path: AssignTables.jsx:225-235 — handleToggleCheckClass
function handleToggleCheckClass(classRow, checked) {
  const studentIds = classRow.studentsIds;
  handleToggleCheckStudents(studentIds, checked); // (1) update selectedStudentIds
  if (!checked) {
    setSelectedClassIds((prevClassIds) =>       // (2) manual update — REDUNDANT
      prevClassIds.filter((l) => l !== classRow.id)
    );
  } else {
    setSelectedClassIds((prevClassIds) => [...prevClassIds, classRow.id]); // (2) manual update — REDUNDANT
  }
}
```

**useEffect yang sudah handle sync otomatis:**

```javascript
// path: AssignTables.jsx:194-203 — automatic sync (sudah benar)
useEffect(() => {
  const classYearsIdOfEveryStudent = classYears
    .filter((cy) =>
      cy.studentsIds?.every((sId) => selectedStudentIds.includes(sId))
    )
    .map((cy) => cy.classroomId);

  setSelectedClassIds(classYearsIdOfEveryStudent);
  if (disableForAssignedStudents) setDisabledStudentIds(defaultStudentIds);
}, [selectedStudentIds]);
```

**Faktor pendukung:** `useEffect` di line 194-203 sudah menjadi source of truth untuk `selectedClassIds` — manual update di `handleToggleCheckClass` tidak diperlukan dan justru merusak.

---

## Bukti dari Jam (https://jam.dev/c/9aed76ec-56b1-4abe-931b-25c922f36d17)

| Sumber | Temuan |
|--------|--------|
| **Video (28s)** | User di halaman `/resources/assign?chapterId=1681`, terlihat klik checkbox student 1 per 1 berulang kali — tidak ada class-level toggle yang berhasil. |
| **Network** | — (tidak ada error network, ini murni frontend state issue) |
| **Console** | — (tidak ada error di console) |
| **User events** | 45 events total. Events 6-9: user klik checkbox `class-1021` berulang kali (4x click dalam ~700ms — tanda checkbox tidak responsif). Events 10-48: user klik individual student checkbox 1 per 1, termasuk navigasi antar page (event 25: klik page "2"). |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend | `bbs/client-teacher/src/views/lessonBuilder/assigns/AssignTables.jsx` (line 225-235) | Checkbox class tidak berfungsi untuk unselect — user harus klik student 1 per 1 |

---

## Proposed Solution Options

### Option A: Hapus manual `setSelectedClassIds` dari `handleToggleCheckClass` (Recommended)

Hapus baris 228-234 (manual `setSelectedClassIds` calls). Biarkan `useEffect` di line 194-203 yang sudah ada handle sync `selectedClassIds` secara otomatis.

```javascript
// FIXED — AssignTables.jsx:225-228
function handleToggleCheckClass(classRow, checked) {
  const studentIds = classRow.studentsIds;
  handleToggleCheckStudents(studentIds, checked);
  // useEffect di line 194-203 sudah handle selectedClassIds sync otomatis
}
```

**Kelebihan:** Minimal change (hapus 7 baris), tidak ada risiko regress di logic lain, `useEffect` sudah terbukti benar.

### Option B: Hapus `useEffect` sync, pertahankan manual update

Hapus `useEffect` di line 194-203 dan biarkan `handleToggleCheckClass` handle `selectedClassIds` secara manual.

**Kekurangan:** Lebih berisiko karena `useEffect` juga handle case lain (mis. ketika student di-toggle individual, class checkbox harus update juga). Option A lebih aman.

---

## Notes

- Fix ini **frontend-only**, tidak ada perubahan backend/API.
- Indeterminate state (checkbox dash icon ketika sebagian student selected) harus tetap berfungsi setelah fix.
- Cross-page selection (pilih student di page 1, navigasi ke page 2, pilih lagi) harus tetap preserved.
- Jam recording tidak memiliki deskripsi atau transcript (microphone tidak aktif saat recording).
