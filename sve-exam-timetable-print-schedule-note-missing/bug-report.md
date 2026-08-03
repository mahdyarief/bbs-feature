---
title: SVE Exam Timetable — Note tidak muncul di PDF Print Schedule
status: open
severity: major
product: BBS LMS
portal: Admin (client)
author: System Analyst
date: 2025-08-03
---

# SVE Exam Timetable — Note "Add Note" Tidak Muncul di PDF Print Schedule

## Summary

Pada halaman **SVE Exam Timetable**, user menambahkan note melalui tombol **"Add Note"** (master Program = **Primary**, Exam Type = **FYE**). Note sudah berhasil tersimpan dan muncul di pop-up Add Note. Namun ketika user menekan tombol **"Print Schedule"** dengan filter yang sama (Primary + FYE), PDF yang ter-generate **tidak menampilkan note** tersebut di bagian bawah tabel schedule.

**Ekspektasi:** Note yang ditambahkan via "Add Note" muncul di bagian bawah tabel schedule pada PDF hasil "Print Schedule".

**Actual Result:** Note tidak muncul di PDF, padahal sudah tersimpan di DB dan benar di pop-up Add Note.

---

## Steps to Reproduce

1. Akses menu **SVE Exam Timetable**
2. Klik tombol **"Add Note"** (label `Note`, icon `BiPlus`)
3. Pilih master Program = **Primary** & Exam Type = **FYE**
4. Isi note — **Note sudah terisi** dan tersimpan (muncul di pop-up Add Note)
5. Klik tombol **"Print Schedule"**
6. Masukkan filter yang sama: **Primary** dan **FYE** → generate PDF

**Actual Result:** PDF ter-generate tetapi **note tidak muncul** di bagian bawah tabel schedule

**Expected Result:** Note yang ada di pop-up Add Note muncul di bagian bawah tabel schedule pada PDF

---

## Root Cause Analysis

### Root Cause (PRIMARY) — Mismatch Tipe Data `masterLevelsIds` (String vs Number)

**File:** `/Users/arielwirawan/Documents/Gawe/api_nest/src/modules/sve-exam-timetable-report/sve-exam-timetable-report.service.ts` **line 302-304**

```typescript
const notesPerSession = notes.find((note) =>
  note.masterLevelsIds.some((ml) => masterLevelsIds.includes(ml)),  // string vs number mismatch
)?.notes?.[0]?.note;
```

Perbandingan ini gagal karena tipe data berbeda:

1. **Frontend menyimpan `masterLevelsIds` sebagai array string** — `SveExamTimetableNote.jsx` line 191-201 `BBSSelect` (isMulti) menghasilkan value string. Disimpan apa adanya ke DB.

2. **Report service membandingkan dengan array number** — line 170-172 `masterLevelsIds` dibangun dari `masterLevel.id` (number), lalu line 302-304 membandingkan `note.masterLevelsIds.includes(ml)`.

3. Karena `"12".includes(12)` → `false` (string ≠ number), `notes.find(...)` mengembalikan `undefined` → `notesPerSession` = `undefined` → di template `<pre>{{{notes}}}</pre>` dirender kosong.

### Jalur Data (Khusus Kasus Primary + FYE)

Kasus Primary + FYE memakai path `generateDefaultTimetable` (default case, line 105-114) → line 302-304 (matching note ke session) → line 309 (`notes: notesPerSession`) → template `sve-exam-timetable-default-schedule.hbs` line 46 `<pre>{{{notes}}}</pre>`.

### Secondary Issue — Note IGCSE/A2 Tidak Pernah Dirender

`generateIgcseA2Timetable` (line 320-348) masih **men-comment** seluruh logic note (line 328-333), dan `notesA2`/`notesIgcse` di-set `''` (line 541-542). Sehingga note IGCSE/A2 tidak pernah dirender di template `igcse-a2`. **Tidak relevan untuk kasus Primary + FYE**, tapi perlu diperbaiki.

---

## Data Flow

| Langkah | File | Keterangan |
|---------|------|------------|
| 1. Tombol "Note" | `SveExamTimetable.jsx` line 224-231 | `setIsNoteOpen(true)` |
| 2. Modal tab "Add Note" | `SveExamTimetableNoteSession.jsx` line 189-200 | Render `SveExamTimetableNote` |
| 3. Input note + pick level | `SveExamTimetableNote.jsx` line 191-220 | `masterLevelsIds` = **string array** |
| 4. Save (bulk) | `SveExamTimetableNoteSession.jsx` line 92-102 | `createBulk` POST `/sveExamTimetableNotes/bulk` |
| 5. Backend save | `sve-exam-timetable-note.service.ts` line 53 | `masterLevelsIds` disimpan **string** |
| 6. Tombol Print Schedule | `SveExamTimetable.jsx` line 200-208 | `setShowPrintSchedule` |
| 7. Generate PDF | `SvePrintScheduleModal.jsx` line 43-50 | `GET /sveExamTimetableReports/schedule` |
| 8. Backend build HTML | `sve-exam-timetable-report.service.ts` line 302-304 | **mismatch string vs number → notes undefined** |
| 9. Template render | `sve-exam-timetable-default-schedule.hbs` line 46 | `<pre>{{{notes}}}</pre>` kosong |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend Admin | `bbs/client/src/views/sves/exam-timetable/noteAndSession/SveExamTimetableNote.jsx` | `masterLevelsIds` disimpan sebagai **string** (line 191-201) |
| Frontend Admin | `bbs/client/src/views/sves/exam-timetable/noteAndSession/SveExamTimetableNoteSession.jsx` | Submit `createBulk` (line 92-102) |
| Backend | `api_nest/src/modules/sve-exam-timetable-note/sve-exam-timetable-note.service.ts` | `createBulk()` line 53 — simpan `masterLevelsIds` tanpa konversi |
| Backend | `api_nest/src/modules/sve-exam-timetable-note/entities/sve-exam-timetable-note.entity.ts` | Deklarasi `masterLevelsIds: Array<Number>` (line 64-65) tapi data string |
| Backend | `api_nest/src/modules/sve-exam-timetable-report/sve-exam-timetable-report.service.ts` | **Mismatch perbandingan** line 302-304 |
| Backend Template | `api_nest/src/templates/sve-report/sve-exam-timetable-default-schedule.hbs` | Line 46 `<pre>{{{notes}}}</pre>` render kosong |

---

## Proposed Solution Options

### Option A: Normalisasi di Backend (Recommended)

Konversi tipe `masterLevelsIds` ke Number **saat save** di `sve-exam-timetable-note.service.ts`:

```typescript
// createBulk() line 53 — dan juga di update()
newNote.masterLevelsIds = (payload.masterLevelsIds || []).map(Number);
```

**Pro:** Data konsisten di DB sejak awal; tidak hanya memperbaiki kasus ini tapi juga semua pembaca lain.

### Option B: Normalisasi di Frontend

Konversi `masterLevelsIds` ke Number sebelum save di `SveExamTimetableNote.jsx` / `SveExamTimetableNoteSession.jsx`.

**Kontra:** Hanya menyentuh satu jalur; data lama yang sudah tersimpan sebagai string tetap perlu migrasi.

### Option C: Normalisasi di Report Service

Konversi saat perbandingan di `sve-exam-timetable-report.service.ts` line 302-304:

```typescript
const notesPerSession = notes.find((note) =>
  (note.masterLevelsIds || []).map(Number).some((ml) => masterLevelsIds.includes(ml)),
)?.notes?.[0]?.note;
```

**Kontra:** Perbaikan hanya di titik baca; data di DB tetap string (tidak konsisten).

---

## Business Rules / Ekspektasi

1. Note yang ditambahkan via "Add Note" harus muncul di bagian bawah tabel schedule pada PDF hasil "Print Schedule"
2. Matching note harus berdasarkan kombinasi: `masterProgrammeId` + `masterSveExamTypeId` + `academicYearId` + level yang cocok dengan session
3. Save dan read harus konsisten dalam tipe data `masterLevelsIds` (number)
4. Untuk kasus Primary + FYE, path yang dipakai `generateDefaultTimetable` — note harus dirender

---

## Notes

- Gunakan `bbs-feature/sve-exam-timetable-print-schedule-note-missing/` untuk spec implementasi setelah keputusan final
- Perlu koordinasi dengan BE Engineer (normalisasi tipe + fix IGCSE/A2 note) — FE Engineer hanya jika memilih Option B
- Download PDF 100% client-side (pdfmake) — tapi **HTML di-generate backend** (`GET /sveExamTimetableReports/schedule`), jadi root cause ada di backend
- Secondary: note IGCSE/A2 di-comment di `generateIgcseA2Timetable` (line 328-333) — perlu perbaikan terpisah

## Reference Files (Dibaca)

### Backend — `api_nest/`

| File | Keterangan |
|------|------------|
| `src/modules/sve-exam-timetable-report/sve-exam-timetable-report.service.ts` | **Root cause** line 302-304 mismatch string/number; `masterLevelsIds` line 170-172; IGCSE/A2 note di-comment line 328-333, 541-542 |
| `src/modules/sve-exam-timetable-note/sve-exam-timetable-note.service.ts` | `createBulk()` line 23-67 — simpan `masterLevelsIds` apa adanya line 53 |
| `src/modules/sve-exam-timetable-note/entities/sve-exam-timetable-note.entity.ts` | Kolom `notes` (jsonb) line 22-23; `masterLevelsIds: Array<Number>` line 64-65 |
| `src/templates/sve-report/sve-exam-timetable-default-schedule.hbs` | Line 46 `<pre>{{{notes}}}</pre>` render notes |

### Frontend — `bbs/`

| File | Keterangan |
|------|------------|
| `client/src/views/sves/exam-timetable/SveExamTimetable.jsx` | Tombol "Note" line 224-231; tombol "Print Schedule" line 200-208; modal line 501-503 |
| `client/src/views/sves/exam-timetable/noteAndSession/SveExamTimetableNoteSession.jsx` | Tab "Add Note" line 189-200; submit `createBulk` line 92-102 |
| `client/src/views/sves/exam-timetable/noteAndSession/SveExamTimetableNote.jsx` | `addNote()` line 72-97; `BBSSelect` pick level line 191-201 (**string array**) |
| `client/src/views/sves/exam-timetable/printSchedule/SvePrintScheduleModal.jsx` | `generateSchedule()` line 43-101; `GET /sveExamTimetableReports/schedule` line 44-50 |
| `client/src/actions/api/sveExamTimetableNotes.js` | `createBulkSveExamTimetableNote` line 44-51 → POST `/sveExamTimetableNotes/bulk` |