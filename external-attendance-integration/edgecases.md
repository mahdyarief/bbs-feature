# Edge Cases — External Attendance Integration

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.

## EC-01: Multiple Check-in di Hari yang Sama
**Scenario:** Student sudah check-in jam 08:00, device kirim check-in lagi jam 09:00 di hari yang sama.

| Opsi | Behavior |
|------|----------|
| (A) Reject | Return error "Already has check-in record" |
| (B) Overwrite | Update check-in lama dengan yang baru |
| (C) Create baru | Allow multiple check-in, isFirstInserted tetap true untuk yang pertama |

**Decision:** (C) Create baru | Allow multiple check-in, isFirstInserted tetap true untuk yang pertama

---

## EC-02: Multiple Check-out di Hari yang Sama
**Scenario:** Student sudah check-out jam 15:00, device kirim check-out lagi jam 16:00.

| Opsi | Behavior |
|------|----------|
| (A) Reject | Return error "Already has check-out record" |
| (B) Overwrite | Update check-out lama dengan yang baru |
| (C) Create baru | Allow multiple check-out |

**Decision:** (A) Reject | Return error "Already has check-out record"

---

## EC-03: Check-in Setelah Check-out
**Scenario:** Student check-in 08:00, check-out 12:00, lalu check-in lagi jam 13:00 (misal keluar makan siang).

| Opsi | Behavior |
|------|----------|
| (A) Reject | Return error "Cannot check-in after check-out" |
| (B) Allow | Create new check-in record (multiple attendance sessions per day) |

**Decision:** (B) Allow | Create new check-in record (multiple attendance sessions per day)

---

## EC-04: Retroactive Attendance (Tanggal Lampau)
**Scenario:** Device kirim attendance untuk tanggal kemarin atau minggu lalu.

| Opsi | Behavior |
|------|----------|
| (A) Reject semua | Hanya boleh attendance untuk hari ini |
| (B) Allow dengan batas | Max N hari ke belakang |
| (C) Allow tanpa batas | Admin bisa input retroactive kapan saja |

**Decision:** (C) Allow tanpa batas | Admin bisa input retroactive kapan saja

---

## EC-05: Future Attendance (Tanggal Mendatang)
**Scenario:** Device kirim attendance untuk besok atau lusa.

| Opsi | Behavior |
|------|----------|
| (A) Reject semua | Tidak boleh attendance untuk masa depan |
| (B) Allow dengan batas | Max N hari ke depan (pre-schedule) |
| (C) Allow tanpa batas | Admin bisa schedule kapan saja |

**Decision:** (C) Allow tanpa batas | Admin bisa schedule kapan saja

---

## EC-06: Holiday / Weekend Attendance
**Scenario:** Student tap card di hari libur atau weekend.

| Opsi | Behavior |
|------|----------|
| (A) Reject | Return error "Cannot attend on holiday/weekend" |
| (B) Allow | Tetap create record (mungkin ada event khusus) |
| (C) Allow dengan flag | Create record tapi mark sebagai "holiday attendance" |

**Decision:** (B) Allow | Tetap create record (mungkin ada event khusus)

---

## EC-07: Campus/Gate Mismatch
**Scenario:** Payload kirim `campusID = 1` tapi `gateID = 5` (gate 5 bukan milik campus 1).

| Opsi | Behavior |
|------|----------|
| (A) Reject | Return error "Gate does not belong to campus" |
| (B) Ignore campusID | Hanya pakai gateID, skip validation |
| (C) Auto-correct | Lookup campus dari gate, ignore campusID di payload |

**Decision:** (C) Auto-correct | Lookup campus dari gate, ignore campusID di payload

---

## EC-08: Concurrent Requests (Race Condition)
**Scenario:** 2 device kirim request bersamaan untuk student yang sama di waktu yang hampir sama.

| Opsi | Behavior |
|------|----------|
| (A) Database lock | Pakai transaction lock, yang kedua tunggu |
| (B) Optimistic lock | Detect conflict, retry otomatis |
| (C) Last write wins | Yang terakhir menang, yang pertama overwrite |

**Decision:** (C) Last write wins | Yang terakhir menang, yang pertama overwrite

---

## EC-09: Student Pindah Class Year di Tengah Hari
**Scenario:** Student check-in jam 08:00 (class year A), admin pindah ke class year B, lalu device kirim check-out jam 15:00.

| Opsi | Behavior |
|------|----------|
| (A) Use current class year | Check-out pakai class year B (yang sekarang) |
| (B) Use original class year | Check-out pakai class year A (yang waktu check-in) |
| (C) Reject check-out | Return error "Class year changed, contact admin" |

**Decision:** (A) Use current class year | Check-out pakai class year B (yang sekarang)

---

## EC-10: Invalid Time Format
**Scenario:** Payload kirim `timelog = "25:00:00"` atau `timelog = "abc"` atau `timelog = ""`.

| Opsi | Behavior |
|------|----------|
| (A) Reject | Return error "Invalid time format" |
| (B) Auto-correct | Clamp ke 23:59:59 atau 00:00:00 |
| (C) Skip record | Skip record ini, lanjut ke record berikutnya |

**Decision:** (B) Auto-correct | Clamp ke 23:59:59 atau 00:00:00

---

## EC-11: Bulk Request Size Limit
**Scenario:** External system kirim ribuan records dalam satu request.

| Opsi | Behavior |
|------|----------|
| (A) Reject | Max N records per request |
| (B) Allow tanpa limit | Process semua, tapi mungkin timeout |
| (C) Chunk processing | Split jadi batch, process sequential |

**Decision:** (C) Chunk processing | Split jadi batch, process sequential

---

## EC-12: Student Inactive di Tengah Proses
**Scenario:** Student check-in jam 08:00 (status CURRENT), admin suspend student jam 10:00, device kirim check-out jam 15:00.

| Opsi | Behavior |
|------|----------|
| (A) Reject check-out | Return error "Student is no longer active" |
| (B) Allow check-out | Tetap process karena check-in sudah ada |
| (C) Allow dengan flag | Mark sebagai "incomplete attendance" |

**Decision:** (B) Allow check-out | Tetap process karena check-in sudah ada

---

## EC-13: Duplicate Card Number
**Scenario:** Ada 2 student dengan `cardNumber` yang sama (data corruption atau human error).

| Opsi | Behavior |
|------|----------|
| (A) Reject semua | Return error "Multiple students found with this card number" |
| (B) Use first match | Pakai student yang pertama ditemukan |
| (C) Use most recent | Pakai student yang terakhir di-create |

**Decision:** (D) Cek student_status yang aktif

---

## EC-14: Timezone Edge Case
**Scenario:** Server di timezone berbeda, payload GMT+7.

| Opsi | Behavior |
|------|----------|
| (A) Always GMT+7 | Ignore server timezone, always convert from GMT+7 |
| (B) Server timezone | Convert dari server timezone ke DB timezone |
| (C) UTC only | Semua data di-convert ke UTC |

**Decision:** Ikuti penyimpanan timezone yang sudah ada

---

## EC-15: Late Time Setting Missing
**Scenario:** Student's level/programme tidak punya `lateTimeSetting` configured.

| Opsi | Behavior |
|------|----------|
| (A) Reject | Return error "Late time setting not configured" |
| (B) Use default | Pakai default late time (misal 08:00:00) |
| (C) Skip late check | Mark semua sebagai PRESENT, skip late detection |

**Decision:** (C) Skip late check | Mark semua sebagai PRESENT, skip late detection
