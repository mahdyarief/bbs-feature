---
title: Loading Lesson Plan di Teacher Portal terkadang sangat lama (gap belasan detik, request utama tidak ke-fire)
status: open
severity: major
product: BBS LMS
portal: Teacher
author: System Analyst
date: 2026-10-02
jam: https://jam.dev/c/eef7b751-1fe5-4259-82ec-a6d6ee073fb7
---

# Loading Lesson Plan di Teacher Portal terkadang sangat lama

## Summary

Pada **Teacher Portal**, halaman **Lesson Plan** (`/lesson-plan`) dan **Lesson Plan Library** (`/lesson-plan/library`) **terkadang** sangat lambat memuat — guru melihat spinner lama sekali (belasan detik). Analisis Jam.dev menunjukkan **bukan API-nya yang lambat**: semua response yang ke-fire cepat (76–130ms), tetapi **request utama list lesson-plan (`GET /api/v1/lesson-plans`) tidak pernah muncul di rekaman** dan terdapat **gap ±12 detik tanpa aktivitas network sama sekali** (5.4s → 18.1s di rekaman).

**Ekspektasi:** List lesson plan muncul dalam hitungan detik setelah navigasi; data utama di-fetch segera tanpa menunggu chain dependensi, dan request yang sama tidak di-refetch dari nol di setiap pindah halaman.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | System Analyst |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | — |
| Portal URL | https://teacher.smartbag.binabangsaschool.com/lesson-plan |
| Environment API | teacher.smartbag.binabangsaschool.com |
| Data konteks | Rekaman 35 detik, klik ke dropdown filter lalu pindah ke Library |
| Browser / OS | Chrome (4G, ~10Mbps, RTT 50ms) |

---

## Steps to Reproduce

1. Login ke **Teacher Portal**.
2. Buka menu **Lesson Plan** (`/lesson-plan`).
3. Perhatikan waktu sampai list lesson plan tampil — terkadang spinner berputar belasan detik.
4. Pindah ke **Library** (`/lesson-plan/library`) dan perhatikan kembali.

**Actual Result (dari Jam):**
- 3.7s: navigasi ke `/lesson-plan` → batch request pertama (`employeePositions/self`, `employees/{id}`, `headDepartments`, `academicYears` **2× identik**) + lazy-load chunk `LessonPlan.js`.
- 5.4s: user klik filter → **tidak ada aktivitas network selama ±12.7 detik** (gap 5.4s → 18.1s).
- 18.1s: user menyerah, klik Library → **semua request di-refetch dari nol**.
- 24.6s: `GET /lesson-plans/file-library` muncul — responsenya hanya **126ms**.
- **`GET /api/v1/lesson-plans` (data utama) tidak pernah muncul** selama rekaman.
- Tiap GET didahului preflight `OPTIONS` CORS (roundtrip ganda).

**Expected Result:** Request list utama fire segera setelah `academicYearId` tersedia; tidak ada gap zero-network; refetch tidak terjadi di setiap navigasi.

---

## Affected Files / Root Cause Analysis

### Frontend — `bbs/client-teacher` (penyebab utama gap)

1. **`src/hooks/useFromApi.js:9-66`** — wrapper `useEffect + dispatch` tanpa cache/dedup/react-query. Setiap mount = 1 request baru; nilai `conditional()` (return value) ikut di dependency array → retrigger liar.
2. **Waterfall yang mem-gate list utama** — `src/views/lessonPlan/LessonPlan.jsx:63-90`:
   ```js
   listApi = useFromApi(getLessonPlans({...}), [..., subjectYears.length],
                        () => academicYearId && subjectYears.length)
   ```
   Request list **tidak bisa fire sebelum** `teacherId` → `academicYearId` → `subjectYears` resolve **berurutan**. Ini penyebab `GET /lesson-plans` tidak muncul / sangat akhir di Jam. Pola sama di `LessonPlanLibrary.jsx`.
3. **Chain user/role refire tiap navigasi** — `src/containers/TheSidebar.jsx:56,63` + `src/containers/TheContent.jsx:27` memanggil `useSecondaryTeacher()` / `usePrincipalOrHod()` (2 consumer). Hook berantai: `getSelfEmployeePositions()` → `getEmployee(id, roles)` → `getHeadDepartments({teacherId})` → di-fetch ulang tiap pindah route.
4. **`academicYears` dobel** — tiap halaman render **2 filter select** dengan `pluralAction={fromApi.getAcademicYears}` (`LessonPlan.jsx:~120-180`, `LessonPlanLibrary.jsx:~52-160`); `src/components/BBSResourceSelect.jsx:110-117` membuat 1 `useFromApi` per instans tanpa shared cache → 2 GET identik. `TheLayout.jsx:134-138` menambah call ketiga (`getCurrentAcademicYear`).
5. **Render diblokir** — `LessonPlan.jsx:180` dan `LessonPlanLibrary.jsx:206`: tabel menampilkan spinner sampai seluruh chain query terakhir resolve.
6. **Lazy chunk dingin** — `src/routes.js`: `LessonPlan.js`, `LessonPlanLibrary.js`, `LessonPlanFileLibrary.js`, `lessonPlanUtils.js`, `lessonPlanConstants.js` semua `React.lazy` → download+parse tiap kunjungan pertama (gap tanpa traffic API).
7. **CORS preflight tiap request** — `src/actions/makeApiRequest.js`: `fetch` dengan header `Authorization` custom ke API cross-origin → setiap GET didahului `OPTIONS`.
8. **Render berat** — `CDataTable` tanpa virtualisasi; `JSON.stringify(relationsObj/selectsObj)` per render.

### Backend — `api_nest` (penyebab "terkadang" / tail latency)

9. **`pageSize=0` = seluruh tabel** — `src/helpers/find-options-helper.ts:~30-60`: `pageSize=0/undefined` → `take` undefined (tanpa limit) → `academicYears/masterProgrammes/masterLevels/subjects?pageSize=0` fetch + `COUNT(*)` **seluruh tabel** tiap navigasi.
10. **`findAndCount` + eager relations berat** — `src/modules/lesson-plan/lesson-plan.service.ts:417-445, 795-850`: `LIST_RELATIONS` (teacher → academicYear → classSubject → subject → classYear → classroom → masterLevel) di-join + `COUNT(*)` tiap halaman; `decorate()` melakukan `JSON.parse/stringify` per baris.
11. **File library query berat** — `src/modules/lesson-plan/lesson-plan-file-library.service.ts:58-70`: 8 `leftJoinAndSelect`; search `:77-84` = 5× `ILIKE '%:search%'` (leading wildcard, tak terindeks); `getCount()` + `getMany()` = dobel query.
12. **Overhead auth per request tanpa cache** — `src/main.ts:39-42` global `JwtAuthGuard` + `PermissionsGuard`; `src/helpers/auth.helper.ts:24+` `validateTokenPassport()` lakukan `findOne` ke DB **di tiap request**; `src/modules/casl/permission.guard.ts:20-43` bangun ability baru per request; `src/helpers/campus-permit.helpers.ts:9-45` query `Campus` tambahan. Buka halaman = ~7-10 request × beberapa query DB masing-masing.
13. **`TransformResponseInterceptor` O(n²)** — `src/interceptors/transform-response.interceptor.ts:19-54`: `flattenData + _.uniqWith(included, _.isEqual)` — quadratic pada payload eager-loaded besar.
14. **Pool DB kecil + tanpa slow-query log** — `src/shared/config/database.service.ts:44-78`: tanpa `poolSize` → default node-pg **~10 koneksi**; `logging:false` + `maxQueryExecutionTime:8000` → query < 8 detik tak pernah tercatat. Fan-out ~8-10 request paralel bisa **antri di pool** → stall sesekali sementara p50 tetap 76-130ms.
15. **CORS tanpa `maxAge` + tanpa throttler** — `src/main.ts:30` `cors:true` (reflect origin) → preflight OPTIONS diulang tiap URL berbeda.

---

## Impact

- Guru menunggu belasan detik dengan spinner tanpa tahu kapan selesai; sering menyerah dan berpindah halaman (terlihat di Jam).
- Setiap pindah halaman mengulang seluruh request dari nol (~10 request + preflight) → beban server & DB membengkak tanpa perlu.
- Dropdown `pageSize=0` memuat seluruh tabel berulang-ulang → memperberat pool DB, menciptakan latensi intermittent untuk semua user.
- UX buruk → keluhan "Lesson Plan lemot" yang tidak reprodukibel (karena tergantung antrean pool).

---

## Saran Solusi

**Quick wins:**
1. **Putus waterfall** — `LessonPlan.jsx:63-90` & `LessonPlanLibrary.jsx`: ubah gate list menjadi hanya `() => !!academicYearId`; jangan tunggu `subjectYears.length` (filter di client / refetch saat subjectYears siap).
2. **Cache/dedup di `useFromApi`** — tambah cache Map + in-flight dedup (atau migrasi ke react-query dengan `staleTime`), agar request tidak di-refetch tiap mount.
3. **`academicYears` cukup sekali** — fetch di parent lalu pass sebagai prop ke kedua select di `BBSResourceSelect`.
4. **Backend: cap `pageSize=0`** — `find-options-helper.ts`: `take` tidak boleh undefined; beri default max (mis. 100).
5. **Naikkan pool + slow-query log** — `database.service.ts`: set `extra.max` (20-30), turunkan `maxQueryExecutionTime` (mis. 200ms), nyalakan logging selektif.

**Medium:**
6. Hoist `useSecondaryTeacher()` / `usePrincipalOrHod()` ke context/redux dengan TTL (5-10 menit) — hilangkan ~6 request berulang per halaman.
7. Same-origin reverse proxy `/api` (hilangkan CORS) atau set `cors: { maxAge: 86400 }` di `main.ts`.
8. Ringankan query file-library (kurangi join, index `pg_trgm` untuk ILIKE, hindari count ganda) + index kolom filter/sort lesson-plan.
9. Ganti `_.uniqWith(_.isEqual)` di `TransformResponseInterceptor` dengan dedup berbasis key (Map/Set by id).
10. Cache auth/permission di Redis (TTL pendek) — `validateTokenPassport`, `checkCampusesPermit`.
11. Prefetch route chunk `LessonPlan*` (webpackPrefetch) + virtualisasi `CDataTable`.

---

## Acceptance Criteria

- [ ] `GET /api/v1/lesson-plans` (list utama) fire dalam < 1 detik setelah navigasi, **tanpa** menunggu `subjectYears` resolve.
- [ ] Tidak ada gap zero-network > 3 detik antara navigasi dan munculnya data saat kondisi normal.
- [ ] `employeePositions/self`, `employees/{id}`, `headDepartments`, `academicYears` **tidak** di-refetch di setiap pindah halaman dalam satu sesi.
- [ ] `academicYears` hanya di-fetch **satu kali** per halaman (bukan 2× identik).
- [ ] `pageSize=0` pada endpoint dropdown membatasi jumlah row (ada cap), bukan seluruh tabel.
- [ ] Preflight OPTIONS tidak terjadi per request (same-origin proxy atau `maxAge` di-set).
- [ ] Tidak ada regresi pada filter (academic year, subject) dan pagination Lesson Plan / Library / File Library.
