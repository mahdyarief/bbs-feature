---
title: Homeroom Class — Daily Attendance Menampilkan Data Berbeda antara "Check Attendance" (tombol atas) dan Alur Table
status: open
severity: major
product: BBS LMS
portal: Teacher
author: System Analyst
date: 2026-10-21
jam: https://jam.dev/c/70b2e5de-49e0-4e20-807e-82b7e0f59a55
---

# Homeroom Class — Daily Attendance Konsistensi Data antara "Check Attendance" dan Table

## Summary

Pada halaman attendance homeroom class (Teacher portal), ketika teacher menekan tombol
**"Check Attendance"** di bagian atas, data kehadiran yang tampil **sudah benar** (terisi
dan sinkron dengan data yang sudah pernah diinput). Namun ketika teacher membuka
**daily attendance lewat alur berbeda** — yaitu memilih baris tanggal di tabel daftar
hari (ikon pensil) atau memuat halaman dengan `daId` yang berbeda — data yang tampil
**berbeda**: kosong / berbeda jumlah present-late-absent.

Investigasi menemukan akar masalahnya: **dua alur tersebut me-resolve `daily_attendance` row
yang berbeda untuk hari yang sama**. Pada rekaman, alur tombol atas memakai
`daId=34723`, sedangkan alur tabel memakai `daId=34724` — dua baris `daily_attendance`
terpisah untuk `class_year_id=100913` pada tanggal yang sama. Hal ini dimungkinkan karena
tabel `daily_attendance` **tidak punya unique constraint** pada `(class_year_id, date)` dan
`DailyAttendanceService.create()` tidak melakukan find-before-create maupun transaksi —
sehingga setiap kali frontend gagal menebak `daId` yang benar, sistem **membuat baris baru
(kosong)** yang langsung dipakai untuk menyimpan penandaan kehadiran.

**Ekspektasi:** kedua alur harus menunjuk pada **baris `daily_attendance` yang sama** untuk
`(class_year_id, date)` yang sama, sehingga data kehadiran yang tampil identik.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Ariel Wirawan Saputra (teacher homeroom) |
| Email (Jam account) | — (tidak terekspos di rekaman) |
| Jam author ID | `a91848ae-261f-4f41-aa59-c6c397fff39f` |
| User ID (dari console log, mis. `selfUser`) | — |
| Portal URL | https://teacher.smartbag.binabangsaschool.com/classrooms/100913?tab=attendance&term=2 |
| Environment API | api.binabangsaschool.dev |
| Data konteks (class ID / daId / tanggal) | class_year_id=**100913**; term=2; alur tombol atas → `daId=34723`; alur tabel → `daId=34724` |
| Browser / OS | — (reaman berbasis network event) |

---

## Steps to Reproduce

1. Login ke Teacher portal, buka homeroom class `100913` → tab **Attendance**.
2. Pastikan term yang dipilih **Term 2** (term aktif saat rekaman).
3. **Alur A (data benar):** klik tombol **"Check Attendance"** di kartu atas
   (bagian "Today"). Layar penandaan per siswa terbuka dengan data kehadiran yang
   **sudah terisi** dan benar. Network: `GET /api/v1/attendances?pageSize=0&dailyAttendanceId=34723&isFirstInserted=true&activeStatus=ACTIVE`.
4. Tekan **Back** (`handleBack()` → `daId` di-reset ke `null`).
5. **Alur B (data berbeda):** pada tabel daftar hari di bawah, klik ikon pensil
   (✏️) pada baris tanggal yang sama. Layar penandaan terbuka dengan
   `daId` yang **berbeda** (34724) dan data **berbeda/kosong**.
   Network: `GET /api/v1/attendances?pageSize=0&dailyAttendanceId=34724&isFirstInserted=true&activeStatus=ACTIVE`.
6. Verifikasi di DB bahwa `daily_attendance` memuat **lebih dari satu baris** untuk
   `(class_year_id=100913, date=<tanggal yang sama>)`.

**Actual Result:**
- Dua alur untuk hari yang sama menunjuk dua `daily_attendance` row berbeda
  (`34723` vs `34724`).
- Baris yang baru dibuat memiliki `present_count`/`late_count`/`absent_count` = 0 dan
  `daily_attendance_status = INCOMPLETE`, padahal baris lama sudah lengkap.
- Riwayat presence siswa bisa terpecah: sebagian data di `daily_attendance` lama, sebagian
  di yang baru — `daily_attendance` yang "yatim" tidak pernah terpakai lagi.
- Jumlah hari kehadiran pada agregat (`onClassYear`) bisa terhitung dobel.

**Expected Result:**
- `daily_attendance` ** unik per `(class_year_id, date)`**.
- Kedua alur memakai `daId` yang sama untuk hari yang sama.
- Membuka daily attendance lewat alur mana pun menampilkan data kehadiran yang identik.

---

## Root Cause Analysis

### Bug #1 (akar masalah utama) — `daily_attendance` tidak punya unique constraint pada `(class_year_id, date)`

Entity hanya mendeklarasikan index **non-unique**:

```typescript
// api_nest/src/modules/daily-attendance/entities/daily-attendance.entity.ts:17-29
@Index(['classYearId', 'date'])            // ← TIDAK unique
export class DailyAttendance extends BaseEntityWithDatesAndEmployee {
  @Index() @Column({ name: 'class_year_id', type: 'int', nullable: true }) classYearId: number;
  @Column({ type: 'date', default: () => 'CURRENT_DATE' }) date: Date;
```

```sql
-- api_nest/src/database/migrations/1768879728475-AddIndexesForDailyAttendance.ts:10,13
CREATE INDEX "IDX_17e16881fa40246cd8faaebe88" ON "daily_attendance" ("date");
CREATE INDEX "IDX_9c540b3c5b5dd1b48c7fd734b7" ON "daily_attendance" ("class_year_id", "date");
-- ^ keduanya NON-UNIQUE
```

Grep `UNIQUE` di seluruh `api_nest/src/database/migrations` tidak menemukan satu pun unique
index pada `daily_attendance`. Akibatnya **dua baris dengan `(class_year_id, date)` yang sama
sangat mungkin dan permanen** di DB.

### Bug #2 (pemicu) — `create()` tidak idempotent: tidak ada find-before-create, tidak ada transaksi

```typescript
// api_nest/src/modules/daily-attendance/daily-attendance.service.ts:40-75
async create(options: CreateDailyAttendanceDto) {
  const employee = this.req.user as Employee;
  const daAttendance = new DailyAttendance();
  if (options.skipChecks) { daAttendance.classYearId = options.classYearId; }
  else {
    const classYear = await ClassYear.findOne({ where: { id: options.classYearId } });
    if (!classYear) NoClassYearFoundError();
    daAttendance.classYear = classYear; daAttendance.classYearId = classYear.id;
  }
  daAttendance.date = options.date;                                  // line 55
  daAttendance.presentCount = options.presentCount || 0;            // lines 56-60 → default 0
  daAttendance.lateCount = options.lateCount || 0;
  daAttendance.absentCount = options.absentCount || 0;
  daAttendance.absentWithoutExcuseCount = options.absentWithoutExcuseCount || 0;
  ...
  await daAttendance.save();                                          // line 73 → INSERT baru, selalu
  return daAttendance;
}
```

Tidak ada pengecekan "sudah ada `daily_attendance` untuk class+tanggal ini?", tidak ada
`@Transactional`, tidak ada `upsert`/`ON CONFLICT`. Setiap POST `POST /api/v1/dailyAttendances`
untuk class+tanggal yang sama **selalu** menghasilkan baris baru dengan counter nol.

DTO-nya juga tidak memaksa keunikan:

```typescript
// api_nest/src/modules/daily-attendance/dto/create-daily-attendance.dto.ts
class CreateDailyAttendanceDto {
  @IsNotEmpty() @IsNumber() classYearId: number;
  @IsNotEmpty() @IsDateString() date: string;
  @IsOptional() @IsEnum(DailyAttendanceStatusEnum) dailyAttendanceStatus: ...;
  @IsOptional() @Min(0) presentCount?: number;   // dst.
}
```

### Bug #3 — frontend membuat baris baru setiap kali `daId` gagal di-resolve

```jsx
// bbs/client-teacher/src/views/classrooms/ClassroomDetailsAttendances.jsx:223-242
function handleChangeActiveDate(dateStr, daId) {
  if (!daId) {                                    // ← falsy (null / undefined) = CREATE
    dispatch(
      fromApi.createDailyAttendance({
        classYearId,
        date: moment(dateStr).endOf("day").toDate()
      })
    ).then((res) => {
      dailyAttendancesApi.refresh();
      currentDailyAttendancesApi.refresh();
      setQuery({ ...query, daId: res.data[0].id });
    });
  } else {
    setQuery({ ...query, daId });
  }
}
```

Fungsi ini dipakai oleh **dua** trigger, dan keduanya bisa mengirim `daId` falsy:

**(a) Tombol "Check Attendance" (atas)** — `daId` diambil dari query **UTC day**:

```jsx
// ClassroomDetailsAttendances.jsx:166-190
const today = useMemo(() => moment().utc(), []);           // ← UTC
const todayStartISO = useMemo(() => today.startOf("day").toISOString(), [today]);
const todayEndISO   = useMemo(() => today.endOf("day").toISOString(),   [today]);
const currentDailyAttendancesAction = useMemo(() =>
  fromApi.getDailyAttendances({
    pageSize: 1, startDate: todayStartISO, endDate: todayEndISO,
    classYearId, sortBy: "-date"
  }), [todayStartISO, todayEndISO, classYearId]);
const currentDailyAttendance = useResourceMapper(
  "dailyAttendances", currentDailyAttendancesApi.sortOrder)[0];
```

```jsx
// ClassroomDetailsAttendances.jsx:339-350
<CButton color="primary" onClick={() =>
  handleChangeActiveDate(
    moment(today).format("YYYY-MM-DD"),
    currentDailyAttendance?.id ?? null      // ← null bila query UTC tidak kena
  )}>
  Check Attendance
</CButton>
```

Kalau jendela UTC tidak memotong baris yang ada (mis. teacher berada di WIB malam =
hari UTC berikutnya, atau baris `date` tercatat di hari sebelumnya), `currentDailyAttendance`
bernilai `undefined` → `daId = null` → **jalur create** → baris baru dengan
`date = moment(utcDateStr).endOf("day")` (dikonversi ke waktu lokal) → berpotensi
**date shift**.

**(b) Ikon pensil pada baris tabel** — `da.id` bisa `undefined` karena baris tabel bisa
merupakan **placeholder sintetis**:

```jsx
// ClassroomDetailsAttendances.jsx:198-212
const tableData = dates
  .map((date) => {
    const dailyAttendance = dailyAttendances.find(
      (da) => da.date === date.dateStr       // ← join by KETIKAN STRING
    );
    ...
    return dailyAttendance || date;          // ← placeholder TIDAK punya .id
  })
  .filter(Boolean);
```

```jsx
// ClassroomDetailsAttendances.jsx:129-149 (placeholder yang dibuat di lines 139-147)
const dates = useMemo(() => getBusinessDaysInRange(termStartDateValue, termEndDateValue, whitelistedDates)
  .map((d) => { const momentDate = moment(d); return {
      date: momentDate.toDate(),
      dateStr: momentDate.format("YYYY-MM-DD"),
      dailyAttendanceStatus: "INCOMPLETE", presentCount: 0, lateCount: 0,
      absentCount: 0, absentWithoutExcuseCount: 0 }; }), ...);
```

```jsx
// ClassroomDetailsAttendances.jsx:483-495
<CButton onClick={() =>
    handleChangeActiveDate(moment(da.date).format("YYYY-MM-DD"), da.id)} ...>
  <BiEdit />
</CButton>
```

Kalau `da` adalah placeholder (`da.id === undefined`) → falsy → **jalur create** → baris
kosong baru dibuat, teacher lalu menandai kehadiran ke baris baru, sementara baris lama
(yang sudah terisi) menjadi "yatim". **Inilah penyebab langsung dari gejala "alur berbeda
menampilkan data berbeda".**

### Bug #4 — dua alur memakai jendela waktu berbeda (UTC vs lokal) dan join rapuh

- Tabel daftar hari memakai rentang **lokal** term: `moment(termStartDateValue).startOf("day")`
  → `moment(termEndDateValue).endOf("day")` (`:88-100`), lalu `.toISOString()` (`:102-108`),
  dipakai untuk `GET /dailyAttendances` (`:115-125`).
- Tombol atas memakai rentang **UTC** "today" (`:166-170`).
- Join tabel ke data API memakai kesamaan string `da.date === date.dateStr` (`:201`).
  Karena `daily_attendance.date` adalah kolom `type: 'date'` (entity `:28`) yang
  diserialisasi `YYYY-MM-DD`, sedangkan `dateStr` dihitung dari `moment(d)` **lokal**,
  setiap skew UTC/lokal di sekitar tengah malam menghasilkan mismatch → fallback ke
  placeholder → `da.id` undefined → create (Bug #3b).

### Bug #5 (penguat) — agregat menghitung semua baris tanpa dedupe, sementara layar per-siswa hanya membaca `is_first_inserted = true`

```typescript
// api_nest/src/modules/daily-attendance/daily-attendance.service.ts:185-197
'COUNT(da.class_year_id)::int as day_of_attends',
"COUNT(CASE WHEN da.daily_attendance_status = '1' THEN 1 END)::int as incomplete",
"COUNT(CASE WHEN da.daily_attendance_status = '2' THEN 1 END)::int as complete",
```

Agregat ini menghitung **semua** baris `daily_attendance` milik class — tanpa filter
dedupe. Duplikat dari Bug #1–#3 membuat `day_of_attends`, `incomplete`, dan `complete`
terhitung dobel, dan status satu hari bisa terlihat **COMPLETE di satu alur dan INCOMPLETE
di alur lain**.

```typescript
// api_nest/src/modules/daily-attendance/daily-attendance.service.ts:283-320
const attendances = await DailyAttendance.find({ where: { classYearId } });
for (const attendance of attendances) {
  const totalAttendanceStudents =
    (attendance.presentCount||0)+(attendance.lateCount||0)+
    (attendance.absentCount||0)+(attendance.absentWithoutExcuseCount||0);
  if (totalAttendanceStudents >= studentCount) {
    attendance.dailyAttendanceStatus = DailyAttendanceStatusEnum.COMPLETE;
    await attendance.save();
  }
}
```

Setiap baris duplikat di-sync **secara independen** → status COMPLETE/INCOMPLETE bisa
berbeda antar baris untuk hari yang sama. Sementara itu layar penandaan per siswa
(`ClassroomDetailsAttendance.jsx`) hanya membaca `attendance` dengan
`isFirstInserted: true` untuk `dailyAttendanceId` tertentu — sehingga **ketika guru
menandai pada baris duplikat yang salah, baris kanonik di `attendance` lama tidak pernah
diperbarui** dan rekap kehadiran siswa jadi tidak sinkron dengan `daily_attendance`.

---

## Bukti dari Jam (https://jam.dev/c/70b2e5de-49e0-4e20-807e-82b7e0f59a55)

| Sumber | Temuan |
|--------|--------|
| **Video (t=...)** | Alur A: klik "Check Attendance" → data attendance per siswa tampil **terisi & benar**. Alur B: dari tabel hari, klik ikon pensil → layar terbuka dengan **data berbeda/kosong** untuk hari yang sama. |
| **Network** | Alur A: `GET /api/v1/attendances?pageSize=0&dailyAttendanceId=34723&isFirstInserted=true&activeStatus=ACTIVE` — `daId=34723`.<br>Alur B: URL halaman `.../classrooms/100913?term=2&pageSize=8&page=0&tab=attendance&daId=34724` lalu `GET /api/v1/attendances?pageSize=0&dailyAttendanceId=34724&isFirstInserted=true&activeStatus=ACTIVE` — `daId=34724`.<br>Terdapat request `POST /api/v1/dailyAttendances` (create) untuk `classYearId=100913` pada tanggal yang **sudah punya** baris → **dua baris `daily_attendance` untuk class+tanggal yang sama**. |
| **Console** | Tidak ada error fatal pada kedua alur — permintaan sukses; perbedaan data berasal dari **resolusi `daId` yang berbeda**, bukan dari kegagalan request. |
| **User events** | Urutan: buka class `100913` tab attendance (term 2) → klik tombol **"Check Attendance"** (atas) → `Back` → klik ikon pensil pada baris tanggal di tabel. Dua aksi tersebut menghasilkan dua `daId` berbeda (34723 vs 34724). |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/daily-attendance/daily-attendance.service.ts` (`:40-75` create, `:77-110` findAll, `:185-197` onClassYear, `:283-320` synchronizeDailyAttendanceStatus) | `create()` tidak idempotent → duplikat row; agregat & status sync menghitung/menandai baris duplikat secara independen |
| Backend DTO | `api_nest/src/modules/daily-attendance/dto/create-daily-attendance.dto.ts`, `dto/get-daily-attendances.dto.ts` | Tidak ada parameter/validasi yang mengunci keunikan `(classYearId, date)`; `findAll` tidak punya opsi dedupe |
| Backend Entity | `api_nest/src/modules/daily-attendance/entities/daily-attendance.entity.ts` (`:17`, `:19-29`, `:41-46`) | `@Index(['classYearId','date'])` non-unique; kolom `date` bertipe `date` (perbedaan serialisasi vs `attendance.date` yang timestamptz) |
| Frontend | `bbs/client-teacher/src/views/classrooms/ClassroomDetailsAttendances.jsx` (`:88-108`, `:115-125`, `:129-149`, `:166-190`, `:198-212`, `:223-242`, `:339-350`, `:483-495`) | Tombol atas & ikon pensil pada baris tabel menyelesaikan `daId` berbeda; `handleChangeActiveDate` membuat row baru saat `daId` falsy; join string `da.date === date.dateStr` rapuh |
| Frontend | `bbs/client-teacher/src/views/classrooms/ClassroomDetailsAttendance.jsx` | Layar per-siswa hanya membaca `attendance` dengan `isFirstInserted: true` untuk `dailyAttendanceId` → data lama tidak ikut berubah bila guru menandai pada baris duplikat |

---

## Proposed Solution Options

### Option A: Upsert + unique constraint di DB, dan resolution `daId` tunggal di frontend (Recommended)

**A1 — Backend: jadikan create idempotent + unique index.**

```typescript
// api_nest/src/modules/daily-attendance/daily-attendance.service.ts
async create(options: CreateDailyAttendanceDto) {
  // ... validasi classYear seperti sekarang ...
  const existing = await DailyAttendance.findOne({
    where: { classYearId: daAttendance.classYearId, date: options.date }
  });
  if (existing) return existing;   // idempotent: kembalikan baris yang sudah ada
  // ... set field & save ...
}
```

Tambahkan migration:

```sql
-- 1) dedupe dulu baris yang sudah terlanjur ganda (pilih baris terbaru/terisi)
-- 2) baru pasang unique index
CREATE UNIQUE INDEX "UQ_daily_attendance_class_year_date"
  ON "daily_attendance" ("class_year_id", "date");
```

Ubah entity menjadi:

```typescript
@Index(['classYearId', 'date'], { unique: true })
```

**A2 — Frontend: hapus dualisme resolusi, satu helper untuk "get-or-create".**

```jsx
// ClassroomDetailsAttendances.jsx — satu sumber kebenaran untuk daId
async function resolveDailyAttendance(dateStr) {
  const res = await dispatch(fromApi.getDailyAttendances({
    pageSize: 1, classYearId, startDate: dateStr, endDate: dateStr
  }));
  const found = res.data?.[0];
  if (found) return found.id;
  const created = await dispatch(fromApi.createDailyAttendance({
    classYearId, date: moment(dateStr).endOf("day").toDate()
  }));
  return created.data[0].id;
}
```

- Tombol atas dan ikon pensil **keduanya** memanggil `resolveDailyAttendance(dateStr)`.
- Gunakan **satu konvensi tanggal** (hapus `moment().utc()` pada `today`, atau normalisasi
  ke key `YYYY-MM-DD` sebelum query) supaya jendela UTC vs lokal tidak lagi berbeda.
- Pada baris tabel, **jangan** fallback ke placeholder yang langsung bisa diklik sebagai
  "create": bedakan aksi `Edit` (butuh `da.id`) vs `Buat` (tombol eksplisit) agar guru
  tidak pernah diam-diam membuat sesi baru.

**A3 — Backend: agregat & status sync hanya menghitung baris kanonik per tanggal.**

```typescript
// daily-attendance.service.ts onClassYear — group by date
'COUNT(DISTINCT da.date)::int as day_of_attends'
```

Dan `synchronizeDailyAttendanceStatus` harus bekerja pada satu baris per tanggal.

**A4 — Data existing:** jalankan script dedupe `daily_attendance` (gabung/soft-delete baris
duplikat, pindahkan `attendance.daily_attendance_id` ke baris kanonik) sebelum mengaktifkan
unique index.

### Option B: Normalisasi di sisi aplikasi saja (tanpa unique index)

- `create()` selalu melakukan find-before-create dan mengembalikan baris yang ada
  (idempotent), dan `findAll()` didedup per `(classYearId, date)` di service layer.
- Frontend memakai `resolveDailyAttendance()` yang sama seperti A2, dengan konvensi tanggal
  tunggal (local, tanpa `.utc()`).
- Tidak ada migration unique index → risiko duplikat tetap ada bila ada insert dari
  proses lain (import, gate check-in, admin) yang tidak lewat `create()`.

> **Trade-off:** Opsi B lebih cepat tanpa risiko migrasi, tetapi tidak menutup celah
> race-condition di level DB. Untuk data kehadiran yang dipakai sebagai dokumen resmi,
> Opsi A lebih aman.

---

## Notes

- `daily_attendance.date` bertipe `date` (serialisasi `YYYY-MM-DD`), sedangkan
  `attendance.date` bertipe `timestamptz` — perbedaan ini yang membuat join string
  `da.date === date.dateStr` di frontend rapuh, dan juga membuat "kunci hari" untuk
  `attendance` adalah `daily_attendance_id`, bukan `date::date`.
- Enum `daily_attendance_status`: `1` = INCOMPLETE, `2` = COMPLETE.
- `DailyAttendanceService.remove()` (`daily-attendance.service.ts:322-327`) adalah
  soft-delete-ish yang tetap `save()` dan mengembalikan entity — **tidak benar-benar
  menghapus** row, sehingga baris "yatim" tetap terhitung di agregat. Perlu
  diverifikasi apakah `remove()` seharusnya melakukan soft-delete atau hanya sebuah no-op.
- Bug ini **berkaitan erat** dengan `bbs-feature/progress-report-tardy-duplicate-attendance`
  (duplikasi pada tabel `attendance`): keduanya berakar dari tidak adanya constraint
  keunikan + insert non-atomik, dan perbaikannya sebaiknya dilakukan bersama.
- Setelah data dibersihkan dan constraint dipasang, **re-generate** rekap/
  progress report agar angka kembali konsisten.
