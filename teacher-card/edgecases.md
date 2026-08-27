# Edge Cases — Teacher Card (Kartu Guru)

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

## EC-01: Foto Tidak Ada

**Scenario:** Staff tidak memiliki foto profil di `employee_identity.photo_url` (null / empty).

| Opsi | Behavior |
|------|----------|
| (A) Placeholder inisial | Tampilkan lingkaran dengan inisial nama (misal "BS" untuk Budi Santoso) — background warna berdasarkan hash employeeId. |
| (B) Icon default user | Tampilkan icon/user avatar default (generic person icon). |
| (C) Area kosong | Area foto dibiarkan kosong tanpa placeholder. |

**Decision:** _TBD_

---

## EC-02: Staff Nonaktif / Keluar / Resigned

**Scenario:** Staff dengan `active_status = INACTIVE` atau `RESIGNED` diakses kartunya.

| Opsi | Behavior |
|------|----------|
| (A) Tetap tampil dengan badge | Kartu tetap ditampilkan dengan badge/label "Nonaktif" atau "Resigned" di bagian atas kartu. Data identity tetap lengkap. |
| (B) Tolak akses | Kembalikan 404 atau 403 — staff nonaktif tidak bisa diakses kartunya. |
| (C) Tampil terbatas | Kartu ditampilkan tanpa foto dan tanpa data kontak (email/telepon). |

**Decision:** _TBD_

---

## EC-03: NIP / NIK Kosong

**Scenario:** Staff tidak memiliki NIP (misal staff honorer baru) atau NIK (data kependudukan belum masuk).

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan "-" (dash) | Field NIP/NIK yang kosong ditampilkan sebagai "-" atau "—". |
| (B) Sembunyikan field | Field yang kosong tidak ditampilkan di kartu. |
| (C) Tampilkan "Belum diisi" | Tampilkan teks "Belum diisi" dengan warna abu-abu. |

**Decision:** _TBD_

---

## EC-04: Staff Pindah Campus

**Scenario:** Staff dipindahkan dari campus A ke campus B, tetapi admin/teacher memilih campus yang lama saat membuka kartu.

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan data terbaru | Kartu selalu menampilkan campus/posisi terbaru dari database (campusId diambil dari `employee.campus_id` saat ini), mengabaikan campusId yang dikirim. |
| (B) Validasi campus mismatch | Backend memvalidasi campusId yang dikirim cocok dengan `employee.campus_id`; jika tidak cocok → 400 "Staff is not in your campus". |
| (C) Tampilkan sesuai request + catatan riwayat | Tampilkan data sesuai campusId request, dengan catatan jika staff sudah pindah (misal badge "Pindah campus"). |

**Decision:** _TBD_

---

## EC-05: Akses Lintas Campus untuk Admin

**Scenario:** Admin/ASD ingin melihat kartu guru dari campus yang berbeda dari campus default user (Admin punya akses lintas campus).

| Opsi | Behavior |
|------|----------|
| (A) Admin boleh kirim campusId berbeda | Admin (role ASD) dapat mengirim `campusId` query param berbeda dan mengakses staff lintas campus. Principal tetap dibatasi campus-nya sendiri. |
| (B) Selalu pakai campus user | Backend selalu memakai `req.user.campusId` — admin tidak bisa mengakses campus lain. |
| (C) Admin tanpa filter campus | Admin/ASD dapat mengakses semua staff tanpa perlu menyebut campusId (scope diabaikan untuk role ASD). |

**Decision:** _TBD_

---

## EC-06: Data Identity Ganda (Duplicate Record)

**Scenario:** Staff memiliki lebih dari satu record di `employee_identity` (duplikat data NIK/NIP/foto karena kesalahan input atau migrasi).

| Opsi | Behavior |
|------|----------|
| (A) Ambil record terbaru | Backend memilih record `employee_identity` dengan `updated_at`/`created_at` terbaru. |
| (B) Ambil record aktif pertama | Backend memilih record pertama dengan `active_status = ACTIVE`. |
| (C) Gabungkan data | Backend menggabungkan field dari semua record aktif (field yang tidak null dipakai). |
| (D) Tandai anomali | Backend mengembalikan data record terbaru dan menulis log warning "duplicate employee_identity" untuk admin. |

**Decision:** _TBD_

---

## EC-07: Foto URL Expired / Broken

**Scenario:** `photoUrl` terisi tetapi URL foto sudah kadaluarsa (signed URL) atau file sudah dihapus dari storage sehingga gambar tidak bisa dimuat.

| Opsi | Behavior |
|------|----------|
| (A) Fallback placeholder | Frontend menggunakan onError handler: jika gambar gagal dimuat, ganti dengan placeholder (inisial/icon default). |
| (B) Validasi backend | Backend melakukan HEAD request ke storage untuk validasi foto ada; jika gagal, set `photoUrl = null`. |
| (C) Biarkan pecah | Tampilkan gambar broken (default browser) tanpa fallback. |

**Decision:** _TBD_
