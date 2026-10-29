# Bug Report: LO Library Subject Filter Menampilkan Subject Secondary

## Metadata
- **Reported by:** User
- **Date:** 2026-10-29
- **Severity:** Medium
- **Module:** Learning Outcomes (LO) — LO Library (Learning Indicators)
- **Environment:** Admin Portal (subject library resource)

## Deskripsi Bug
Pada LO Library, kolom/filter **Subject** menampilkan **semua subject** termasuk **secondary subject**. Semestinya hanya **primary subject** yang ditampilkan, karena LO (Learning Outcomes) hanya diperuntukkan untuk primary subject, dan secondary subject tidak relevan (tidak terbawa).

## Steps to Reproduce
1. Buka halaman **LO Library** di Admin Portal.
2. Perhatikan kolom **Subject** dan filter single-select subject-nya.
3. Filter dropdown menampilkan **primary dan secondary subject** sekaligus.
4. Secondary subject yang muncul di filter tetap bisa dipilih, padahal LO hanya berlaku untuk primary subject.

## Expected Behavior
- Kolom/filter **Subject** pada LO Library hanya menampilkan **primary subject** (`category = PRIMARY`).
- Secondary subject (`category = SECONDARY`) **tidak terbawa** ke filter LO, karena LO hanya diperuntukkan untuk primary subject.

## Actual Behavior
- Filter dropdown menampilkan **semua subject (primary + secondary)**.
- Tidak ada `category` filter yang dilewatkan dari frontend ke backend, sehingga backend mengembalikan seluruh subject tanpa pembatasan kategori.

## Root Cause Analysis

### Frontend — `bbs/client/src/views/learningOutcomes/LearningOutcomes.jsx` (kolom subject)
```jsx
{
  key: "subjectId",
  title: "Subject",
  dataType: DataTypes.SINGLE_SELECT,
  reducerName: "subjects"
}
```
- Tidak ada `query`/`category` yang dilewatkan untuk column filter. `BBSResourceSelect` memanggil `get subjects` → `GET /subjects` tanpa parameter filter, sehingga backend mengembalikan semua subject.

### Backend — `api_nest/src/modules/subject/subject.service.ts`
- Method **`findAll()`** (lines ~321-324) **mendukung** filtering berdasarkan kategori:
  ```typescript
  if (options.category) {
    validateSubjectCategoryTypeEnum(options.category);
    whereFilters.category = SubjectCategoryTypeEnum[options.category];
  }
  ```
  namun `category` tidak pernah dipanggil dari frontend LO.
- Method **`findAllSubjectBySubjectLibrary()`** (lines ~383-409), yang dipakai LO untuk fetch subject by subject library, **tidak menerapkan** category filter — hanya `libraryOnly`.

### Backend DTO — `api_nest/src/modules/subject/dto/get-subject.dto.ts`
- Field `category` (baris 84-86) sudah ada dan menerima `SubjectCategoryTypeEnum` (`PRIMARY`/`SECONDARY`), sehingga kontrak API mendukung filter kategori — hanya tidak pernah dilewatkan oleh frontend LO.

## Saran Fix

### Option A (Frontend only — Recommended)
Pada `LearningOutcomes.jsx`, tambahkan `category: "PRIMARY"` ke query filter kolom Subject:
```jsx
{
  key: "subjectId",
  title: "Subject",
  dataType: DataTypes.SINGLE_SELECT,
  reducerName: "subjects",
  query: { category: "PRIMARY" }
}
```
**Efek:** request ke `/subjects?category=PRIMARY` (atau via `findAllSubjectBySubjectLibrary`), sehingga dropdown hanya menampilkan primary subject.

### Option B (Backend — hardcode LO-only)
Di `subject.service.ts` → `findAllSubjectBySubjectLibrary()`, selalu terapkan `whereFilters.category = SubjectCategoryTypeEnum.PRIMARY` (karena LO hanya berlaku untuk primary subject).

### Option C (Keduanya)
Frontend pass `category: PRIMARY` (transparan, reusable untuk non-LO context) **dan** backend mempertahankan validasi `validateSubjectCategoryTypeEnum` (sudah ada).

## Files Terkait
| Layer | File | Peran |
|---|---|---|
| Frontend | `bbs/client/src/views/learningOutcomes/LearningOutcomes.jsx` | halaman LO Library — subject column filter |
| Frontend | `bbs/client/src/components/BBSResourceSelect.jsx` | component render filter dropdown subject |
| Frontend | `bbs/client/src/actions/api/subject.js` | `getSubjects` action |
| Backend | `api_nest/src/modules/subject/subject.service.ts` | `findAll()` (category filter) + `findAllSubjectBySubjectLibrary()` (tanpa category filter) |
| Backend | `api_nest/src/modules/subject/dto/get-subject.dto.ts` | parameter `category` |
| Backend | `api_nest/src/types/enums/subject-category-type.ts` | enum `SubjectCategoryTypeEnum` (PRIMARY/SECONDARY) |

## Notes
- Bug bersifat **filter leak**, bukan mutasi data — secondary subject tidak seharusnya tampil di filter LO.
- LO (Learning Outcomes) hanya diperuntukkan untuk primary subject, sehingga secondary subject yang ikut tampil di dropdown adalah data noise / risk of mis-assignment.
- Sebaiknya fix di filter side (frontend/backend) agar konsisten, bukan di data side.
```, "file_path">
/Users/arielwirawan/Documents/Gawe/bbs-feature/LO filter bug/bug-report.md