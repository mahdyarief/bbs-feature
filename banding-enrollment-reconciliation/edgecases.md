# Edge Cases — Banding Enrollment Reconciliation

> **Status: DECIDED (2026-09-30)** — keputusan EC-01 diambil user setelah bukti
> lengkap (LIG soft-deleted via UI 2026-09-30 03:20-03:21 UTC). EC-02..EC-05 tetap
> PENDING sebagai panduan operasional.

## EC-01: Delta terbukti karena perubahan manual user (bukan drift)
**Scenario:** User mengonfirmasi ia menghapus siswa dari banding 111981/111985/111986/111987 via UI (mis. Character First tidak diikuti sebagian siswa).

| Opsi | Behavior |
|------|----------|
| (A) Keep current state | Tidak ada pemulihan; baseline di `actions/05-assign-banding-students/result.md` di-update dengan catatan perubahan manual |
| (B) Restore baseline | Semua junction dikembalikan ke 48 — menimpa keputusan manual user |

**Decision: (A) KEEP CURRENT STATE** — diputuskan 2026-09-30.

### Bukti pendukung keputusan

Cross-check TC-035 (LIG soft-deleted) ↔ TC-010 (banding delta) menunjukkan pola
**unenroll yang disengaja dan konsisten**, bukan drift:

| Subject | Banding kehilangan | LIG soft-deleted (via UI, 03:20-03:21 UTC) |
|---|---|---|
| 112398 PPKn (banding 111986) | 100334 keluar (tersisa 100215, 100312) | 100334 t1+t2 di-soft-delete ✅ cocok |
| 112399 Indonesian Studies (banding 111987) | 100215, 100312 keluar (tersisa 100334) | 100215 t1 + 100312 t1 di-soft-delete ✅ cocok |
| 112393 Character First (111981) | 100215, 100312 keluar (tersisa 100334) | LIG subject ini memang tidak ada (5 subject tanpa LO) |
| 112397 Faith Builder (111985) | 100334 keluar (tersisa 100215, 100312) | LIG memang tidak ada |

Artinya: unenroll via UI **ikut menghapus data turunan LIG** (perilaku benar —
kebalikan bug `unenroll-orphan-learning-indicator-grade`). State sekarang adalah
hasil operasi UI yang disengaja oleh user, bukan kerusakan data.

### Tindak lanjut yang dilakukan

1. Baseline di `operate-smartbag/actions/05-assign-banding-students/result.md`
   dan `test-cases/TC-010-banding-promote/result.md` di-update: state 42 junction
   adalah **baseline baru** (hasil unenroll manual), bukan delta yang harus dipulihkan.
2. `test-cases/TC-037-cross-module-consistency/result.md` menyimpan snapshot profil
   siswa sebagai pembanding eksekusi berikutnya.
3. Tidak ada write pemulihan yang dijalankan.

---

## EC-02: Siswa yang hilang dari banding sudah punya LO grade di subject itu
**Scenario:** Siswa yang dihapus dari banding 111981 (Character First) masih punya baris `learning_indicator_grade` untuk subject_year 112393 (dari action 06, 3 siswa per subject).

| Opsi | Behavior |
|------|----------|
| (A) Restore banding, pertahankan LO grade | Konsisten kembali ke baseline; nilai tetap sah |
| (B) Restore banding + bersihkan LO grade subject itu untuk siswa tsb | Konsisten dengan pola "unenroll = bersihkan data turunan" (lihat `unenroll-orphan-learning-indicator-grade`) |

**Decision:** _TBD_

---

## EC-03: Muncul junction "ekstra" yang tidak ada di baseline
**Scenario:** Saat rekonsiliasi ditemukan siswa terdaftar di banding yang di baseline tidak ada (tambahan, bukan pengurangan).

| Opsi | Behavior |
|------|----------|
| (A) Keep | Tambahan dianggap perubahan sah; hanya dilaporkan |
| (B) Remove | Kembali persis ke baseline (write berisiko) |

**Decision:** _TBD_

---

## EC-04: Endpoint API banding tidak menerima assignment per-student (hanya create banding)
**Scenario:** Investigasi menunjukkan `POST /bandings` membuat banding baru beserta siswanya, tidak ada endpoint "tambah siswa ke banding existing".

| Opsi | Behavior |
|------|----------|
| (A) POST ulang banding dengan daftar siswa lengkap | Mengikuti perilaku UI; risiko overwrite tergantung service (wajib baca service dulu) |
| (B) DB insert ke junction dengan persetujuan | Terkontrol tapi tanpa jejak `updatedBy` aplikasi |

**Decision:** _TBD_

---

## EC-05: Baseline berubah setelah rekonsiliasi dijalankan
**Scenario:** Rekonsiliasi selesai (junction = 48), lalu user kembali mengubah enrollment via UI minggu depan dan TC-010 dieksekusi ulang menemukan delta baru.

| Opsi | Behavior |
|------|----------|
| (A) Snapshot baseline otomatis tiap selesai action | `actions/*/result.md` selalu menjadi sumber kebenaran terbaru |
| (B) Treat setiap delta sebagai keputusan manual baru | Tidak ada pemulihan otomatis; selalu tanya user |

**Decision:** _TBD_
