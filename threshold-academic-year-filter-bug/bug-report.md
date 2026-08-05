---
title: Threshold — Nilai Threshold Tidak Ikut Berubah Sesuai Academic Year yang Dipilih
status: open
severity: major
product: BBS LMS
portal: Admin (client)
author: System Analyst
date: 2026-08-05
jam-link: https://jam.dev/c/a14dd129-1e23-4441-ac55-0a89f71fe242
---

# Threshold — Nilai Threshold Tidak Ikut Berubah Sesuai Academic Year yang Dipilih

## Summary

Pada halaman **Threshold** (`bbs/client/src/views/threshold/Threshold.jsx`), user memilih **Subject** dan **Academic Year** untuk melihat / mengelola nilai threshold. Saat user **mengganti Academic Year**, nilai threshold yang tampil **tidak ikut berubah** — nilai yang muncul tetap milik kombinasi subject + academic year sebelumnya.

**Ekspektasi:** Ketika Academic Year diubah, nilai threshold (dan data terkaitnya) yang tampil harus sesuai dengan Academic Year yang baru dipilih.

---

## Steps to Reproduce

1. Login sebagai **Admin** ke Admin Portal
2. Buka halaman **Threshold**
3. Pilih **Subject** (misal subject X) dan **Academic Year A** — catat nilai threshold yang muncul
4. Ganti **Academic Year** ke **B** (yang sudah memiliki data threshold berbeda)
5. Perhatikan nilai threshold yang tampil

**Actual Result:** Nilai threshold **tetap sama** seperti Academic Year A — tidak berubah meskipun Academic Year sudah diganti ke B.

**Expected Result:** Nilai threshold berubah sesuai data threshold milik Academic Year B.

---

## Root Cause Analysis

Ada **dua jalur pengambilan data** di halaman Threshold (`Threshold.jsx`):

1. **`currentDraftThresholdApi`** → `fromApi.getThreshold(...)` → `GET /v1/threshold` → `findOne()` — **SUDAH BENAR** (academicYear-aware)
2. **`thresholdApi`** → `fromApi.getTresholdBySubjectId(subjectId)` → `GET /v1/threshold/subject/:subjectId` → `findBySubjectId()` — **SALAH** (tidak academicYear-aware)

Jalur kedua inilah yang menyebabkan bug: ia **tidak pernah mengirim `academicYearId`** (baik di frontend maupun backend), sehingga data threshold dari subject yang sama tetap muncul walau Academic Year diganti.

### Jalur 1 — `findOne()` (BENAR, bukan penyebab)

**File:** `api_nest/src/modules/threshold/threshold.service.ts` line 246-262

```typescript
async findOne(options: FindThresholdDto) {
  const data = await Threshold.findOne({
    where: {
      academicYearId: options.academicYearId,   // ✅ filter AY ada
      subjectId: options.subjectId,
      statusType: options.thresholdStatus || IThresholdStatusType.DRAFT,
      thresholdType: options.thresholdType || ThresholdTypeEnum.IGCSE_ALEVEL,
    },
    ...
```

DTO-nya juga sudah mensyaratkan `academicYearId`:

**File:** `api_nest/src/modules/threshold/dto/find-threshold.dto.ts` line 7-12

```typescript
export class FindThresholdDto {
  @ApiProperty()
  @IsNumber()
  @IsPositive()
  @Type(() => Number)
  academicYearId: number;   // ✅ required
```

Dan di frontend, `currentDraftThresholdApi` sudah mengirim `academicYearId` + dependency `[academicYearId, subjectId]`:

**File:** `bbs/client/src/views/threshold/Threshold.jsx` line 72-82

```jsx
const currentDraftThresholdApi = useFromApi(
  () => !!subjectId && !!academicYearId &&
    fromApi.getThreshold({
      academicYearId,
      subjectId,
      thresholdStatus: THRESHOLD_STATUS_ENUM[THRESHOLD_STATUS_ENUM.DRAFT],
      thresholdType: THRESHOLD_TYPE_ENUM.AS_PRELIM,
    }),
  [academicYearId, subjectId],
  () => !!subjectId && !!academicYearId,
);
```

### Jalur 2 — `findBySubjectId()` (PENYEBAB BUG)

#### Lapisan 1 (PRIMARY) — Backend tidak menerima `academicYearId`

**File:** `api_nest/src/modules/threshold/threshold.controller.ts` line 45-56

```typescript
@Get('subject/:subjectId')
async findOneBySubjectId(
  @Param('subjectId', ParseIntPipe) subjectId: number,
  @Query('thresholdType') thresholdType?: ThresholdTypeEnum,
) {
  return {
    data: await this.thresholdService.findBySubjectId(subjectId, thresholdType),
  };
}
```

Endpoint hanya menerima `thresholdType` sebagai query param — **tidak ada `academicYearId`**.

**File:** `api_nest/src/modules/threshold/threshold.service.ts` line 264-278

```typescript
async findBySubjectId(subjectId: number, thresholdType?: ThresholdTypeEnum) {
  const where: any = {
    subjectId,   // ⛔ hanya filter subject — SEMUA academic year ikut terambil
  };
  if (thresholdType) {
    where.thresholdType = thresholdType;
  }
  const data = await Threshold.find({
    where,
    relations: { thresholdConfig: true },
  });
  return data.map((d) => new ThresholdDto(d));
}
```

`findBySubjectId` mengembalikan **semua** threshold untuk subject tersebut di semua Academic Year. Data yang dirender di halaman tidak pernah terfilter oleh AY yang sedang dipilih user.

#### Lapisan 2 (PRIMARY) — Frontend tidak mengirim `academicYearId` & dependency salah

**File:** `bbs/client/src/actions/api/threshold.js` (fungsi `getTresholdBySubjectId`, line 18-26)

```js
export const getTresholdBySubjectId = (subjectId) =>
  api.get(`/threshold/subject/${subjectId}`);
```

Fungsi ini tidak menerima / mengirim `academicYearId` sama sekali.

**File:** `bbs/client/src/views/threshold/Threshold.jsx` line ~52-56

```jsx
const thresholdApi = useFromApi(
  () => fromApi.getTresholdBySubjectId(subjectId),
  [subjectId],   // ⛔ dependency TIDAK menyertakan academicYearId
  ...
);
```

Karena dependency `useFromApi` hanya `[subjectId]`, perubahan `academicYearId` **tidak memicu refetch** → data threshold di halaman tetap data lama dari subject yang sama. (Hook `useFromApi` mem-fetch ulang ketika dependency berubah — lihat `bbs/client/src/hooks/useFromApi.js`.)

---

## Data Flow

| Langkah | File | Keterangan |
|---------|------|------------|
| 1. Pilih Subject + Academic Year | `Threshold.jsx` line 30-35 (`useQueryString`) | `subjectId`, `academicYearId` dari query string |
| 2. Fetch data threshold | `Threshold.jsx` line ~52-56 (`thresholdApi`) | `fromApi.getTresholdBySubjectId(subjectId)` — **tanpa academicYearId**, deps `[subjectId]` |
| 3. API call | `actions/api/threshold.js` line 18-26 | `GET /v1/threshold/subject/:subjectId` |
| 4. Controller | `threshold.controller.ts` line 45-56 | Hanya forward `subjectId` + `thresholdType` (opsional) |
| 5. Service `findBySubjectId()` | `threshold.service.ts` line 264-278 | Filter hanya `subjectId` → **semua AY ikut** |
| 6. Render | `Threshold.jsx` | Data threshold dari AY lama tetap dirender |
| 7. (Pembanding) `currentDraftThresholdApi` | `Threshold.jsx` line 72-82 | `GET /v1/threshold` + `findOne()` — **sudah benar** |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Frontend Admin | `bbs/client/src/views/threshold/Threshold.jsx` | `thresholdApi` deps `[subjectId]` saja (line ~52-56) — refetch tidak terpicu saat AY berubah |
| Frontend Admin | `bbs/client/src/actions/api/threshold.js` | `getTresholdBySubjectId` (line 18-26) tidak menerima param `academicYearId` |
| Backend | `api_nest/src/modules/threshold/threshold.controller.ts` | `GET /threshold/subject/:subjectId` (line 45-56) tidak menerima query param `academicYearId` |
| Backend | `api_nest/src/modules/threshold/threshold.service.ts` | `findBySubjectId()` (line 264-278) tidak filter `academicYearId` |

---

## Proposed Solution Options

### Option A: Tambahkan `academicYearId` di seluruh jalur `subject/:subjectId` (Recommended)

1. **Backend DTO baru / param opsional** — terima `academicYearId` (opsional) di `GET /threshold/subject/:subjectId`, misal via query param `@Query('academicYearId', ParseIntPipe) academicYearId?: number`
2. **`findBySubjectId`** — tambahkan filter:
   ```typescript
   async findBySubjectId(subjectId: number, thresholdType?: ThresholdTypeEnum, academicYearId?: number) {
     const where: any = { subjectId };
     if (thresholdType) { where.thresholdType = thresholdType; }
     if (academicYearId) { where.academicYearId = academicYearId; }
     ...
   }
   ```
3. **`getTresholdBySubjectId`** — terima `academicYearId` dan kirim sebagai query param:
   ```js
   export const getTresholdBySubjectId = (subjectId, academicYearId) =>
     api.get(`/threshold/subject/${subjectId}`, { params: { academicYearId } });
   ```
4. **`Threshold.jsx`** — panggil `fromApi.getTresholdBySubjectId(subjectId, academicYearId)` dan masukkan `academicYearId` ke dependency `useFromApi`:
   ```jsx
   const thresholdApi = useFromApi(
     () => fromApi.getTresholdBySubjectId(subjectId, academicYearId),
     [subjectId, academicYearId],
     () => !!subjectId && !!academicYearId,
   );
   ```

**Pro:** Konsisten dengan jalur `findOne()`; perubahan minimal di 4 file; perilaku halaman jadi academicYear-aware penuh.

### Option B: Hapus `thresholdApi` dan pakai `currentDraftThresholdApi` saja

Jika `thresholdApi` ternyata tidak menyediakan data unik yang tidak bisa didapat dari `GET /v1/threshold`, hapus pemakaiannya dan andalkan `currentDraftThresholdApi` yang sudah benar. Perlu verifikasi FE Engineer dulu apakah ada data dari `thresholdApi` yang dipakai untuk fungsi lain (misal dropdown/select threshold lain di halaman).

**Pro:** Menghilangkan sumber bug; **Kontra:** perlu audit semua pemakaian `thresholdApi` di `Threshold.jsx` (dan file lain yang memakai `getTresholdBySubjectId`).

### Opsi Verifikasi Sebelum Fix

Sebelum implementasi, pastikan perilaku yang diharapkan untuk AY tanpa data threshold: apakah halaman harus tampil kosong, atau fallback ke data AY terdekat? (lihat **Edge Cases** di bawah — keputusan ini butuh konfirmasi Product Owner)

---

## Business Rules / Ekspektasi

1. Nilai threshold yang tampil harus **selalu sesuai dengan kombinasi Subject + Academic Year yang sedang dipilih**
2. Mengganti Academic Year harus **memicu reload data threshold** untuk AY baru
3. Jika AY yang dipilih belum memiliki data threshold, halaman harus menampilkan state kosong / pesan yang jelas (bukan menampilkan data AY lain)

---

## Edge Cases (Perlu Keputusan)

| Skenario | Pertanyaan | Keputusan |
|----------|-----------|-----------|
| AY baru dipilih belum punya data threshold | Apakah tampil kosong atau fallback ke AY lain? | **OPEN** — perlu konfirmasi Product Owner |
| Ganti AY saat `subjectId` masih sama | `thresholdApi` harus refetch (deps + param) | Harus refetch dengan `academicYearId` baru |
| `academicYearId` tidak dikirim (opsional di API) | Backend mengembalikan semua AY (perilaku lama) | Biarkan untuk kompatibilitas call-site lain yang memang butuh semua AY |
| Duplikat threshold per subject+AY (status berbeda) | `findBySubjectId` mengembalikan array — halaman harus pilih yang mana? | Perlu cek logika render di `Threshold.jsx` |

---

## Notes

- Bug dilaporkan via Jam recording: https://jam.dev/c/a14dd129-1e23-4441-ac55-0a89f71fe242
- Jalur `findOne()` (`GET /v1/threshold`) **sudah benar** — jangan diubah
- Ada `console.log` debug yang tertinggal di backend (`threshold.service.ts` line 41, 247-248, 260: `'ASKJDKJS'`, `{ options }`, `data`) — sekalian dibersihkan saat implementasi fix
- File ini hanya tiket bug report — **tidak ada perubahan kode** di FE/BE (sesuai aturan READ ONLY di AGENT.md)

## Reference Files (Dibaca)

### Backend — `api_nest/`

| File | Keterangan |
|------|------------|
| `src/modules/threshold/threshold.controller.ts` | `GET /threshold` (line 38-43) & `GET /threshold/subject/:subjectId` (line 45-56) — yang kedua tidak terima `academicYearId` |
| `src/modules/threshold/threshold.service.ts` | `findOne()` line 246-262 (benar, filter AY); `findBySubjectId()` line 264-278 (salah, tanpa filter AY) |
| `src/modules/threshold/dto/find-threshold.dto.ts` | `academicYearId` required (line 7-12) — dipakai jalur `GET /threshold` |

### Frontend — `bbs/`

| File | Keterangan |
|------|------------|
| `client/src/views/threshold/Threshold.jsx` | `useQueryString` line 30-35; `thresholdApi` line ~52-56 (deps `[subjectId]`); `currentDraftThresholdApi` line 72-82 (benar) |
| `client/src/actions/api/threshold.js` | `getThreshold(query)` (line 9-16) & `getTresholdBySubjectId(subjectId)` (line 18-26 — tanpa `academicYearId`) |
| `client/src/hooks/useFromApi.js` | Hook `useFromApi(actionCreator, dependencyList, conditional, ...)` — refetch saat dependency berubah (line 67-73) |
