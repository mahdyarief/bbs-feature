---
title: Gradebook Junior College — kolom CA1 tidak muncul di Term 2
status: open
severity: major
product: BBS LMS
portal: Teacher
author: System Analyst
date: 2026-11-05
jam:
---

# Gradebook Junior College — kolom CA1 tidak muncul di Term 2

## Summary

Di teacher portal, gradebook **Junior College** Term 2 tidak menampilkan kolom **CA1**. Hasil pengecekan acak oleh reporter: satu kelas JC di Term 2 tidak menampilkan CA1.

JC memakai komponen gradebook yang **berbeda** dari Secondary, dan akar masalahnya juga berbeda: **daftar kolom (gradebook table) untuk JC Term 2 tidak memuat `CA1`**, sehingga nilai CA1 yang sudah dihitung backend tidak dirender.

Untuk department **Chinese** CA1 sudah tampil, karena cabang Chinese JC di `defaultGradebookTables.js` memuat `EXAM_TYPES.CA`. Department lain (Science, Mathematics, Business, English Language, Aesthetics, Music) jatuh ke cabang umum yang `extraFields`-nya kosong.

**Ekspektasi:** Gradebook Term 2 untuk semua level JC menampilkan kolom CA1.

---

## Penting: CA1 sudah dihitung dan disimpan backend

Sama seperti kasus Secondary, backend **sudah** menghitung CA1 untuk master level secondary **7-17** (termasuk JC 13-17):

`api_nest/src/modules/student-secondary-grade/student-secondary-grade.service.ts`

```ts
// line 31
const SECONDARY_MASTER_LEVEL_IDS = [7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17];

// line 254 — gate recalculate
if (!masterLevelId || !SECONDARY_MASTER_LEVEL_IDS.includes(masterLevelId)) return sg;

// line 299
const ca1 = this.rollupTermAverage(grades, [1, 2], weight);

// line 444 — disimpan sebagai Grade row term 2 order 1
await this.upsertCalculatedGrade(sg.id, GradeExamTypeEnum.CA, 2, 1, ca1);
```

`rollupTermAverage(grades, [1, 2], weight)` (line 502-571) menjumlahkan komponen term 1 dan term 2 sesuai `term1Config` + `term2Config` dari `SecondaryGradeWeight`, lalu membagi total persentase yang terisi. Formula ini **tidak membedakan secondary vs JC** — yang membedakan adalah weight yang dipilih lewat `findEffectiveWeight(masterLevelId, subjectId, targetYear)`.

Jadi Grade row `examType = CA`, `term = 2`, `order = 1` seharusnya sudah ada di response `GET /v1/grades` maupun `studentSecondaryGrades`. **CA1 tidak tampil semata-mata karena gradebook table JC tidak memintanya.**

---

## Steps to Reproduce

1. Login teacher portal, buka **Gradebook** (bukan Secondary Gradebook).
2. Pilih subject kelas **JC 1 / JC 2 / JC Bridging**, department non-Chinese.
3. Pilih **Term 2**.
4. Amati header kolom tabel gradebook.

**Actual Result:** Hanya kolom `TT1`, `TT2` (input) dan kolom average. Tidak ada `CA1`.

**Expected Result:** Kolom `CA1` tampil dengan nilai dari backend.

---

## Root Cause

### Bagian 1 — Table JC Term 2 tidak memuat CA1

`bbs/client-teacher/src/views/gradebook/defaultGradebookTables.js`

Struktur fungsi (line 12-19, 21, 301):

```js
const isPrimary = PRIMARY_COHORT_LEVELS.includes(cohortLevel);
const isSecondary = SECONDARY_COHORT_LEVELS.includes(cohortLevel);
const isJuniorCollege = JUNIOR_COLLEGE_LEVELS.includes(cohortLevel);

switch (department) {
  case DEPARTMENTS.CHINESE: ...
  case DEPARTMENTS.ENGLISH_LANGUAGE: ...
  default: break;          // line 297-298 — department lain langsung break
}

switch (academicType) {
  case "ACADEMIC": ...     // line 303
  default: ...             // line 483 — fajar ke useMarkingType, tanpa CA1
```

Setelah kedua switch, department selain Chinese dan English Language jatuh ke cabang `ACADEMIC` → `else if (isSecondary || isJuniorCollege)`:

```js
// line 412-431 — JC & Secondary berbagi cabang yang sama
} else if (isSecondary || isJuniorCollege) {
  switch (term) {
    case 2:
      return {
        fields: [{ type: "TT", amount: 2, startFrom: 3, percentage: 100 }],
        fieldPercentages: [...],
        extraFields: [
          // { type: EXAM_TYPES.MYE, order: 1 },
          // { type: EXAM_TYPES.CA, order: 1 },     <-- DIKOMENTARI
          // { type: EXAM_TYPES.SA, order: 1 },     <-- DIKOMENTARI
        ],
        showAverage: true
      };
```

`extraFields` kosong = CA1, MYE, dan SA1 tidak diminta dari data.

### Bagian 1b — Khusus English Language, JC tidak punya cabang sama sekali

Cabang `DEPARTMENTS.ENGLISH_LANGUAGE` (line 124-296) hanya menangani:

- `if (isPrimary)` → term 1-4
- `else if (isSecondary)` → term 1-4 (term 2 CA1 juga dikomentari, line 245-246)

Tidak ada `else if (isJuniorCollege)`. Jadi **English Language JC** keluar dari `case` tersebut tanpa `return`, lalu jatuh ke `switch (academicType)`. Berarti English Language JC mengikuti cabang umum di Bagian 1, bukan cabang English Language. Perlu dicek saat QA apakah ini memang disengaja.

### Bagian 1c — Kondisi `academicType` harus ACADEMIC

`getDefaultGradebookTable(term, level, subject.category, subject.department)`. Bila `subject.category` bukan `"ACADEMIC"`, fungsi jatuh ke `default:` (line 483) yang mengembalikan `useMarkingType: true`, satu field `TT`, dan **tanpa `extraFields`** — CA1 juga tidak muncul. Perlu dicek nilai `category` pada subject JC yang dilaporkan.

### Bagian 2 — Chinese JC (reference implementasi yang benar)

```js
// line 22-42
case DEPARTMENTS.CHINESE:
  if (isJuniorCollege) {
    switch (term) {
      case 2:
        return {
          fields: [
            { type: "PU", amount: 1, startFrom: 2 },
            { type: "IC", amount: 1, startFrom: 1 }
          ],
          extraFields: [
            { type: EXAM_TYPES.CA, order: 1 },     // CA1
            { type: EXAM_TYPES.MYE, order: 1 },
            { type: EXAM_TYPES.SA, order: 1 }
          ],
          showAverage: true
        };
```

Ini bukti struktur CA1 untuk JC Term 2 sudah pernah dibuat, hanya belum diterapkan ke department lain.

### Bagian 3 — Grade row diabaikan bila tidak ada di allFields

`bbs/client-teacher/src/components/GradebookFlexibleTable.jsx`

```js
// line 379
const allFields = [...allFieldOrders, ...extraFields];

// line 397-411 — loop grade rows siswa
for (const gradeOfStudent of gradesOfStudent) {
  const examOrder = gradeOfStudent?.examType + (gradeOfStudent?.order || "");
  if (allFields.includes(examOrder)) {         // <-- filter
    ...
  }
}
```

Karena `extraFields` kosong, `allFields` tidak memuat `"CA1"`. Grade row CA1 yang dikirim backend **dibuang di sini**, sehingga nilainya tidak pernah masuk ke form.

### Bagian 4 — Sumber table & fallback

```js
// GradebookFlexibleTable.jsx:122-147
const currentGradebookTablesApi = useFromApi(
  fromApi.getGradebookTables({ getLatest: true, subjectYearId }), ...
);
const currentGradebookTable = currentGradebookTables.find((gdt) => gdt.term === parseInt(term));

const gradebookTable =
  currentGradebookTable ||
  (subject && classYear
    ? getDefaultGradebookTable(parseInt(term), classYear.level, subject.category, subject.department)
    : {});
```

Ada dua lapis yang harus diperhatikan:

1. Bila sudah ada **gradebook table tersimpan** untuk subject+term tersebut di DB, default di atas tidak dipakai sama sekali. Artinya perubahan default saja tidak akan berpengaruh pada subject yang tabelnya sudah ter-generate.
2. `getDefaultGradebookTable` menerima `academicType = subject.category`. Bila `subject.category` bukan `"ACADEMIC"`, fungsi jatuh ke `default:` di line 483 yang mengembalikan tabel `useMarkingType` dan `showAverage: false` — di kondisi ini CA1 juga tidak muncul. Perlu dicek nilai `category` pada subject JC yang dilaporkan.

### Referensi definisi

`bbs-wiki/secondary-gradebook.md:22`

```
CA1 = weighted T1 + T2
CA2 = weighted T3 + T4
FYA = CA1 * ca1% + MYE * mye% + CA2 * ca2% + FYE * fye%
```

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend | `bbs/client-teacher/src/views/gradebook/defaultGradebookTables.js:412-431` | Cabang `isSecondary \|\| isJuniorCollege` Term 2 — `extraFields` kosong |
| Frontend | `bbs/client-teacher/src/views/gradebook/defaultGradebookTables.js:22-42` | Chinese JC — pola yang benar (CA, MYE, SA) |
| Frontend | `bbs/client-teacher/src/views/gradebook/defaultGradebookTables.js:124-296` | Cabang English Language — tidak ada `isJuniorCollege`, jadi English JC jatuh ke cabang umum |
| Frontend | `bbs/client-teacher/src/views/gradebook/defaultGradebookTables.js:301-303,483` | Cabang `academicType` — perlu dicek untuk JC non-ACADEMIC |
| Frontend | `bbs/client-teacher/src/components/GradebookFlexibleTable.jsx:379,397-411` | Filter `allFields` — CA1 dibuang bila tidak diminta |
| Frontend | `bbs/client-teacher/src/components/GradebookFlexibleTable.jsx:122-147` | Tabel tersimpan menimpa default |
| Backend | `api_nest/src/modules/student-secondary-grade/student-secondary-grade.service.ts:31,254,299,444` | CA1 sudah dihitung & disimpan — tidak perlu diubah |
| Data | `secondary_grade_weight`, `gradebook_table` | Weight untuk level JC harus ada; tabel tersimpan harus memuat CA1 |

---

## Proposed Solution

Tambahkan `EXAM_TYPES.CA` (order 1) ke `extraFields` pada cabang `isSecondary || isJuniorCollege` Term 2, mengikuti pola Chinese JC. Perlu keputusan Product:

1. **Apakah `MYE` dan `SA1` juga perlu ditampilkan** di JC Term 2, atau cukup CA1?
2. **Cakupan**: cabang ini dipakai bersama `isSecondary` dan `isJuniorCollege`. Apakah perubahan harus dibatasi hanya untuk JC (`if (isJuniorCollege)` terpisah), atau boleh berlaku untuk keduanya?
3. **Data tersimpan**: bila `gradebook_table` untuk JC Term 2 sudah ada di DB, perlu regenerate/update agar memuat CA1. Ini perlu dikoordinasikan agar tidak menimpa konfigurasi yang sudah disesuaikan sekolah.
4. **Weight**: pastikan `SecondaryGradeWeight` untuk master level JC (13-17) sudah dikonfigurasi. Tanpa weight, `rollupTermAverage` mengembalikan `null` (line 507) sehingga CA1 tetap kosong walau kolomnya sudah muncul.

---

## Notes

- Tidak ada perubahan kode. Seluruh isi adalah hasil pembacaan codebase dan wiki.
- Issue referensi: https://github.com/bbs-lms/bbs-teacher/issues/288 (private, tidak dapat diakses publik).
- Tiket Secondary dipisah ke `features/gradebook-ca1-missing-term2-secondary/`.
- Kalau ternyata CA1 juga hilang di Chinese JC, akar masalahnya bukan cabang department dan brief ini perlu diperluas.
- `operate-smartbag` belum punya test case yang memverifikasi kolom CA1 di gradebook.