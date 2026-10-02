# Notes — Subject Folder Auto-Load

## Root Cause Analysis (kode tidak diubah — analisis saja, sesuai instruksi)

### 1. Fetch gate di `useFromApi` — `bbs/client-teacher/src/hooks/useFromApi.js` (lines 60-66)

```js
useEffect(() => {
  if (conditional()) {
    fetch();
  } else {
    setLoading(false);
  }
}, [dispatch, conditional(), refreshState, ...dependencyList]);
```

Argumen ke-3 `conditional()` adalah gerbang: jika false saat mount, **tidak ada request sama sekali** — hook hanya `setLoading(false)`. Ini mekanisme umum hook ini (dipakai banyak halaman), jadi perbaikannya ada di pemakaian di `SubjectFiles.jsx`, bukan di hook-nya.

### 2. Pemakaian di `SubjectFiles.jsx` — `bbs/client-teacher/src/views/fileSharing/SubjectFiles.jsx`

**Files (lines 64-72)** — digate `activeSubject` (pill subject belum dipilih saat halaman dibuka):

```jsx
const sfApi = useFromApi(
  fromApi.getSubjectFiles({
    ...query,
    pageSize: 0,
    subjectId: activeSubject?.id
  }),
  [...Object.values(query), activeSubject?.id],
  () => activeSubject          // ← false di awal → fetch dilewati
);
```

**Subjects (lines 81-90)** — digate `masterLevelId` (dropdown level belum dipilih):

```jsx
const subjectApi = useFromApi(
  fromApi.getSubjects({
    ...query,
    pageSize: 0,
    breakthrough: true,
    teacherId: selfUser?.id
  }),
  [...Object.values(query)],
  () => masterLevelId          // ← false di awal → fetch dilewati
);
```

**Render gate (line 147):** `{masterLevelId && subjects.map(...)}` — pills subject tidak dirender sama sekali sebelum level dipilih, sehingga user tidak bisa memilih subject tanpa memilih level dulu.

Akibat alur wajib: pilih level → fetch subjects → muncul pills → klik pill → fetch files → tabel terisi. Dua gerbang berurutan ini yang membuat data tidak langsung muncul.

### 3. Backend tidak menghalangi (semua opsional)

- `GET /api/v1/subjectFile` → `GetSubjectFileDto` (`api_nest/src/modules/subject-file/dto/get-subject-file.dto.ts`): `fileName`, `subjectId`, `level`, `masterLevelId`, `createdByTeacherId`, `teacherName` — **semua `@IsOptional()`**. Tanpa parameter pun valid → return semua subject files.
- Controller `subject-file.controller.ts` `findAll(@Query() options)` → `service.findAll(options)` tanpa validasi wajib.
- `GET /api/v1/subjects` → `GetSubjectDto` mendukung `academicYearId` (line 25), `breakthrough` (line 108), `teacherId` (line 114). `subject.service.ts`:
  - line 175: `if (options.teacherId)` → scope ke subjects yang diajar guru tersebut
  - line 237 (komentar eksisting): "`options.breakthrough` used in order to pass the headDepartments check n masterSubject coordinator check, **used on the subject-folder feature in teacher-portal**"

Jadi fetch tanpa filter level/subject sudah didukung API — perubahan murni frontend (hapus/ubah argumen `conditional` + render gate).

## Referensi pola serupa

- `features/bmt-files-subject-filter-race-condition/bug-report.md` — race condition akibat fetch bergantung pada dependency yang belum resolve (mirip pola "fetch menunggu input filter").
- `features/sve-assignment-subject-filter-inconsistency/` — inkonsistensi filter subject.

## Data verifikasi (DB `binabangsa_prod_mig_v01`)

Untuk akun contoh **elena.kjp** (`employee.id 20854`), AY **2026/2027** (`academic_year.id 27`):

| class_year | classroom | Subjek |
|---|---|---|
| 100931 | Primary 4 Hope (id 37) | English Language (subject id 408) |
| 100932 | Primary 4 Faith (id 36) | English Language (subject id 408) |

Catatan: rows pivot lain miliknya (subject_year 101575/101591 → class_year 100111/100112) berada di `academic_year_id 26` (2025/2026), bukan tahun berjalan — dengan auto-load, kedua folder 2026/2027 ini langsung tampil tanpa filter.

## Open questions

- Lihat `edgecases.md` EC-01..EC-06 (semua masih _TBD_) — terutama EC-03 (cara menentukan AY berjalan di fetch awal) menentukan apakah ada perubahan query param.
