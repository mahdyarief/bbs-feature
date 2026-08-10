# Edge Cases — FTP Evaluation Skip Empty Term

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

## EC-01: Student masuk di term 3, tidak ada conduct rows term 1/2 sama sekali
**Scenario:** Student baru terdaftar di term 3 (misal student 103033, student_report 3758). `student_ftp_conduct` hanya berisi term 3 & 4; term 1/2 = 0 rows. Saat generate report FTP Evaluation, `ftpEvaluation` menerima semua conduct (tidak difilter term), sehingga score term 1/2 dihitung dari data term 3/4.

| Opsi | Behavior |
|------|----------|
| (A) Skip term tanpa rows di `termsToEvaluate` + filter `groupByTerm` | Term 1/2 tidak dibuat eval row; score hanya dari term 3/4. Paling sesuai intent bisnis. |
| (B) Tetap buat eval row term 1/2 tapi score = 0 | Report tetap menampilkan "Term 1: 0 BELOW" — masih menyesatkan (menunjukkan student dinilai di term 1/2). |
| (C) Biarkan seperti sekarang | Score term 1/2 = akumulasi data term 3/4 (salah semantik, kondisi bug saat ini). |

**Decision:** **A — Term yang TIDAK punya conduct rows sama sekali (null) diabaikan:** tidak dibuat eval row, tidak menyumbang point. (Sesuai kondisi yang disepakati: "null = diabaikan".)

---

## EC-02: Ada conduct rows term 1/2 tapi badges 0/null di semua Character/Traits/Values
**Scenario:** Guru pernah membuka form grading term 1/2 lalu menyimpan kosong, sehingga rows `student_ftp_conduct` term 1/2 ada dengan badges 0 (default kolom). `getConductPoint(0)` saat ini return **1** (ftp-evaluation.ts:21-23) — term 1/2 "kosong" tetap menambah point minimum.

| Opsi | Behavior |
|------|----------|
| (A) Skip group yang `totalBadges <= 0` | Term yang badges-nya 0 semua → 0 point. Konsisten dengan EC-01 (prinsip "tidak ada data = tidak ada kontribusi"). |
| (B) Hitung point minimum (perilaku sekarang) | Term badges 0 tetap +1 point per table. Student dapat nilai "gratis". |
| (C) Skip berdasarkan `reportStatus !== DONE` | Hanya rows berstatus DONE yang dihitung; ON_PROGRESS/NOT_STARTED di-skip. Perlu pastikan semua flow grading mengisi reportStatus dengan benar. |

**Decision:** **B — Row yang ADA dengan badges 0 tetap dihitung (point minimum 1).** Hanya **ketiadaan row (null)** yang diabaikan. (Sesuai kondisi yang disepakati: "ada rownya dan nilainya 0 = tetap dihitung".)

---

## EC-03: Eval row term 1/2 sudah terlanjur ada di DB (data tercemar)
**Scenario:** Student 103033 sudah punya eval rows term 1/2 (score 21/22, BELOW_EXPECTATIONS) di `student_ftp_conduct_evaluation` meskipun tidak ada conduct data term 1/2. Setelah enhancement diimplementasi, rows ini tetap ada di DB.

| Opsi | Behavior |
|------|----------|
| (A) Skip saat render saja (`createStudentFtpEvaluationReport` filter term valid) | Rows tetap di DB, tapi tidak tampil di report. Tidak ada perubahan data. |
| (B) Hapus/soft-delete rows invalid via script cleanup | Data bersih, tapi butuh migration script/backfill + keputusan tim sebelum jalan di production. |
| (C) Biarkan + recompute dengan rule baru | Rows tetap tampil; score akan berubah (kemungkinan jadi 0/BELOW). Perlu cek apakah report menampilkan row dengan score 0. |

**Decision:** **C — Biarkan rows tetap ada; nilai akan diperbarui (recompute) saat report di-update setelah implementasi kalkulasi baru.** Tidak ada script cleanup terpisah. Eval rows term 1/2 yang tidak punya conduct data akan otomatis berubah nilainya (di-skip / tidak dihitung) ketika report FTP Evaluation di-generate ulang dengan kalkulasi baru. (Keputusan tim 2026-08-10.)

---

## EC-04: Student full-year dengan conduct lengkap term 1-4
**Scenario:** Student terdaftar sejak term 1 dan punya conduct rows di semua term. Enhancement tidak boleh mengubah hasil evaluasinya.

| Opsi | Behavior |
|------|----------|
| (A) Filter term hanya yang punya rows (minTerm = 1) | Semua term 1-4 tetap dihitung → score tidak berubah. Regression aman. |
| (B) Tanpa pengecekan | Tidak relevan — ini baseline. |

**Decision:** **A — Filter term hanya yang punya rows (minTerm = 1):** semua term 1-4 tetap dihitung → score tidak berubah. Regression aman. (Ikut rekomendasi.)

---

## EC-05: Student tidak punya conduct data sama sekali di semua term
**Scenario:** Student terdaftar di class year tapi belum pernah diisi grading FTP di term manapun (misal baru daftar akhir term 4, atau guru belum mengisi).

| Opsi | Behavior |
|------|----------|
| (A) Tidak buat eval row sama sekali | Report FTP Evaluation kosong / menampilkan pesan "belum ada data". |
| (B) Buat 1 eval row untuk term yang diminta dengan score 0 BELOW | Report menampilkan term tsb dengan nilai kosong — menyesatkan. |
| (C) Buat eval row hanya jika `syncFtpEvaluation`/generate dipanggil eksplisit (perilaku sekarang) | Perlu diputuskan apakah generate tetap jalan atau di-skip. |

**Decision:** **A — Tidak buat eval row sama sekali:** report FTP Evaluation kosong / menampilkan pesan "belum ada data". (Ikut rekomendasi.)

---

## EC-06: `ftpGradeSetting` threshold tidak ditemukan untuk term tertentu
**Scenario:** `FtpGradeSetting` untuk academic year + master level + term tidak ada (belum di-set admin).

| Opsi | Behavior |
|------|----------|
| (A) `getConductPoint` return 0 (perilaku existing, ftp-evaluation.ts:26) | Semua point 0 → score 0 BELOW. Tidak ada crash. |
| (B) Throw error / block generate | Generate report gagal; perlu pesan error jelas untuk admin. |

**Decision:** **A — `getConductPoint` return 0 (perilaku existing):** semua point 0 → score 0 BELOW. Tidak ada crash; pertahankan perilaku existing, jangan ubah di scope brief ini. (Ikut rekomendasi.)
