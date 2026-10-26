# Edge Cases — GPA Report: Subject dari Tahun Lain Muncul di Kolom Tahun Berjalan

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.
>
> Konteks: `secondary-gpa-report.helper.ts` merender `student.classYears.slice(0, 4)`
> (baris 73-74) tanpa urutan terjamin, dan daftar subject per kolom diambil dari
> `student.bandings` yang difilter `subjectYear.classYearId` (baris 111-114). Grid 2 kolom
> (baris 347-387) membuat baris tahun lama tampak menyatu ke kolom tahun berjalan.

---

## EC-01: Berapa tahun yang dirender?

**Scenario:** Report GPA Transcript sekarang merender 4 tahun terakhir (`slice(0, 4)`).
Siswa dengan riwayat 7 tahun (Aaronne) kehilangan 3 tahun pertama, dan 4 tahun yang
dirender masih tercampur kolomnya.

| Opsi | Behavior |
|------|----------|
| (A) Tetap 4 tahun terakhir, tapi **diurutkan deterministik** | Perilaku minimal-cabang: `.sort(academicYear DESC).slice(0,4)`. Aaronne tetap lihat 2022/23–2025/26, tapi kolom 25/26 bersih. |
| (B) Render **semua** tahun riwayat (tanpa slice) | Transkrip lengkap, cocok untuk dokumen resmi. Perlu layout multi-halaman/handle panjang. |
| (C) Render **hanya AY yang dipilih** (single column) | Report = transkrip 1 tahun. Paling bersih untuk kebutuhan "GPA 25/26", tapi menghilangkan sifat transkrip historis. |

**Decision:** _TBD_

---

## EC-02: Cara mengurutkan riwayat tahun

**Scenario:** `student.classYears` adalah `@ManyToMany` tanpa `order` (student.entity.ts:155-158),
jadi urutan array tidak dijamin. `.slice(0,4)` bisa mengambil tahun yang salah.

| Opsi | Behavior |
|------|----------|
| (A) Sort di aplikasi setelah load: `classYears.sort((a,b) => b.academicYear.year.localeCompare(a.academicYear.year))` | Cepat, tanpa ubah query loader. Format `year` harus konsisten (`2025/2026`). |
| (B) Tambah `order` di relasi TypeORM: `@ManyToMany(..., { order: { academicYearId: 'DESC' } })` | Deklaratif & berlaku semua pemakaian relasi. Perlu migrasi metadata entities. |
| (C) Tambah `order` di `getAnotherReportDataFromRelations` untuk relasi ini | Terlokalisasi di report; tidak mempengaruhi consumer lain. |

**Decision:** _TBD_

---

## EC-03: Tahun diwakili oleh apa?

**Scenario:** `class_year` per tahun ajaran (mis. 100392 untuk JC2 Mendel 25/26), tapi
`student.classYears` menghubungkan student ↔ class_year (many-to-many). Urutan yang benar
harus berdasarkan `academicYear`, bukan `class_year.id` (id tidak selalu mencerminkan urutan
tahun — 1857 vs 100392).

| Opsi | Behavior |
|------|----------|
| (A) Sort berdasarkan `academicYear.year` (string `YYYY/YYYY`) | Benar untuk format konsisten; `localeCompare` DESC aman. |
| (B) Sort berdasarkan `academicYear.id` (urutan insert) | Bergantung urutan seed tiap tahun; rapuh bila ada backfill. |
| (C) Sort berdasarkan `class_year.startDate` | Paling tepat secara semantik; perlu cek kolom tersedia di entity. |

**Decision:** _TBD_

---

## EC-04: Penanda batas tahun di template 2 kolom

**Scenario:** `tables[0]` = tahun index genap, `tables[1]` = tahun index ganjil, dirender
sebagai 2 `<td>` berdampingan (`secondary-student-gpa-report.hbs:126-194`). Baris tahun
2023/2024 di kolom kanan tampak menyambung ke blok 25/26 di kolom yang sama.

| Opsi | Behavior |
|------|----------|
| (A) Header tahun dibuat tegas (bold + border atas + shading), plus baris pemisah `[' ']` antar blok tahun | Minimal, tanpa perubahan struktur grid. Menghilangkan ambiguitas visual. |
| (B) Satu tabel per tahun, ditumpuk vertikal (bukan 2 kolom) | Paling jelas, tapi mengubah layout & panjang halaman. |
| (C) Label tahun diulang di setiap baris subject (prefix kecil) | Selalu terbaca, tetapi mengotori tabel. |

**Decision:** _TBD_

---

## EC-05: Siswa dengan < 4 tahun riwayat

**Scenario:** Siswa baru (1–2 tahun) memakai `classYears` kurang dari 4. Grid 2 kolom dan
logika padding `tableLengthDifference` bisa menghasilkan blok kosong aneh.

| Opsi | Behavior |
|------|----------|
| (A) Render apa adanya: N = `min(classYears.length, 4)`; kolom kanan kosong bila hanya 1 tahun | Perilaku existing, aman. |
| (B) Paksa 2 kolom dengan placeholder `[' ']` | Tampilan tetap "seimbang", tapi ada area kosong. |
| (C) Jika ≤ 2 tahun, render 1 kolom penuh (tidak bagi 2) | Tampilan lebih rapi untuk siswa baru. |

**Decision:** _TBD_

---

## EC-06: Banding tanpa `subjectYear` (relasi hilang / soft-deleted)

**Scenario:** `currentSubjectYearBandings` memfilter `banding?.subjectYear?.classYearId`.
Bila `banding.subjectYear` null atau `subject_year` soft-deleted, subject bisa hilang
diam-diam atau error saat akses `.subjectYear.subject`.

| Opsi | Behavior |
|------|----------|
| (A) Skip banding yang `subjectYear` null / deleted (guard) | Aman, report tidak crash; subject tidak tampil. |
| (B) Tampilkan subject dengan nilai kosong | Menampilkan nama yang tidak diketahui sumbernya. |
| (C) Lempar error | Tidak diinginkan — report tidak jadi. |

**Decision:** _TBD_

---

## EC-07: Banding duplikat untuk tahun yang sama

**Scenario:** Bila ada dua banding untuk subject yang sama di class_year yang sama
(duplikat migrasi/manual), baris subject bisa muncul dua kali di kolom tahun itu.

| Opsi | Behavior |
|------|----------|
| (A) Dedupe per `subjectId` dalam satu class_year (ambil satu) | Hindari baris ganda. |
| (B) Tampilkan semua | Sesuai data, tapi laporan membingungkan. |
| (C) Dedupe hanya saat `subjectId + classYearId` identik, sisanya tampil | Konservatif. |

**Decision:** _TBD_

---

## EC-08: Term 2 — nilai SA1 untuk tahun terakhir

**Scenario:** Untuk `term == 2` dan `classYearIdx == classYearHistory.length - 1`, report
membaca `SA` term 2 (SA1), bukan FYA (baris 193-198). Karena `classYearHistory` saat ini
tidak deterministik, "tahun terakhir" bisa bukan AY 25/26 — nilai yang dipilih pun bisa
salah tahun.

| Opsi | Behavior |
|------|----------|
| (A) Setelah EC-02 diurutkan DESC, "tahun terakhir" = AY berjalan (benar) | Perbaikan ikut dari urutan deterministik. |
| (B) Tentukan "tahun berjalan" eksplisit dari `studentReport.classYear`, bukan index array | Lebih eksplisit & tidak bergantung slicing. |
| (C) Biarkan | Bug lanjutan: range FYA/SA bisa membingungkan. |

**Decision:** _TBD_

---

## EC-09: GPA kumulatif (`weightedCumulativeGpa`)

**Scenario:** `cumulativeGpa / classYearInfo.length` (baris 344, 392-394) membagi dengan
jumlah tahun yang dirender. Bila komposisi tahun berubah (slice/urutan), nilai GPA kumulatif
ikut berubah.

| Opsi | Behavior |
|------|----------|
| (A) Tetap rata-rata dari N tahun yang dirender | Sesuai definisi report saat ini. |
| (B) Rata-rata hanya dari tahun yang punya nilai valid (`gpaDenominator > 0`) | Lebih akurat; tahun kosong tidak menurunkan rata-rata. |
| (C) Weighted by subject unit saja | Perubahan besar; di luar scope. |

**Decision:** _TBD_

---

## EC-10: Data bukan penyebab — jangan "fix" dengan hapus subject

**Scenario:** Muncul dorongan menghapus banding subject 724 (Bahasa Indonesia Level 4) dari
Aaronne agar report bersih.

| Opsi | Behavior |
|------|----------|
| (A) **Jangan** ubah data; perbaiki report | Subject 724 adalah record sah tahun 2023/2024. Menghapusnya merusak transkrip historis. |
| (B) Soft-delete banding 724 | Salah — menghilangkan nilai Sec 4 yang valid. |
| (C) Reassign class_year banding 724 ke 25/26 | Salah — membuat data palsu. |

**Decision:** _TBD_

---

## EC-11: Regression siswa tanpa data

**Scenario:** Siswa tanpa `classYears`, tanpa `bandings`, atau tanpa `conducts`/attendance
dalam rentang tahun yang dirender.

| Opsi | Behavior |
|------|----------|
| (A) Render report kosong tanpa crash (200) | Perilaku yang diinginkan; guard di setiap akses relasi. |
| (B) Return 400/500 | Tidak diinginkan. |
| (C) Fallback ke data terakhir | Tidak jelas semantiknya. |

**Decision:** _TBD_
