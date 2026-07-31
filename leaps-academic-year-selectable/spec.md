---
feature: LEAPS Academic Year Selectable
slug: leaps-academic-year-selectable
type: feature-changes
status: draft
author: BBS Team
date: 2026-07-31
target_release: TBD
---

# LEAPS — Academic Year Selectable

## Overview

Perubahan kecil di **teacher portal** (`client-teacher/`) pada halaman LEAPS Event form (`/leaps/add` dan `/leaps/:id/edit`). Saat ini field **Academic Year** di form di-**disable** (`isDisabled`) sehingga hanya bisa menggunakan **current academic year**. Perubahan ini membuat field Academic Year **bisa dipilih**, sehingga teacher dapat membuat/mengedit LEAPS event untuk **academic year sebelumnya**.

## Problem / Motivation

Saat ini teacher tidak bisa membuat atau mengedit LEAPS event untuk academic year yang sudah lewat. Contoh kasus:

- Teacher ingin menambah LEAPS event yang terlewat di AY sebelumnya (retroactive entry)
- Teacher ingin mengoreksi data LEAPS event yang dibuat di AY sebelumnya

Field Academic Year di form di-hardcode `isDisabled` sehingga terkunci ke current academic year. Ini menghambat data entry untuk AY lampau.

## Root Cause

**File:** `bbs/client-teacher/src/views/form-class/leaps/LeapsForm.jsx`

Line 188 — prop `isDisabled` di `BBSResourceSelect` untuk field `academicYearId`:

```jsx
<BBSResourceSelect
  control={control}
  pagination={false}
  name="academicYearId"
  searchKey="year"
  pluralName="academicYears"
  singularName="academicYear"
  isDisabled                        // <-- root cause: AY terkunci
  defaultValue={
    editMode ? leapsEvent?.academicYearId : currentAyear?.id
  }
  query={{
    selectsObj: JSON.stringify({
      id: true,
      year: true
    })
  }}
/>
```

Selain itu, `useEffect` di line 74-78 selalu set `academicYearId` ke `currentAyear?.id`:

```jsx
useEffect(() => {
  if (currentAyear?.id) {
    setValue("academicYearId", currentAyear?.id);
  }
}, [currentAyear]);
```

## Scope

### In Scope
- Hapus `isDisabled` dari field Academic Year di `LeapsForm.jsx` (create & edit mode)
- Adjust `useEffect` yang meng-override academicYearId ke current AY — hanya set default saat create mode (belum ada value), jangan override pilihan user
- Verifikasi field Academic Year di halaman list (`Leaps.jsx`) — sudah selectable, tidak perlu diubah

### Out of Scope
- Perubahan di backend (`api_nest/`)
- Perubahan di admin portal (`client/`)
- Perubahan validasi inclusive date vs academic year range (jika ada kebijakan baru, pisahkan jadi brief terpisah)
- Perubahan logic programme di form (tetap seperti sekarang — user memilih programme secara manual)

## User Stories

### As a Teacher
I want to select the academic year when creating or editing a LEAPS event
So that I can add or correct LEAPS events for previous academic years

## Acceptance Criteria

- [ ] Field Academic Year di form `/leaps/add` dapat dipilih (tidak disabled)
- [ ] Field Academic Year di form `/leaps/:id/edit` dapat dipilih (tidak disabled)
- [ ] Saat create mode, default value Academic Year tetap current academic year
- [ ] Saat edit mode, Academic Year menampilkan AY dari event yang sedang diedit
- [ ] User dapat mengganti Academic Year ke AY lain dan submit berhasil
- [ ] LEAPS event tersimpan dengan academicYearId sesuai pilihan user
- [ ] User dapat mengedit event AY sebelumnya dan mengganti AY-nya

## UI / UX Changes

### Affected Portals
- [ ] Admin (client/)
- [ ] Student (client-student/)
- [x] Teacher (client-teacher/) — **target**

### Halaman `/leaps/add` dan `/leaps/:id/edit`

**Sebelum (current behavior):**

```
Add LEAPS Event
┌─────────────────────────────────────────────┐
│ Academic Year: [ 2025/2026        ▾ (disabled) ] │  ← terkunci
│ Leaps Type:    [ ----------------- ▾ ]        │
│ Name:          [ _________________ ]           │
│ Inclusive Date: [ ______________ ]             │
│ Programme:     [ (multi select)    ]           │
│ Description:   [ _________________ ]           │
└─────────────────────────────────────────────┘
```

**Sesudah (target behavior):**

```
Add LEAPS Event
┌─────────────────────────────────────────────┐
│ Academic Year: [ 2025/2026        ▾        ] │  ← bisa dipilih (default current AY)
│ Leaps Type:    [ ----------------- ▾ ]        │
│ Name:          [ _________________ ]           │
│ Inclusive Date: [ ______________ ]             │
│ Programme:     [ (multi select)    ]           │
│ Description:   [ _________________ ]           │
└─────────────────────────────────────────────┘
```

## Implementation Detail

### File: `bbs/client-teacher/src/views/form-class/leaps/LeapsForm.jsx`

**Change 1 — Hapus `isDisabled` (line 188):**

```diff
 <BBSResourceSelect
   control={control}
   pagination={false}
   name="academicYearId"
   searchKey="year"
   pluralName="academicYears"
   singularName="academicYear"
-  isDisabled
   defaultValue={
     editMode ? leapsEvent?.academicYearId : currentAyear?.id
   }
```

**Change 2 — Adjust useEffect (line 74-78):**

Saat ini useEffect selalu meng-override value ke current AY setiap kali `currentAyear` berubah — ini akan menimpa pilihan user. Ubah agar hanya set default di create mode (saat belum ada value).

```diff
 useEffect(() => {
-  if (currentAyear?.id) {
-    setValue("academicYearId", currentAyear?.id);
-  }
-}, [currentAyear]);
+  // Set default ke current AY hanya di create mode (bukan edit mode)
+  if (currentAyear?.id && !leapsEventId) {
+    setValue("academicYearId", currentAyear?.id);
+  }
+}, [currentAyear, leapsEventId]);
```

### Catatan: Halaman List (`Leaps.jsx`)

Field Academic Year di halaman list **sudah selectable** (tidak ada `isDisabled`). Berarti user sudah bisa filter list berdasarkan AY. Setelah perubahan ini, konsistensi terjaga: user bisa filter list ke AY lama → bisa masuk ke detail → bisa edit.

## API Changes

**Tidak ada perubahan endpoint.** 

Endpoint yang digunakan sudah support `academicYearId` di payload create/update:

| Method | Path | Keterangan |
|--------|------|-----------|
| POST | `/api/v1/leapsEvents` | Create LEAPS event dengan `academicYearId` |
| PATCH/PUT | `/api/v1/leapsEvents/:id` | Update LEAPS event dengan `academicYearId` |
| GET | `/api/v1/academicYears` | Dropdown list academic year (sudah digunakan) |

## Database Changes

**Tidak ada perubahan database.** Kolom `academic_year_id` di tabel LEAPS event sudah ada.

## Business Rules / Validation

1. **Academic Year optional flow:** Tidak ada batasan bahwa LEAPS event hanya boleh untuk current AY — validasi di form hanya `required`
2. **Default value:** Create mode default ke current academic year; edit mode menampilkan AY event tersebut
3. **Programme tetap manual:** Programme dipilih manual oleh user (tidak auto-sync dengan AY). Jika user memilih AY lama + programme yang tidak aktif di AY tersebut, tetap bisa disubmit (tidak ada validasi baru)
4. **Inclusive Date:** Tidak ada validasi bahwa inclusive date harus dalam rentang AY yang dipilih (tetap seperti sekarang)

## Error Handling

| Error | HTTP Code | Message | Behavior |
|-------|-----------|---------|----------|
| Academic Year not found | 404 | "Academic year not found" | Error toast dari existing API |
| Invalid academicYearId | 400 | Validation error | Error toast dari existing API |
| Loading academic years | — | — | Skeleton/disabled dropdown saat fetch |

## Dependencies

- **Existing component:** `BBSResourceSelect` (teacher FE) — dropdown Academic Year
- **Existing file:** `LeapsForm.jsx` (teacher FE) — satu-satunya file yang diubah
- **Existing endpoint:** `leapsEvents` (BE) — create/update sudah support `academicYearId`
