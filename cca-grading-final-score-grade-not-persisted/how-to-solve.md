---
title: How to Solve — CCA Grading final_score & grade tidak persist ke DB
status: draft
owner: engineering
reviewed-by: backend, frontend
date: 2026-09-17
source-jam: https://jam.dev/c/b492aaaf-d766-4301-9b07-912af45d87ab
severity: major
related: bug-report.md (folder yang sama)
---

# How to Solve: CCA Grading — final_score & grade tampil di UI tapi tidak tersimpan ke DB

> ⚠️ Ini panduan implementasi, BUKAN code fix. Kerjakan sesuai urutan langkah di bawah.
> Jangan ubah kontrak API di luar yang disebutkan. Jangan refactor file lain.

## 🐛 Ringkasan Bug (baca `bug-report.md` untuk detail penuh)

1. **Payload tanpa hasil hitungan:** `handleSubmit` (`CCADetail.jsx:299-309`) hanya kirim `attendance` + `activeParticipation`; `finalScore`/`grade` di-comment-out. Perhitungan 100% di backend.
2. **UI tampilkan angka basi:** slot `finalScore` (539-553) & `grade` (554-568) render nilai DB lama, bukan hasil hitungan live — user tertipu mengira nilai sudah benar.
3. **Backend gagal diam-diam:** `ccaGrading()` return `{0, null}` bila setting/studentReport tak ketemu; guard `NoXxxFoundError()` dipanggil **tanpa `throw`**; `classYearId:"0"` lolos; hasil rusak tetap di-save dengan status 201.

## 🔍 Akar Masalah per Layer

### Frontend — `bbs/client-teacher/src/views/cca/CCADetail.jsx`
- `handleSubmit` (~280-310): payload tanpa `finalScore`/`grade` (commented-out 307-308).
- `handleChangeGrade` (234-267): hanya update field yang diubah, tidak recompute `finalScore`/`grade`.
- Slot `finalScore`/`grade` (539-568): baca `studentCcaFields[index]` (stale DB), tidak panggil `ccaFormulation` live seperti slot `score` (508-538).
- Init form (150-179): seed `finalScore`/`grade` dari DB apa adanya.

### Backend — `api_nest/src/modules/student-cca-grade/student-cca-grade.service.ts`
- `createUpdateBulk` (27-44): loop tanpa transaksi; return `[]` sehingga response bulk kosong.
- `create` (46-151): `if (!ccaYear) NoCcaYearFoundError();` (line 53), `if (!teacher)...` (58), `if (!classYear)...` (64) — semua **tanpa `throw`**, eksekusi lanjut dengan entity null.
- `update` (225-283): pola guard yang sama (233, 236, 247).
- `classYearId: "0"` dari frontend → `ClassYear.findOne({id: 0})` → null → lookup/buat `StudentReport` di atas relasi salah.

### Backend — `api_nest/src/helpers/grading/cca-grading.ts`
- Early return `{ finalScore: 0, grade: null }` bila `!studentReportId || !ccaYear` (28) atau `!ccaGradeSetting` (40) — tanpa log/error.
- Selector setting pakai OR (35-38): `cgs.isDefault === true || cgs.academicYearId === ccaYear.academicYearId` — bisa pilih setting AY yang salah.
- `termSetting` undefined → threshold `gradeAMin..DMin` undefined → semua perbandingan `>=` false → grade `null`.

---

## ✅ Yang Harus Dikerjakan

### 1. Frontend — tampilkan hasil hitungan LIVE di kolom Final Score & Grade (wajib)

Di `CCADetail.jsx`, ubah slot `finalScore` (539-553) dan `grade` (554-568) agar menghitung ulang dari nilai form saat ini memakai `ccaFormulation`, meniru pola slot `score` (508-538):

```jsx
// pola yang sudah benar di slot `score` — tiru untuk finalScore & grade:
gradeResult = ccaFormulation(
  parseInt(gradeWatch(`studentCcas.${index}.attendance`) || "0"),
  parseInt(gradeWatch(`studentCcas.${index}.activeParticipation`) || "0"),
  0,
  ccaGradeTermSettings,
  parseInt(term)
);
// lalu render gradeResult.finalScore / gradeResult.grade
```

Constraints:
- Pakai `gradeWatch` (nilai form live), BUKAN `studentCcaFields[index]` (stale DB).
- Untuk `previousScore` (param ke-3): Term 1 & 3 → `0`; Term 2 & 4 → ambil finalScore term sebelumnya dari data yang sudah di-load (lihat fungsi `findPreviousStudentCcaGrade` yang saat ini di-comment-out di ~283-297 — hidupkan kembali bila perlu, jangan tulis ulang logika dari nol).
- Kolom tetap `disabled` (display-only); tidak ada perubahan UX selain angkanya kini benar.
- JANGAN uncomment pengiriman `finalScore`/`grade` di payload (307-308) — kontrak "dihitung server" tetap; UI hanya display.

### 2. Frontend — validasi payload sebelum kirim (wajib)

Di `handleSubmit` (~280-310):
- Skip row bila `attendance`/`activeParticipation` `NaN` (contoh: `parseInt("")` → NaN terkirim hari ini).
- Pastikan `classYearId` tidak `"0"`: ambil dari `student.currentClassYearId` / `foundStudentReport` yang valid; bila tidak ada, skip row + beri toast per-baris agar user tahu baris mana yang gagal.
- Setelah `createOrUpdateBulkStudentCca` sukses, `refresh()` daftar nilai (sudah ada di line 315 — pertahankan) supaya angka server (finalScore/grade hasil `ccaGrading`) kembali tampil.

### 3. Backend — buat guard benar-benar menghentikan eksekusi (wajib)

Di `student-cca-grade.service.ts`, ubah semua guard tanpa `throw` menjadi throw:

```ts
// create(): lines 53, 58, 64 — update(): lines 233, 236, 247
if (!ccaYear) throw NoCcaYearFoundError();
if (!teacher) throw NoTeacherFoundError();
if (!classYear) throw NoClassYearFoundError();
if (!studentCcaGrade) throw NoStudentCcaGradeFoundError();
```

Cek dulu signature `NoXxxFoundError` di `src/errors/ResourceError` — bila ia me-return (bukan throw) HttpException, maka `throw` di depan adalah fix yang tepat. Bila ia sudah throw sendiri, cukup pastikan tidak ada kode yang menelannya.

Constraints:
- JANGAN ubah DTO (`create-student-cca-grade.dto.ts`) — field `finalScore`/`grade` tetap commented-out.
- `createUpdateBulk` tetap loop per-row; opsional: bungkus dalam transaksi bila pola transaksi sudah ada di codebase, jika belum — biarkan loop, jangan perkenalkan infrastruktur transaksi baru.

### 4. Backend — perketat selector `ccaGradeSetting` + tangani threshold hilang (wajib)

Di `cca-grading.ts:35-38`, ganti OR dengan prioritas eksplisit:

```ts
const ccaGradeSetting =
  ccaGradeSettings.find((cgs) => cgs.academicYearId === ccaYear.academicYearId)
  ?? ccaGradeSettings.find((cgs) => cgs.isDefault === true);
```

Dan setelah `termSetting` diambil (line 71): bila `termSetting` undefined → return `{ finalScore, grade: null }` SECARA EKSPLISIT dengan log warn (bukan jatuh diam-diam ke bawah), supaya Term tanpa konfigurasi tidak dikira "nilai 0".

### 5. Backend — tolak `classYearId` invalid lebih awal (recommended)

Di `create()` sebelum query: bila `Number(options.classYearId)` falsy/0 → throw `NoClassYearFoundError()` langsung (hemat 1 query + cegah StudentReport salah relasi). Berlaku juga untuk `update()` bila ia memakai `classYearId`.

---

## ❌ Yang TIDAK Boleh Diubah

| Area | Constraint |
|------|------------|
| **DTO** | `CreateStudentCcaGradeDto.finalScore`/`grade` tetap commented-out — server satu-satunya penulis kedua kolom. |
| **API contract** | `POST /api/v1/studentCcaGrades/bulk` request/response shape tidak berubah. |
| **Rumus grading** | `ccaFormulation` (frontend) dan rumus `ccaGrading` Term 1/3 vs 2/4 tidak berubah — hanya plumbing-nya yang diperbaiki. |
| **File lain** | Jangan sentuh modul LEAPS/report lain, jangan sentuh `cca-registration/*`, `class-year.v2.service.ts`, `leaps-event.service.ts`. |
| **Code di smartbag** | Task ini hanya menulis di `features/` — JANGAN edit file di `smartbag/`. |

## 🧩 Yang Diasumsikan Tersedia

| Artifact | Lokasi / Catatan |
|----------|------------------|
| Rumus UI benar | `bbs/client-teacher/src/utils/utilFunctions.js:1304-1352` (`ccaFormulation`) — pakai ulang, jangan duplikasi. |
| Pola live-compute | Slot `score` di `CCADetail.jsx:508-538` — tiru untuk `finalScore`/`grade`. |
| Helper cari term sebelumnya | `findPreviousStudentCcaGrade` (commented-out, ~283-297 & ~200-232) — hidupkan bila perlu. |
| Error factory | `src/errors/ResourceError` (`NoCcaYearFoundError` dkk) — cek apakah return atau throw. |
| Bulk endpoint | `POST /api/v1/studentCcaGrades/bulk` → `createUpdateBulk` (27-44). |

## 🚨 Non-Negotiable Constraints

1. **Angka di UI harus bisa dipercaya** — setelah fix, nilai Final Score/Grade yang terlihat = hasil `ccaFormulation` dari input saat ini, bukan sisa DB.
2. **Save gagal harus berisik** — tidak ada lagi `NoXxxFoundError()` tanpa throw; tidak ada lagi save `0/null` diam-diam dengan status 201.
3. **Satu submit cukup** — user tidak boleh diminta "ganti nilai, isi lagi" untuk mendapatkan nilai benar.
4. **Paritas rumus** — Term 1/3 = `attendance + activeParticipation`; Term 2/4 = rata-rata kumulatif; tidak berubah.

## 📋 Verification Checklist (For PR Review)

- [ ] **Repro (pre-fix):** buka `/ccaYear/100306/27` Term 3, isi Attendance/Active Participation → Final Score/Grade tampil; Submit → refresh → DB `student_cca_grade` berisi `finalScore` 0 / `grade` null (atau tidak sesuai UI).
- [ ] **Frontend live:** ubah Attendance di Term 3 → kolom Final Score langsung berubah (= attendance + activeParticipation); Grade ikut berubah sesuai threshold term setting.
- [ ] **Satu submit:** Submit sekali → refresh → DB `finalScore`/`grade` sama dengan yang tampil di UI.
- [ ] **Payload:** `POST .../bulk` body tetap tanpa `finalScore`/`grade`, `classYearId` tidak pernah `"0"`.
- [ ] **Backend guard:** `POST .../bulk` dengan `classYearId: "0"` → error 4xx (bukan 201 dengan data rusak).
- [ ] **Term tanpa setting:** Term yang belum punya `ccaGradeTermSettings` → tersimpan dengan `grade: null` + ada log warn (bukan crash, bukan angka ngawur).
- [ ] **Regresi Term 2/4:** kolom `score` live + rata-rata kumulatif tidak berubah.
- [ ] **Tidak ada diff** di `smartbag/` pada task ini (laporan saja); diff `features/` hanya 2 file baru.

## Related Context

- Jam: https://jam.dev/c/b492aaaf-d766-4301-9b07-912af45d87ab
- Bug report: `bug-report.md` (folder yang sama)
- Frontend: `bbs/client-teacher/src/views/cca/CCADetail.jsx` (handleSubmit 280-310, slot 508-568, handleChangeGrade 234-267, init 150-179)
- Rumus UI: `bbs/client-teacher/src/utils/utilFunctions.js:1304-1352`
- Backend service: `api_nest/src/modules/student-cca-grade/student-cca-grade.service.ts` (bulk 27-44, create 46-151, update 225-283)
- Grading helper: `api_nest/src/helpers/grading/cca-grading.ts:16-90`
- DTO: `api_nest/src/modules/student-cca-grade/dto/create-student-cca-grade.dto.ts:44-52`
