---
title: CCA Grading — final_score & grade terlihat di UI tapi tidak tersimpan ke DB (halaman/term 3)
status: open
severity: major
product: Smartbag (BBS)
portal: Teacher (client-teacher)
author: System Analyst
date: 2026-09-17
jam: https://jam.dev/c/b492aaaf-d766-4301-9b07-912af45d87ab
---

# CCA Grading — final_score & grade tampil di UI tapi tidak persist ke DB

## Summary

Di halaman **CCA Grading** portal Teacher (`/ccaYear/:ccaYearId/:academicYearId`, contoh Jam: `ccaYear/100306/27` — CCA "Beading Fun", AY 2025–2026), kolom **Final Score** dan **Grade** sudah terlihat ada nilainya di UI (terutama di tab/term ke-3), tetapi saat user klik **Submit Changes**, nilai yang tersimpan ke DB **hanya `attendance` + `activeParticipation`** — `finalScore`/`grade` tidak ikut terkirim dan hasil hitung backend tidak kembali ke UI tanpa edit ulang. Akibatnya user harus **mengubah nilai, mengisi lagi**, baru nilai benar-benar tersimpan.

**Ekspektasi:** nilai Final Score & Grade yang terlihat di UI harus sama dengan yang tersimpan di DB setelah save, tanpa perlu edit ulang.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Mahdy Arief |
| Email (Jam account) | versaproject001@gmail.com |
| Portal URL | https://teacher.smartbag.binabangsaschool.com/ccaYear/100306/27 |
| Environment API | api.binabangsaschool.dev (dev) |
| Data konteks | ccaYearId=100306, academicYearId=27, CCA "Beading Fun", Term 1–4 |
| Browser / OS | Chrome-Headless 140 / Linux x86_64 |

---

## Steps to Reproduce

1. Login ke portal Teacher, buka `/ccaYear/100306/27`.
2. Pindah ke tab Term 3 (halaman ketiga). Isi Attendance & Active Participation beberapa siswa.
3. Perhatikan kolom Final Score & Grade **sudah terisi di UI**.
4. Klik **Submit Changes** (klik terkonfirmasi di Jam @8961ms, toast "Successfully submit data").
5. Refresh halaman / cek DB `student_cca_grade` — `attendance` & `activeParticipation` tersimpan, tapi `finalScore`/`grade` **tidak sesuai dengan yang tampil di UI** (stale/kosong). User harus mengubah nilai dan submit ulang agar benar.

**Actual Result:** Final Score & Grade di UI ≠ DB setelah save pertama.

**Expected Result:** sekali Submit, DB berisi `attendance`, `activeParticipation`, `finalScore`, `grade` yang konsisten dengan tampilan UI.

---

## Root Cause Analysis

### Bug #1 — Frontend mengirim payload TANPA `finalScore`/`grade` (kontrak API memang begitu) — `CCADetail.jsx:299-309`

`handleSubmit` membangun payload bulk hanya dengan `attendance` + `activeParticipation`:

```js
// bbs/client-teacher/src/views/cca/CCADetail.jsx:299-309
payload.push({
  id: data?.studentCcaId,
  term: data?.term,
  ccaYearId: ccaYear?.id?.toString(),
  studentId: Number(data?.id ?? "0"),
  classYearId: data?.targetClassYearId ?? "0",
  attendance,
  activeParticipation
  // finalScore: parseInt(gradeResult.finalScore),  // ← COMMENTED OUT
  // grade: gradeResult.grade                        // ← COMMENTED OUT
});
```

Bukti Jam: body `POST /api/v1/studentCcaGrades/bulk` (201, @15758ms & @16567ms) hanya berisi `{"id","term","ccaYearId","studentId","classYearId":"0","attendance":10,"activeParticipation":10}` — tidak ada `finalScore`/`grade`. DTO backend (`create-student-cca-grade.dto.ts:44-52`) juga meng-comment-out kedua field tersebut, jadi penghitungan **sepenuhnya didelegasikan ke backend** (`ccaGrading()`).

### Bug #2 — Kolom Final Score & Grade di tabel me-render nilai STALE dari DB, bukan hasil hitungan berjalan — `CCADetail.jsx:539-568`

```jsx
// finalScore slot (539-553) & grade slot (554-568) — render langsung dari form state
// yang diinisialisasi dari DB, TIDAK dihitung ulang saat attendance berubah:
value={studentCcaFields?.[index]?.finalScore || ""}
value={studentCcaFields?.[index]?.grade || ""}
```

Berbeda dengan kolom `score` (term 2/4, lines 508-538) yang memanggil `ccaFormulation(...)` secara live dari `gradeWatch(...)`. Akibatnya: user melihat angka lama dari DB (atau kosong), mengira itu hasil inputnya, padahal itu bukan yang akan tersimpan. Inilah "nilai cuma di UI aja" — **angka yang dilihat user bukan angka yang dihitung dari input saat ini**.

`handleChangeGrade` (234-267) juga hanya meng-update field yang diubah (`[name]: value`), tanpa me-recompute `finalScore`/`grade` di form state.

### Bug #3 — Backend `ccaGrading()` bisa mengembalikan `finalScore: 0, grade: null` secara diam-diam — `cca-grading.ts:28-40`

```ts
if (!studentReportId || !ccaYear) return result;   // → { finalScore: 0, grade: null }
...
if (!ccaGradeSetting) return result;               // → setting tidak ketemu = 0/null
```

Pemicu di Jam ini:
- Payload mengirim `classYearId: "0"` untuk semua row (terlihat di request body). `create()` mencari `ClassYear` id 0 → tidak ada → `NoClassYearFoundError()` **dipanggil tanpa `throw`** (`student-cca-grade.service.ts:64`), sehingga eksekusi lanjut dengan `classYear` null dan lookup/buat `StudentReport` berjalan di atas relasi yang salah.
- `ccaGradeSetting` dipilih dengan `.find(cgs => cgs.isDefault === true || cgs.academicYearId === ccaYear.academicYearId)` — kondisi OR ini bisa memilih setting yang salah untuk AY berjalan, sehingga `termSetting` untuk term 3 tidak ketemu → threshold `gradeAMin..DMin` undefined → grade jatuh ke `null`.
- Hasil `0/null` ini tetap di-`save()` ke DB tanpa error (POST tetap 201), jadi dari sisi user "save sukses" tapi nilai rusak.

### Kenapa terasa di "halaman ketiga" (Term 3)

Term 1 & 3 memakai rumus `finalScore = attendance + activeParticipation` (tanpa histori), sedangkan Term 2 & 4 memakai rata-rata dengan term sebelumnya + kolom `score` live. Di Term 3 tidak ada kolom `score` live sebagai pembanding, dan `finalScore`/`grade` yang tampil murni nilai stale DB — sehingga selisih UI vs DB paling jelas terlihat di tab ini. Setelah user "ganti nilai, isi lagi", row sudah punya `studentCcaId` + `targetClassYearId` yang benar sehingga `update()` menemukan relasi yang tepat dan nilai tersimpan — itulah kenapa submit kedua "menyembuhkan" baris tersebut.

---

## Bukti dari Jam (https://jam.dev/c/b492aaaf-d766-4301-9b07-912af45d87ab)

| Sumber | Temuan |
|--------|--------|
| **Video** | User mengisi Attendance/Active Participation Term 1–4; Term 1 tampil Final 10, Term 2 tampil Score 5 + Grade D; klik Submit Changes @8961ms → toast sukses |
| **Network** | 2× `POST /api/v1/studentCcaGrades/bulk` → 201 (@15758ms, @16567ms). Request body **tanpa** `finalScore`/`grade`, `classYearId:"0"` di semua row |
| **Console** | 0 error — save terlihat sukses, bug bersifat silent |
| **User events** | Navigasi `ccaYear/100306/27`, puluhan `change` event di Term 1–4, 1× klik Submit |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend | `bbs/client-teacher/src/views/cca/CCADetail.jsx` (handleSubmit 280-310, slot finalScore 539-553, grade 554-568, handleChangeGrade 234-267, init 150-179) | Payload tanpa finalScore/grade; kolom tampilkan stale DB; tidak ada recompute saat edit |
| Frontend | `bbs/client-teacher/src/utils/utilFunctions.js:1304-1352` (`ccaFormulation`) | Rumus UI benar, tapi tidak dipakai untuk slot finalScore/grade |
| Backend | `api_nest/src/modules/student-cca-grade/student-cca-grade.service.ts` (createUpdateBulk 27-44, create 46-151, update 225-283) | Guard `NoXxxFoundError()` tanpa throw; `classYearId:"0"` lolos; hasil grading 0/null tetap disave |
| Backend | `api_nest/src/helpers/grading/cca-grading.ts:16-90` | Silent fallback 0/null bila setting/studentReport tidak ketemu; selector setting pakai OR |
| Backend DTO | `api_nest/src/modules/student-cca-grade/dto/create-student-cca-grade.dto.ts:44-52` | `finalScore`/`grade` di-comment-out (kontrak: dihitung server) |

---

## Notes

- **JANGAN benerin code** (instruksi task) — file ini hanya laporan + panduan solve terpisah di `how-to-solve.md`.
- Environment Jam adalah **dev** (`api.binabangsaschool.dev`, `teacher.smartbag...com`); verifikasi ulang di staging sebelum rilis fix.
