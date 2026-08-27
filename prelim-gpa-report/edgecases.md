---
feature: Prelim Report & GPA June/Nov
slug: prelim-gpa-report
---

# Edge Cases — Prelim Report & GPA June/Nov

## EC-01: Level tidak sesuai dengan reportType

**Skenario:** User memilih reportType CAMBRIDGE_IGCSE untuk class JC2 (yang seharusnya AS/A Level).

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Batasi pilihan reportType berdasarkan level class + term (seperti `cambridgeReportTypesForTerm.js`) | Mencegah error dari awal |
| **B** | Izinkan semua reportType, backend return error 422 jika data tidak sesuai | Fleksibel tapi rawan error |
| **C** | Deteksi otomatis level → pilihkan reportType yang tepat | User-friendly |

**Decision:** _TBD — rekomendasi Opsi A (FE filtering) + C (auto-detect fallback)._

---

## EC-02: Data moderation tidak ada

**Skenario:** Prelim Report di-generate sebelum proses moderation dilakukan.

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Generate report dengan PUM gradebook (tanpa moderation) | Report tidak akurat |
| **B** | Return error "Data moderation belum ada" | Admin harus melakukan moderation dulu |
| **C** | Tampilkan peringatan tapi tetap generate | Transparan |

**Decision:** _TBD — rekomendasi Opsi A (PUM gradebook sebagai fallback, moderation jika ada)._

---

## EC-03: GP Value tidak terdefinisi untuk grade tertentu

**Skenario:** Nilai siswa menghasilkan grade yang tidak ada di `GpaGradingScale.config`.

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Skip subject tersebut dari kalkulasi GPA | GPA tidak lengkap |
| **B** | Gunakan GP Value = 0 untuk grade yang tidak terdefinisi | Menurunkan GPA |
| **C** | Return error | Admin harus update grading scale |

**Decision:** _TBD — rekomendasi Opsi B (GP Value = 0) dengan catatan di report._

---

## EC-04: Subject unit tidak terdefinisi

**Skenario:** Subject tidak memiliki entry di `gpa_subject_unit` untuk AY tertentu.

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Default unit = 1.0 | Aman, standar |
| **B** | Skip subject | GPA tidak lengkap |
| **C** | Return error | Admin harus konfigurasi unit |

**Decision:** _TBD — rekomendasi Opsi A (default 1.0, sesuai kode `subjectUnit?.unit || 1.0`)._

---

## EC-05: Tidak ada data Cambridge grade (PRELIMINARY)

**Skenario:** Siswa belum memiliki data `student_cambridge_grade` dengan `gradeType=PRELIMINARY`.

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Tampilkan nilai kosong/placeholder di report | Dokumen tidak valid |
| **B** | Return error 422 "No preliminary data" | Admin harus input nilai dulu |
| **C** | Gunakan data PUM dari student_grade saja | Tidak akurat |

**Decision:** _TBD — rekomendasi Opsi C (PUM gradebook sebagai fallback)._

---

## EC-06: Class campuran level (JC2 + Sec 4)

**Skenario:** Satu class berisi siswa dari dua level berbeda (misal JC2 dan Sec 4).

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Deteksi level per siswa, terapkan threshold berbeda | Akurat tapi kompleks |
| **B** | Gunakan threshold class level | Tidak akurat untuk sebagian siswa |

**Decision:** _TBD — rekomendasi Opsi A (per-siswa, seperti di legacy)._

---

## EC-07: A Level composite data tidak lengkap

**Skenario:** Prelim A Level butuh AS actual marks sebagai overlay, tapi data AS belum ada.

| Opsi | Deskripsi | Dampak |
|------|-----------|--------|
| **A** | Generate hanya dari data prelim yang ada | Tidak komposit |
| **B** | Return error | Admin harus input AS dulu |
| **C** | Tampilkan peringatan + PUM prelim saja | Transparan |

**Decision:** _TBD — rekomendasi Opsi A (fallback ke prelim-only)._
