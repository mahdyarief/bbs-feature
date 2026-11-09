---
feature: Threshold & Syllabus — Threshold Tidak Muncul / Salah Setelah Memilih AY
slug: threshold-syllabus-ay-scoping-bug
status: draft
author: OpenClaude (dari laporan user + investigasi kode)
date: 2026-10-30
target_release: TBD
---

# Threshold & Syllabus — Threshold Tidak Muncul / Salah Setelah Memilih AY

## Overview

Halaman **Threshold** (menu `Threshold IGCSE/A Level` → `views/threshold/Threshold.jsx`) dan
**Threshold AS Prelim** (`views/threshold/ThresholdAsPrelim.jsx`) punya dropdown **Academic
Year (AY)** di atas daftar kartu threshold dan section **Threshold Syllabus**. Saat user
mengganti AY, isi halaman tidak konsisten: kartu threshold bisa menampilkan nilai AY lain,
dan section Syllabus bisa kosong padahal threshold untuk AY tersebut ada.

Investigasi statis (tanpa perubahan kode) menemukan **tiga defect** yang saling menumpuk:
(1) data threshold tidak di-scope per AY saat dibaca ulang dari Redux, (2) section Syllabus
tidak pernah tahu AY yang dipilih, (3) pembuatan threshold baru bisa menulis
`threshold_config` dengan `syllabusId` kosong sehingga threshold AY tersebut kehilangan
kaitan ke syllabus. Brief ini merinci root cause, dampak, dan spesifikasi perbaikannya.

## Problem / Motivation

User melaporkan: "di bagian threshold syllabus ketika milih AY, threshold-nya ga muncul
bener". Reproduksi:

1. Buka menu Threshold (mis. subject IGCSE/A Level).
2. Pilih AY 2025/2026 → kartu threshold & section Syllabus muncul normal.
3. Ganti AY ke 2024/2025 (atau sebaliknya).
4. Kartu threshold menampilkan grade/boundary yang tidak sesuai AY terpilih, dan/atau
   section Syllabus kosong atau tetap menampilkan grup tahun lama.

### Akar Masalah

**1. Pembacaan threshold tidak di-scope per AY (client).**
`views/threshold/Threshold.jsx:72-82` dan `ThresholdAsPrelim.jsx:73-82` memanggil
`fromApi.getThreshold({ academicYearId, subjectId, thresholdType })`, lalu membaca hasilnya
dengan `useResourceMapper("threshold", currentDraftThresholdApi.sortOrder)`.
`getThreshold` memakai `ACTION_TYPES.MERGE` (`actions/api/threshold.js:9-16`), sementara
endpoint `GET /api/v1/threshold/subject/:subjectId` (`threshold.controller.ts:45-56`,
`threshold.service.ts:276-301`) mengembalikan **semua** threshold subject tersebut lintas AY.
`findBySubjectId` memang hanya *meranking* baris AY terpilih di depan; ia **tidak**
memfilter. Hasilnya slice `threshold` di Redux berisi campuran baris dari banyak AY, dan
`sortOrder` tidak menyaring per AY.

**2. `ThresholdStatistic` memilih baris dengan `find` tanpa filter AY (client).**
`views/threshold/ThresholdStatistic.jsx` mengambil baris DRAFT dan ACTIVE lewat
`threshold.find((d) => d.statusType === ...)`, yaitu **kemunculan pertama** di seluruh slice.
Tanpa filter `academicYearId`, baris AY lain (atau baris yang sudah di-soft-delete) bisa
terpilih; `avgValue`-nya kosong atau tidak cocok sehingga kartu grade kosong/salah.

**3. Section Syllabus tidak menerima AY (client).**
`views/threshold/ThresholSyllabus.jsx` hanya membaca `subjectId` dari query string
(`object().shape({ subjectId: string() })`); AY tidak dipakai sama sekali. Grup syllabus
dibentuk dari semua syllabus subject itu (`GET /api/v1/syllabus/bySubject/:subjectId`,
`syllabus.service.ts:128-140` — filter hanya `subjectId` + `syllabusType`), lalu
`threshold` yang dipakai untuk mengaitkan threshold↔grup adalah `currentDraftThreshold?.[0]`
— yaitu baris pertama daftar campuran AY di atas.

**4. `create` menulis `threshold_config` dengan `syllabusId` kosong (server).**
`threshold.service.ts:79-116` membangun array `syllabi` dari 3 AY sebelumnya. Bila salah
satu AY sebelumnya tidak punya syllabus, elemen array bernilai `undefined`. Pada baris
229-238 array itu dipetakan tanpa filter:

```ts
const thresholdConfigOptions = syllabi.map((syllabus) => ({
  year: syllabus.year,        // TypeError bila syllabus undefined
  syllabusId: syllabus.id,
  thresholdId: thresholdNew.id,
}));
```

Selain risiko `TypeError` (transaksi gagal), pada jalur `activatedThreshold`
(`threshold.service.ts:362-376`) config dengan `syllabusId` kosong ikut tersalin ke
threshold ACTIVE. Threshold ACTIVE AY tersebut jadi tidak punya kaitan syllabus →
section Syllabus tidak menampilkan apa pun meski threshold-nya ada.

**5. `handleSave` komponen syllabus mengirim `rowIndex` yang tidak pernah diisi (client).**
`views/threshold/components/ThresholdSyllabusPart.jsx:196-231` men-destructure
`{ syllabusId, rowIndex, ...rest }` dari baris tabel, padahal `tableData`
(`ThresholdSyllabusPart.jsx:86-118`) tidak pernah mengisi `rowIndex` per baris; nomor baris
hanya berasal dari indeks render. Nilai `rowIndex` yang dikirim ke
`POST /api/v1/syllabusComponent/updateGroup` bisa `undefined`/salah, sehingga penyimpanan
komponen grup bisa mengincar baris yang keliru.

### Dampak

- Kartu threshold di halaman admin menampilkan boundary grade AY yang salah → dasar penilaian
  Cambridge (gradebook) berpotensi keliru bila operator membandingkan/aktivasi.
- Section Syllabus kosong/salah AY → operator tidak bisa memeriksa/mengedit komponen
  syllabus untuk AY yang dipilih.
- `POST /threshold` (create) bisa gagal senyap (500) atau menghasilkan threshold tanpa
  konfigurasi syllabus.

## Scope

### In Scope
- Scope daftar threshold yang dibaca UI ke AY yang dipilih (filter/kunci `academicYearId`).
- Penerusan `academicYearId` ke section Syllabus dan pemakaiannya untuk memilih threshold
  yang tepat (bukan `[0]`).
- Perbaikan pemilihan baris DRAFT/ACTIVE di `ThresholdStatistic` agar per AY dan mengabaikan
  baris terhapus.
- Guard pada `ThresholdService.create` agar tidak menulis `threshold_config` dengan
  `syllabusId` kosong, dan perbaikan penyalinan config pada `activatedThreshold`.
- Perbaikan `rowIndex` pada `handleSave` di `ThresholdSyllabusPart`.
- (Opsional, lihat edge case) penambahan parameter `academicYearId` pada
  `GET /threshold/subject/:subjectId` agar server bisa memfilter.

### Out of Scope
- Perubahan aturan perhitungan threshold (`calculateThreshold`) dan grade clamping.
- Perubahan skema database (tidak perlu kolom baru; `threshold.academic_year_id` sudah ada).
- Redesign UI halaman Threshold.
- Perubahan pada halaman GPA / report lain yang juga memakai threshold (kecuali bila
  ditentukan lain di edge case).

## User Stories

### As an admin/kurikulum
I want memilih AY lalu melihat kartu threshold dan daftar syllabus **hanya** untuk AY itu
So that saya yakin nilai batas grade dan komponen syllabus yang saya tinjau memang milik
tahun ajaran tersebut

### As an admin/kurikulum
I want memilih AY yang belum punya threshold dan mendapat pesan jelas
So that saya tidak mengira data hilang/rusak dan tahu harus membuat threshold baru

### As an admin/kurikulum
I want menyimpan komponen syllabus dan yakin perubahan tersimpan ke baris yang benar
So that konfigurasi threshold AY tersebut akurat

## Acceptance Criteria

- [ ] Setelah memilih AY X, kartu threshold menampilkan hanya threshold dengan
      `academicYearId === X` untuk subject & `thresholdType` aktif di halaman.
- [ ] Baris threshold yang sudah di-soft-delete (`deletedAt` terisi) tidak pernah dipilih.
- [ ] Jika AY X tidak punya threshold DRAFT maupun ACTIVE, halaman menampilkan kartu
      kosong secara eksplisit (bukan nilai AY lain), dan section Syllabus tetap
      menampilkan daftar syllabus (read-only tanpa threshold) atau empty-state yang jelas.
- [ ] Section Syllabus menerima dan memakai AY terpilih; `threshold` yang dipakai untuk
      lookup group adalah baris milik AY tersebut.
- [ ] `POST /api/v1/threshold` tidak pernah menulis `threshold_config` dengan `syllabusId`
      null/undefined; elemen `undefined` di array `syllabi` dilewati.
- [ ] `POST /api/v1/threshold/active` menyalin hanya config dengan `syllabusId` valid.
- [ ] `handleSave` pada `ThresholdSyllabusPart` mengirim `rowIndex` sesuai posisi baris yang
      diedit.
- [ ] Ganti AY bolak-balik (X → Y → X) menampilkan data identik untuk AY yang sama
      (tidak ada sisa state dari AY lain).

## UI / UX Changes

### Affected Portals
- [x] Admin (client/)
- [ ] Student (client-student/)
- [ ] Teacher (client-teacher/)

Deskripsi fungsional:
- Dropdown **Academic Year** (`views/threshold/Threshold.jsx:211-235`,
  `ThresholdAsPrelim.jsx:213-235`) tetap sumber kebenaran AY. Tidak ada perubahan visual.
- Kartu threshold (`ThresholdStatistic`) harus menampilkan boundary untuk AY terpilih saja;
  bila kosong, tampilkan state "No threshold for this academic year".
- Section Syllabus (`ThresholSyllabus` → `ThresholdSyllabusGroup` →
  `ThresholdSyllabusComponent` → `ThresholdSyllabusPart`) menampilkan grup syllabus untuk
  subject, dengan indikator AY aktif yang sama dengan dropdown.
- Halaman `Threshold.jsx` saat ini juga memuat `thresholdApi` dengan
  `getTresholdBySubjectId(subjectId, IGCSE_ALEVEL)` tetapi `threshold` dari situ tidak
  dipakai; perlu diputuskan apakah dihapus (edge case EC-06).

## API Changes

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/threshold` | Tidak berubah; dipakai untuk AY spesifik (`findOne`). Pastikan tidak mengembalikan baris soft-deleted. |
| GET | `/api/v1/threshold/subject/:subjectId` | **Opsional**: tambah query `academicYearId` untuk memfilter (bukan hanya meranking). Lihat EC-04. |
| POST | `/api/v1/threshold` | Perbaiki pembentukan `threshold_config` (lewati syllabus `undefined`). Bentuk request/response tidak berubah. |
| POST | `/api/v1/threshold/active` | Perbaiki penyalinan config (hanya `syllabusId` valid). Bentuk tidak berubah. |
| GET | `/api/v1/syllabus/bySubject/:subjectId` | Tidak berubah (syllabus = rentang `effectiveFrom`/`effectiveTo`, bukan per AY). |

Detail kontrak lihat `api-contract.md`.

## Database Changes

Tidak ada perubahan skema. Kolom yang sudah ada dimanfaatkan:

- `threshold.academic_year_id` (dipakai untuk scoping).
- `threshold_config.syllabus_id` (harus diisi valid; tidak boleh null untuk baris baru).
- `syllabus.academic_year_id`, `syllabus.effective_from`, `syllabus.effective_to`
  (dipakai `findEffectiveForSubjectYear` oleh gradebook).

### Migrations
- Tidak ada. (Bila ada data historis dengan `threshold_config.syllabus_id IS NULL`, lihat
  edge case EC-07 untuk strategi backfill/cleanup data — di luar scope kode.)

## Business Rules / Validation

1. **Satu threshold per (AY, subject, thresholdType, status).** Backend
   (`checkDuplicate`, `threshold.service.ts:398-412`) dan `activatedThreshold` sudah
   menegakkan ini; UI harus mengandalkannya, bukan `find` liar.
2. **Sumber AY di UI = query string `academicYearId`.** Semua komponen yang menampilkan
   threshold/syllabus harus menerima AY ini secara eksplisit, bukan menyimpulkan dari baris
   pertama.
3. **Baris soft-deleted tidak boleh tampil.** Query baca (`findOne`) harus mengecualikan
   `deletedAt != null`.
4. **Threshold tanpa syllabus valid secara data?** Tidak — jika threshold dibuat untuk AY
   yang tidak punya syllabus sama sekali, `threshold_config` kosong dan gradebook
   (`student-cambridge-grade.service.ts` → `findEffectiveForSubjectYear`) mengembalikan
   payload kosong. UI harus memberi peringatan bahwa AY tersebut belum punya syllabus.
5. **Aktivasi threshold** harus menyalin config yang valid saja; bila draft tidak punya
   config valid, aktivasi ditolak dengan error yang jelas (lihat EC-03).

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| AY tanpa syllabus (create) | 400 | "Tidak ada syllabus untuk subject ini pada AY terpilih. Buat syllabus terlebih dahulu." |
| Aktivasi threshold tanpa config valid | 400 | "Threshold ini tidak memiliki konfigurasi syllabus yang valid sehingga tidak dapat diaktifkan." |
| Duplikat threshold (AY+subject+type+status) | 409 | "Threshold untuk AY dan subject ini sudah ada." (sudah ada di `checkDuplicate`) |
| Subject/AY tidak ditemukan | 400 | Sudah ditangani `NoSubjectFoundError` / `NoAcademicYearFoundError`. |

Perbaikan kode dapat mengembalikan error nyata alih-alih `return NoAcademicYearFoundError()`
(konstruktor error yang di-*return*, bukan di-*throw*) — lihat EC-05.

## Dependencies

- Backend: modul `threshold`, `threshold-config`, `syllabus`, `syllabus-component`,
  `student-cambridge-grade` (konsumen threshold via `findBySubjectId`).
- Frontend: `views/threshold/*`, `actions/api/threshold.js`, `actions/api/syllabus.js`,
  `hooks/useFromApi.js`, `hooks/useResourceMapper.js`, `hooks/useQueryString.js`.
- Referensi terkait: `features/gpa-threshold-config/spec.md` (memperkenalkan
  `academic_year_id` per threshold), `features/gpa-report-stale-subject-from-other-year`.
