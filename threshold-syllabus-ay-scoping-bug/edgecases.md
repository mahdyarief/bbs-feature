# Edge Cases — Threshold & Syllabus AY Scoping

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update `spec.md` jika ada perubahan behavior.
>
> Konteks kode:
> - `bbs/client/src/views/threshold/Threshold.jsx:52-82` dan `ThresholdAsPrelim.jsx:54-82`
>   memuat `getTresholdBySubjectId` (semua AY) + `getThreshold` (AY spesifik) ke slice
>   `threshold` yang sama via MERGE.
> - `ThresholdStatistic.jsx` memilih DRAFT/ACTIVE dengan `find` tanpa filter AY.
> - `ThresholSyllabus.jsx` hanya tahu `subjectId`; threshold yang dipakai `[0]`.
> - `api_nest/.../threshold.service.ts:229-238` memetakan `syllabi` yang bisa berisi
>   `undefined` ke `ThresholdConfig`.
> - `ThresholdSyllabusPart.jsx:196-231` mengirim `rowIndex` yang tidak pernah diisi.

---

## EC-01: How to scope threshold data per AY on the client

**Scenario:** `getThreshold` (MERGE) menambahkan baris AY baru ke slice `threshold` yang
sudah berisi baris AY sebelumnya. `useResourceMapper` tidak memfilter; siapa pun yang
`find` di slice bisa mendapat baris AY lain.

| Opsi | Behavior |
|------|----------|
| (A) Filter di consumer: `threshold.find(d => d.academicYearId === Number(academicYearId) && d.statusType === ...)` di setiap tempat | Perubahan minimal, tidak menyentuh hook/global. Risiko: setiap consumer baru harus ingat untuk filter manual. |
| (B) Buat helper/hook `useScopedThreshold(entity, { academicYearId, thresholdType })` yang selalu memfilter + buang soft-deleted | Satu titik kebenaran, dipakai ulang oleh `ThresholdStatistic`, `ThresholSyllabus`, `SetThresholdModal`. Sedikit lebih banyak kode baru. |
| (C) Filter di level fetch/hook `useFromApi` dengan flag `scopeByAcademicYear` | Trasparan, tapi menyentuh hook global yang dipakai ratusan view → risiko regresi luas. |

**Decision:** _TBD_

---

## EC-02: Perilaku section Syllabus ketika AY terpilih belum punya threshold

**Scenario:** AY X punya syllabus (efektif tahunnya mencakup X) tetapi belum ada threshold
DRAFT/ACTIVE untuk AY X. Saat ini kartu threshold kosong dan grup syllabus mungkin tetap
tampil karena tidak bergantung pada AY.

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan semua grup syllabus read-only + banner "Belum ada threshold untuk AY ini" | Operator tetap bisa melihat komponen syllabus dan tahu harus klik tombol Threshold untuk membuatnya. |
| (B) Sembunyikan seluruh section Syllabus sampai threshold dibuat | Halaman lebih bersih tetapi menghilangkan konteks; operator tidak tahu apakah syllabus ada. |
| (C) Tampilkan grup syllabus + tombol "Create Threshold for this AY" di dalam empty-state | Paling actionable, perlu sedikit UI tambahan (bukan redesign besar). |

**Decision:** _TBD_

---

## EC-03: Aktivasi threshold yang tidak punya config syllabus valid

**Scenario:** Draft hasil `create` lama menyimpan `threshold_config` dengan `syllabusId`
kosong (bug saat ini). Ketika user menekan aktivasi, `activatedThreshold` menyalin config
kotak ke ACTIVE.

| Opsi | Behavior |
|------|----------|
| (A) Tolak aktivasi (400) bila draft tidak punya minimal 1 config valid | Mencegah threshold ACTIVE tanpa syllabus; user harus regenerate/create ulang. |
| (B) Aktifkan tetapi buang config kosong, dan kembalikan warning di response | Aktivasi tetap jalan (untuk data lama yang sebagian valid); UI menampilkan warning. |
| (C) Perbaiki data lama dulu (backfill) baru terapkan (A) | Paling bersih untuk jangka panjang, butuh koordinasi data. |

**Decision:** _TBD_

---

## EC-04: Apakah endpoint `GET /threshold/subject/:subjectId` perlu parameter AY?

**Scenario:** `findBySubjectId` (dipakai gradebook) sengaja mengembalikan semua AY dengan
AY terpilih di depan (ada komentar di `threshold.service.ts:268-275`). UI memakai endpoint
ini merah-campur lalu memfilter sendiri.

| Opsi | Behavior |
|------|----------|
| (A) Jangan ubah endpoint; UI memfilter sendiri (lihat EC-01) | Backend tidak terdampak → gradebook aman. Paling kecil blast radius. |
| (B) Tambah query `academicYearId` untuk mode UI, tapi default tetap semua AY | Backend bisa memfilter tanpa merusak gradebook (yang tidak mengirim parameter ini). |
| (C) Ubah endpoint menjadi filter eksklusif per AY | Berisiko merusak gradebook yang mengandalkan fallback ke AY lain. |

**Decision:** _TBD_

---

## EC-05: `return NoXFoundError()` vs `throw`

**Scenario:** `threshold.service.ts:47,55` melakukan `return NoSubjectFoundError()` /
`return NoAcademicYearFoundError()` — mengembalikan objek error, bukan melempar. Akibatnya
transaksi bisa "sukses" dengan respons berisi objek error. (Di luar bug utama, tetapi
muncul ketika memperbaiki `create`.)

| Opsi | Behavior |
|------|----------|
| (A) Perbaiki menjadi `throw` di seluruh `create` dan `activatedThreshold` | Error benar-benar dikembalikan sebagai HTTP error; perilaku API lebih jelas. |
| (B) Biarkan seperti sekarang; hanya perbaiki bagian `threshold_config` | Perubahan minimal terkait bug ini. |

**Decision:** _TBD_

---

## EC-06: `thresholdApi` yang tidak terpakai di `Threshold.jsx`

**Scenario:** `Threshold.jsx:52-58` memuat `getTresholdBySubjectId(subjectId, IGCSE_ALEVEL)`
dan `useResourceMapper("threshold", ...)` tetapi hasilnya tidak dipakai di JSX. Pemuatan ini
juga yang mencemari slice `threshold` dengan semua AY.

| Opsi | Behavior |
|------|----------|
| (A) Hapus pemuatan `thresholdApi` yang tidak terpakai dari `Threshold.jsx` | Mengurangi pencemaran slice & kode mati. Perlu cek `refetch` yang memanggilnya. |
| (B) Pertahankan, tetapi hentikan pemakaian slice `threshold` untuk tampilan berbasis AY | Perubahan lebih hati-hati, tetapi kode mati tetap ada. |

**Decision:** _TBD_

---

## EC-07: Data lama dengan `threshold_config.syllabus_id` kosong

**Scenario:** Threshold yang dibuat sebelum perbaikan bisa punya `threshold_config` dengan
`syllabus_id` null. Setelah perbaikan kode, data lama tetap tidak punya kaitan syllabus.

| Opsi | Behavior |
|------|----------|
| (A) Script cleanup: hapus baris `threshold_config` yang `syllabus_id IS NULL` | Bersih, tapi threshold lama tanpa config menjadi "tidak lengkap" (perlu aktivasi ulang). |
| (B) Rekontruksi: cari syllabus terbaik (`effectiveFrom <= year <= effectiveTo`) dan isi `syllabus_id` | Memulihkan data lama; risiko salah pasang bila ada >1 syllabus yang cocok. |
| (C) Tidak lakukan apa-apa; biarkan user create ulang threshold per AY | Aman, tetapi membebani user. |

**Decision:** _TBD_

---

## EC-08: Subject dengan `cambridgeProgramme` tidak standar (bukan IGCSE/ALEVEL/ASLEVEL)

**Scenario:** `Threshold.jsx:39-46` menentukan `syllabusType` dari `cambridgeProgramme`. Bila
program tidak cocok, `syllabusType` `undefined`, sehingga `getSyllabusBySubject` dipanggil
tanpa filter `syllabusType` (`ThresholSyllabus.jsx` `enabled: !!subjectId && syllabusType != null`).
Akibatnya section Syllabus bisa kosong (tidak fetch) untuk subject tersebut.

| Opsi | Behavior |
|------|----------|
| (A) Tetap tidak fetch + tampilkan pesan "Programme tidak mendukung threshold" | Jelas ke user, tidak menampilkan data ambigu. |
| (B) Fetch tanpa filter `syllabusType` sebagai fallback | Bisa menampilkan grup syllabus yang bukan milik program; berisiko misleading. |

**Decision:** _TBD_

---

## EC-09: Duplikasi syllabus lintas AY menghasilkan grup `effectiveFrom-effectiveTo` yang sama

**Scenario:** `syllabus.service.ts:112-114` membuat salinan syllabus untuk 3 AY sebelumnya
dengan rentang `effectiveYear-3 .. effectiveYear-1`. Jika rentang tersebut overlap dengan
syllabus asal, `syllabusGroupBy` di `ThresholSyllabus.jsx` menggabungkan dua baris syllabus
ke key yang sama → tampilan ganda/tercampur.

| Opsi | Behavior |
|------|----------|
| (A) Deduplikasi grup berdasarkan `syllabus.id` sebelum render | Mencegah baris ganda; key grup tetap informatif. |
| (B) Bila overlap, prioritaskan syllabus dengan `academicYearId` terbesar | Tampilan sesuai AY terbaru, tapi bisa menyembunyikan riwayat. |
| (C) Anggap di luar bug ini (data seed sudah disiplin) | Tidak ada perubahan. |

**Decision:** _TBD_
