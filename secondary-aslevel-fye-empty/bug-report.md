---
title: Secondary Gradebook — Nilai FYE kosong pada AS Level (JC) dan A Level (JC) meski PUM Prelim sudah terisi
status: open
severity: major
product: BBS LMS
portal: Teacher Portal
author: System Analyst
date: 2026-10-16
jam:
---

# Secondary Gradebook — FYE Kosong pada AS Level (JC) dan A Level (JC) Meski PUM Prelim Sudah Ada

## Summary

Di Teacher Portal → Gradebook (Secondary/JC), kolom **FYE** pada subjek Cambridge **AS Level** (`cambridge_programme = '2'`) DAN **A Level** (`cambridge_programme = '3'`) selalu kosong, padahal nilai **PUM** (Prelim Ujian Matrikulasi) untuk subjek tersebut sudah terisi dari hasil Prelim.

**Dua masalah yang teridentifikasi:**

- **Bug #1 — AS Level:** `pumFieldsForProgramme()` case `ASLEVEL` hanya mengembalikan `mye`, tanpa `fye`. Akibatnya `student_grade.fye` tetap `NULL` untuk baris AS Level, dan frontend `resolveFyePum()` tidak membaca `asPredictedPum` untuk FYE.
- **Bug #2 — A Level:** Backend `pumFieldsForProgramme()` case `ALEVEL` memetakan FYE dari `sg.predictedPum` saja, sedangkan frontend `resolveFyePum()` juga membaca `ssg.pum` / `ssg.predictedPum` — **tidak membaca `ssg.asPredictedPum`**. Karena prelim A Level disimpan ke `as_predicted_pum` (melalui `ThresholdTypeEnum.AS_PRELIM`), kolom `predicted_pum` dan `pum` tetap NULL, sehingga `resolveFyePum()` mengembalikan null dan FYE kosong di UI.

Masalah ini terjadi di level **Secondary 3 s/d JC 2** (master_level_id 11–17).

**Ekspektasi:** Nilai FYE pada subjek AS Level dan A Level harus terisi dari PUM Prelim, konsisten dengan perilaku IGCSE (`fye = predicted_pum`), atau kolom FYE tidak ditampilkan sama sekali jika memang programme tersebut tidak memiliki FYE.

---

## Test Identity / Akun Akses

Bug ini dianalisis lewat code reading + query database produksi (bukan via Jam recording), jadi sebagian field tidak tersedia.

| Field | Nilai |
|-------|-------|
| Reporter / Tester | System Analyst (analisis kode + DB) |
| Email (Jam account) | — (tanpa Jam recording) |
| Jam author ID | — |
| User ID | — (repro membutuhkan akun Teacher) |
| Portal URL | `https://teacher.smartbag.binabangsaschool.com` |
| Environment API | produksi (query read-only via `db-access-tools`) |
| Data konteks | master_level_id 11–17 (Sec3 Accelerated s/d JC2); subjek `cambridge_programme IN ('1','2','3')` |
| Browser / OS | — |

---

## Steps to Reproduce

1. Login sebagai **Teacher** ke Teacher Portal, buka **Gradebook** untuk kelas JC1/JC2 (master_level_id 16/17)
2. Pilih subjek Cambridge **AS Level** (mis. General Paper (AS), `cambridge_programme = '2'`) — pastikan `as_predicted_pum` pada `student_grade` sudah terisi (hasil Prelim)
3. **atau** pilih subjek Cambridge **A Level** (`cambridge_programme = '3'`) — pastikan `as_predicted_pum` pada `student_grade` sudah terisi (hasil Prelim)
4. Buka tampilan gradebook pada kolom **FYE** (term 4 / laporan akhir)

---

## Actual Result

- Kolom FYE kosong (null) untuk semua baris siswa subjek **AS Level** dan **A Level**, meski PUM Prelim sudah terisi.
- Query produksi (master_level 11–17, subjek Cambridge):
  - Programme `'2'` (AS Level): **1.567** baris `as_predicted_pum` terisi, `predicted_pum` = 0, `pum` = 0.
  - Programme `'3'` (A Level): **7.387** baris total, hanya **92** `predicted_pum` terisi, **68** `pum` terisi, **5.889** `as_predicted_pum` terisi. **5.879** baris memiliki prelim data **hanya** di `as_predicted_pum` (keduanya `predicted_pum` dan `pum` NULL).

---

## Expected Result

- Kolom FYE terisi dari PUM Prelim (`as_predicted_pum`) untuk subjek **AS Level**, konsisten dengan perilaku IGCSE (`fye = predicted_pum`) dan A Level.
- Kolom FYE terisi dari PUM Prelim untuk subjek **A Level** — backend harus membaca `as_predicted_pum` bila `predicted_pum`/`pum` NULL, atau frontend harus fallback ke `asPredictedPum`.
- **Atau** jika programme tersebut tidak memiliki FYE, kolom FYE tidak dirender di UI untuk programme tersebut.

---

## Root Cause Analysis

### Bug #1 — Pemetaan PUM AS Level tidak mengisi `fye` — `student-secondary-grade.service.ts` (line 619-620)

Method privat `pumFieldsForProgramme()` menentukan kolom mana (`mye`/`fye`) yang diisi dari PUM. Case AS Level **hanya mengembalikan `mye`**, tanpa `fye`:

```typescript
// api_nest/src/modules/student-secondary-grade/student-secondary-grade.service.ts:611-629
private pumFieldsForProgramme(
  programme: CambridgeProgrammeTypeEnum,
  sg: StudentGrade,
): { mye?: number | null; fye?: number | null } | null {
  switch (programme) {
    case CambridgeProgrammeTypeEnum.IGCSE:
      return { fye: sg.predictedPum ?? null };          // IGCSE: PUM → FYE ✓
    case CambridgeProgrammeTypeEnum.ASLEVEL:
      return { mye: sg.asPredictedPum ?? null };        // AS LEVEL: hanya MYE — fye TIDAK pernah di-set ✗
    case CambridgeProgrammeTypeEnum.ALEVEL:
      return {
        mye: sg.asPredictedPum ?? null,
        fye: sg.predictedPum ?? null,                    // ← hanya membaca predicted_pum, bukan as_predicted_pum
      };
    default:
      return null;
  }
}
```

Method ini dipanggil dari `recalculate()` (line 223-247) yang merupakan roll-up satu-satunya yang mengisi kolom `mye`/`fye` pada `student_grade` untuk level secondary (guard `SECONDARY_MASTER_LEVEL_IDS = [7..17]`, line 31 & 244).

Karena `fye` tidak pernah dikembalikan untuk AS Level, `student_grade.fye` tetap `NULL` untuk baris AS Level — berapa pun PUM-nya.

### Bug #2 — A Level FYE kosong karena frontend tidak membaca `asPredictedPum` — `SecondaryGradebookFlexibleTable.jsx` (line 193-215) + `student-cambridge-grade.service.ts`

Adapter frontend `resolveFyePum(ssg)` dirancang hanya membaca `ssg.pum` / `ssg.predictedPum`:

```javascript
// bbs/client-teacher/src/components/SecondaryGradebookFlexibleTable.jsx:203-215
function resolveFyePum(ssg) {
  // Both IGCSE and A Level use predictedPum for FYE.
  if (cambridgeProg !== 1 && cambridgeProg !== "1" &&
      cambridgeProg !== 3 && cambridgeProg !== "3" &&
      cambridgeProg !== "IGSCE" && cambridgeProg !== "ALEVEL") return null;
  // A Level: prefer actual pum when official results are in
  if ((cambridgeProg === 3 || cambridgeProg === "3" || cambridgeProg === "ALEVEL") &&
      ssg?.pum != null) return ssg.pum;
  if (ssg?.predictedPum != null) return ssg.predictedPum;
  if (ssg?.pum != null) return ssg.pum;
  return null;
}
```

Selain itu, pemetaan `fyeFromPum` di frontend (line 193-198) juga tidak mencakup programme `"2"` (AS Level):

```javascript
// bbs/client-teacher/src/components/SecondaryGradebookFlexibleTable.jsx:193-198
const fyeFromPum =
  isCambridge &&
  (String(cambridgeProg) === "1" ||
    String(cambridgeProg) === "3" ||
    cambridgeProg === "IGCSE" ||
    cambridgeProg === "ALEVEL");
```

**Mechanism decisif:** Saat nilai Prelim A Level disimpan ke database, `student-cambridge-grade.service.ts` membaginya ke enum `ThresholdTypeEnum.AS_PRELIM` sehingga PUM masuk ke `as_predicted_pum` / `as_pum`, **bukan** `predicted_pum` / `pum`. Setelahnya `recalculateForSubjectYear()` dipanggil, tapi backend `pumFieldsForProgramme()` ALEVEL case hanya membaca `sg.predictedPum` untuk `fye` — sehingga baris `grade` FYE yang di-aggregate tetap NULL. Di frontend, `resolveFyePum()` juga tidak pernah melihat `ssg.asPredictedPum`, sehingga mengembalikan null dan sel FYE kosong.

---

## Bukti Database (produksi, read-only via `db-access-tools`)

Query pada `student_grade` JOIN `subject_year`/`class_year`/`subject`, filter `master_level_id BETWEEN 11 AND 17` (Sec3–JC2), `sg.deleted_at IS NULL`, subjek Cambridge:

| Programme | Total baris | `predicted_pum` terisi | `pum` terisi | `as_predicted_pum` terisi | `predicted_pum`=NULL & `pum`=NULL & `as_predicted_pum` terisi |
|-----------|------------|------------------------|--------------|---------------------------|---------------------------------------------------------------|
| `'1'` IGCSE | 18.663 | 10.639 | 1.337 | 0 | 0 |
| `'2'` AS Level | 3.014 | 0 | 0 | 1.567 | 1.567 |
| `'3'` A Level | 7.387 | 92 | 68 | 5.889 | 5.879 |

→ **AS Level:** hanya punya `as_predicted_pum` (1.567 baris). Backend case ASLEVEL tidak memakai `as_predicted_pum` untuk `fye` → `student_grade.fye` kosong.

→ **A Level:** 5.879 dari 7.387 baris memiliki prelim data **hanya** di `as_predicted_pum`. Karena backend `fye` = `sg.predictedPum` dan frontend `resolveFyePum` hanya membaca `ssg.pum` / `ssg.predictedPum` (tidak pernah `ssg.asPredictedPum`), kedua sisi kembalikan null → FYE kosong.

**Baris `grade` per-exam tetap ada (bukan masalah input):**

| Programme | Baris FYE (`exam_type::text::int = 4`) | Baris MYE (= 3) |
|-----------|----------------------------------------|-----------------|
| `'1'` IGCSE | 16.247 | 17.354 |
| `'2'` AS Level | 2.449 | 3.476 |
| `'3'` A Level | 4.741 | 9.966 |

→ Data mentah FYE/MYE per siswa **ada** untuk semua programme; yang gagal adalah **agregasi roll-up ke `student_grade.fye`** untuk AS Level dan **sumber nilai FYE** untuk A Level.

---

## Bukti dari Code Reading

| Sumber | Temuan |
|--------|--------|
| **Code (BE)** — Bug #1 | `pumFieldsForProgramme()` (student-secondary-grade.service.ts:611-629): case `ASLEVEL` hanya return `{ mye }`, tanpa `fye`. Case `ALEVEL` FYE = `sg.predictedPum` saja. |
| **Code (BE)** — Bug #2 | `student-cambridge-grade.service.ts` (1383-1391): `ThresholdTypeEnum.AS_PRELIM` → `asPredictedPum`/`asPum` (bukan `predictedPum`/`pum`). |
| **Code (FE)** | `SecondaryGradebookFlexibleTable.jsx` line 193-198: `fyeFromPum` hanya cover programme "1"/"3" — programme "2" (AS Level) tidak dicover. |
| **Code (FE)** | `resolveFyePum()` (line 203-215): hanya membaca `ssg.pum` / `ssg.predictedPum`, **tidak pernah** `ssg.asPredictedPum` untuk A Level. |
| **Code (FE)** | Cell renderer (line 1011-1042): `isPumSourced` FYE → `resolveFyePum(ssg)` → null → sel kosong. |
| **DB (produksi)** | 1.567 baris AS Level dengan `as_predicted_pum` terisi, `predicted_pum` = 0, `pum` = 0. |
| **DB (produksi)** | 5.879 baris A Level dengan prelim data **hanya** di `as_predicted_pum` (keduanya `predicted_pum` dan `pum` NULL). |
| **DB (produksi)** | Baris `grade` FYE (exam_type=4) untuk programme AS Level di level 11–17: 2.449 — data input ada. |
| **DB (produksi)** | Baris `grade` FYE (exam_type=4) untuk programme A Level di level 11–17: 4.741 — data input ada. |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/student-secondary-grade/student-secondary-grade.service.ts` | `pumFieldsForProgramme()` (line 611-629): case `ASLEVEL` (line 619-620) tidak return `fye` → `student_grade.fye` selalu NULL untuk AS Level. Case `ALEVEL` FYE = `sg.predictedPum` saja (line 621-625). |
| Backend Writer PUM | `api_nest/src/modules/student-cambridge-grade/student-cambridge-grade.service.ts` (line 1383, 1391-1393) | `ThresholdTypeEnum.AS_PRELIM` → `as_predicted_pum`/`as_pum`, bukan `predicted_pum`/`pum`. |
| Backend Entity | `api_nest/src/modules/student-grade/entities/student-grade.entity.ts` | Kolom `predicted_pum`, `as_pum`, `as_predicted_pum` — sumber PUM. `predicted_pum` NULL untuk AS Level; juga NULL untuk A Level yang hanya punya prelim. |
| Frontend Teacher | `bbs/client-teacher/src/components/SecondaryGradebookFlexibleTable.jsx` | `fyeFromPum` (line 193-198) tidak cover programme "2" (AS Level); `resolveFyePum` (line 203-215) hanya membaca `ssg.pum` / `ssg.predictedPum`, tidak `ssg.asPredictedPum` untuk A Level. |
| Frontend Teacher | `bbs/client-teacher/src/views/gradebook/defaultGradebookTables.js` | Tabel default JC (PU/IC/CA/MYE/SA) tidak punya kolom FYE — FYE hanya muncul di konfigurasi grade weight fleksibel. |

---

## Proposed Solution Options

### Option A: Samakan perilaku AS Level dan A Level dengan IGCSE — PUM → `fye` juga (Recommended)

**Backend `student-secondary-grade.service.ts:619-625`** — tambahkan pemetaan `fye` dari `asPredictedPum` untuk AS Level, dan retrofit A Level agar membaca `asPredictedPum` bila `predicted_pum` NULL:

```typescript
case CambridgeProgrammeTypeEnum.ASLEVEL:
  return {
    mye: sg.asPredictedPum ?? null,
    fye: sg.asPredictedPum ?? null,   // ← PUM Prelim AS Level juga menjadi FYE
  };
case CambridgeProgrammeTypeEnum.ALEVEL:
  return {
    mye: sg.asPredictedPum ?? null,
    fye: sg.predictedPum ?? sg.asPredictedPum ?? null,  // ← fallback ke asPredictedPum
  };
```

**Frontend `SecondaryGradebookFlexibleTable.jsx`:**

- Tambahkan programme `"2"` ke kondisi `fyeFromPum` (sedikitnya sebagai sinyal UI).
- Ubah `resolveFyePum(ssg)` asal eksplisit A Level, sehingga membaca `ssg.asPredictedPum` bila `ssg.predictedPum` dan `ssg.pum` NULL:

```javascript
function resolveFyePum(ssg) {
  if (cambridgeProg === "2" || cambridgeProg === "ASLEVEL") {
    // AS Level: FYE dari asPredictedPum (prelim)
    return ssg?.asPredictedPum ?? null;
  }
  if (cambridgeProg === 3 || cambridgeProg === "3" || cambridgeProg === "ALEVEL") {
    // A Level: prefer actual pum when official results are in, fallback to predictedPum, then asPredictedPum
    if (ssg?.pum != null) return ssg.pum;
    if (ssg?.predictedPum != null) return ssg.predictedPum;
    return ssg?.asPredictedPum ?? null;
  }
  // IGCSE
  return ssg?.predictedPum ?? null;
}
```

**Catatan:** Perlu konfirmasi business rule dengan pihak sekolah: apakah FYE AS Level memang harus sama dengan PUM Prelim (seperti IGCSE), atau FYE AS Level seharusnya berasal dari input exam `grade` `exam_type = 4` (2.449 baris sudah ada). Jika yang terakhir, pilih Option B.

### Option B: AS Level FYE berasal dari input exam (roll-up `grade` biasa)

Jika AS Level memiliki ujian FYE sendiri (bukan duplikat PUM), maka `pumFieldsForProgramme()` case AS LEVEL harus `return null` untuk `fye` **dan** `recalculate()` harus meng-aggregate baris `grade` `exam_type = 4` ke `student_grade.fye` untuk AS Level. Saat ini fallback hanya mengisi `fye` dari `grade` ketika pemetaan PUM tidak dipakai — perlu dipastikan jalur aggregasi `grade → fye` aktif untuk AS Level.

### Option C: Sembunyikan kolom FYE untuk AS Level

Jika FYE memang tidak berlaku untuk AS Level: frontend tidak merender kolom FYE sama sekali untuk `cambridge_programme = '2'`, dan grade weight config tidak meng-expose `fye` untuk subjek AS Level. Bug "kolom kosong" hilang, tapi tidak memenuhi ekspektasi awal user (nilai FYE seharusnya tampil dari PUM).

---

## Notes

- Analisis: code reading (`api_nest`, `bbs`) + query database produksi read-only via `db-access-tools` — tidak ada Jam recording.
- Level cakupan: master_level_id 11–17 (Sec3 Accelerated, Sec3 Express, Sec4 Accelerated, Sec4 Express, JC Bridging, JC1, JC2). Guard backend: `SECONDARY_MASTER_LEVEL_IDS = [7..17]` (`student-secondary-grade.service.ts:31`).
- Cambridge programme di tabel `subject` adalah enum string: `'1'`=IGCSE/Upper Secondary, `'2'`=AS Level, `'3'`=A Level, `'4'`=None (lihat `db-access-tools/docs/enum_mappings.md` — label `'1'`–`'4'` di `subject_cambridge_programme_enum`).
- Backend `CambridgeProgrammeTypeEnum`: IGCSE=1, ASLEVEL=2, ALEVEL=3, NONE=4 — cocok dengan nilai enum DB.
- Setelah fix backend diterapkan, baris `student_grade.fye` yang lama (NULL) perlu **backfill** — jalankan ulang `recalculate()` / `bulkRecalculate()` untuk subject year terdampak (AS Level 1.567 baris, A Level 5.879 baris).
- Perlu konfirmasi business rule: FYE AS Level = PUM Prelim (Option A) atau input exam terpisah (Option B) — menentukan mana yang direkomendasikan.
- Bagian Bug #1 (AS Level) dan Bug #2 (A Level) adalah dua mekanisme yang saling terkait: keduanya berujung pada `student_grade.fye` / `resolveFyePum()` menghasilkan null karena sumber PUM yang ada berada di kolom `as_predicted_pum`/`as_pum`.
- Setelah keputusan final, buat spec implementasi di folder ini (ikuti `_templates/spec-template.md`).
- AI operate tidak mengubah kode `api_nest`/`bbs` — implementasi oleh engineer.
