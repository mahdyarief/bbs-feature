---
title: Gradebook Secondary (Lower/Upper) — kolom CA1 tidak muncul di Term 2
status: open
severity: major
product: BBS LMS
portal: Teacher
author: System Analyst
date: 2026-11-05
jam:
---

# Gradebook Secondary — kolom CA1 tidak muncul di Term 2

## Summary

Di teacher portal, gradebook secondary Term 2 tidak menampilkan kolom **CA1**.

Pembagian tier yang dipakai codebase dan wiki **berbeda**:

| Sumber | Pembagian | Arti |
|--------|-----------|------|
| Wiki `secondary-gradebook.md` | Sec1-3 (master level 7-12) vs Sec4-JC (13-17) | Sec 1, 2, 3 satu tier |
| Frontend `SecondaryGradebookFlexibleTable.jsx` | `isSec1To2` (Sec 1-2) vs sisanya (Sec 3-4) | Sec 3 dipindah ke tier atas |

Akibatnya ada dua perilaku berbeda di Term 2:

- **Sec 1-2**: cabang `isSec1To2` mengembalikan `CAT2` saja (plus `MYE`/`SA1` bila `hasMye`). **CA1 tidak pernah dibuat maupun ditampilkan.**
- **Sec 3-4**: `getCalculatedColumns` mengembalikan `["CA1", "SA1"]`, jadi CA1 tampil.

Karena wiki mendefinisikan tier sebagai **Sec1-3**, Sec 3 seharusnya ikut aturan lower secondary. Yang benar-benar hilang CA1-nya adalah **Sec 1 dan Sec 2**.

**Ekspektasi:** seluruh level secondary (Sec 1-4) menampilkan CA1 di Term 2, dengan `CA1 = weighted T1 + T2`.

---

## Penting: CA1 sudah dihitung dan disimpan backend

Ini membedakan tiket ini dari dugaan awal. Backend **sudah** menghitung CA1 untuk **semua** master level secondary 7-17 (termasuk JC):

`api_nest/src/modules/student-secondary-grade/student-secondary-grade.service.ts`

```ts
// line 299-300
const ca1 = this.rollupTermAverage(grades, [1, 2], weight);
const ca2 = this.rollupTermAverage(grades, [3, 4], weight);

// line 444-446 — disimpan sebagai Grade row (CA1, CA2, SA1, SA2)
await this.upsertCalculatedGrade(sg.id, GradeExamTypeEnum.CA, 2, 1, ca1);
await this.upsertCalculatedGrade(sg.id, GradeExamTypeEnum.CA, 4, 2, ca2);
await this.upsertCalculatedGrade(sg.id, GradeExamTypeEnum.SA, 2, 1, sa1);
```

`rollupTermAverage(grades, [1, 2], weight)` (line 502-571) menjumlahkan seluruh komponen term 1 **dan** term 2 sesuai `term1Config` + `term2Config` dari `SecondaryGradeWeight`, lalu membagi total persentase yang terisi. Rumus ini konsisten dengan wiki (`CA1 = weighted T1 + T2`), dan **tidak membedakan lower vs upper secondary** — weight yang membedakan lewat `findEffectiveWeight(masterLevelId, subjectId, targetYear)`.

Grade row CA1 dengan `term = 2, order = 1` juga sudah ditulis, sehingga CA1 tersedia di response `GET /v1/studentSecondaryGrades`.

**Kesimpulan:** CA1 bukan masalah data atau perhitungan. Masalahnya murni **kolom tidak dirender** di frontend untuk Sec 1-2.

---

## Steps to Reproduce

1. Login teacher portal, buka **Secondary Gradebook**.
2. Pilih subject untuk kelas **Sec 1** atau **Sec 2**.
3. Pilih **Term 2**.
4. Amati header kolom gradebook.

**Actual Result:** Kolom calculated Term 2 hanya `CAT2` (plus `MYE`/`SA1` bila `gradeWeight.mye > 0`). Tidak ada `CA1`.

**Actual Result setelah fix CA1 di `develop` (commit `131946432`):** Kolom `CAT2` hilang, tergantikan oleh `CA1`. Murid dengan `gradeWeight.mye > 0` juga kehilangan `CAT2` meski `SA1` masih tampil.

**Expected Result:** `CAT2` **tetap tampil** dan `CA1` **ditambahkan** di ujung kanan.

| Kondisi | Kolom calculated Term 2 yang harus tampil (urutan kiri → kanan) |
|---------|------------------------------------------------------------------|
| `gradeWeight.mye > 0` | `CAT 2` → `MYE` → `SA 1` → `CA 1` |
| `gradeWeight.mye = 0` | `CAT 2` → `CA 1` |

Catatan: `CA1` harus tetap tampil walau `hasMye` false, karena perhitungannya hanya memakai `cat1` + komponen Term 2.

---

## Root Cause

`bbs/client-teacher/src/components/SecondaryGradebookFlexibleTable.jsx`

```js
// line 171-177 — tingkat granularitas level
const SEC_1_2_LEVELS = [
  "SECONDARY1",
  "SECONDARY1_ACC",
  "SECONDARY2",
  "SECONDARY2_ACC"
];
const isSec1To2 = SEC_1_2_LEVELS.includes(classYear?.level);

// line 233-242 — kolom terhitung
function getCalculatedColumns(t) {
  if (isSec1To2) {
    if (t === 2 && hasMye) return [`CAT${t}`, "SA1"];
    return t <= 3 ? [`CAT${t}`] : ["CA2", "FYA"];   // <-- CA1 tidak ada
  }
  if (t === 1) return ["CAT1"];
  if (t === 2) return ["CA1", "SA1"];               // Sec 3-4 benar
  if (t === 3) return ["CAT3"];
  return ["CA2", "FYA"];
}

// line 276-286 — kolom yang dirender (urutan display)
function getDisplayColumns(t) {
  if (isSec1To2) {
    if (t === 2 && hasMye) return [`CAT${t}`, "MYE", "SA1"];
    if (t <= 3) return [`CAT${t}`];                 // <-- CA1 tidak ada
    return ["CA2", ...(hasFye ? ["FYE"] : []), "FYA"];
  }
  if (t === 1) return ["CAT1"];
  if (t === 2) return ["CA1", ...(hasMye ? ["MYE"] : []), "SA1"];
  if (t === 3) return ["CAT3"];
  return ["CA2", ...(hasFye ? ["FYE"] : []), "FYA"];
}
```

Dua fungsi ini harus diubah bersamaan. `getCalculatedColumns` menentukan kolom read-only, `getDisplayColumns` menentukan urutan render (line 288-293).

Nilai CA1 sendiri sudah tersedia di frontend. `calculatedColumnValue` (line 709-720) memanggil `computeSemesterAverage(studentId, 1)` untuk kolom `CA1`:

```js
// line 709-716
function calculatedColumnValue(studentId, col) {
  ...
  if (col === "CA1") return computeSemesterAverage(studentId, 1);
  if (col === "CA2") return computeSemesterAverage(studentId, 3);
```

`computeSemesterAverage` (line 536-577) mengambil `ssg.cat{prevTerm}` — nilai rollup CAT yang sudah dipersist backend — lalu menambah komponen term berjalan sesuai `fieldPercentages` dan `dcaFields`, dan membagi total persentase yang terisi. Jadi perhitungan lower dan upper secondary memakai rumus yang sama; yang berbeda hanya komposisi weight per tier.

Perlu dicatat: backend `rollupTermAverage` (line 511-566) **menghitung DCA juga**, padahal wiki (`secondary-gradebook.md:79`) menyatakan *"DCA exam type is excluded from CA1 and CA2 until the DCA module is live"*. Ini inkonsistensi yang perlu dikonfirmasi ke tim akademik, tapi terpisah dari bug kolom CA1 tidak muncul. Jangan dicampur dalam fix tiket ini.

**Kesimpulan:** CA1 sudah dihitung, sudah disimpan, dan sudah punya fungsi pembaca di frontend. Masalahnya murni **kolom tidak dimasukkan ke daftar** untuk Sec 1-2.

---

## Catatan soal perhitungan CA1 (lower vs upper secondary)

Perhitungan tidak berbeda antara lower dan upper secondary. Yang berbeda adalah **komposisi weight per tier**:

| Tier | Master level | Komponen Term 2 (default) |
|------|--------------|---------------------------|
| Sec1-3 | 7-12 | TT x2 (18.75% each) + DCA (12.5%) |
| Sec4-JC | 13-17 | TT (25%) + DCA (12.5%) |

Sumber: `bbs/client/src/views/secondaryGradeWeights/SecondaryGradeWeightForm.jsx:24-69` (`SEC1_3_DEFAULTS`, `SEC4_JC_DEFAULTS`).

`rollupTermAverage` membaca `term1Config` dan `term2Config` dari weight yang aktif, jadi hasil CA1 otomatis mengikuti tier yang benar — **asalkan weight untuk master level tersebut sudah dikonfigurasi**. Kalau `findEffectiveWeight` mengembalikan `null` (belum ada konfigurasi weight untuk level itu), `rollupTermAverage` langsung `return null` (line 507) dan CA1 kosong di semua level.

**Perlu dicek saat QA:** apakah CA1 kosong di Sec 1-2 karena kolom tidak dirender, atau juga karena weight belum dikonfigurasi untuk level tersebut.

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend | `bbs/client-teacher/src/components/SecondaryGradebookFlexibleTable.jsx:171-177` | `isSec1To2` membagi tier berbeda dari wiki |
| Frontend | `bbs/client-teacher/src/components/SecondaryGradebookFlexibleTable.jsx:233-242` | `getCalculatedColumns` — CA1 tidak dibuat untuk Sec 1-2. Fix di `develop` menimpa `CAT2`; harus ditambahkan, bukan replace |
| Frontend | `bbs/client-teacher/src/components/SecondaryGradebookFlexibleTable.jsx:276-286` | `getDisplayColumns` — CA1 tidak dirender untuk Sec 1-2. Fix di `develop` menghilangkan kolom CAT2 |
| Backend helper | `api_nest/src/helpers/default-gradebook-table.helpers.ts:1019-1034` | Referensi — Term 2 Sec 1-3 punya 2 komponen Test + DCA (total 50%) |
| Backend | `api_nest/src/modules/student-secondary-grade/student-secondary-grade.service.ts:299,444` | CA1 sudah dihitung & disimpan — tidak perlu diubah |
| Backend | `api_nest/src/modules/student-secondary-grade/student-secondary-grade.service.ts:502-571` | `rollupTermAverage` — logika benar, tier-agnostic |
| Data | `secondary_grade_weight` | Weight per master level harus ada, jika tidak CA1 = null |

---

## Koreksi pada fix yang sudah dirilis (WAJIB DIBACA)

Fix "CA1 incorrectly not appearing in term 2" (PR #2584, commit `131946432`) **salah arah**: kode yang masuk ke `develop` **menimpa** `CAT2` dengan `CA1`, bukan menambahkan `CA1`.

```jsx
// client-teacher/src/components/SecondaryGradebookFlexibleTable.jsx:234-237 — versi di develop SEBELUM fix
if (isSec1To2) {
  if (t === 2 && hasMye) return [`CAT${t}`, "SA1"];        // ada MYE -> CAT2 + SA1
  return t <= 3 ? [`CAT${t}`] : ["CA2", "FYA"];           // tanpa MYE -> CAT2
}

// versi di develop SESUDAH fix (commit 131946432) — CAT2 hilang
if (isSec1To2) {
  if (t === 2) return ["CA1", ...(hasMye ? ["SA1"] : [])];
  return t <= 3 ? [`CAT${t}`] : ["CA2", "FYA"];
}
```

**Koreksi yang benar: `CA1` DITAMBAHKAN, `CAT2` TETAP TAMPIL.**

- `CA1` adalah hasil akhir (`T1 + T2`), sehingga posisinya di **ujung kanan**, sejajar dengan `SA1` — bukan di kiri menggantikan `CAT2`.
- `CAT2` adalah rollup komponen Term 2 (`Test 2` 18.75% + `Test 3` 18.75% + `DCA T2` 12.5% = 50%) dan harus tetap ada agar guru bisa memantau nilai T2 sebagai unit mandiri.
- **Logika MYE tidak diubah oleh fix ini dan tidak perlu diubah.** `hasMye` (`termNum === 2 && gradeWeight?.mye > 0`, line 179) tetap memakai definisi yang sama. Bug ini murni soal `CAT2` yang tertimpa `CA1`; perilaku MYE/`SA1` tetap seperti sebelumnya.
- Munculnya `CAT2` **tidak bergantung pada `hasMye`**. Di kode lama, `hasMye` hanya menentukan apakah `MYE` dan `SA1` ikut dirender; `CAT2` selalu muncul lewat fallback `return t <= 3 ? [`CAT${t}`] : ...`.

### Perilaku yang benar per kondisi

| `hasMye` | Kolom calculated Term 2 (urutan tampil) |
|---|---|
| `true` | `CAT 2` → `MYE` → `SA 1` → `CA 1` |
| `false` | `CAT 2` → `CA 1` |

### Patch yang diusulkan

```jsx
// getCalculatedColumns (line 233-242)
function getCalculatedColumns(t) {
  if (isSec1To2) {
    if (t === 2) return [`CAT${t}`, "CA1", ...(hasMye ? ["SA1"] : [])];
    return t <= 3 ? [`CAT${t}`] : ["CA2", "FYA"];
  }
  if (t === 1) return ["CAT1"];
  if (t === 2) return ["CA1", "SA1"];
  if (t === 3) return ["CAT3"];
  return ["CA2", "FYA"];
}

// getDisplayColumns (line 276-286) — CA1 di ujung kanan
function getDisplayColumns(t) {
  if (isSec1To2) {
    if (t === 2)
      return [
        `CAT${t}`,
        ...(hasMye ? ["MYE"] : []),
        ...(hasMye ? ["SA1"] : []),
        "CA1"
      ];
    if (t <= 3) return [`CAT${t}`];
    return ["CA2", ...(hasFye ? ["FYE"] : []), "FYA"];
  }
  if (t === 1) return ["CAT1"];
  if (t === 2) return ["CA1", ...(hasMye ? ["MYE"] : []), "SA1"];
  if (t === 3) return ["CAT3"];
  return ["CA2", ...(hasFye ? ["FYE"] : []), "FYA"];
}
```

### Masih perlu keputusan Product

1. **Apakah `isSec1To2` perlu diganti jadi `isSec1To3`** agar sesuai wiki (Sec 1-3 satu tier)? Ini mempengaruhi Sec 3 juga, jadi harus diputuskan bersamaan dengan patch ini.
2. **`SA1`/`MYE` — di luar scope.** Keduanya **tidak diubah** oleh fix ini; patch di atas memakai kondisi `hasMye` yang persis sama seperti kode lama. Yang perlu dipastikan hanya bahwa `CA1` tetap tampil saat `hasMye` false, karena `computeSemesterAverage(studentId, 1)` hanya membaca `cat1` + komponen Term 2 dan tidak butuh MYE. Apakah `SA1` perlu selalu tampil adalah pertanyaan terpisah, bukan bagian dari bug CA1.

---

## Notes

- Tidak ada perubahan kode. Seluruh isi adalah hasil pembacaan codebase, wiki, dan admin form.
- **Regresi di `develop`:** fix CA1 (PR #2584, commit `131946432`) sudah dirilis dan membuat kolom `CAT2` hilang di Term 2 untuk Sec 1-2. Patch pada bagian "Koreksi pada fix yang sudah dirilis" harus menggantikan fix tersebut, bukan ditambahkan di atasnya.
- `CAT2` tidak dipersist sebagai field sendiri; kolom calculated dikirim saat submit sebagai `examType = CAT`, `order = 2`. Jadi tidak ada data yang hilang dari database.
- Issue referensi: https://github.com/bbs-lms/bbs-teacher/issues/288 (private, tidak dapat diakses publik).
- Tiket JC dipisah ke `features/gradebook-ca1-missing-term2-jc/`.
- Feed `operate-smartbag` TC-014 hanya memverifikasi pemetaan level dan tabel weight; belum ada test case untuk kolom CA1.