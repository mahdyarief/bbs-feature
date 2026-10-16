---
title: Secondary Gradebook — Nilai FYE kosong pada AS Level (JC) meski PUM Prelim sudah terisi
status: open
severity: major
product: BBS LMS
portal: Teacher Portal
author: System Analyst
date: 2026-10-16
jam:
---

# Secondary Gradebook — FYE Kosong pada AS Level Meski PUM Prelim Sudah Ada

## Summary

Di Teacher Portal → Gradebook (Secondary/JC), kolom **FYE** pada subjek Cambridge **AS Level** (`cambridge_programme = '2'`) selalu kosong, padahal nilai **PUM** (`as_predicted_pum`) untuk subjek tersebut sudah terisi dari hasil Prelim. Masalah ini terjadi di level **Secondary 3 s/d JC 2** (master_level_id 11–17).

Akar masalahnya ada di backend: fungsi `pumFieldsForProgramme()` hanya memetakan PUM → `mye` untuk programme AS Level, dan **tidak pernah mengisi field `fye`** — berbeda dengan IGCSE (PUM → `fye`) dan A Level (PUM → `mye` + `fye`). Akibatnya `student_grade.fye` tetap `NULL` terus untuk baris AS Level, dan karena kolom FYE di UI membaca dari `ssg.fye` / `ssg.predictedPum` (yang juga `NULL` untuk AS Level), user melihat kolom FYE kosong.

**Ekspektasi:** Nilai FYE pada subjek AS Level harus terisi dari PUM Prelim (konsisten dengan perilaku IGCSE & A Level), atau kolom FYE tidak ditampilkan sama sekali jika memang AS Level tidak memiliki FYE.

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
3. Buka tampilan gradebook pada kolom **FYE** (term 4 / laporan akhir)

**Actual Result:**
- Kolom FYE kosong (null) untuk semua baris siswa subjek AS Level, meski PUM Prelim sudah terisi.
- Query produksi (master_level 11–17, subjek Cambridge): programme `'2'` (AS Level) punya **1.567** baris `as_predicted_pum` terisi, tapi `student_grade.fye` tidak pernah terisi dari PUM tersebut.

**Expected Result:**
- Kolom FYE terisi dari PUM Prelim (as_predicted_pum) untuk subjek AS Level, konsisten dengan IGCSE (`fye = predicted_pum`) dan A Level (`fye = predicted_pum`); **atau** jika AS Level memang tidak memiliki FYE, kolom FYE tidak dirender di UI untuk programme AS Level.

---

## Root Cause Analysis

### Bug #1 — Pemetaan PUM AS Level tidak mengisi `fye` — `student-secondary-grade.service.ts` (line 611-629)

Method privat `pumFieldsForProgramme()` menentukan kolom mana (`mye`/`fye`) yang diisi dari PUM. Case AS Level **hanya mengembalikan `mye`**, tanpa `fye`:

```typescript
// api_nest/src/modules/student-secondary-grade/student-secondary-grade.service.ts:611-629
private pumFieldsForProgramme(
  programme: CambridgeProgrammeTypeEnum,
  sg: StudentGrade,
): { mye?: number | null; fye?: number | null } | null {
  switch (programme) {
    case CambridgeProgrammeTypeEnum.IGCSE:
      // Preliminary exam PUM is used for FYE throughout the academic year.
      return { fye: sg.predictedPum ?? null };          // IGCSE: PUM → FYE ✓
    case CambridgeProgrammeTypeEnum.ASLEVEL:
      return { mye: sg.asPredictedPum ?? null };        // AS LEVEL: hanya MYE — fye TIDAK pernah di-set ✗
    case CambridgeProgrammeTypeEnum.ALEVEL:
      return {
        mye: sg.asPredictedPum ?? null,                 // A LEVEL: keduanya ✓
        fye: sg.predictedPum ?? null,
      };
    default:
      return null;                                      // NONE/null → fallback bobot CA/SA manual
  }
}
```

Method ini dipanggil dari `recalculate()` (line 223-247) yang merupakan roll-up satu-satunya yang mengisi kolom `mye`/`fye` pada `student_grade` untuk level secondary (guard `SECONDARY_MASTER_LEVEL_IDS = [7..17]`, line 31 & 244):

```typescript
// student-secondary-grade.service.ts:244-247
if (!masterLevelId || !SECONDARY_MASTER_LEVEL_IDS.includes(masterLevelId)) {
  // Not a secondary subject — leave the row alone (primary owns roll-up).
  return sg;
}
```

Karena `fye` tidak pernah dikembalikan untuk AS Level, `student_grade.fye` tetap `NULL` untuk baris AS Level — berapa pun PUM-nya.

### Faktor pendukung (Frontend)

- `bbs/client-teacher/src/components/SecondaryGradebookFlexibleTable.jsx` (line 193-198): `fyeFromPum` hanya `true` untuk programme `"1"` (IGCSE) / `"3"` (A Level) — **programme `"2"` (AS Level) tidak dicover** untuk sumber FYE dari PUM. Komentar di line 174-175 juga menyatakan: *"IGCSE → FYE from PUM | ASLEVEL → MYE from PUM | ALEVEL → both from PUM"* — pemetaan yang sama dengan backend, sehingga FYE AS Level tidak pernah di-source dari PUM di UI.
- `SecondaryGradebookFlexibleTable.jsx` (line 203-207): `resolveFyePum(ssg)` mengembalikan `ssg.predictedPum` — kolom `predicted_pum` memang `NULL` untuk AS Level (AS Level memakai `as_predicted_pum`), jadi read-only FYE dari PUM juga kosong.
- `bbs/client-teacher/src/views/gradebook/defaultGradebookTables.js` (line 210-213): tabel default Sec1–2 term 4 memakai `["CA2", "FYA"]` (bukan FYE); tabel default JC (PU/IC/CA/MYE/SA) **tidak punya kolom FYE** — sehingga FYE hanya muncul pada konfigurasi grade weight fleksibel, dan di situ tetap kosong.

### Bukti Database (produksi, read-only via `db-access-tools`)

Query pada `student_grade` JOIN `subject_year`/`class_year`/`subject`, filter `master_level_id BETWEEN 11 AND 17` (Sec3–JC2), `sg.deleted_at IS NULL`, subjek Cambridge:

**Kehadiran PUM per programme:**

| Programme | Total baris | `predicted_pum` terisi | `as_predicted_pum` terisi |
|-----------|------------|------------------------|---------------------------|
| `'1'` IGCSE | — | 10.639 | — |
| `'2'` AS Level | — | 0 | 1.567 |
| `'3'` A Level | — | (ada) | 5.889 |
| `'4'` None | 70.995 | 0 | 0 |

→ AS Level **hanya** punya `as_predicted_pum` (1.567 baris), `predicted_pum` = 0. Karena case AS Level tidak memakai `as_predicted_pum` untuk `fye`, `student_grade.fye` kosong.

**Baris `grade` per-exam tetap ada (bukan masalah input):**

| Programme | Baris FYE (`exam_type::text::int = 4`) | Baris MYE (= 3) |
|-----------|----------------------------------------|-----------------|
| `'1'` IGCSE | 16.247 | 17.354 |
| `'2'` AS Level | 2.449 | 3.476 |
| `'3'` A Level | 4.741 | 9.966 |

→ Data mentah FYE/MYE per siswa **ada** untuk semua programme; yang gagal adalah **agregasi roll-up ke `student_grade.fye`** untuk AS Level.

---

## Bukti dari Jam

Tidak ada Jam recording untuk bug ini — analisis dilakukan via code reading (`api_nest`, `bbs`) dan query database produksi read-only.

| Sumber | Temuan |
|--------|--------|
| **Code (BE)** | `pumFieldsForProgramme()` line 619-620: case ASLEVEL hanya return `{ mye }`, tanpa `fye` |
| **Code (FE)** | `SecondaryGradebookFlexibleTable.jsx` line 193-198: `fyeFromPum` hanya cover programme "1"/"3" |
| **DB (produksi)** | 1.567 baris AS Level dengan `as_predicted_pum` terisi; `predicted_pum` = 0; `student_grade.fye` tidak terisi dari PUM |
| **DB (produksi)** | Baris `grade` FYE (exam_type=4) untuk programme AS Level di level 11–17: 2.449 — data input ada |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/student-secondary-grade/student-secondary-grade.service.ts` | `pumFieldsForProgramme()` (line 611-629): case `ASLEVEL` (line 619-620) tidak return `fye` → `student_grade.fye` selalu NULL untuk AS Level. Dipanggil dari `recalculate()` (line 223-247) |
| Backend Entity | `api_nest/src/modules/student-grade/entities/student-grade.entity.ts` (line 89, 92, 95) | Kolom `predicted_pum`, `as_pum`, `as_predicted_pum` — sumber PUM; `predicted_pum` NULL untuk AS Level |
| Backend (writer PUM) | `api_nest/src/modules/student-cambridge-grade/student-cambridge-grade.service.ts` (line 1383, 1393, 1860, 1871) | Menulis `predictedPum` & `asPredictedPum` — tidak bermasalah, data PUM masuk dengan benar |
| Frontend Teacher | `bbs/client-teacher/src/components/SecondaryGradebookFlexibleTable.jsx` | `fyeFromPum` (line 193-198) tidak cover AS Level; `resolveFyePum` (line 203-207) baca `predictedPum` yang NULL untuk AS Level; FYE input hanya render jika `hasFye && !fyeFromPum` (line 247-249) |
| Frontend Teacher | `bbs/client-teacher/src/views/gradebook/defaultGradebookTables.js` (line 210-213) | Tabel default: Sec1-2 term 4 = FYA (bukan FYE); tabel JC tanpa kolom FYE — FYE hanya muncul di grade weight fleksibel |

---

## Proposed Solution Options

### Option A: Samakan perilaku AS Level dengan A Level — PUM → `fye` juga (Recommended)

Backend `student-secondary-grade.service.ts:619-620` — tambahkan pemetaan `fye` dari `asPredictedPum`:

```typescript
case CambridgeProgrammeTypeEnum.ASLEVEL:
  return {
    mye: sg.asPredictedPum ?? null,
    fye: sg.asPredictedPum ?? null,   // ← PUM Prelim AS Level juga menjadi FYE
  };
```

Frontend `SecondaryGradebookFlexibleTable.jsx` — tambahkan programme `"2"` ke kondisi `fyeFromPum` dan sesuaikan `resolveFyePum` agar AS Level membaca `ssg.asPredictedPum` (bukan `ssg.predictedPum`).

**Catatan:** Perlu konfirmasi business rule dengan pihak sekolah: apakah FYE AS Level memang harus sama dengan PUM Prelim (seperti IGCSE), atau FYE AS Level seharusnya berasar dari input exam `grade` `exam_type = 4` (2.449 baris sudah ada). Jika yang terakhir, pilih Option B.

### Option B: AS Level FYE berasal dari input exam (roll-up `grade` biasa)

Jika AS Level memiliki ujian FYE sendiri (bukan duplikat PUM), maka `pumFieldsForProgramme()` case AS LEVEL harus `return null` untuk `fye` **dan** `recalculate()` harus meng-aggregate baris `grade` `exam_type = 4` ke `student_grade.fye` untuk AS Level (perilaku fallback bobot CA/SA manual). Saat ini fallback hanya mengisi `fye` dari `grade` ketika pemetaan PUM tidak dipakai — perlu dipastikan jalur aggregasi `grade → fye` aktif untuk AS Level.

### Option C: Sembunyikan kolom FYE untuk AS Level

Jika FYE memang tidak berlaku untuk AS Level: frontend tidak merender kolom FYE sama sekali untuk `cambridge_programme = '2'`, dan grade weight config tidak meng-expose `fye` untuk subjek AS Level. Bug "kolom kosong" hilang, tapi tidak memenuhi ekspektasi awal user (nilai FYE seharusnya tampil dari PUM).

---

## Notes

- Analisis: code reading (`api_nest`, `bbs`) + query database produksi read-only via `db-access-tools` — tidak ada Jam recording.
- Level cakupan: master_level_id 11–17 (Sec3 Accelerated, Sec3 Express, Sec4 Accelerated, Sec4 Express, JC Bridging, JC1, JC2). Guard backend: `SECONDARY_MASTER_LEVEL_IDS = [7..17]` (`student-secondary-grade.service.ts:31`).
- Cambridge programme di tabel `subject` adalah enum string: `'1'`=IGCSE/Upper Secondary, `'2'`=AS Level, `'3'`=A Level, `'4'`=None (lihat `db-access-tools/docs/enum_mappings.md` — label `'1'`–`'4'` di `subject_cambridge_programme_enum`).
- Backend `CambridgeProgrammeTypeEnum`: IGCSE=1, ASLEVEL=2, ALEVEL=3, NONE=4 — cocok dengan nilai enum DB.
- Setelah fix backend diterapkan, baris `student_grade.fye` yang lama (NULL) perlu **backfill** — jalankan ulang `recalculate()` / `bulkRecalculate()` untuk subject year terdampak.
- Perlu konfirmasi business rule: FYE AS Level = PUM Prelim (Option A) atau input exam terpisah (Option B) — menentukan mana yang direkomendasikan.
- Setelah keputusan final, buat spec implementasi di folder ini (ikuti `_templates/spec-template.md`).
- AI operate tidak mengubah kode `api_nest`/`bbs` — implementasi oleh engineer.
