# API Contract — Threshold & Syllabus AY Scoping

Endpoints yang terlibat. Tidak ada endpoint baru yang wajib; satu parameter opsional
(`academicYearId` pada `GET /threshold/subject/:subjectId`) menunggu keputusan EC-04.

Base path: `/api/v1`

---

## GET `/threshold`

Mengambil satu threshold untuk AY + subject (+ optional status/type).

**Query**

| Param | Tipe | Wajib | Keterangan |
|-------|------|-------|------------|
| academicYearId | number | ya | AY target. |
| subjectId | number | ya | Subject. |
| thresholdStatus | enum `IThresholdStatusType` (0=DRAFT, 1=ACTIVE) | tidak | Filter status. |
| thresholdType | enum `ThresholdTypeEnum` (`IGCSE_ALEVEL`, `AS_PRELIM`) | tidak | Filter tipe. |

**Impl:** `threshold.controller.ts:38-43` → `threshold.service.ts:246-266` (`findOne`).

**Perbaikan yang diperlukan**
- `findOne` saat ini tidak menyertakan filter `deletedAt`. TypeORM `findOne` harus
  mengecualikan soft-deleted secara default **hanya jika** entity memakai `@DeleteDateColumn`.
  Verifikasi `Threshold` entity; bila memakai `softRemove` tetapi tidak punya
  `@DeleteDateColumn`, `findOne` akan ikut mengembalikan baris terhapus → harus difilter
  eksplisit (`deletedAt: IsNull()`).
- Buang `console.log` debug (`threshold.service.ts:247-248,264`).

**Response 200**

```json
{
  "data": {
    "id": 123,
    "academicYearId": 42,
    "subjectId": 100003,
    "thresholdType": "IGCSE_ALEVEL",
    "statusType": 0,
    "effectiveFrom": 2025,
    "effectiveTo": 2025,
    "avgValue": { "a*": 90, "a": 80, "...": 0 },
    "thresholdConfig": [ { "year": 2022, "syllabusId": 9001 } ]
  }
}
```

---

## GET `/threshold/subject/:subjectId` (opsional berubah)

**Impl:** `threshold.controller.ts:45-56` → `threshold.service.ts:276-301`.

Perilaku saat ini: mengembalikan **semua** threshold subject (optional filter type),
dengan baris yang cocok `academicYearId` diurutkan paling depan. Dikonsumsi oleh:
- UI `views/threshold/Threshold.jsx`, `ThresholdAsPrelim.jsx` (via `getTresholdBySubjectId`).
- Gradebook `student-cambridge-grade.service.ts` (memanggil service langsung dengan
  `classYear.academicYearId`, lalu `find(ACTIVE)`).

**Perubahan opsional (EC-04 opsi B)**

| Param | Tipe | Wajib | Keterangan |
|-------|------|-------|------------|
| thresholdType | enum | tidak | Sudah ada. |
| academicYearId | number | tidak | Bila ada, hanya kembalikan baris AY ini. |

Default (tanpa `academicYearId`) tetap: semua AY, AY terpilih di depan. Gradebook tidak
terpengaruh.

---

## POST `/threshold`

Membuat draft threshold untuk AY + subject.

**Body**

```json
{ "academicYearId": 42, "subjectId": 100003, "thresholdType": "AS_PRELIM" }
```

**Impl:** `threshold.controller.ts:24-29` → `threshold.service.ts:40-244` (`create`).

**Perbaikan yang diperlukan**
1. Baris 79-116: `syllabi` bisa memuat `undefined`. Sebelum membuat `ThresholdConfig`,
   filter: `syllabi.filter(Boolean)`.
2. Baris 47,55: `return NoSubjectFoundError()` / `return NoAcademicYearFoundError()`
   seharusnya `throw` (EC-05) agar response benar-benar error.
3. Jika setelah filter `syllabi` kosong (tidak ada syllabus sama sekali untuk AY itu),
   kembalikan 400 dengan pesan jelas (lihat spec Error Handling) daripada membuat
   threshold dengan 0 config.
4. Buang `console.log({ options })` (baris 41).

**Response 200**

```json
{ "data": { "id": 124, "academicYearId": 42, "statusType": 0, "avgValue": { "...": 0 } } }
```

---

## POST `/threshold/active`

Mengaktifkan draft threshold: men-soft-delete ACTIVE sebelumnya untuk (AY, subject, type),
lalu menyimpan salinan ACTIVE.

**Body**

```json
{ "thresholdId": 124 }
```

**Impl:** `threshold.controller.ts:31-36` → `threshold.service.ts:317-381`.

**Perbaikan yang diperlukan**
- Baris 362-376: `activeThresholdConfigPayload` bisa berisi entri dengan `syllabusId`
  `undefined`. Filter entri tersebut sebelum `save`.
- Jika tidak ada config valid, tolak aktivasi (EC-03) dengan error yang jelas.
- Baris 327: `if (!draftThreshold) NoThresholdFoundError();` tidak me-*throw*; perbaiki.

---

## GET `/syllabus/bySubject/:subjectId`

**Impl:** `syllabus.controller.ts:102-126` → `syllabus.service.ts:128-140`
(`findAllBySubject`). Filter: `subjectId` + optional `syllabusType`. **Tidak** berubah —
syllabus berlaku berdasarkan rentang `effectiveFrom`/`effectiveTo`, bukan satu AY.

**Query**

| Param | Tipe | Wajib | Keterangan |
|-------|------|-------|------------|
| syllabusType | 1=IGCSE, 2=ASLEVEL, 3=ALEVEL | tidak | Filter program. |

**Response 200**

```json
{
  "data": [
    {
      "id": 9001,
      "subjectId": 100003,
      "academicYearId": 42,
      "year": 2025,
      "effectiveFrom": 2022,
      "effectiveTo": 2024,
      "syllabusType": 2,
      "syllabusComponentIds": [71, 72],
      "syllabusComponent": [ { "id": 71, "type": "AVERAGEVALUE", "values": {} } ]
    }
  ]
}
```

---

## POST `/syllabusComponent/updateGroup`

Menyimpan komponen (weight / marks / grade) untuk satu grup komponen syllabus.

**Impl:** `syllabus-component.controller.ts` → `updateComponentGroup` service.

**Body (dari `ThresholdSyllabusPart.handleSave`)**

```json
{
  "syllabusId": 9001,
  "rowIndex": 0,
  "year": 2022,
  "components": [
    { "key": "Weight", "val": 50 },
    { "key": "Components", "val": "P1" },
    { "key": "Total Marks", "val": 100 },
    { "key": "a*", "val": 90 }
  ],
  "thresholdType": "AS_PRELIM"
}
```

**Perbaikan yang diperlukan (client):** `rowIndex` harus berasal dari posisi baris yang
diedit (mis. dari render index), bukan dari destructuring baris yang tidak pernah berisi
`rowIndex` (`ThresholdSyllabusPart.jsx:196-231`). Kontrak server tetap; yang diperbaiki
adalah nilai `rowIndex` yang dikirim.
