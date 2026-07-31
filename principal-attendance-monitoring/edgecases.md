# Edge Cases — Principal Attendance Monitoring

> **Status: FINAL** — semua keputusan sudah ditentukan berdasarkan spec.

---

## EC-01: No Attendance Data for Selected Date
**Scenario:** Principal memilih tanggal yang tidak memiliki data attendance (weekend, holiday, atau hari pertama tahun ajaran belum ada attendance).

| Opsi | Behavior |
|------|----------|
| (A) Show empty state | Tampilkan ilustrasi + "No attendance data for this date" |
| (B) Show blank table | Tabel kosong tanpa pesan |
| (C) Show "No School Day" | Deteksi weekend/holiday dan tampilkan label khusus |

**Decision:** **(A) Show empty state** — konsisten dengan acceptance criteria: "Empty state muncul jika tidak ada data untuk filter yang dipilih". Tidak perlu deteksi weekend/holiday khusus.

**Rationale:** API sudah handle empty data dengan return 200 + empty array. FE cukup tampilkan empty state illustration + message.

---

## EC-02: Class Has No Homeroom Teacher
**Scenario:** Ada class yang belum memiliki homeroom teacher assigned.

| Opsi | Behavior |
|------|----------|
| (A) Show dash `—` | Kolom Homeroom Teacher menampilkan `—` |
| (B) Show "Not Assigned" | Tampilkan teks "Not Assigned" dengan warna abu-abu |
| (C) Hide class | Class tanpa homeroom teacher tidak muncul di list |

**Decision:** **(A) Show dash `—`** — reuse existing behavior dari admin DailyAttendance. Homeroom teacher yang null ditampilkan sebagai dash.

**Rationale:** Principal perlu tahu kelas mana yang belum punya homeroom teacher — hiding class akan menyembunyikan informasi penting.

---

## EC-03: User is HOD (Not Principal)
**Scenario:** User yang login adalah HOD (Head of Department) mencoba mengakses halaman ini.

| Opsi | Behavior |
|------|----------|
| (A) Redirect to landing | HOD tidak bisa akses, redirect ke landing page |
| (B) Show 403 page | Tampilkan halaman "Access Denied" |
| (C) Allow with scoped data | HOD tetap bisa akses dengan data terbatas pada department-nya |

**Decision:** **(A) Redirect to landing** — Principal only. HOD tidak perlu akses. Guard `requirePrincipalOrHod` sudah ada di teacher portal, tapi FE harus mengecek `isPrincipalOrVp` (bukan `isPrincipalOrHod`) untuk halaman ini.

**Rationale:** Spec already decided: "Principal only, no HOD." Existing guard name tetap `requirePrincipalOrHod` untuk reuse, tapi di TheContent.jsx perlu tambahan pengecekan bahwa user adalah Principal/VP, bukan HOD.

---

## EC-04: Selected Date is in the Future
**Scenario:** Principal memilih tanggal yang belum terjadi.

| Opsi | Behavior |
|------|----------|
| (A) Disable future dates | Date picker tidak mengizinkan selection tanggal > hari ini |
| (B) Show error + clear | Tampilkan error toast "Date cannot be in the future" dan reset date ke hari ini |
| (C) Show empty | Tampilkan tabel kosong tanpa error |

**Decision:** **(A) Disable future dates** — Date picker mengatur `maxDate={today}` sehingga tanggal > hari ini tidak bisa dipilih.

**Rationale:** Konsisten dengan business rule di spec: "Date range: Tidak boleh pilih future date (max: today)". Preventif lebih baik daripada reaktif.

---

## EC-05: No Active Academic Year
**Scenario:** Sistem tidak memiliki academic year yang aktif (misal transisi tahun ajaran).

| Opsi | Behavior |
|------|----------|
| (A) Error banner | Tampilkan error banner "No active academic year found" — halaman tidak bisa digunakan |
| (B) Allow manual select | Biarkan user pilih academic year secara manual dari dropdown |
| (C) Redirect | Redirect ke halaman lain dengan notifikasi |

**Decision:** **(A) Error banner** — Tampilkan error banner "No active academic year found" di halaman. Halaman tidak dapat digunakan sampai ada active academic year.

**Rationale:** Halaman ini tidak memiliki dropdown AY (default active AY). Jika tidak ada active AY, tidak bisa menampilkan data. Error banner sudah di-spec di error handling table.

---

## EC-06: Class Has No Students Enrolled
**Scenario:** Ada class yang aktif tetapi tidak memiliki student enrolled (class kosong).

| Opsi | Behavior |
|------|----------|
| (A) Show with 0 student | Tampilkan class di tabel dengan total student = 0 dan status "N/A" |
| (B) Hide class | Class tanpa student tidak muncul di tabel |
| (C) Show with warning | Tampilkan class dengan badge "No Students" |

**Decision:** **(A) Show with 0 student** — Tampilkan class di tabel dengan total = 0. Kolom Complete/Incomplete menampilkan "N/A" atau dash.

**Rationale:** Principal perlu tahu ada kelas kosong — mungkin ada masalah enrollment yang perlu ditindaklanjuti.

---

## EC-07: API Loading / Slow Response
**Scenario:** Data attendance besar (banyak class) menyebabkan API response lambat.

| Opsi | Behavior |
|------|----------|
| (A) Skeleton loading | Tampilkan skeleton placeholder di tabel dan summary section |
| (B) Spinner | Tampilkan spinner/loading spinner di tengah halaman |
| (C) Progressive loading | Load tabel dulu, baru summary section setelahnya |

**Decision:** **(A) Skeleton loading** — Tampilkan skeleton placeholder di tabel dan summary section selama loading.

**Rationale:** Konsisten dengan acceptance criteria: "Loading state (skeleton) saat data sedang di-fetch". Skeleton memberikan persepsi performa lebih baik daripada spinner.

---

## EC-08: Multiple Attendance Records Per Day
**Scenario:** Ada student yang memiliki multiple attendance record dalam satu hari (check-in dan check-out).

| Opsi | Behavior |
|------|----------|
| (A) Use first record | Hanya hitung record pertama (is_first_inserted = true) — reuse existing logic |
| (B) Show all records | Tampilkan multiple records untuk student yang sama |
| (C) Show latest | Hanya hitung record terakhir |

**Decision:** **(A) Use first record** — Reuse existing logic yang sudah menggunakan `is_first_inserted = true`.

**Rationale:** Existing API (`GET /api/v1/dailyAttendances/onClassYear`) sudah handle ini. Tidak perlu perubahan logic.

---

## EC-09: Level Filter Shows No Data
**Scenario:** Principal memilih Level filter yang tidak memiliki class (misal level "Primary 6" tapi sekolah hanya sampai Primary 3).

| Opsi | Behavior |
|------|----------|
| (A) Empty table + message | Tabel kosong dengan "No classes found for this level" |
| (B) Disable Level option | Level yang tidak memiliki class tidak muncul di dropdown |
| (C) Show count next to Level | Tampilkan jumlah class di samping nama Level di dropdown |

**Decision:** **(A) Empty table + message** — Tabel kosong dengan pesan "No classes found for this level".

**Rationale:** Opsi B dan C membutuhkan logic tambahan di API untuk menghitung class per level. Opsi A reuse empty state yang sudah ada. Simple dan jelas.

---

## EC-10: Programme Has No Levels
**Scenario:** Principal memilih Programme yang tidak memiliki Level assignment (misal programme baru belum di-setup).

| Opsi | Behavior |
|------|----------|
| (A) Empty Level dropdown | Dropdown Level menampilkan "(No levels available)" dan disabled |
| (B) Hide Level filter | Level filter tidak muncul jika Programme tidak memiliki level |
| (C) Allow all | Abaikan filter Programme, tampilkan semua class |

**Decision:** **(A) Empty Level dropdown** — Dropdown Level menampilkan placeholder "(No levels available)" dan disabled.

**Rationale:** Opsi A memberikan feedback jelas ke user. Opsi B bisa membingungkan (user pilih programme, Level hilang). Opsi C melanggar filter logic.

---

## EC-11: Discipline — No Data for Selected Term
**Scenario:** Principal memilih Term yang belum memiliki data discipline (misal term baru mulai, belum ada laporan).

| Opsi | Behavior |
|------|----------|
| (A) Empty table | Tabel kosong dengan pesan "No discipline data for this term" |
| (B) Show all students with 0 count | Tampilkan semua student dengan late=0, absent=0 |
| (C) Hide table + message | Jangan tampilkan tabel, hanya pesan |

**Decision:** **(A) Empty table** — Tabel kosong dengan pesan "No discipline data for this term".

**Rationale:** API `findDiscipline` hanya mengembalikan student yang memiliki late/absent > 0, jadi empty result berarti tidak ada pelanggaran di term tersebut. Konsisten dengan pattern yang sudah ada.

---

## EC-12: Discipline — Filter Level/Class Mismatch
**Scenario:** Principal memilih Level "Primary 1" lalu Class "P2A" (class dari Level berbeda).

| Opsi | Behavior |
|------|----------|
| (A) Class filter reset on Level change | Saat Level berubah, Class filter di-reset ke kosong |
| (B) Allow mismatch | Biarkan user memilih, class akan filter berdasarkan nama saja |
| (C) Filter Class by Level | Class dropdown hanya menampilkan class dari Level yang dipilih |

**Decision:** **(B) Allow mismatch** — Class filter adalah text search (classroom name), bukan dropdown. Tidak ada relasi otomatis dengan Level.

**Rationale:** Admin juga menggunakan classroom name search (free text). Jika user mencari class yang tidak cocok dengan Level, hasilnya akan kosong — ini sudah di-handle oleh empty state.

---

## EC-13: Principal Assigned to Multiple Campuses
**Scenario:** Principal memiliki EmployeePosition records di lebih dari satu campus.

| Opsi | Behavior |
|------|----------|
| (A) Aggregate all | Tampilkan semua class dari semua campus yang di-assign |
| (B) Campus filter | Tambahkan filter Campus untuk memilih |
| (C) First campus only | Hanya tampilkan data campus pertama |

**Decision:** **(A) Aggregate all** — Tampilkan semua class dari semua programme di semua campus yang di-assign ke principal.

**Rationale:** Existing `usePrincipalOrHod` hook dan employee position handling sudah support multi-campus. Principal melihat semua data di scope-nya. Tidak perlu tambahan filter campus — reuse existing behavior.
