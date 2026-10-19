# Edge Cases — National Report Sequence

> **Status: DECIDED 2026-10-19 (user confirmed via Q&A)** — scope se-KJ per AY, urut abjad nama (A–Z), persist di tabel baru. Implementasi kode menyusul di task terpisah.

## EC-01: Siswa ditambah setelah sequence sudah di-generate
**Scenario:** Class sudah di-generate (nomor 002–033 assigned), lalu ada siswa baru (mis. pindahan masuk) yang namanya jatuh di tengah urutan abjad.

| Opsi | Behavior |
|------|----------|
| (A) Assign nomor baru di akhir (next max+1), tidak renumber yang lama | Stabil, tidak ubah nomor yang sudah tercetak. Rekomendasi. |
| (B) Regenerate penuh — semua nomor berubah | Berisiko: dokumen yang sudah dicetak jadi tidak cocok. |

**Decision:** **(A)** — nomor yang sudah assigned tidak boleh berubah; siswa baru dapat nomor lanjutan.

---

## EC-02: Siswa pindah class dalam AY yang sama
**Scenario:** Siswa pindah dari JC2 Mendel ke JC2 Forbes di AY yang sama setelah nomor assigned.

| Opsi | Behavior |
|------|----------|
| (A) Nomor tetap (dibawa) — nomor tied ke siswa+AY, bukan class | Sesuai keputusan se-KJ per AY. Rekomendasi. |
| (B) Nomor di-revoke, dapat nomor baru di class baru | Merusak stabilitas + audit trail. |

**Decision:** **(A)** — nomor mengikuti siswa dalam AY tersebut.

---

## EC-03: Siswa tidak lulus / dikeluarkan setelah nomor assigned
**Scenario:** Siswa assigned nomor lalu dinyatakan tidak lulus atau keluar sekolah.

| Opsi | Behavior |
|------|----------|
| (A) Nomor tetap tercatat (soft), tidak dipakai ulang | Audit trail utuh; ada gap di sequence — wajar. |
| (B) Nomor dicabut dan dipakai ulang untuk siswa lain | Merusak keunikan historis. |

**Decision:** **(A)** — gap di sequence dapat diterima (sudah ada di legacy: 003, 006, 007, 008 dst. kosong).

---

## EC-04: Data test / sample student ikut ter-generate
**Scenario:** "Sample Student S3A Taylor" (nomor 41 di PDF) ikut dalam daftar generate.

| Opsi | Behavior |
|------|----------|
| (A) Filter: skip record bertanda test/sample sebelum assign | Produksi bersih. Rekomendasi. |
| (B) Ikut di-assign seperti siswa biasa | Mengotori sequence produksi. |

**Decision:** **(A)** — test data di-exclude dari sequence produksi.

---

## EC-05: Generate ulang (re-run) dengan data sama
**Scenario:** Admin menekan generate dua kali berturut-turut.

| Opsi | Behavior |
|------|----------|
| (A) Idempotent — nomor yang sudah ada tidak berubah (upsert) | Aman. Rekomendasi. |
| (B) Regenerate penuh — nomor bisa berubah | Berbahaya untuk dokumen yang sudah dicetak. |

**Decision:** **(A)** — upsert idempotent per (student_id, academic_year_id).

---

## EC-06: Nama siswa sama persis (duplikat)
**Scenario:** Dua siswa dengan nama lengkap identik di AY yang sama.

| Opsi | Behavior |
|------|----------|
| (A) Tiebreak stabil: NISN, lalu student_id | Deterministik, reproducible. Rekomendasi. |
| (B) Urutan acak / by id saja | Tidak deterministik terhadap nama. |

**Decision:** **(A)** — tiebreak NISN → student_id.

---

## EC-07: Format tahun di nomor
**Scenario:** Nomor memakai tahun berapa — tahun kelulusan (2025), AY (2024/2025), atau tahun generate (2026)?

**Observed (PDF JC2 Mendel):** `002/CJCE/KJ/2025` untuk AY 2024/2025 dengan tanggal kelulusan 4 Mei 2026 — tahun di nomor = **2025** (tahun ajaran berakhir / tahun kelulusan), bukan tahun generate.

**Decision:** tahun = **tahun akhir AY kelulusan** (mis. AY 2024/2025 → 2025).
