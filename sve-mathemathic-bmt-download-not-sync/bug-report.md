---
title: SVE portal tampil "Mathematics" sedangkan BMT download tampil "Science" — subject mismatch untuk guru yang sama
status: open
severity: major
product: BBS LMS
portal: Teacher
author: System Analyst
date: 2026-11-09
jam: https://jam.dev/c/d051feef-6290-4a0d-afbb-2ccbd82d52d3
---

# SVE portal tampil "Mathematics", BMT download tampil "Science" — subject mismatch guru yang sama (Alice DP Shajima Bright, emp 517, AY 2026/2027)

## Summary

**Ekspektasi:** Portal SVE dan BMT download untuk guru yang sama (Alice DP Shajima Bright, employee_id 517, academic year 2026/2027) seharusnya menampilkan subject yang konsisten, yaitu Mathematics (subject_id 431) berdasarkan tugas pengajaran guru di `subject_year`.

**Fakta:** SVE portal menunjukkan subject "mathemathic" (Mathematics), sedangkan BMT download menunjukkan "Science" untuk guru yang sama — padahal data live DB menunjukkan semua record `sve_exam_type` milik emp 517 di AY 2026/2027 justru terhubung ke subject **Mathematics**, bukan Science. Kemungkinan besar BMT download membaca `sve.subject_id` yang berbeda dari `subject_year.subject_id` (two-column subject_id divergence), atau ada record `sve` yang terhubung ke subject Science yang tidak terekspos di portal SVE.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | System Analyst |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | 517 (Alice DP Shajima Bright) |
| Portal URL | https://teacher.smartbag.binabangsaschool.com/sves?pageSize=12&page=0&academicYearId=27 |
| Environment API | api_nest (NestJS / TypeORM) — production binabangsa_prod_mig_v01 |
| Data konteks (class ID / daId / tanggal) | Academic Year 27 = `2026/2027`, kelas 9 (level 9) |
| Browser / OS | — |

---

## Steps to Reproduce

1. Login ke portal Teacher (BBS LMS) dengan akun karyawan 517 (Alice DP Shajima Bright).
2. Buka halaman SVE portal (`/sves?academicYearId=27`).
3. Lihat subject yang ditampilkan untuk record Sve — terlihat **"mathemathic"/Mathematics**.
4. Buka halaman BMT File Download (`/bmt-file-download`).
5. Lihat subject yang ditampilkan untuk record BMT — terlihat **"Science"**.
6. (Opsional) Buka DB live dan query `sve_exam_type` untuk employee_id 517 di AY 27 — semua record justru berhubungan dengan Mathematics, bukan Science.

**Actual Result:** SVE portal → "mathemathic", BMT download → "Science" (tidak konsisten untuk guru yang sama).

**Expected Result:** Keduanya menampilkan subject yang sama, yaitu Mathematics (subject_id 431) sesuai tugas pengajaran guru di `subject_year` (subject_id 431, kelas 9, 2026/2027).

---

## Root Cause Analysis

### Bug #1 — Dua kolom `subject_id` yang berbeda dan bisa divergen (backend)

**`api_nest/src/modules/sve-exam-type/sve-exam-type.service.ts`**

Dua endpoint berbagi file yang sama tetapi membaca `subject_id` dari sumber yang **berbeda**:

- `findAllByUser` (SVE portal): filter `SubjectYear.teachers` berdasarkan `teacherId` + `classYear.academicYear.id`, lalu baca **`Subject.subject_id`** — subject yang di-assign ke guru melalui `subject_year`.
- `findBmtOnly` (BMT download): filter `SveExamType` berdasarkan `bmtWeek` **dan** `sve.subjectId`, lalu baca **`Sve.subject_id`** — subject yang tersimpan pada record `Sve` yang dibuat ketika BMT diset-up.

```
guru (emp 517) ──► subject_year.subject_id ──► SVE portal → "Mathematics"
                                                      ▲
                                                      │ divergen
                                                      ▼
                                              sve.subject_id ──► BMT download → "Science"
```

Data live DB (AY 2026/2027, emp 517):

- `subject_year_teachers_teacher` + `subject_year_teachers_employee`: **tidak ada** baris untuk emp 517 di academic_year_id = 27 (tugas pengajaran guru di tahun ini belum tercatat/ada gap data). Tugas 517: Mathematics (431) di 2023/2024, 2024/2025; Mathematics (399) di 2025/2026.
- `sve_exam_type` emp 517 di AY 27 (setter/expert/vetter): semuanya `subject_id = 431` → **Mathematics**. Tidak ada baris Science (407/415/432/442).
- `bmt_week` di AY 27 memiliki baris subject **Science** (mis. `id=50` subject_id 407 "English Language"? faktanya Science: 407, 415, 432, 442) yang **tidak terhubung** ke sve_exam_type emp 517.

**Sinkronisasi:** `sve.subject_id` tidak selalu sinkron dengan `subject_year.subject_id` milik guru yang sama. Saat BMT disimpan/diubah, `sve.subject_id` bisa diisi subject yang berbeda dari tugas pengajaran guru, sehingga BMT download menampilkan "Science" sementara SVE portal menampilkan "Mathematics".

### Bug #2 — Frontend membaca dari `sve.subjectId` (frontend)

**`bbs/client-teacher/src/views/sve/Sve.jsx` line 318** → `subjects?.[sve?.subjectId]`

**`bbs/client-teacher/src/views/bmt-file-download/BmtFileDownload.jsx`** → `subjects?.[sve?.subjectId]`

Kedua view membaca subject dari `sve.subjectId`, tetapi endpoint backend yang mengisi data tersebut berbeda (`findAllByUser` vs `findBmtOnly`), sehingga hasil render bisa berbeda untuk guru yang sama.

---

## Bukti dari Jam

| Sumber | Temuan |
|--------|--------|
| **Video (t=...)** | SVE portal menampilkan subject "mathemathic" (Mathematics); BMT download menampilkan subject "Science" untuk guru yang sama (emp 517, AY 2026/2027). |
| **Network** | `GET /sve-exam-type` (SVE portal) vs `GET /sve-exam-type/bmt-only` (BMT download) → dua endpoint yang berbeda, masing-masing membaca `subject_id` berbeda. |
| **Console** | Mungkin ada warning/API error di frontend karena data subject tidak cocok — perlu dicek di recording. |
| **User events** | Teacher login sebagai emp 517, akses /sves dan /bmt-file-download, lihat subject yang berbeda. |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/sve-exam-type/sve-exam-type.service.ts` | `findAllByUser` vs `findBmtOnly` membaca `subject_id` berbeda (Subject vs Sve) |
| Backend DTO | (DTO sve_exam_type) | `sve.subjectId` mungkin tidak konsisten dengan `subject_year.subject_id` |
| Backend Entity | `sve.entity.ts`, `subject-year.entity.ts` | `sve.subject_id` dan `subject_year.subject_id` dua kolom terpisah |
| Frontend | `Sve.jsx:318`, `BmtFileDownload.jsx` | `subjects?.[sve?.subjectId]` — render dari sumber data yang berbeda |

---

## Proposed Solution Options

### Option A: Single source of truth + sync validation (Recommended)

- Pastikan `Sve.subject_id` selalu sinkron dengan `subject_year.subject_id` guru yang mengampu saat pembuatan/edit BMT.
- Di service `findBmtOnly`, tambahkan validasi/penyesuaian bahwa jika `sve.subject_id` != `subject_year.subject_id` guru yang terkait, lakukan fallback ke `subject_year.subject_id` atau kembalikan error.
- Query `sve_exam_type` di kedua endpoint menggunakan sumber yang sama (mis. `subject_year.subject_id` terlebih dahulu, `sve.subject_id` sebagai fallback).

```typescript
// sve-exam-type.service.ts — contoh penyesuaian findBmtOnly
const teacherSubjectId = await this.subjectYearService.getSubjectIdByTeacher(
  sve.subjectId,  // atau filter guru
  academicYearId,
);
const finalSubjectId = teacherSubjectId ?? sve.subjectId; // fallback jika data gap
```

- (Opsional) Hapus kolom `Sve.subject_id` yang redundan, atau wajikan semua BMT menggunakan `subject_id` yang sama dengan tugas pengajaran guru.

### Option B: Reconcile existing data

- Jalankan migration untuk sinkronisasi ulang `sve.subject_id` dengan `subject_year.subject_id` untuk seluruh record yang sudah divergent di seluruh academic year (dapatkan dari live DB).
- Contoh transformasi (PostgreSQL):

```sql
UPDATE sve s
SET subject_id = sy.subject_id
FROM subject_year sy
WHERE s.id = sy.sve_id
  AND sy.subject_id <> s.subject_id;
```

---

## Notes

- Environment API: production `binabangsa_prod_mig_v01` (PostgreSQL 14.22). Credential DB_PASSWORD: `MigrasiSudahBeres!`.
- Data live yang dikumpulkan: AY 27 = `2026/2027`; emp 517 **tidak ada** `subject_year` di AY 27 (gap data tugas pengajaran guru); `sve_exam_type` emp 517 di AY 27 semuanya Mathematics (431, 406, 414, 441); `bmt_week` AY 27 ada subject Science (407, 415, 432, 442) yang tidak terhubung ke emp 517.
- `subject_year_teachers_teacher` vs `subject_year_teachers_employee`: tabel join yang benar adalah `subject_year_teachers_employee` (employee_id), bukan `subject_year_teachers_teacher`.
- `class_year` memakai `academic_year_id` (bukan `year`), `employee` menggunakan `full_name` (bukan `name`).
- Prioritas: pilih Option A untuk mencegah kasus di masa depan, lalu Option B untuk membersihkan data yang sudah divergen.
