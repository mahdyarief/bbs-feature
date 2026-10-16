# Edge Cases — GPA Report FYE/PUM Mismatch — FYE Kosong Saat Nilai Telah Diinput

> **Status: DECIDED** — root cause sudah ditentukan lewat investigasi di DB produksi.

---

## EC-01: FYE ditampilkan di GPA?

**Scenario:** Guru input FYE di gradebook, tapi GPA report tetap menampilkan `0` atau kosong.

| Opsi | Behavior |
|------|----------|
| (A) Report langsung baca FYE | Terobosan besar — akan mengubah arsitektur report dan rentan breaking change. |
| (B) Report tetap baca FYA; FYE menjadi bahan baku FYA yang digenerate via recalculate() | Perilaku existing yang sudah benar, FYE tidak pernah tampil di GPA. |
| (C) Report tampilkan kedua nilai (FYE + FYA) | Tidak sesuai spesifikasi current GPA report; butuh perubahan yang lebih besar. |

**Decision:** (B). GPA report dibatasi untuk menampilkan FYA (non-Cambridge) dan PUM (Cambridge). FYE hanya bahan baku, tidak pernah dibaca langsung.

---

## EC-02: FYE = -1 (ignored) ini sudah diabaikan?

**Scenario:** Guru mengisi nilai FYE = -1 di form.

| Opsi | Behavior |
|------|----------|
| (A) FYE = -1 dianggap "diabaikan", tidak ikut hitung FYA | Ini adalah perilaku yang sudah didefinisikan di `recalculate()` — `fyeIgnored = true`, `fyeActive = false`. |
| (B) FYE = -1 melempar error / crash | Tidak dibenarkan, ada validasi di form. |
| (C) FYE = -1 dianggap nilai 0 (null) | Salah, akan merusak weighted average FYA. |

**Decision:** (A) — FYE = -1 dianggap ignored. `fyeReady = !needsFye || fyeIgnored || fye != null`, jadi -1 tetap "ready" tanpa ikut hitung.

---

## EC-03: CA1/CA2 belum diisi

**Scenario:** Guru input FYE tapi CA1 atau CA2 masih kosong (null). Akibatnya FYA = null, row FYA dihapus oleh `upsertCalculatedGrade(score == null → grade.remove())`, GPA menampilkan `0`.

| Opsi | Behavior |
|------|----------|
| (A) GPA menampilkan 0 dan FYA column hilang dari report | Perilaku existing: `subjectFinalMark=''` → `finalMark='0'`, baris tidak masuk gpaNumerator/denominator. |
| (B) GPA menampilkan 0 tapi tetap memasukkan ke gpaNumerator/denominator | Tidak sesuai: finalMark='' (empty string) bukan angka 0. |
| (C) GPA menampilkan "incomplete" / peringatan | Perlu UI/preference baru; tidak dalam scope ini. |

**Decision:** (A) — GPA menampilkan `0` / kosong, baris tidak masuk perhitungan GPA. Ini bukan bug, ini bucket "belum lengkap".

---

## EC-04: Subjek Cambridge menggunakan PUM, bukan FYE/FYA

**Scenario:** Subject dengan `cambridgeProgramme ≠ NONE`. Guru input FYE/FYA di form, tetapi GPA tetap menggunakan `predictedPum`/`asPredictedPum` dari `student_grade`.

| Opsi | Behavior |
|------|----------|
| (A) Cambridge report baca FYE/FYA dari grade tabel | Tidak sesuai dengan arsitektur: Cambridge nilai berasal dari PUM, FK FYE/FYA biasanya null. |
| (B) Cambridge report tetap baca PUM (`predictedPum`/`asPredictedPum`) | Perilaku existing yang sudah benar. |
| (C) Tampilkan kedua PUM dan FYE/FYA | Bisa bikin kebingungan; PUM dan FYE/FYA nilainya biasanya sama (PUM adalah input FYE). |

**Decision:** (B) — subjek Cambridge tetap menggunakan PUM. PUM yang di-input guru adalah sumber FYE/FYA (via `resolvePumSource` di `recalculate()`), yang kemudian ditulis ulang sebagai Grade record FYE/FYA. Report baca `predictedPum`/`asPredictedPum` dari `student_grade`.

---

## EC-05: Bobot `fye` = 0 di `SecondaryGradeWeight`

**Scenario:** Guru input FYE, tapi bobot `fye` di config weight = 0. FYE tidak ikut hitung FYA.

| Opsi | Behavior |
|------|----------|
| (A) Bobot fye = 0 → FYE tidak diperlukan, `fyeReady = true` (karena `!needsFye`), FYA tetap terhitung tanpa FYE | Perilaku existing. |
| (B) Bobot fye = 0 → FYE dianggap tidak valid, error | Tidak dibenarkan; bobot config boleh 0 tapi itu maksudnya FYE tidak berlaku. |
| (C) Bobot fye = 0 → gagal input FYE di UI | Adegan spesifik UI, bukan di service. |

**Decision:** (A) — `needsFye = Number(weight.fye) > 0`. kalau fye = 0, FYE tidak dibutuhkan dan tidak mempengaruhi FYA. Engineer harus pastikan `fye` weight > 0 kalau FYE ingin dihitung.

---

## EC-06: `recalculate()` belum dijalankan setelah input FYE

**Scenario:** Guru input FYE di form, form `recalculate()` otomatis, tapi putus koneksi / rate limit / error → FYA belum di-generate.

| Opsi | Behavior |
|------|----------|
| (A) `recalculate()` otomatis di-trigger saat input FYE (event handler) | Inggih — perlu diverifikasi apakah event trigger-nya terjaga. |
| (B) Guru harus manual klik "recalculate" tombol | UX yang juga ada di form gradebook sekunder. |
| (C) FYE input otomatis trigger `bulkRecalculate()` dan berhasil | Ideal, tapi butuh infra yang stabil. |

**Decision:** FYE input → `recalculate()` dipanggil. Kalau tidak otomatis, engineer perlu cek event trigger-nya atau tambahkan manual "recalculate" button di form. Ini isu opsional yang perlu dicatat di notes.md.

---

## EC-07: `upsertCalculatedGrade()` menghapus row FYA kalau null

**Scenario:** `fya = null` di `recalculate()`, lalu `upsertCalculatedGrade(sg.id, FYA, 4, null, null)` dijalankan → `grade.remove()` — row FYA dihapus dari database. Tidak ada lagi FYA, GPA menampilkan `0`.

| Opsi | Behavior |
|------|----------|
| (A) `upsertCalculatedGrade` delete kalau null (perilaku existing) | Inggih — ini perilaku design; FYA tidak pernah null di DB karena generated oleh service. |
| (B) `upsertCalculatedGrade` skip kalau null (tidak insert/hapus) | Perubahan behavior; FYA row tetap ada tapi null, GPA tetap menampilkan 0. |
| (C) `upsertCalculatedGrade` hanya insert kalau tidak null (bukan update) | Perubahan behavior; FYA row yang lama nggak terhapus. |

**Decision:** (A) — perilaku existing langkang dipertahankan. FYA row di-hapus kalau null, dan ini membuat GPA report tidak menemukan nilai. Masalah utama bukan di `upsertCalculatedGrade`, tapi di `recalculate()` yang return `fya = null` karena syarat belum terpenuhi.

---

## EC-08: `recalculate()` pakai `findEffectiveWeight` dari `findEffectiveWeight`

**Scenario:** `recalculate()` memanggil `findEffectiveWeight(cy.id, sg.subjectYear.subject.id, ...)`, yang mengambil bobot dari `secondary_grade_weight` via weight config. Kalau bobot-nya overriden / expired / deleted, `weight` = null → `fya` tetap null.

| Opsi | Behavior |
|------|----------|
| (A) `findEffectiveWeight` mengembalikan weight yang valid; kalau tidak ada, `weight = null` | Perilaku existing. |
| (B) Jika `weight` null, `recalculate()` melempar error | Tidak. |
| (C) Jika `weight` null, gunakan bobot default / fallback | Perubahan behavior; potensi off-by-one. |

**Decision:** (A) — `weight` null → `fya` tetap null. Ini bagian dari root cause "CA1/CA2 tidak ada" atau "bobot tidak valid".

---

## EC-09: `student_grade` data Aarone (AKSI)

**Scenario:** `check_aarone_v3.py` (student_id = 21416) dengan `class_year_id = 100392` menunjukkan data AKSI:

| Subject | CA1 | CA2 | SA1 | SA2 | final_grade (FYA) |
|---|---|---|---|---|---|
| Bahasa Indonesia | -1.0 | -1.0 | NULL | NULL | NULL |
| Chemistry | 53.08 | -1.0 | 54.70 | -1.0 | 54.70 |
| Computer Science | 88.67 | 86.67 | 88.85 | 79.24 | 83.57 |
| English GP | 74.89 | -1.0 | 84.95 | 93.0 | 88.47 |
| Mathematics | 54.22 | 56.54 | 55.21 | 62.56 | 59.25 |
| ... | ... | ... | ... | ... | ... |

Beberapa mapel CA2 = -1.0, SA1/SA2 = NULL. `final_grade` (FYA) hanya ada di Chemistry, Computer Science, English GP, Mathematics (mapel non-Cambridge dengan final_grade terisi). Mapel yang CA2 = -1 atau SA kosong → `ca2` ilegal (score = -1) → FYA tidak bergenerasi.

**Decision:** Aku coba direct student_grade, FP=3/4 (MYE/FYE/FYA), bobot `fye`=35 untuk JC2. Report GPA baca `final_grade` (FYA). Yang error: mapel-mapel dengan CA2=-1 → FYA null.

**Decision:** (A) — FYA = null, row FYA dihapus. Mapel yang CA2=-1 atau SA yang kosong → tidak akan muncul nilai di GPA. Ini bucket "incomplete".

---

## EC-10: PUM (predictedPum / asPredictedPum) vs FYE/FYA untuk Cambridge

**Scenario:** `secondary-gpa-report.helper.ts:171-198` — subjek Cambridge (`cambridgeProgramme != NONE`) menggunakan `studentGrade?.predictedPum` (term != 2) atau `studentGrade?.asPredictedPum` (term == 2) dari `student_grade`.

| Opsi | Behavior |
|------|----------|
| (A) Cameroon report baca `predictedPum`/`asPredictedPum` | Perilaku existing, benar. |
| (B) Cameroon report baca `ca1`/CA2/MYE/FYE/FYA | Salah — untuk Cambridge nilai origin dari PUM. |

**Decision:** (A) — Cameroon report baca PUM dari `student_grade`. `predictedPum`/`asPredictedPum` di `student_grade` adalah nilai input dari guru untuk Cambridge subjects, yang kemudian di-parse ke FYE/FYA oleh `recalculate()` (via `resolvePumSource`). Tapi GPA report langsung baca PUM, bukan FYE/FYA.

---

## EC-11: Bug `null !== undefined` di `resolvePumSource` menghapus row FYE dari PUM

**Scenario:** Untuk subjek Cambridge, `recalculate()` mengambil `predictedPum`/`asPredictedPum` (PUM) dan menulisnya ulang sebagai Grade record FYE:

```ts
if (pumSource?.fye !== undefined) {
  await this.upsertCalculatedGrade(
    sg.id,
    GradeExamTypeEnum.FYE,
    4,
    null,
    pumSource.fye,   // ← bisa NULL
  );
}
```

Ketika `predictedPum` di DB = NULL, `pumSource.fye` bernilai `NULL` (bukan `undefined`). Karena `null !== undefined` adalah `true`, blok ini tetap dieksekusi, dan `upsertCalculatedGrade` dipanggil dengan `score = null`, yang pada baris 463-465 akan **menghapus** row FYE yang sudah ada:

```ts
if (score == null) {
  if (grade) await grade.remove();
  return;
}
```

Akibat: setiap kali `recalculate()` dijalankan untuk subjek Cambridge, row FYE dari PUM ikut terhapus. Jadi FYE dari PUM tidak pernah tersimpan sebagai Grade record yang stabil. Data Aarone menunjukkan **semua subjek Cambridge punya `predicted_pum = NULL`**, sehingga semua row FYE berpotensi hapus saat `recalculate()` berjalan.

| Opsi | Behavior |
|------|----------|
| (A) Cek `pumSource.fye != null` sebelum upsert, biar kalau PUM null row FYE tidak ikut terhapus | Perilaku existing (bug). |
| (B) Upsert tetap dieksekusi walau `pumSource.fye` = null | Memperparah: row FYE terus terhapus. |
| (C) `resolvePumSource` return null kalau `predictedPum` = null | Perubahan behavior; FYE dari PUM tidak pernah ditulis. |

**Decision:** (A) — tambahkan pengecekan `pumSource.fye != null` sebelum upsert, atau ganti `!== undefined` menjadi `!= null` (handles both `null` and `undefined`). Ini memperbaiki lapis 2. Lapis 1 (CA1/CA2 = -1) tetap menjadi penyebab utama FYA kosong di GPA.
