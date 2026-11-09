---
title: "Attendance — Scan gate kedua membuat baris duplikat & menimpa status (PIKP, laporan Ms Lina 2/11/2026)"
status: open
severity: critical
product: BBS LMS
portal: "Admin | Teacher"
author: OpenClaude
date: 2026-11-04
jam: ""
---

# Attendance — Scan gate kedua membuat baris duplikat & menimpa status kehadiran

## Summary

Laporan Ms Lina (email 4/11/2026) atas data **Attendance General Overview 2/11/2026 (PIKP: P1, P4, P6)** tidak bisa direkonsiliasi dengan data aktual. Setelah audit read-only di production, ditemukan **dua masalah terpisah**:

1. **Bug #1 (data):** Setiap kali sebuah kartu di-scan ulang di gate, sistem membuat **baris `attendance` baru** (`is_first_inserted = false`) alih-alih meng-update baris yang sudah ada. Baris kedua ini berstatus **LATE**, sehingga sistem mencatat satu siswa punya dua status berbeda di hari yang sama (PRESENT + LATE), dan pada sebagian kasus tidak ada baris kanonik sama sekali.
2. **Bug #2 (roster):** Siswa dummy ber-nama **"Sample …"** (5 di P1/P4/P6) masih berstatus aktif sehingga ikut terhitung sebagai siswa nyata. Setelah dikeluarkan, P6 cocok persis dengan angka Ms Lina (101), P1 sisa selisih 1, sedangkan **P4 justru kurang 3** — perlu klarifikasi definisi roster per level.

**Ekspektasi:** Satu siswa hanya punya **satu** baris attendance final per hari. Scan ulang di gate harus meng-update baris yang sama (mis. mengisi `check_out_date`), bukan membuat baris baru. Siswa dummy/testing tidak boleh dihitung dalam roster maupun summary attendance.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Ms Lina (Principal PIKP) via email |
| Email (Jam account) | - |
| Jam author ID | - |
| User ID (dari console log, mis. `selfUser`) | - (audit memakai DB read-only) |
| Portal URL | Admin `client/` → Attendance → General Overview |
| Environment API | `https://api.binabangsaschool.dev` |
| Data konteks | Programme `6` (PIK-P), academic year `27` (2026/2027), tanggal `2026-11-02`, level enum `5` = P1, `7` = P4, `10` = P6, campus `3` (PIKP) |
| Browser / OS | - |

---

## Steps to Reproduce

1. Buka Admin → Attendance → General Overview, pilih programme **PIK-P** (programmeId `6`), tanggal **2/11/2026**.
2. Bandingkan angka Present/Late/Absent per level P1/P4/P6 dengan roster aktual.

**Actual Result (dari Ms Lina):**

| Level | Actual | Smartbag (Absent / Late / Present) |
|-------|--------|-------------------------------------|
| P1 | 105 | 2 / 4 / 97 |
| P4 | 82 | 1 / 5 / 79 |
| P6 | 101 | 7 / 0 / 97 |

**Expected Result:** Present + Late + Absent = jumlah siswa aktual, dan tidak ada siswa yang dihitung lebih dari sekali.

**Data DB (read-only, 2026-11-02, `deleted_at IS NULL`):**

| Level | Baris `is_first_inserted = true` (Present/Late/Absent) | Baris duplikat (`false`, gate) (Present/Late) | Total baris |
|-------|--------------------------------------------------------|-----------------------------------------------|-------------|
| P1 | 97 / 4 / 2 | 7 / 32 | 142 |
| P4 | 75 / 1 / 1 | 4 / 34 | 115 |
| P6 | 97 / 0 / 7 (Absent = 2 status `3` + 5 status `4`) | 2 / 12 | 123 |

---

## Root Cause Analysis

### Bug #1 — Scan ulang gate tidak meng-update baris, tetapi membuat baris baru berstatus LATE

`createAttendanceIntegrationBulk()` memproses record dalam chunk secara concurrent (`Promise.all`), dan jalur `processIntegrationRecord` dipanggil di luar advisory lock. Ketika gate mengirim scan kedua untuk kartu yang sama, pencarian baris existing **tidak menemukan** baris hari itu (kunci `daily_attendance` tidak konsisten / row belum terlihat pada saat concurrent read), lalu kode melakukan insert baru:

- `is_first_inserted = false` → baris kedua **tidak** dihitung sebagai row kanonik.
- `attendance_status` di-set **LATE** karena jam `timelog` scan kedua melewati batas on-time.
- `check_out_date` **tidak diisi** → scan kedua bukan diperlakukan sebagai check-out.

Bukti paling jelas — siswa yang sama, satu hari, dua baris:

| student_id | Nama | `check_in_date` (JKT) | `check_out_date` | Status | `is_first_inserted` | Gate |
|-----------|------|----------------------|------------------|--------|---------------------|------|
| 25483 | Kenrich Henva Maximilian | 06:39 | 15:17 | PRESENT | true | 23 (Preschool Lobby 2) |
| 25483 | Kenrich Henva Maximilian | 06:39 | — | PRESENT | **false** | 23 |
| 25483 | Kenrich Henva Maximilian | 06:42 | — | PRESENT | **false** | 19 (Prim Lobby Lane 1) |
| 25483 | Kenrich Henva Maximilian | 15:17 | — | **LATE** | **false** | 21 (Prim Lobby Lane 3) |

Contoh lain (PRESENT + LATE pada siswa yang sama, 2/11): `Alessa Grace Gunawan`, `Charlotte Widjaja`, `Angelynn Dior`, `Maeveline Reia Martin`, dll.

**Distribusi scan kedua:** semua baris duplikat yang LATE dibuat antara **08:24–09:50 WIB** (jam masuk terlambat), sedangkan baris PRESENT sudah dibuat **06:39–07:30 WIB**. Artinya ini bukan scan pulang — gate melaporkan *event masuk* kedua di jam yang sama dengan jam masuk siswa lain, dan backend menafsirkannya sebagai attendance baru.

**Kesimpulan:** angka Ms Lina tidak bisa dicocokkan karena sebagian siswa punya 2–4 baris per hari dengan status berbeda, dan ada siswa yang baris PRESENT-nya duplikat sehingga tidak pernah menjadi baris kanonik.

### Dampak skala besar (satu academic year)

Baris duplikat gate (`is_first_inserted = false AND campus_gate_id IS NOT NULL`) di programme 6:

| Level | Baris duplikat | Siswa terlibat |
|-------|----------------|-----------------|
| P1 (`5`) | 1.996 | 104 |
| P4 (`7`) | 2.103 | 81 |
| P6 (`10`) | 1.226 | 101 |
| **Total** | **5.325** | |
| Tanggal | 2026-07-16 s/d 2026-11-03 (88 hari) | |
| `check_out_date` terisi | **0** dari 5.325 | |
| **Tidak punya baris `is_first_inserted = true` sama sekali** | **2** (Regent Sevenseas Prayudi, P4, 2026-08-07) | |

### Bug #2 — Siswa dummy "Sample" berstatus aktif dan ikut dihitung

| Level | Aktif di sistem | Siswa "Sample" | **Siswa nyata** | Actual Ms Lina | Selisih |
|-------|-----------------|----------------|-----------------|----------------|---------|
| P1 (`5`) | 107 | 1 (`Sample P1 Kindness`) | **106** | 105 | +1 |
| P4 (`7`) | 83 | 4 (`Sample P3 …`) | **79** | 82 | **−3** |
| P6 (`10`) | 105 | 4 (`Sample P6 …`) | **101** | 101 | **0** |

Daftar siswa Sample yang masih aktif:

| Level | student_id | Nama |
|-------|-----------|------|
| P1 | `adsap1kindness2` | Sample P1 Kindness |
| P4 | `203900006` | Sample P3 Courage |
| P4 | `20202001` | Sample P3 Initiative |
| P4 | `201020002` | Sample P3 Obedience |
| P4 | `202020200` | Sample P3 Respect |
| P6 | `20102013` | Sample P6 Attentiveness |
| P6 | `20202008` | Sample P6 Compassion |
| P6 | `20202007` | Sample P6 Determination |
| P6 | `20202010` | Sample P6 Tolerance |

Catatan: roster dihitung dengan `student.active_status = '1'` **dan** `student.student_status = '1'`. P6 setelah exclude Sample = **101**, sama persis dengan Ms Lina — mengonfirmasi bahwa selisih P6 murni dari bug #1 + siswa Sample.

P4 memiliki **79 < 82**, arah selisih terbalik. Hipotesis: angka Ms Lina untuk P4 mencakup 2 siswa **Immersion Programme Primary 3** (classroom terpisah di level yang sama) dan/atau siswa yang belum ter-assign. **Perlu klarifikasi daftar siswa P4 dari Ms Lina** sebelum menentukan tindakan.

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/attendance/attendance.service.ts` — `createAttendanceIntegrationBulk` (~line 1497-1498), `processIntegrationRecord` (~line 1514), `createOrUseExistingDailyAttendance` (~line 1961-1979) | Insert baris baru pada scan ulang; `is_first_inserted` tidak dikunci |
| Backend — attendance summary | `findOverview` (memfilter `is_first_inserted === true`) | Menghitung angka yang tidak boleh direkonsiliasi dengan roster |
| Backend — roster | `student.current_class_year_id` + `active_status`/`student_status` | Siswa "Sample" ikut terhitung |
| Data | `attendance` table untuk `academic_year_id = 27`, `programme_id = 6`, `level IN ('5','7','10')` | 5.325 baris duplikat (2026-07-16 s/d 2026-11-03) |
| Frontend | `bbs/client/src/views/attendance/generalOverview/attendanceOverview/AttendanceOverview.jsx` | Hanya menampilkan hasil API; tidak menyaring siswa Sample |

Tiket terkait (remediation sebelumnya): `features/external-attendance-integration-bulk-race-condition/`, `features/attendance-duplicate-daily-attendance-race/`.

---

## Proposed Solution Options

### Option A: Perbaiki jalur integrasi + bersihkan data (Recommended)

**Fix kode**
1. Serialisasi `processIntegrationRecord` per `(student_id, date)` dengan advisory lock yang **kunci stabil** (mis. `pg_advisory_xact_lock(student_id, yyyymmdd)`), bukan `daily_attendance_id` hasil insert.
2. Jika baris hari itu sudah ada:
   - Scan kedua **tidak** membuat baris baru.
   - Isi `check_out_date` bila kontak kedua lebih dari N jam setelah `check_in_date`.
   - **Jangan** mengubah `attendance_status` dari PRESENT ke LATE.
3. Tambahkan unique constraint/index parsial: satu `is_first_inserted = true` per `(student_id, daily_attendance_id)`.

**Bersihkan data (setelah fix deploy)**
- Identifikasi baris duplikat: `is_first_inserted = false AND campus_gate_id IS NOT NULL AND attendance_status = '2'`.
- Cek apakah `check_out_date` pada baris tersebut perlu dipindahkan ke baris kanonik sebelum di-soft-delete.
- Untuk 2 siswa tanpa baris kanonik, tentukan status final manual (preserve data) sebelum diseragamkan.

### Option B: Dedupe di aplikasi saja (tanpa migration)

Tambahkan logika normalisasi di `findOverview`/summary: ambil status dari baris `is_first_inserted = true` per siswa/hari, abaikan baris duplikat. Cepat dipakai, tetapi **tidak** menyelesaikan akar masalah (data tetap menumpuk, dampak ke report lain tetap).

---

## Yang perlu engineer cek (checklist)

- [ ] Konfirmasi mekanisme scan kedua dari gate: apakah ini benar-benar scan ulang masuk, atau scan pulang yang salah ditafsirkan? (Cek `request->records[0]->statusGate` pada `attendance_log` untuk 91 baris 2/11 — semua statusGate `1`.)
- [ ] Kenapa baris duplikat tidak terhubung ke `attendance_log` (`attendance_log.attendance_id` NULL untuk baris 2/11)? Pastikan logging tidak kehilangan jejak.
- [ ] Cek `daily_attendance` untuk programme 6 tanggal 2/11: **tidak ada** kelas dengan > 1 batch, jadi ini bukan duplikat batch — murni duplikat baris `attendance`.
- [ ] Tentukan kebijakan siswa "Sample": nonaktifkan, atau beri flag `is_test` agar dikecualikan dari roster/summary/report.
- [ ] Validasi 5.325 baris duplikat se-AY 27 dengan query verifikasi sebelum remediation.
- [ ] Kirim pertanyaan ke Ms Lina: daftar siswa aktual P1/P4/P6 + apakah P4 mencakup Immersion Programme.

---

## Notes

- Sumber data: audit read-only production (`binabangsa-db-tools`), tanggal 2026-11-04.
- Angka summary (Present/Late/Absent) diambil dari `attendance.attendance_status` dengan filter `is_first_inserted = true`; baris duplikat dihitung terpisah.
- Latar belakang pendorong: laporan Ms Lina tanggal 4/11/2026 atas data 2/11/2026. Masalah serupa pernah ditangani sebagai "data nimpa karena dua tombol hit data beda" — kasus gate ini adalah varian lain dari race condition yang sama.
