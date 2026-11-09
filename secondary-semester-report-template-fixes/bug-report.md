---
title: Secondary Semester Report — Cohort tidak muncul, persentase CA1 salah, dan logo masih tampil di print
status: open
severity: major
product: BBS LMS
portal: Admin | Teacher | Student
author: System Analyst
date: 2026-11-05
jam:
---

# Secondary Semester Report — Cohort hilang, persentase CA1 salah, logo muncul di print

## Summary

Template laporan semester Secondary (`REPORT_SEMESTER` untuk Secondary & Junior College) punya tiga cacat terpisah yang semuanya muncul di e-report maupun print report:

1. **Bug #1 — Cohort tidak muncul.** Header laporan menampilkan `Name / Class / Student's ID` saja. Field Cohort (level siswa, mis. "Secondary 2") tidak pernah dibangun di payload header, tidak diterima sebagai parameter oleh `headerSecondary`, dan tidak ada di template `.hbs`.
2. **Bug #2 — Persentase CA1 salah.** Kolom Term 1 / Term 2 di template semester 1 diisi dari `ca1Pct / 2`. Dengan bobot default `ca1 = 35`, hasilnya `17.5` hanya bila bobot efektifnya 35; jika bobot yang tersimpan berbeda, header ikut berubah dan tidak sesuai kontrak 17,5. Nilai juga dirender dengan titik desimal (`17.5`), bukan koma (`17,5`) seperti brief.
3. **Bug #3 — Logo masih ada di print report.** Untuk Secondary, `withLogo` dihitung `!useSemestralLetterhead`. Karena `useSemestralLetterhead` hanya true untuk e-report (`isSemestralRapot && !isPrint`), print report Secondary selalu merender logo inline — bertentangan dengan aturan "print report Secondary tanpa logo".

**Ekspektasi:** header Secondary menampilkan Cohort, kolom CA1 semester 1 tertulis `Term 1 (17,5%)` dan `Term 2 (17,5%)`, dan print report Secondary tidak menampilkan logo.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | _(diisi QA)_ |
| Email (Jam account) | _(diisi QA)_ |
| Jam author ID | _(diisi QA)_ |
| User ID (dari console log, mis. `selfUser`) | _(diisi QA)_ |
| Portal URL | Admin `/student-reports`, Teacher `/classrooms/:id/reports`, Student `/report` |
| Environment API | api_nest (current) |
| Data konteks (class ID / daId / tanggal) | Siswa level Secondary (ML 7–17), kelas dengan `classYear.level` berawalan `SECONDARY`/`JUNIOR_COLLEGE`, term 2 (semester 1) |
| Browser / OS | _(diisi QA)_ |

---

## Steps to Reproduce

1. Login sebagai Admin, buka Student Reports untuk siswa Secondary (mis. Secondary 2).
2. Generate report tipe `REPORT_SEMESTER` untuk Term 2 (semester 1).
3. Buka report di viewer (e-report), lalu tekan Print.
4. Perhatikan header (identitas siswa), header tabel kolom CA1, dan apakah logo BBS tampil.

**Actual Result:**
- Header hanya berisi `Name`, `Class`, `Student's ID`. Tidak ada Cohort.
- Header tabel berbunyi `Term 1 (17.5%)` / `Term 2 (17.5%)` (atau nilai lain yang mengikuti bobot CA1 tersimpan), dengan pemisah desimal titik.
- Print report Secondary menampilkan logo BBS inline di kiri atas.

**Expected Result:**
- Header memuat Cohort (level siswa) di samping Class.
- Kolom CA1 tertulis persentase `17,5` (format koma).
- Print report Secondary tanpa logo.

---

## Root Cause Analysis

### Bug #1 — Cohort tidak pernah dibangun di payload header — `pdf-report-headers.jsx` (line 44-76)

`generateHeaderPayload` hanya mengisi `reportDate`, `studentName`, `studentId`, `classroom`, `ageDate`, `studentAgeOnAgeDate`, `campusProgramme`, `term`. Tidak ada field cohort/level.

```jsx
// smartbag/bbs/client/src/utils/pdf-generator/pdf-report-headers.jsx:62-73
const headerPayload = {
  reportDate,
  studentName: student?.fullName,
  studentId: student?.studentId,
  classroom: classroom?.name || "",
  ageDate: age.ageDate,
  studentAgeOnAgeDate: age.studentAgeOnAgeDate,
  campusProgramme: programme?.name || "",
  term: isSemestral
    ? `Semester ${term / 2} ${withAyear ? "AY " + academicYear?.year : ""}`
    : `Term ${term} ${withAyear ? "AY " + academicYear?.year : ""}`
};
```

`headerSecondary` juga tidak punya parameter cohort, dan baris identitasnya hard-coded tiga field:

```jsx
// smartbag/bbs/client/src/utils/pdf-generator/pdf-report-headers.jsx:349 & 412-447
const idLineText = `Name: ${studentName || ""} Class: ${classroom || ""} Student's ID: ${studentId || ""}`;   // line 351
// ...
// STUDENT IDENTITY - single row: Name + Class clustered at the left
// border, Student's ID pushed to the right border.
{ margin: [0, 18, 0, 0], columns: [ /* Name */, /* Class */, /* spacer */, /* Student's ID */ ] }
```

Template `.hbs` juga tidak punya placeholder cohort:

```hbs
{{! smartbag/api_nest/src/templates/student-reports/secondary-semester1-report.hbs }}
{{! blok identitas hanya berisi {{studentName}} / {{classroom}} / {{studentId}} }}
```

**Faktor pendukung:** datanya sudah tersedia di komponen pemanggil. `classYear?.level` (mis. `SECONDARY2`, `JUNIOR_COLLEGE1`) ada di `ReportList.jsx:319-321`, dan `masterLevelId`/`masterLevel` sudah dipakai backend (`secondary-semester-report.helper.ts:32`, `campus-and-programme.helper.ts`). Jadi ini murni rantai yang tidak tersambung: payload → parameter header → render. Kode `Cohort` tidak ada sama sekali di tiga salinan `pdf-report-headers.*` (admin, teacher, student).

### Bug #2 — Persentase CA1 dihitung ulang dari bobot, bukan kontrak 17,5 — `secondary-semester-report.helper.ts` (line 51-54, 578)

```ts
// smartbag/api_nest/src/helpers/reports/secondary-semester-report.helper.ts:51-54
const ca1Pct = defaultWeight?.ca1 ?? 35;
const myePct = defaultWeight?.mye ?? 0;
const ca2Pct = defaultWeight?.ca2 ?? 25;
const fyePct = defaultWeight?.fye ?? 40;
```

```ts
// smartbag/api_nest/src/helpers/reports/secondary-semester-report.helper.ts:574-585 (lihat 578, 582)
if (semester === 1) {
  if (hasMye) {
    templatePath = '.../secondary-semester1-mye-report.hbs';
    data = { ca1Percentage: ca1Pct, myePercentage: myePct, ... };
  } else {
    const halfCa1 = parseFloat((ca1Pct / 2).toFixed(1));   // 35 / 2 = 17.5
    templatePath = '.../secondary-semester1-report.hbs';
    data = {
      ca1Fields: [`Term 1 (${halfCa1}%)`, `Term 2 (${halfCa1}%)`],
      ...
    };
  }
}
```

Dua masalah:

1. **Nilai bergantung pada bobot tersimpan.** Angka 17,5 di template tidak hard-coded; ia turunan `ca1 / 2`. Selama bobot efektif `ca1 = 35` hasilnya 17.5, tetapi bila admin mengubah bobot atau bila baris bobot per level/subject terbaca lain, header ikut berubah (mis. 25 → 12.5, 50 → 25). Kontrak layar "harus 17,5" tidak dijamin oleh kode.
2. **Pemisah desimal.** `toFixed(1)` menghasilkan `17.5` dengan titik. Brief meminta format `17,5` (koma). Ini juga menyentuh `secondary-semester1-mye-report.hbs` (`CA1 ({{ca1Percentage}}%)`), `secondary-semester2-report.hbs` (`CA1 ({{ca1Percentage}}%)`, `CA2 ({{ca2Percentage}}%)`, `FYE ({{fyePercentage}}%)`) dan `secondary-semester2-mye-report.hbs` (`{{sem1Percentage}}`, `{{ca2Percentage}}`, `{{fyePercentage}}`).

**Faktor pendukung:** terdapat ketidaksesuaian antara default kode (`35/0/25/40`) dan rumus terdokumentasi (`FYA = CA1 20% + MYE 25% + CA2 20% + FYE 35%`, lihat `Panduan_Fitur_Secondary_vs_Primary.md:10,26`). Perlu keputusan bisnis mana yang benar sebelum angka di template dijadikan tetap. Brief lama untuk 17,5 tidak ditemukan di `features/` maupun riwayat git (`git log -S "17.5"` hanya mengenai helper, bukan template), jadi angka 17,5 perlu dikonfirmasi sebagai kontrak tampilan tetap.

### Bug #3 — Print report Secondary tetap merender logo — `ReportList.jsx` (line 328, 396-417)

```jsx
// smartbag/bbs/client/src/views/studentReports/ReportList.jsx:328
const useSemestralLetterhead = isSemestralRapot && !isPrint;

// smartbag/bbs/client/src/views/studentReports/ReportList.jsx:412-416
// E-report uses the letterhead background (which already shows the logo), so suppress the inline logo.
// For the physical print: Primary 3–6 omits the logo entirely; secondary keeps the inline logo.
withLogo: isPrimarySemester ? false : !useSemestralLetterhead
```

`useSemestralLetterhead` bernilai false saat print, sehingga `!useSemestralLetterhead` = true dan logo Secondary tercetak. Komentar di kode bahkan menyatakan aturan lama "secondary keeps the inline logo", yang sekarang berlawanan dengan permintaan.

**Faktor pendukung:** teacher portal sudah benar — `client-teacher/src/views/classrooms/viewer/ClassReportViewer.jsx:985` dan `ClassroomDetailsReports.jsx:454` selalu `withLogo: false`. Admin class report viewer masih salah dengan pola yang sama (`ClassReportViewer.jsx:941`). Student portal memakai varian `!useSemestralLetterhead` dengan `useSemestralLetterhead = isSemestralRapot` (selalu true), sehingga print-nya ikut menampilkan logo (`client-student/src/views/Report/components/ReportList.jsx:318`).

---

## Bukti dari Jam

_(belum ada link Jam)_

| Sumber | Temuan |
|--------|--------|
| **Video (t=...)** | |
| **Network** | |
| **Console** | |
| **User Actions** | |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Helper | `api_nest/src/helpers/reports/secondary-semester-report.helper.ts` (51-54, 574-630) | Nilai persentase CA1/CA2/FYE yang dikirim ke template |
| Backend Template | `api_nest/src/templates/student-reports/secondary-semester{1,2}[-mye]-report.hbs` | Format persentase; tidak ada slot Cohort |
| Frontend Admin | `bbs/client/src/utils/pdf-generator/pdf-report-headers.jsx` (44-76, 327-450) | Cohort tidak ada di payload & header |
| Frontend Teacher | `bbs/client-teacher/src/utils/pdf-generator/pdf-report-headers.js` (15-47, 353-477) | Cohort tidak ada di payload & header |
| Frontend Student | `bbs/client-student/src/utils/pdf-generator/pdf-report-headers.jsx` (44-76, 588-711) | Cohort tidak ada di payload & header |
| Frontend Admin Viewer | `bbs/client/src/views/studentReports/ReportList.jsx` (319-328, 396-417) | Logo muncul di print Secondary |
| Frontend Admin Class | `bbs/client/src/views/classReports/ClassReportViewer.jsx` (927-943) | Logo muncul di print Secondary |
| Frontend Student | `bbs/client-student/src/views/Report/components/ReportList.jsx` (302-319) | Logo muncul di print Secondary |
| Frontend Teacher | `bbs/client-teacher/src/views/classrooms/viewer/ClassReportViewer.jsx` (972-987), `ClassroomDetailsReports.jsx` (441-456) | Sudah `withLogo: false`; hanya perlu tambah Cohort |

---

## Proposed Solution Options

### Option A: Satu sumber Cohort + format tersentral (Recommended)

1. **Cohort** — tambahkan `cohort` ke `generateHeaderPayload` di ketiga salinan header (nilai dari `masterLevels[classYear.masterLevelId].name`, fallback label dari `classYear.level`). Tambahkan parameter `cohort` ke `headerSecondary` di ketiga salinan dan render sebagai kolom keempat di baris identitas (`Name … Class … Cohort … Student's ID`). Sisipkan ke setiap pemanggil: `ReportList.jsx` (student/student-report), `ClassReportViewer.jsx` (admin class + teacher classroom), `ClassroomDetailsReports.jsx`, dan `ReportList.jsx` (student portal). Tidak perlu perubahan `.hbs` karena identitas dirender client-side.
2. **Persentase** — jadikan angka header tetap mengikuti kontrak: kirim `ca1Fields: ["Term 1 (17,5%)", "Term 2 (17,5%)"]` (hard-coded format) atau hitung dari bobot tetapi format dengan koma (`toLocaleString("id-ID")` / `replace(".", ",")`). Konfirmasi dulu apakah bobot efektif memang harus 35, atau default diganti ke 20 agar konsisten dengan panduan.
3. **Logo** — ubah `withLogo` untuk Secondary menjadi `useSemestralLetterhead` (bukan `!useSemestralLetterhead`) di `ReportList.jsx` admin, `ClassReportViewer.jsx` admin, dan student portal, sehingga logo hanya ada bila letterhead background aktif (e-report) dan hilang saat print. Teacher portal sudah benar.

### Option B: Cohort via template backend + flag logo lokal

1. **Cohort** — ambil `MasterLevel.name` di `secondary-semester-report.helper.ts` (relasi `campusAndProgramme` sudah menyediakan `masterLevelId`) dan sisipkan ke data `.hbs` sebagai baris identitas di template. Header client hanya menampilkan tabel grade.
2. **Persentase** — ganti placeholder ke nilai literal `17,5` di `.hbs` untuk template semester 1 non-MYE. Cepat, tetapi tidak fleksibel bila bobot berubah.
3. **Logo** — di tiap pemanggil, set `withLogo: isPrint ? false : !useSemestralLetterhead` untuk Secondary.

---

## Notes

- **Perlu konfirmasi owner bisnis (poin terbuka):**
  1. **Isi Cohort** — apakah yang dimaksud "Cohort Level" (level siswa, mis. "Secondary 2" dari `MasterLevel.name`) atau cohort tahun masuk/kelulusan? UI aplikasi lain (`ClassInYearDetails.jsx:78`) memakai `masterLevels[classInYear.masterLevelId]` dengan label "Cohort Level", jadi arah yang dipilih di Option A mengikuti konvensi itu.
  2. **Posisi Cohort** — apakah menggantikan kolom `Class` (karena `classroom.name` biasanya sudah memuat level, mis. "Sec 2A") atau ditambah sebagai field keempat. Saat ini `Name … Class … Student's ID` harus tetap satu baris di kertas A5; menambah field keempat berisiko wrap, jadi ukuran font perlu diturunkan (sudah ada logika `idFontSize` berbasis panjang teks di line 351).
  3. **Sumber angka 17,5** — apakah 17,5 adalah setengah dari bobot CA1 35% (default saat ini) sehingga boleh dihitung, atau angka tetap yang tidak boleh berubah walau bobot diubah admin? Jika bobot bisa diubah, 17,5 tidak konsisten dengan bobot efektif dan perlu keputusan mana yang benar.
  4. **Scope logo** — apakah "print tanpa logo" berlaku untuk semua portal (Admin, Teacher, Student) dan semua semester Secondary, atau hanya Admin? Teacher sudah tanpa logo di semua mode.
- **Temuan tambahan yang berdekatan:** `Panduan_Fitur_Secondary_vs_Primary.md` menyatakan FYA = 20/25/20/35, sementara `secondary-semester-report.helper.ts` default ke 35/0/25/40. Ketidaksesuaian ini kemungkinan sumber angka persentase yang tidak lagi 17,5 pada data tertentu, dan layak ditelusuri terpisah.
- **Cara verifikasi setelah fix:** generate semester 1 Secondary (no-MYE) di Admin e-report dan print; pastikan header memuat Cohort, kolom bertuliskan `Term 1 (17,5%)` / `Term 2 (17,5%)`, dan tidak ada gambar logo. Ulangi untuk semester 2 (term 4) dan kedua portal lainnya.
