---
feature: Remark Max Length 535
slug: remark-max-length-535
status: draft
author: OpenClaude
date: 2026-09-28
target_release: TBD
---

# Remark Max Length 535

## Overview

Perubahan batas maksimum karakter **remark** dari **837 → 535**, dengan validasi dilakukan di **frontend** (Teacher Portal) dan **tanpa memicu autosave/save**. Saat ini limit 837 hardcode di `RemarkDetail.jsx` (yup schema) dan tidak ada validasi panjang di backend; validasi justru dijalankan lewat `trigger()` di dalam alur autosave, sehingga pemeriksaan max-length terikat pada siklus save. Brief ini mensyaratkan: limit baru 535, pengecekan max di sisi frontend, dan pengecekan tersebut **tidak** memicu atau men-gate proses save/autosave.

## Problem / Motivation

- Batas karakter remark saat ini **535**, di-hardcode di frontend (`const maxRemarkLength = 535`)
- Pemeriksaan max-length saat ini berjalan di dalam `autosaveStudentRef.current` via `await trigger(...)` sebelum `dispatch(fromApi.autosaveBulkRemark(...))` — artinya validasi dan siklus save terikat bersama: validasi mempengaruhi kapan/apa yang disave.
- Kebutuhan bisnis: pengecekan max-length harus terjadi **di frontend** dan **tidak trigger save** — user melihat feedback melebihi batas tanpa menyebabkan request autosave terkirim/terblokir oleh mekanisme validasi.
- Backend (`CreateRemarkDto`) saat ini hanya punya `@IsString()` — **tidak ada** `@MaxLength` sama sekali, jadi batas 535 murni keputusan frontend (lihat edge case EC-04).

## Scope

### In Scope
- Mengubah limit maksimum remark dari 837 → **535** di frontend (Teacher Portal, `RemarkDetail.jsx`).
- Memindahkan/melakukan pengecekan max-length **di frontend** tanpa terikat ke siklus autosave/save.
- Feedback error max-length tetap tampil di bawah textarea (`BBSTextArea` `error` prop) — `Max. 535 characters`.
- Autosave tetap berjalan normal untuk input yang **valid** (≤ 535).

### Out of Scope
- Perubahan database / migration — kolom `remark` di entity `Remark` tidak berubah.
- Endpoint API baru — tidak ada perubahan route `remarks`.
- Validasi `@MaxLength(535)` di backend DTO (opsional, lihat EC-04).
- Component `BBSTextArea` di `bbs-client-common` — tidak wajib diubah; jika ditambah counter/maxLength harus opt-in agar tidak mempengaruhi consumer lain.
- Portal Admin (`client/`) dan Student (`client-student/`) — halaman remark hanya ada di Teacher Portal.
- Perubahan perilaku manual Save button (`handleSubmitRemark`) di luar relasi dengan validasi max-length.

## User Stories

### As a Teacher
I want a remark field that limits my input to 535 characters with clear feedback
So that I know the exact limit before submitting and my text isn't silently rejected later.

### As a Teacher
I want the max-length check to run without triggering or blocking the autosave
So that typing over the limit shows an error immediately while my valid changes still save automatically.

## Acceptance Criteria

- [ ] **AC-1:** Limit maksimum remark di frontend adalah **535** (bukan 837); pesan error menampilkan `Max. 535 characters`.
- [ ] **AC-2:** Pengecekan max-length dilakukan di frontend (saat mengetik / onBlur), **bukan** hanya saat save.
- [ ] **AC-3:** Pengecekan max-length **tidak memicu** request autosave (`POST /api/v1/remarks/bulk` tidak terkirim hanya karena validasi berjalan).
- [ ] **AC-4:** Pengecekan max-length **tidak men-gate/menunda** autosave untuk input valid — autosave tetap jalan sesuai debounce 800ms.
- [ ] **AC-5:** Input ≤ 535 karakter: autosave berfungsi normal (status "Saving…" → "All changes saved").
- [ ] **AC-6:** Input > 535 karakter: error tampil di bawah textarea terkait; autosave tidak mengirim payload invalid (atau data tidak tersimpan melebihi 535 — lihat EC-04).
- [ ] **AC-7:** Manual Save button tetap memvalidasi seluruh form sebelum submit (perilaku `trigger()` pada `handleSubmitRemark` dipertahankan).

## UI / UX Changes

- Tidak ada perubahan layout. Perubahan hanya pada **behavior validasi**:
  - Pesan error di bawah textarea: `Max. 535 characters`.
  - Error muncul mengikuti `mode: "onTouched"` react-hook-form (setelah touch/blur) atau saat validasi inline berjalan — tanpa memicu save.
  - Opsional (usulan): counter karakter `123 / 535` di bawah textarea agar teacher tahu sisa kuota (lihat EC-05).

### Affected Portals
- [ ] Admin (client/)
- [ ] Student (client-student/)
- [x] Teacher (client-teacher/)

### Lokasi Halaman
- Route: `/remark/:classYearId/:academicYearId` (name: "Add Remark") — `bbs/client-teacher/src/routes.js`
- View: `bbs/client-teacher/src/views/form-class/remark/RemarkDetail.jsx`

## API Changes

Tidak ada perubahan endpoint. Endpoint yang terlibat (perilaku tidak berubah):

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/remarks/bulk` | Autosave & manual save bulk remark — **tidak boleh terpicu oleh pengecekan max-length** |
| GET | `/api/v1/remarks` | Load remarks per term / studentReportIds |

## Database Changes

### New Tables
- Tidak ada.

### Modified Tables
- Tidak ada. Entity `Remark` (`remark` column, `nullable: true`, tanpa length constraint) tidak berubah.

### Migrations
- Tidak ada.

## Business Rules / Validation

1. **Limit:** Panjang maksimum isi remark adalah **535 karakter** (diukur pada string value textarea).
2. **Validasi di frontend:** Pengecekan max-length dilakukan di frontend pada field `studentRemark.{i}.remarks.{j}.remark` — bukan di backend, dan bukan hanya saat save.
3. **Validasi tidak trigger save:** Menjalankan/memunculkan pengecekan max-length **dilarang** memicu `fromApi.autosaveBulkRemark` atau `fromApi.createOrUpdateBulkRemark`. Pengecekan harus terpisah dari siklus autosave (autosave hanya menangani payload valid).
4. **Autosave tetap berjalan:** Autosave (debounce 800ms) tetap memproses baris dengan input valid; keberadaan error max-length di baris lain tidak men-stop autosave baris valid.
5. **Payload invalid tidak dikirim:** Jika isi remark melebihi 535, baris/ts itu tidak dikirim ke backend dalam keadaan invalid — autosave dilewati untuk payload tersebut (behavior detail: lihat EC-04), sehingga data > 535 tidak pernah persist.
6. **Manual save:** Tombol Save tetap menjalankan validasi seluruh form (`trigger()`) dan hanya submit bila seluruh field valid.
7. **Nilai yang sudah ada > 535:** Data lama dengan panjang 536–837 yang sudah tersimpan tidak di-truncate/dihapus otomatis (lihat EC-03).

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| Remark melebihi 535 karakter (frontend validation) | — (client-side) | `Max. 535 characters` |
| Autosave di-skip karena payload invalid | — (client-side, tidak ada request) | Tidak ada request terkirim; error tampil di field |
| Backend menolak remark terlalu panjang (jika EC-04 opsi B) | 400 | Pesan validasi class-validator bawaan |
| Autosave gagal (network/server) | 400/500 | Status autosave `error`: "Couldn't autosave changes" (perilaku existing, tidak berubah) |

## Dependencies

- Frontend (`bbs/client-teacher/`):
  - `src/views/form-class/remark/RemarkDetail.jsx` — `maxRemarkLength` (baris 60), yup schema (baris 71–86), `autosaveStudentRef.current` + `trigger()` (baris 257–303), debounced autosave (baris 305–313), render `BBSTextArea` + `error` prop (baris 452–498), manual save `handleSubmitRemark` (baris 324–362).
  - `bbs-client-common` — `BBSTextArea` (menampilkan `error?.message`), `bbsToaster`, `bbsConfirm`.
  - `react-hook-form` (`useForm`, `useFieldArray`, `trigger`), `yup` + `yupResolver`, `lodash/debounce`.
- Backend (`api_nest/`):
  - `src/modules/remark/dto/create-remark.dto.ts` — `remark: string` (`@IsString`, `@IsOptional`, **tanpa MaxLength**).
  - `src/modules/remark/remark.service.ts` — `create` / `update` / `createUpdateBulk` (tidak ada validasi panjang).
  - `src/modules/remark/entities/remark.entity.ts` — kolom `remark` nullable, tanpa length constraint.
- Cross-reference:
  - `features/appraisal-lock/` — lock remark mematikan input (`disabled={isLocked && passDueDateInTerm}`).
