---
title: Academic Year — Form edit tidak langsung menampilkan weeks 1-10, Add Week menumpuk di Week 1, refresh menghapus tanggal
status: open
severity: major
product: BBS LMS
portal: Admin
author: System Analyst
date: 2026-10-08
jam: https://jam.dev/c/78df0815-4735-4851-8478-59213b65cdb3
---

# Academic Year — Weeks tidak auto-load saat edit, Add Week salah posisi, refresh me-reset tanggal

## Summary

Dari rekaman Jam (akun Ms Lily, halaman Academic Year `/academic-year/:academicYearId/edit`,
komponen `bbs/client/src/views/academicYears/AcademicYearForm.jsx`):

1. **Edit tidak langsung menampilkan weeks 1-10.** Saat form edit dibuka, daftar
   week yang sudah tersimpan (week 1-10) tidak langsung muncul. Ekspektasi:
   buka edit → weeks 1-10 langsung terlihat tanpa aksi tambahan.
2. **Add Week menumpuk di Week 1.** Saat menekan "Add Week", week baru tidak
   tersusun di bawah (week N+1) — Week 1 tetap di atas dan week baru berperilaku
   seolah week 1 lagi (nomor tidak auto-increment, urutan salah).
3. **Refresh menghilangkan semua tanggal.** Setelah refresh browser di halaman
   edit, semua tanggal weeks yang sudah diisi hilang / ter-reset ke kosong.

## Expected Result

- Buka halaman edit Academic Year → seluruh weeks tersimpan (mis. week 1-10
  per term) langsung tampil lengkap dengan tanggalnya.
- Tekan "Add Week" di Term X → satu baris baru muncul **di bawah** week
  terakhir term tersebut dengan nomor berurutan (mis. week 11), Week 1 tetap
  di posisinya di atas.
- Refresh browser → data tersimpan tetap tampil (form diisi ulang dari server),
  tidak ada tanggal yang hilang.

## Actual Result

- Weeks tersimpan tidak langsung muncul saat edit dibuka (butuh trigger / hanya
  muncul sebagian).
- Week baru dari "Add Week" tidak terurut di bawah; Week 1 tetap di atas dan
  penomoran tidak berlanjut.
- Refresh → seluruh rows weeks + tanggal hilang / kembali ke default kosong.

## Root Cause (analisis kode, belum fix)

File: `smartbag/bbs/client/src/views/academicYears/AcademicYearForm.jsx`
(553 baris), hook: `src/hooks/useResourceMapper.js`, API: `src/actions/api/academicYear.js`.

### RC-1: Effect inisialisasi form balapan dengan fetch async (penyebab #1 dan #3)

`AcademicYearForm` **tidak pernah fetch single AY**. Satu-satunya fetch adalah
daftar semua AY (lines 46-49):

```js
const academicYearsApi = useFromApi(
  fromApi.getAcademicYears({ pageSize: 0 }),
  []
);
const academicYear = useSelector(
  (state) => state.academicYears?.[academicYearId]
);
const academicYearWeeks = useResourceMapper(
  "academicYearWeeks",
  academicYear?.academicYearWeeksIds
);
```

Effect pengisi form (lines 85-135) berjalan **setiap kali** `academicYear` atau
`academicYearWeeks` berubah, dan selalu diawali `remove()` (kosongkan seluruh
field array):

```js
useEffect(() => {
  remove();                       // ← selalu clear dulu
  ...
  if (editMode && academicYear) { /* setValue(...) */ }
  if (academicYearWeeks.length) {
    academicYearWeeks.forEach((ayWeek) => { append(ayWeek); });
  } else {
    [1, 2, 3, 4].map((t) => {     // ← fallback: 4 baris default
      append({ weekNumber: 1, term: parseInt(t),
               startDate: undefined, endDate: undefined });
    });
  }
}, [editMode, academicYearsApi.loading, academicYear, academicYearWeeks]);
```

Alurnya saat buka edit / refresh: render pertama `academicYear === undefined`
(masih loading) → effect jalan → `remove()` + append 4 baris default kosong.
Saat data tiba, effect jalan ulang → `remove()` + append data asli. Antara dua
tahap ini user melihat form kosong/default, dan setiap perubahan referensi
`academicYear` / `academicYearWeeks` (object identity baru dari Redux
normalizer) mengulang `remove()` + `append()` — inilah yang terlihat sebagai
"tanggal hilang / kereset", terutama saat refresh (store kosong → tahap default
dulu, dan bila `academicYearWeeksIds` tidak ikut ter-normalize dari response
list, tahap kedua pun tetap fallback default).

### RC-2: Add Week tanpa nomor + sort NaN (penyebab #2)

Tombol Add Week (lines 396-408) append tanpa nomor:

```js
append({ weekNumber: undefined, term: parseInt(t),
         startDate: undefined, endDate: undefined });
```

Render me-sort per term (lines 413-415):

```js
fields
  .filter((f) => f?.term === parseInt(t))
  .sort((a, b) => a?.weekNumber - b?.weekNumber)
```

`undefined - number = NaN` → comparator tidak valid → urutan week baru tidak
terjamin (terlihat "Week 1 tetap di atas", week baru tidak jatuh di bawah).
Input nomor (`FINAControlledNumberInput`, lines 424-434) menampilkan
`watch(...) || 0`, jadi week baru tampil sebagai "0"/kosong, bukan N+1.

### RC-3 (pendukung): `shouldUnregister: true` (line 75)

Dengan `useFieldArray` + `remove()` berulang, field yang sempat ter-unregister
kehilangan value-nya — memperkuat efek "tanggal hilang" dari RC-1.

## Repro Steps

1. Login sebagai Ms Lily (akun non-teacher biasa), buka admin portal.
2. Buka Academic Year → pilih satu AY → klik Edit
   (`/academic-year/:academicYearId/edit`).
3. Perhatikan: weeks 1-10 tidak langsung muncul.
4. Tekan "Add Week" di salah satu term → perhatikan posisi & nomor week baru.
5. Refresh browser → perhatikan seluruh tanggal weeks hilang / reset.

## Scope

### In Scope
- `bbs/client/src/views/academicYears/AcademicYearForm.jsx` — guard effect
  inisialisasi (hanya isi form sekali saat data siap, jangan `remove()` buta),
  auto-number pada Add Week (max+1 per term), sort stabil untuk week baru.
- Pastikan `academicYearWeeksIds` tersedia untuk entity dari response list
  (atau fetch single `GET /academicYears/:id` saat edit).

### Out of Scope
- Perubahan backend (`api_nest` `academic-year` module) — `findAll` sudah
  include relasi `academicYearWeeks`, `findOne` tersedia; kemungkinan tidak
  diperlukan.
- Migrasi database.

## Referensi kode

| File | Keterangan |
|------|------------|
| `bbs/client/src/views/academicYears/AcademicYearForm.jsx` lines 46-62 | Fetch list + selector + `useResourceMapper` weeks |
| `bbs/client/src/views/academicYears/AcademicYearForm.jsx` lines 79-135 | Effect init form (`remove()` + `append`) |
| `bbs/client/src/views/academicYears/AcademicYearForm.jsx` lines 396-408 | Tombol Add Week (`weekNumber: undefined`) |
| `bbs/client/src/views/academicYears/AcademicYearForm.jsx` lines 412-435 | Render rows + sort `weekNumber` |
| `bbs/client/src/actions/api/academicYear.js` lines 26-51 | `getAcademicYears`, `getAcademicYear` (single tidak dipakai form) |
| `bbs/client/src/views/academicYears/AcademicYearDetails.jsx` lines 30-42 | Halaman detail (pembanding: fetch single + mapper weeks) |
| `api_nest/src/modules/academic-year/academic-year.service.ts` lines 101-103 | `findAll` include `academicYearWeeks: true` |

## Open Questions

- Apakah portal yang benar Admin atau Teacher? Kode form AY hanya ada di
  `bbs/client` (admin portal) — konfirmasi Ms Lily membuka portal admin.
- Berapa weeks per term yang diharapkan (10)? Apakah nomor week unik per term
  atau global per AY? (Menentukan rumus auto-number Add Week.)
- Apakah week boleh dihapus (tombol trash lines 505-512) atau hanya tambah?
