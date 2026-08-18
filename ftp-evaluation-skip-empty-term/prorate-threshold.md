---
title: FTP Evaluation — Prorate Threshold Mark untuk Student Partial-Year
status: draft
author: BBS Team
date: 2026-08-18
depends_on: ftp-evaluation-skip-empty-term (skip empty term enhancement)
---

# FTP Evaluation — Prorate Threshold Mark untuk Student Partial-Year

## Latar Belakang

Setelah enhancement **skip empty term** diimplementasi, term tanpa conduct data tidak lagi dihitung. Ini memperbaiki score yang sebelumnya "tercemar" oleh data dari term lain. Namun muncul masalah baru: **threshold mark (`getEvaluationMark`) tetap absolut** — tidak disesuaikan dengan jumlah term yang dimiliki student.

### Masalah

| Kondisi | Max Score Possible | Threshold EXCEEDING (≥42) | Threshold MEETING (≥25) |
|---------|-------------------|---------------------------|-------------------------|
| Full-year (4 terms) | 4 × 4 = **16** per eval... | 42 | 25 |

Tunggu — mari kita hitung ulang max score secara tepat.

## Analisis Max Score

### Struktur Kalkulasi (dari `ftp-evaluation.ts`)

```
Score = Σ (per evaluation table) Σ (per term with data) getConductPoint(sumBadges)
```

- `getConductPoint` return **1–4** (4 = always, 3 = mostOfTheTime, 2 = sometimes, 1 = less)
- **Per evaluation term**, ada **3 tables** yang dievaluasi (dari `ftp_evaluation_setting`)
- Score per term = 3 tables × max 4 points = **12 points**

### Max Score per Jumlah Term

| Jumlah term dengan data | Max Score | Bisa EXCEEDING (≥42)? | Bisa MEETING (≥25)? |
|-------------------------|-----------|----------------------|---------------------|
| 4 terms (full-year) | 4 × 12 = **48** | ✅ Ya (≥42) | ✅ Ya (≥25) |
| 3 terms | 3 × 12 = **36** | ❌ Tidak mungkin | ✅ Ya (≥25) |
| 2 terms (masuk term 3) | 2 × 12 = **24** | ❌ Tidak mungkin | ❌ Tidak mungkin (max 24 < 25) |
| 1 term (masuk term 4) | 1 × 12 = **12** | ❌ Tidak mungkin | ❌ Tidak mungkin |

**Kesimpulan:** Student yang masuk di term 3 (2 terms data) **secara matematis tidak bisa mencapai MEETING** (max 24 < threshold 25). Student masuk term 2 (3 terms data) bisa MEETING tapi tidak bisa EXCEEDING. Ini tidak adil.

---

## Usulan Formulasi Prorate

### Opsi A: Prorate berdasarkan jumlah term dengan data (Recommended)

Threshold disesuaikan proporsional dengan jumlah term yang dimiliki student relatif terhadap full-year (4 terms).

```typescript
const totalTerms = 4; // full-year
const activeTerms = numberOfTermsWithData; // 1-4

const prorateRatio = activeTerms / totalTerms;

const exceedingThreshold = Math.ceil(42 * prorateRatio);
const meetingThreshold = Math.ceil(25 * prorateRatio);
```

**Tabel threshold prorate:**

| Jumlah term | Ratio | EXCEEDING (≥) | MEETING (≥) | BELOW (<) |
|-------------|-------|---------------|-------------|-----------|
| 4 terms | 4/4 = 1.0 | **42** | **25** | < 25 |
| 3 terms | 3/4 = 0.75 | **32** (31.5→ceil) | **19** (18.75→ceil) | < 19 |
| 2 terms | 2/4 = 0.5 | **21** | **13** (12.5→ceil) | < 13 |
| 1 term | 1/4 = 0.25 | **11** (10.5→ceil) | **7** (6.25→ceil) | < 7 |

**Verifikasi dengan case student 103033 (masuk term 3, 2 terms data):**
- Score saat ini (setelah skip empty term): ~16-20 (dari term 3+4 saja)
- Threshold MEETING prorate: ≥13
- Threshold EXCEEDING prorate: ≥21
- → Student dengan score 20 akan mendapat **MEETING_EXPECTATIONS** (adil, bukan selalu BELOW)

### Opsi B: Prorate berdasarkan entry term

Threshold dihitung dari term berapa student mulai masuk (bukan jumlah term dengan data).

```typescript
const entryTerm = 3; // student masuk term 3
const termsAvailable = totalTerms - entryTerm + 1; // 3,4 = 2 terms
const prorateRatio = termsAvailable / totalTerms;
```

**Kelebihan:** Lebih eksplisit, cocok jika field `entry_term` ditambahkan ke skema.
**Kekurangan:** Memerlukan field baru atau derive dari data; secara matematis identik dengan Opsi A jika `termsAvailable = activeTerms`.

### Opsi C: Normalisasi score ke skala full-year

Score dinormalisasi ke skala 48 (full-year max) sebelum dibandingkan dengan threshold absolut.

```typescript
const normalizedScore = Math.round((rawScore / (activeTerms * 12)) * 48);
// normalizedScore kemudian dibandingkan dengan threshold 42/25 seperti biasa
```

**Kelebihan:** Threshold `getEvaluationMark` tidak perlu diubah.
**Kekurangan:** Score yang ditampilkan di report bukan score mentah — bisa membingungkan. Perlu keputusan apakah `student_ftp_conduct_evaluation.score` menyimpan raw atau normalized.

---

## Rekomendasi: Opsi A

**Alasan:**
1. **Minimal perubahan** — hanya `getEvaluationMark` perlu menerima parameter tambahan (`activeTerms`).
2. **Score tetap mentah** — `student_ftp_conduct_evaluation.score` tetap menyimpan raw score, tidak ada disonansi antara data di DB dan yang ditampilkan.
3. **Deterministik** — `activeTerms` bisa di-derive dari data existing (count distinct term di `student_ftp_conduct` untuk `student_report_id` tersebut), tidak perlu field baru.
4. **Backward-compatible** — untuk student full-year (4 terms), threshold tetap 42/25 (ratio = 1.0).

---

## Implementasi Detail

### File yang Berubah

| File | Perubahan |
|------|-----------|
| `api_nest/src/helpers/reports/ftp-evaluation.ts` | (1) Modifikasi `getEvaluationMark` menerima `activeTerms`. (2) Hitung `activeTerms` dari conduct data. (3) Prorate threshold sebelum assign evaluation mark. |

### Kode Usulan

```typescript
// ftp-evaluation.ts

// SEBELUM:
const getEvaluationMark = (score = 0): FtpEvaluationMarkTypeEnum => {
  if (score >= 42) return FtpEvaluationMarkTypeEnum.EXCEEDING_EXPECTATIONS;
  if (score >= 25) return FtpEvaluationMarkTypeEnum.MEETING_EXPECTATIONS;
  return FtpEvaluationMarkTypeEnum.BELOW_EXPECTATIONS;
};

// SESUDAH:
const FULL_YEAR_TERMS = 4;
const FULL_YEAR_EXCEEDING_THRESHOLD = 42;
const FULL_YEAR_MEETING_THRESHOLD = 25;

const getEvaluationMark = (
  score = 0,
  activeTerms = FULL_YEAR_TERMS,
): FtpEvaluationMarkTypeEnum => {
  const prorateRatio = activeTerms / FULL_YEAR_TERMS;
  const exceedingThreshold = Math.ceil(FULL_YEAR_EXCEEDING_THRESHOLD * prorateRatio);
  const meetingThreshold = Math.ceil(FULL_YEAR_MEETING_THRESHOLD * prorateRatio);

  if (score >= exceedingThreshold) {
    return FtpEvaluationMarkTypeEnum.EXCEEDING_EXPECTATIONS;
  }
  if (score >= meetingThreshold) {
    return FtpEvaluationMarkTypeEnum.MEETING_EXPECTATIONS;
  }
  return FtpEvaluationMarkTypeEnum.BELOW_EXPECTATIONS;
};
```

### Perhitungan `activeTerms`

Di dalam `ftpEvaluation`, `activeTerms` = jumlah distinct term yang punya conduct rows untuk student_report tersebut **dan** table-nya termasuk dalam evaluation setting:

```typescript
// Di dalam ftpEvaluation(), setelah filteredFtpConducts:
const activeTermsSet = new Set(filteredFtpConducts.map((sfc) => sfc.term));
const activeTerms = activeTermsSet.size || FULL_YEAR_TERMS; // fallback ke 4 jika kosong

// ... kemudian:
studentEval.evaluationMark = getEvaluationMark(total, activeTerms);
```

**Catatan:** `activeTerms` dihitung dari `filteredFtpConducts` (sudah difilter oleh `ftpReportTableId` dari evaluation setting). Ini memastikan hanya term yang benar-benar berkontribusi ke score yang dihitung.

### Panggilan `getEvaluationMark` di `ftpEvaluation`

```typescript
// Line ~91 (existing eval update):
studentEval.evaluationMark = getEvaluationMark(total, activeTerms);

// Line ~99 (new eval create):
newStudentEval.evaluationMark = getEvaluationMark(total, activeTerms);
```

---

## Edge Cases

### EC-P01: Student full-year (4 terms) — regression check
- `activeTerms = 4`, `prorateRatio = 1.0`
- Threshold: ≥42 EXCEEDING, ≥25 MEETING — **sama persis dengan sebelumnya**
- ✅ Tidak ada regression

### EC-P02: Student masuk term 4 (1 term data)
- `activeTerms = 1`, `prorateRatio = 0.25`
- Threshold: ≥11 EXCEEDING, ≥7 MEETING
- Max score = 12 → bisa EXCEEDING jika badges tinggi
- ✅ Adil

### EC-P03: Student tidak punya conduct data sama sekali
- `activeTerms = 0` → fallback ke `FULL_YEAR_TERMS (4)` (division by zero prevention)
- Tapi score = 0 → tetap BELOW_EXPECTATIONS
- ✅ Aman (juga ditangani oleh skip-empty-term: tidak dibuat eval row)

### EC-P04: Student masuk term 2 (3 terms data)
- `activeTerms = 3`, `prorateRatio = 0.75`
- Threshold: ≥32 EXCEEDING, ≥19 MEETING
- Max score = 36 → bisa EXCEEDING (36 ≥ 32) dan MEETING
- ✅ Adil

### EC-P05: Rounding — apakah `Math.ceil` tepat?
- `Math.ceil(42 * 0.75) = Math.ceil(31.5) = 32` — threshold sedikit lebih tinggi dari exact proportional
- `Math.ceil(25 * 0.5) = Math.ceil(12.5) = 13` — sama
- Alternatif `Math.round`: `Math.round(31.5) = 32`, `Math.round(12.5) = 13` — hasil sama untuk case umum
- **Keputusan:** `Math.ceil` dipilih agar threshold sedikit lebih ketat (student harus usaha sedikit lebih untuk reach kategori)

---

## Verifikasi dengan Data Produksi (Student 103033)

Dari `notes.md`: student 103033 masuk term 3, academic year 2025/2026.

### Sebelum fix (kondisi sekarang):
- Score term 1 = 21 (dihitung dari data term 3+4 yang table-nya cocok setting term 1)
- Score term 3 = 21, Score term 4 = 19
- Evaluation mark: semua BELOW_EXPECTATIONS (< 25)
- **Masalah:** term 1/2 seharusnya tidak ada; score term 3/4 benar tapi mark salah karena threshold absolut

### Setelah skip-empty-term fix (tanpa prorate):
- Term 1/2: tidak dibuat eval row ✅
- Score term 3 = ~12 (hanya dari conduct term 3, 3 tables × 4 points max)
- Score term 4 = ~12 (hanya dari conduct term 4)
- Mark: BELOW_EXPECTATIONS (12 < 25) — **masih tidak adil**

### Setelah skip-empty-term + prorate:
- Term 3: score ~12, `activeTerms = 2`, threshold MEETING ≥ 13 → **BELOW** (12 < 13)
- Term 4: score ~12, `activeTerms = 2`, threshold MEETING ≥ 13 → **BELOW** (12 < 13)
- Jika badges lebih tinggi (misal 3+ per table): score bisa 3×3 = 9 per term, atau 3×4 = 12 per term
- Jika student konsisten "always" (badges ≥ 5): score = 3×4 = 12 per term → 12 < 13 → BELOW
- Jika student sangat bagus: perlu hampir sempurna di semua table untuk reach MEETING dengan prorate

**Catatan:** Dengan data actual student 103033 (badges campuran 1-2-3-4-5), score per term setelah skip-empty-term kemungkinan sekitar 8-12. Threshold MEETING prorate = 13. Student ini masih BELOW tapi **lebih dekat ke MEETING** — ini adil karena performanya memang di bawah rata-rata.

---

## Scope & Constraints

### In Scope
- Modifikasi `getEvaluationMark` di `ftp-evaluation.ts` untuk menerima `activeTerms` dan prorate threshold
- Perhitungan `activeTerms` dari `filteredFtpConducts`
- Verifikasi dengan data produksi

### Out of Scope
- Perubahan skema database (tidak perlu field `entry_term`)
- Perubahan `getConductPoint` atau kalkulasi score mentah
- Perubahan UI / report template
- Normalisasi score di DB (score tetap raw)

### Dependencies
- Enhancement **skip-empty-term** harus sudah diimplementasi terlebih dahulu
- Threshold prorate hanya bermakna jika empty terms sudah di-skip (jika tidak, `activeTerms` akan selalu 4 karena data term lain " Bocor ")

---

## Pertanyaan untuk Tim

1. **Rounding:** `Math.ceil` atau `Math.round`? (Rekomendasi: `Math.ceil` — sedikit lebih ketat)
2. **Fallback `activeTerms = 0`:** gunakan `FULL_YEAR_TERMS (4)` atau return `BELOW_EXPECTATIONS` langsung? (Rekomendasi: fallback ke 4, karena case ini sudah ditangani skip-empty-term — tidak dibuat eval row)
3. **Apakah prorate perlu ditampilkan di report PDF?** (misal footnote: "Threshold disesuaikan untuk student yang masuk di term 3") — saat ini tidak ada rencana UI change
