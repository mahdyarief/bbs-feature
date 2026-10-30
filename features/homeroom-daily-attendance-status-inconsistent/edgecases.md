---
feature: Homeroom Daily Attendance Status Inconsistency (100% vs Incomplete)
slug: homeroom-daily-attendance-status-inconsistent
status: draft
author: BBS Team
date: 2026-10-30
---

# Edge Cases — Homeroom Daily Attendance Status Inconsistency

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom. Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

## EC-01: Beberapa absensi siswa dicatat oleh subject teacher (bukan homeroom)
**Scenario:** Sebagian siswa absensinya dicatat oleh guru mata pelajar, bukan homeroom teacher. Record ada di DB tapi disembunyikan oleh filter `!bySubjectTeacher` di list homeroom.

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan semua catatan absensi (homeroom + subject teacher) di list homeroom | Setiap baris siswa muncul bila ada catatan apa pun untuk tanggal tersebut |
| (B) Tampilkan hanya catatan homeroom teacher, tandai baris sebagai "belum dicatat homeroom" | Baris subject-teacher tetap kosong tapi diberi indikator |
| (C) Custom: | |

**Decision:** _TBD_

## EC-02: Jumlah siswa terisi turun di bawah jumlah total siswa (status sebelumnya COMPLETE)
**Scenario:** Hari sebelumnya status harian COMPLETE, lalu ada data absensi yang dihapus/dibatalkan sehingga tidak semua siswa terisi.

| Opsi | Behavior |
|------|----------|
| (A) Kembalikan status harian ke INCOMPLETE otomatis | `daily_attendance_status` berubah INCOMPLETE saat `total < studentCount` |
| (B) Biarkan status tetap COMPLETE (stale) | Tidak ada perubahan; header tetap 100% |
| (C) Custom: | |

**Decision:** _TBD_

## EC-03: `unmarkedAmount` threshold saat `checkCompleteness`
**Scenario:** Saat save absensi, gate `unmarkedAmount < 2` menentukan apakah backend menghitung ulang status harian.

| Opsi | Behavior |
|------|----------|
| (A) Ganti menjadi `unmarkedAmount === 0` | Sinkronisasi hanya saat semua siswa terisi |
| (B) Hapus gate; selalu hitung ulang pada setiap save | Status harian selalu akurat setelah setiap perubahan |
| (C) Custom: | |

**Decision:** _TBD_

## EC-04: Detail siswa dibuka tapi list masih INCOMPLETE
**Scenario:** Detail menunjukkan data lengkap, tapi baris di list tetap INCOMPLETE karena filter homeroom.

| Opsi | Behavior |
|------|----------|
| (A) Sinkronkan logika list dengan detail (tampilkan status aktual dari DB) | List konsisten dengan detail |
| (B) Biarkan list menggunakan filter homeroom, detail tetap menampilkan semua | List tetap bisa INCOMPLETE padahal detail lengkap |
| (C) Custom: | |

**Decision:** _TBD_
