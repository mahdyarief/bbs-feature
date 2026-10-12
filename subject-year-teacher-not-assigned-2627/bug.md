---
title: Subject in Year 26/27 — "Students assigned to" tampil tapi kolom Teacher "not assigned" (Secondary & JC)
status: open
severity: major
product: BBS LMS
portal: Admin
author: System Analyst
date: 2026-10-12
jam:
---

# Subject in Year 26/27 — Student assigned tapi Teacher "not assigned" (semua subject, terparah Secondary & Junior College)

## Summary

Di halaman Subject in Year AY 26/27, semua subject (terkhusus Secondary dan Junior College) menampilkan blok "Students assigned to {nama}" tetapi kolom Teacher menunjukkan tidak ada guru ("-" di list / "No teacher has been assigned" di detail). Kondisinya konsisten di seluruh subject tahun ajaran baru, bukan satu-dua baris yang lupa di-assign manual.

Dua sumber data yang tidak sinkron menyebabkan ini:

1. **Backend — rollover membuat SubjectYear tanpa guru.** `ensureSubjectYearsExist` (`transfer-student-batch.service.ts:556-707`) membuat SubjectYear 26/27 baru dengan `teachers = [superAdmin ID 206]` atau `[]` bila ID 206 tidak ada (`:631-669`), tanpa menyalin guru dari SubjectYear tahun sebelumnya. Fungsi koreksi `assignTeachersToSubjectYears` (`:930-989`) hanya meniru guru dari SubjectYear lain **dalam classYear target yang sama + kategori sama** (`:949-974`) — kalau classYear 26/27 masih kosong (kasus umum Sec/JC yang subject-nya banyak), map kosong → warn `:981` dan SubjectYear dibiarkan tanpa guru.
2. **Frontend — teacher dan student dibaca dari 2 API independen.** Kolom teacher dari `GET /subjectYears` (`teachersIds`), sedangkan teks "Students assigned to" dari `GET /bandings?subjectYearId=` (`banding.teacherId`). Banding/student tetap tercipta via jalur `currentClassYearId` + `createBandingRecords` (`:870-920`, fallback superadmin 206) walau `teachers=[]` — jadi student terlihat assigned sementara teacher kosong.

**Ekspektasi:** SubjectYear 26/27 membawa teacher assignment yang benar (carry-over dari AY sebelumnya atau hasil assign yang sinkron dengan banding), sehingga kolom teacher terisi dan konsisten dengan blok "Students assigned to".

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | (diisi QA — temuan dari analisa code, bukan rekaman Jam) |
| Portal URL | Admin Portal → Subject in Years, filter Academic Year 26/27 |
| Data konteks | AY 26/27, level Secondary & Junior College (semua subject); Primary tidak / kurang terdampak |
| Browser / OS | - |

---

## Steps to Reproduce

1. Login ke Admin Portal.
2. Buka menu **Subject in Years** → filter **Academic Year 26/27**, level Secondary / Junior College.
3. Perhatikan kolom **Subject Teachers** di list → tampil `-`.
4. Buka salah satu SubjectYear → blok banding menampilkan judul **"Students assigned to {nama}"** tetapi bagian teacher menampilkan `<i>No teacher has been assigned</i>`.

**Actual Result:**
- `subjectYear.teachersIds = []` → list tampil `-`, detail tampil "No teacher has been assigned" (`SubjectInYearDetails.jsx:586`).
- Banding tetap ada isinya (`banding.teacherId` + `studentsIds` terisi) → judul "Students assigned to X" tetap tampil (`SubjectInYearDetails.jsx:175`, `BandingForm.jsx:123`).

**Expected Result:**
- Kolom teacher terisi guru yang benar untuk setiap SubjectYear 26/27.
- Guru di kolom teacher konsisten dengan nama pada blok "Students assigned to".

---

## Root Cause Analysis

### Bug #1 — SubjectYear 26/27 dibuat tanpa carry-over guru — `transfer-student-batch.service.ts:556-707`

```typescript
// api_nest/src/modules/class-year/transfer-student-batch.service.ts:631-669
const superAdminEmployee = await transactionManager.findOne(Employee, {
  where: { id: 206 },
});

if (!superAdminEmployee) {
  console.warn(
    'Super admin employee (ID 206) not found, creating subject years without teacher assignment',  // ← :637
  );
}
...
const subjectYear = new SubjectYear();
subjectYear.classYearId = targetClassYear.id;
subjectYear.subjectId = subject.id;
subjectYear.subject = subject;
subjectYear.lockGradebook = true;
subjectYear.teachers = superAdminEmployee ? [superAdminEmployee] : [];  // ← :669: TIDAK copy dari SY sumber
```

Guru tahun sebelumnya tidak pernah dibaca/di-copy — satu-satunya sumber adalah hardcoded superadmin ID 206. Bila ID 206 tidak ada di DB (atau nantinya diganti user asli), SubjectYear tercipta dengan `teachers = []` (join table `subject_years_teachers_employee` kosong).

### Bug #2 — Fungsi koreksi guru hanya meniru dalam classYear yang sama — `transfer-student-batch.service.ts:930-989`

```typescript
// api_nest/src/modules/class-year/transfer-student-batch.service.ts:944-983
const existingSubjectYears = await transactionManager.find(SubjectYear, {
  where: { classYearId: targetClassYear.id },   // ← hanya classYear TARGET (26/27), bukan sumber
  relations: { teachers: true, subject: true },
});

const categoryTeacherMap = new Map<string, any[]>();
existingSubjectYears.forEach((sy) => {
  if (sy.subject?.category && sy.teachers?.length > 0) {  // ← butuh SY lain yg SUDAH berguru
    ...
  }
});
...
} else {
  console.warn(
    `No teachers found for subject category ${subjectYear.subject.category} in class year ${targetClassYear.id}`,  // ← :981
  );
}
```

Dipanggil dari `:1460`. Pada rollover massal Sec/JC (banyak subject dibuat sekaligus ke classYear 26/27 yang masih kosong), tidak ada SubjectYear berguru untuk ditiru → semua dibiarkan tanpa guru. Inilah kenapa Secondary & JC (subject banyak + banding per subject) kena paling parah, sementara Primary (subject sedikit / sudah di-assign manual) lolos.

### Bug #3 (pendukung, bukan penyebab utama) — Banding tetap tercipta walau `teachers=[]` — `transfer-student-batch.service.ts:870-920`

`createBandingRecords` memakai fallback `subjectYear.teachers?.[0]?.id || superAdmin(206)`, sehingga `banding.teacherId` + `studentsIds` tetap terisi. Akibatnya frontend menampilkan "Students assigned to X" dari data banding, padahal `subjectYear.teachersIds` kosong.

### Konteks frontend — dua API independen, bukan bug display

- Teacher: `GET /subjectYears` (`client/src/actions/api/subjectYear.js:16`, `relationsObj: {teachers: true}`, `SubjectInYears.jsx:144-173`) → `subjectYear.teachersIds` via `resourceMapper("employees", ...)` (`:234-243`); detail via `getSubjectYear(..., {relationsObj:{teachers:true}})` (`SubjectInYearDetails.jsx:312-357`, `:103-107, :382-385`).
- Student: `GET /bandings?subjectYearId=...` (`client/src/actions/api/banding.js:8`, `SubjectInYearDetails.jsx:359-366`); judul dari `banding.teacherId` (`:175-188`), bukan dari `subjectYear.teachersIds`.
- Logika `teachersWithoutBandings` (`:425-434`) bahkan tetap merender banding untuk teacher yang tidak ada di `teachersIds` — mempertegas kedua sumber memang tidak disinkronkan.
- Tidak ada hardcode 26/27 atau mapping khusus Sec/JC di frontend (`SubjectInYears.jsx:29-39` hanya default filter `getCurrentAcademicYear`).

Entity: `SubjectYear.teachers` = ManyToMany ke `Employee` via `subject_years_teachers_employee` (`subject-year.entity.ts`); murid tidak ada relasi langsung — dibaca via `classYear.students` / `bandings.students`. Endpoint list: `GET /v1/subjectYears` → `subject-year.controller.ts:40-54` → `subject-year.service.ts:245-264` (`findAll`, relations `teachers:true`).

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/class-year/transfer-student-batch.service.ts:556-707` (`ensureSubjectYearsExist`, khususnya `:631-669`) | SY 26/27 dibuat tanpa carry-over guru |
| Backend Service | `api_nest/src/modules/class-year/transfer-student-batch.service.ts:930-989` (`assignTeachersToSubjectYears`, khususnya `:944-983`, dipanggil `:1460`) | Koreksi guru gagal saat classYear target masih kosong |
| Backend Service | `api_nest/src/modules/class-year/transfer-student-batch.service.ts:870-920` (`createBandingRecords`) | Banding tetap dibuat walau teachers kosong → data tidak sinkron |
| Backend Entity | `api_nest/src/modules/subject-year/entities/subject-year.entity.ts` (`teachers` ManyToMany via `subject_years_teachers_employee`) | Join table kosong untuk AY 26/27 = sumber "not assigned" |
| Backend API | `api_nest/src/modules/subject-year/subject-year.controller.ts:40-54`, `subject-year.service.ts:245-264` | `GET /v1/subjectYears` mengembalikan `teachers: []` |
| Frontend List | `bbs/client/src/views/subjectInYears/SubjectInYears.jsx:144-173, :234-243` | Kolom teacher tampil `-` |
| Frontend Detail | `bbs/client/src/views/subjectInYears/SubjectInYearDetails.jsx:175, :312-385, :425-434, :586` | "No teacher has been assigned" vs "Students assigned to X" |
| Frontend Form | `bbs/client/src/views/subjectInYears/BandingForm.jsx:123` | Judul banding dari `banding.teacherId` |

---

## Proposed Solution Options

### Option A: Carry-over guru dari SubjectYear sumber saat rollover (Recommended — permanent fix)

**Backend** — di `ensureSubjectYearsExist`, saat membuat SubjectYear baru untuk classYear target, cari SubjectYear sumber (AY sebelumnya, `subjectId` sama atau `classYear` asal siswa) beserta `teachers`-nya, lalu copy:

```typescript
// sebelum :664-669 — load SY sumber + teachers
const sourceSY = await transactionManager.findOne(SubjectYear, {
  where: { classYearId: sourceClassYearId, subjectId: subject.id },
  relations: { teachers: true },
});
subjectYear.teachers = sourceSY?.teachers?.length
  ? sourceSY.teachers
  : (superAdminEmployee ? [superAdminEmployee] : []);
```

Kelebihan: memperbaiki akar masalah — semua rollover berikutnya otomatis membawa guru. Kekurangan: perlu menentukan `sourceClassYearId` yang benar per siswa/kelas (parameter sudah tersedia di alur transfer).

### Option B: Data fix sekali jalan — backfill join table 26/27 (cepat, tanpa ubah kode)

Isi `subject_years_teachers_employee` untuk AY 26/27 dengan menyalin `teachers` dari SY AY 25/26 yang `subjectId` + level/grade-nya sama (via SQL/API admin). Verifikasi: `GET /v1/subjectYears?classYear_academicYearId=<id 26/27>` harus mengembalikan `teachers` non-kosong.

Kelebihan: langsung memulihkan tampilan tanpa deploy. Kekurangan: tidak mencegah terulang di rollover berikutnya; tetap butuh Option A/C sebagai tindak lanjut.

### Option C: Sinkronisasi tampilan — fallback ke guru banding (ringan, frontend atau read-model)

Bila `subjectYear.teachersIds` kosong tetapi banding punya `teacherId`, tampilkan guru dari banding (atau auto-backfill `teachersIds` saat baca). Bisa di `SubjectInYearDetails.jsx:382-385` / resolver `teachersIds`.

Kelebihan: murah, menutup inkonsistensi tampilan. Kekurangan: menyembunyikan masalah data — join table tetap kosong dan fitur lain yang membaca `subjectYear.teachers` langsung tetap "not assigned".

---

## Notes

- Bukan bug filter level: tidak ada perbedaan logika Primary vs Secondary vs JC di frontend maupun endpoint — Sec/JC kena lebih parah murni karena volume subject + banding per subject lebih besar sehingga classYear 26/27 kosong saat koreksi guru berjalan.
- Query deteksi cepat (read-only): cari SubjectYear AY 26/27 yang tidak punya baris di `subject_years_teachers_employee` tetapi punya banding berisi siswa.
- Tidak ada perubahan kode di `api_nest` / `bbs` pada pembuatan tiket ini (dokumentasi saja).
