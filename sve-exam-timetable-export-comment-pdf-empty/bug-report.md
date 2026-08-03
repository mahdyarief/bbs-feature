---
title: SVE Exam Timetable — Export Comment as a PDF menghasilkan PDF kosong
status: open
severity: major
product: BBS LMS
portal: Admin (client)
author: System Analyst
date: 2025-08-03
---

# SVE Exam Timetable — Export Comment as a PDF Menghasilkan PDF Kosong

## Summary

Pada halaman **SVE Exam Timetable**, user sudah:
1. Assign teacher di SVE list untuk AY **26/27 FYE**
2. Set exam date untuk beberapa subject & paper

Namun ketika fitur **"Export Comment as a PDF"** dijalankan, **data yang ter-export pada PDF kosong** — baris tabel PDF muncul tanpa isi (subject, level, component, exam type, session, dll. kosong/blank).

**Ekspektasi:** PDF yang di-generate berisi data comment history lengkap sesuai dengan AY yang dipilih, termasuk kolom User Comment, Exam Type, Subject, Component, Level, Draft Type, Session, Date, dll.

---

## Steps to Reproduce

1. Login sebagai **Admin** ke Admin Portal
2. Buka halaman **SVE → Exam Timetable** (`/Users/arielwirawan/Documents/Gawe/bbs/client/src/views/sves/exam-timetable/SveExamTimetable.jsx`)
3. Pastikan SVE list sudah memiliki:
   - Teacher yang di-assign untuk AY **26/27 FYE**
   - Subject & paper dengan **exam date** yang sudah di-set
4. Klik tombol **"Export Comment PDF"** (line 210-219)
5. Isi AY (26/27 FYE) dan Master SVE Exam Type di modal
6. Generate PDF

**Actual Result:** PDF ter-generate tetapi **semua baris data kosong** (tidak ada subject, level, component, exam type, session, comment)

**Expected Result:** PDF berisi daftar comment history yang lengkap untuk AY yang dipilih

---

## Root Cause Analysis

### Lapisan 1 (PRIMARY) — Typo Parameter API: `relationsObje` → `relationsObj`

**File:** `/Users/arielwirawan/Documents/Gawe/bbs/client/src/views/sves/exam-timetable/SveExamTimeTableHIstoriesPDFModal.jsx` **line 60**

```jsx
// BUG: salah ketik — seharusnya "relationsObj"
relationsObje: JSON.stringify({
  sveExamType: { sve: { subject: true } }
})
```

Backend DTO mendefinisikan properti **`relationsObj`**:
- `/Users/arielwirawan/Documents/Gawe/api_nest/src/common/dto/page-options.dto.ts` (line 87-100)

Karena frontend mengirim `relationsObje` (salah eja), backend **tidak pernah membaca** request relations → `find-options-helper.ts` (line 57-67) fallback ke default `{}` → **relasi `sveExamType → sve → subject` tidak di-load**.

Akibatnya, response `GET /v1/sveExamTypeHistories` hanya berisi baris history polos di `body.data`, dan **tidak ada entity `sveExamType` di `body.included`**.

### Lapisan 2 — Lookup Store yang Fragile (Paginated Redux)

**File:** `/Users/arielwirawan/Documents/Gawe/bbs/client/src/views/sves/exam-timetable/SveExamTimeTableHIstoriesPDFModal.jsx` (line 70-76)

Modal merekonstruksi metadata parent dari Redux store:

```jsx
const sveExamType = resourceMapper("sveExamTypes", [d.attributes.sveExamTypeId])[0];
const sve = sves?.[sveExamType?.sveId];
const level = masterLevels?.[sve?.masterLevelId];
const subject = subjects?.[sve?.subjectId];
const ay = academicYears?.[sve?.academicYearId];
const mse = masterSveExamTypes?.[data.masterSveExamTypeId];
```

Store `sveExamTypes` hanya terisi dari query BBSDataTable halaman utama yang **paginated & filtered** (`SveExamTimetable.jsx` line 456-459 — `excludeBMT: true`, `academicYearId: currentAy?.id`).

Jika record `sveExamType` yang bersangkutan **tidak ada di halaman yang sedang dimuat**, maka `sveExamType = undefined` → semua lookup turunannya (`sve`, `subject`, `level`, `session`, `component`, `examType`) jadi `undefined` → **baris PDF render blank**.

Bahkan jika history rows ada, typo `relationsObje` membuat API tidak pernah mengembalikan `sveExamType` yang ter-couple → store tidak terpopulasi → lookup gagal.

### Lapisan 3 (SECONDARY) — Filter Overwrite di Backend `findAll()`

**File:** `/Users/arielwirawan/Documents/Gawe/api_nest/src/modules/sve-exam-type-history/sve-exam-type-history.service.ts` (line 55-67)

```typescript
if (options.masterSveExamTypeId) {
  whereFilters.sveExamType = { masterSveExamTypeId: options.masterSveExamTypeId };
}
if (options.academicYearId) {
  whereFilters.sveExamType = { sve: { academicYearId: options.academicYearId } }; // overwrite!
}
```

Ketika keduanya dikirim (modal selalu mengirim keduanya), block `academicYearId` **menimpa** block `masterSveExamTypeId` → filter exam type di-drop → query mengembalikan *lebih banyak* rows dari seharusnya (semua exam type untuk AY tersebut). Ini bukan penyebab utama PDF kosong, tapi logika filternya salah.

---

## Data Flow

| Langkah | File | Keterangan |
|---------|------|------------|
| 1. Tombol "Export Comment PDF" | `SveExamTimetable.jsx` line 210-219 | `setShowHistoriesGeneratePDF(true)` |
| 2. Buka modal | `SveExamTimeTableHIstoriesPDFModal.jsx` | Pilih AY + Master SVE Exam Type |
| 3. API call | `actions/api/sveExamTypeHistory.js` line 17-23 | `GET /v1/sveExamTypeHistories?page=0&pageSize=0&masterSveExamTypeId=...&academicYearId=...&relationsObje=...` |
| 4. Controller | `sve-exam-type-history.controller.ts` line 38-44 | Forward ke service |
| 5. Service `findAll()` | `sve-exam-type-history.service.ts` line 45-83 | Filter + relasi (relationsObj tidak terbaca) |
| 6. Redux dispatch | `makeApiRequest.js` line 112-154 | Hanya dispatch `body.data` + `body.included` (included kosong) |
| 7. PDF generation | `utilFunctions.js` line 775 `generateSveCommentHistories()` | Build HTML table dari store |
| 8. Generate PDF | `pdf-make.js` `generatePDF.createPdf(dd).download(...)` | PDF kosong |

---

## Scope Note — Apa yang Di-Export

Export ini hanya mencakup **comment/change-request history rows**. Kolom-kolomnya: User Comment, Exam Type, Subject, Component, Level, Draft Type, Session, Date, Change Request, Reason, Status.

**TIDAK termasuk** nama teacher (setter/expert/vetter) atau exam datetime — itu berada di `SveExamType` (`sve-exam-type.entity.ts` line 50-58, 106-135) dan tampil di tabel timetable, bukan di PDF comment ini.

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend Admin | `bbs/client/src/views/sves/exam-timetable/SveExamTimeTableHIstoriesPDFModal.jsx` | **Typo `relationsObje`** (line 60) — relasi tidak di-load |
| Frontend Admin | `bbs/client/src/views/sves/exam-timetable/SveExamTimetable.jsx` | Store paginated — lookup parent sering undefined |
| Backend | `api_nest/src/modules/sve-exam-type-history/sve-exam-type-history.service.ts` | Filter `masterSveExamTypeId` di-overwrite oleh `academicYearId` (line 55-67) |
| Backend | `api_nest/src/common/dto/page-options.dto.ts` | Properti `relationsObj` (line 87-100) — nama benar, frontend salah |

---

## Proposed Solution Options

### Option A: Fix Typo + Gunakan Response (Recommended)

1. **`SveExamTimeTableHIstoriesPDFModal.jsx` line 60** — Perbaiki typo: `relationsObje` → `relationsObj`
2. **Jangan andalkan store paginated** — gunakan entity `sveExamType` dari response `body.included`, atau fetch `sveExamTypes` by ids sebelum mapping, ganti `resourceMapper("sveExamTypes", ...)`
3. **`sve-exam-type-history.service.ts`** — Merge filter, jangan overwrite:
   ```typescript
   whereFilters.sveExamType = {
     masterSveExamTypeId: options.masterSveExamTypeId,
     sve: { academicYearId: options.academicYearId },
   };
   ```

### Option B: Minimal Fix (Typo Saja)

1. Perbaiki typo `relationsObje` → `relationsObj` saja
2. **Risiko:** Masih bergantung pada store paginated — PDF bisa tetap kosong untuk record yang parent-nya tidak ada di halaman yang sedang dimuat

### Option C: Backend-Driven PDF

1. Pindahkan generasi PDF ke backend — endpoint khusus yang mengembalikan PDF jadi dengan data lengkap (query langsung join `sve_exam_type_history` → `sve_exam_type` → `sve` → `subject` + master exam type)
2. **Pro:** Tidak ada ketergantungan pada Redux store, data selalu lengkap
3. **Kontra:** Perubahan besar; butuh backend PDF engine

---

## Business Rules / Ekspektasi

1. PDF harus berisi **semua comment history** untuk kombinasi AY + Master SVE Exam Type yang dipilih
2. Setiap baris PDF harus menampilkan: User Comment, Exam Type, Subject, Component, Level, Draft Type, Session, Date, Change Request, Reason, Status
3. Jika AY 26/27 FYE dipilih, hanya data AY tersebut yang tampil
4. Teacher assignment & exam date yang sudah di-set di SVE list harus ikut konsisten dengan data di tabel timetable

---

## Notes

- Gunakan `bbs-feature/sve-exam-timetable-export-comment-pdf-empty/` untuk spec implementasi setelah keputusan final
- Perlu koordinasi dengan FE Engineer (perbaikan modal + utilFunctions) dan BE Engineer (fix filter overwrite)
- "FYE" hanyalah display label (dipetakan ke "Final Year Examination" di `SveExamTimeTablePDFLayout.js` line 14) — tidak ada filter FYE terpisah di path ini; filter hanya `academicYearId`
- PDF 100% client-side (pdfmake) — tidak ada intervensi backend di pembuatan PDF