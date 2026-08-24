---
title: CCA Registration — Race Condition pada Quota Validation Memungkinkan Over-Enrollment
status: open
severity: critical
product: BBS LMS
portal: Student Portal
author: System Analyst
date: 2026-08-24
---

# CCA Registration Quota — Race Condition Mengizinkan Melebihi Quota

## Summary

Ketika sebuah CCA-Year memiliki quota sangat terbatas (misal: `quota = 1`), dan **2 atau lebih siswa mendaftar secara bersamaan** dari akun berbeda, **kedua registrasi berhasil** meskipun quota hanya 1. Sistem menampilkan jumlah pendaftar melebihi quota (misal: `2/1`).

**Ekspektasi:** Sistem harus menolak registrasi kedua ketika quota sudah terpenuhi, bahkan jika pendaftaran dilakukan secara bersamaan (concurrent requests).

---

## Steps to Reproduce

1. Login sebagai **Admin** ke Admin Portal
2. Buat CCA-Year dengan konfigurasi:
   - `quota = 1`
   - Status: **Open** untuk pendaftaran
3. Login sebagai **Student A** ke Student Portal
4. Login sebagai **Student B** ke Student Portal (akun berbeda, browser/device berbeda)
5. **Student A** dan **Student B** membuka CCA yang sama di halaman CCA Registration
6. **Kedua siswa klik tombol "Daftar" secara bersamaan** (dalam selisih milidetik)
7. Amati hasil registrasi

**Actual Result:**
- Kedua registrasi **berhasil**
- Sistem menampilkan `2/1` pada CCA tersebut (melebihi quota)
- Tidak ada error message yang muncul

**Expected Result:**
- Hanya **1 siswa** yang berhasil mendaftar
- Siswa kedua mendapatkan error: `"Quota for this CCA has been fully subscribed. Please choose another CCA."`
- Counter menampilkan `1/1`

---

## Root Cause Analysis

### Backend — `cca-registration.service.ts` (line 137-164)

Method `validateQuota()` melakukan **check-then-act** tanpa mekanisme locking atau transaction isolation:

```typescript
// Line 158-159: COUNT query - check current registrations
const currentCount = await this.ccaRegistrationRepository.count({
  where: { ccaYearId, status: 'approved' },
});

// Line 161-164: Compare with quota
if (ccaYear.quota && currentCount >= ccaYear.quota) {
  QuotaExceededError();
}
```

**Masalah:**
1. **Request A** menjalankan `count()` → result: `0` (belum ada yang daftar)
2. **Request B** menjalankan `count()` → result: `0` (karena Request A belum commit)
3. **Request A** lolos validasi → `create()` registration
4. **Request B** lolos validasi → `create()` registration
5. **Result:** 2 registrasi berhasil, quota terlampaui

Ini adalah **classic race condition** dalam concurrent programming — tidak ada database-level locking, optimistic locking, atau transaction isolation yang mencegah kedua request membaca data yang sama sebelum salah satu commit.

### Database Schema

Tidak ada **unique constraint** atau **database trigger** yang membatasi jumlah registrasi per `ccaYearId`. Validasi hanya dilakukan di application layer.

---

## Impact

### Severity: **Critical** 🔴

- **Fairness Issue:** Siswa yang seharusnya tidak kebagian quota tetap berhasil mendaftar, merugikan siswa lain yang menunggu
- **Data Integrity:** Counter menampilkan data tidak konsisten (`2/1`, `3/1`, dst.)
- **Operational Burden:** Admin harus manual review dan cancel registrasi yang melebihi quota
- **Trust Issue:** Siswa dan orang tua kehilangan kepercayaan terhadap sistem jika quota tidak ditegakkan dengan benar
- **Compliance Risk:** Untuk CCA dengan quota ketat (misal: limited resources, safety constraints), over-enrollment bisa melanggar kebijakan sekolah

---

## Recommended Fix

### Option 1: Database-Level Locking (Recommended)

Gunakan **pessimistic locking** dengan `FOR UPDATE` dalam transaction:

```typescript
return await this.dataSource.transaction(async (manager) => {
  // Lock the ccaYear row
  const ccaYear = await manager
    .createQueryBuilder(CcaYear, 'ccaYear')
    .setLock('pessimistic_write')
    .where('ccaYear.id = :ccaYearId', { ccaYearId })
    .getOne();

  // Count with lock
  const currentCount = await manager
    .createQueryBuilder(CcaRegistration, 'reg')
    .setLock('pessimistic_write')
    .where('reg.ccaYearId = :ccaYearId', { ccaYearId })
    .andWhere('reg.status = :status', { status: 'approved' })
    .getCount();

  if (ccaYear.quota && currentCount >= ccaYear.quota) {
    QuotaExceededError();
  }

  // Create registration within same transaction
  return await manager.save(CcaRegistration, registrationData);
});
```

### Option 2: Optimistic Locking dengan Version Column

Tambahkan `version` column di `CcaYear` entity. Setiap update increment version. Jika version berubah saat save, retry atau reject.

### Option 3: Database Constraint

Tambahkan **trigger** atau **check constraint** di database level yang membatasi jumlah `approved` registrations per `ccaYearId`.

### Option 4: Redis Distributed Lock

Gunakan Redis lock dengan TTL untuk serialize quota checks:

```typescript
const lockKey = `cca-registration:${ccaYearId}`;
const lock = await redis.acquireLock(lockKey, 5000); // 5s TTL

try {
  // validate quota
  // create registration
} finally {
  await lock.release();
}
```

---

## Additional Considerations

### Edge Cases

1. **Retry Logic:** Jika user klik "Daftar" berkali-kali karena loading lama

### Testing Requirements

- Concurrent registration test dengan 2+ simultaneous requests
- Load testing untuk validate locking mechanism under high concurrency
- Verify rollback behavior jika transaction gagal

---

## References

- Code: `api_nest/src/modules/cca-registration/cca-registration.service.ts:137-164`
- Related: `cca-registration-quota-filter/bug-report.md` (gender quota filter issue)
- Database: `cca_year` dan `cca_registration` tables
- Pattern: Check-Then-Act Race Condition
