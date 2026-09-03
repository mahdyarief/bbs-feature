---
title: Pickup Persons — Tambah Penjemput Gagal dengan "Key (id) already exists" (Duplikat Primary Key)
status: open
severity: critical
product: BBS LMS
portal: Student Portal
author: System Analyst
date: 2026-09-03
jam: https://jam.dev/c/64c373a1-c8f2-4b06-b782-4f1a727b543b
---

# Pickup Persons — Save Selalu Gagal: "Key (id)=(N) already exists"

## Summary

Di halaman **Pickup Persons** pada Student Portal Smartbag, ketika user mencoba **menambah pickup person baru** lewat form "Add Pickup Person" dan menekan **Save**, request `POST /api/v1/studentPickUpPersons` selalu gagal dengan **HTTP 400** berisi error `Key (id)=(N) already exists.` — pelanggaran constraint primary key di tabel `student_pick_up_person`. Nilai `N` terus bertambah setiap percobaan (289, 290, 291, … sampai 328), menandakan **sequence auto-increment PostgreSQL tidak sinkron dengan data yang sudah ada di tabel**. Fitur ini **100% tidak bisa dipakai** sampai sequence diperbaiki; user tidak punya jalan lain selain mengulang Save tanpa hasil.

**Ekspektasi:** Pickup person baru tersimpan, daftar bertambah, dan tidak ada error. ID baru dihasilkan otomatis tanpa konflik.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | Mahdy Arief |
| Email (Jam account) | mahdy.arief.torche.indonesia@gmail.com |
| Jam author ID | 7ebe5b87-970c-4f3d-95f5-c4f39b901a03 |
| User ID (dari console log, mis. `selfUser`) | *(tidak terekam di Jam — tidak ada metadata/console selfUser)* |
| Portal URL | `https://student.smartbag.binabangsaschool.com/pickup` |
| Environment API | `https://api.binabangsaschool.dev` (dev/staging) |
| Data konteks (class ID / daId / tanggal) | — |
| Browser / OS | Brave 150 / Windows 11 (screen 1710x1283) |

---

## Steps to Reproduce

1. Login sebagai **Student** ke Student Portal Smartbag (`student.smartbag.binabangsaschool.com`).
2. Buka halaman **Pickup** → bagian **Pickup Persons** → klik **Add Pickup Person**.
3. Isi form dengan data valid (contoh dari recording: `full_name = "asd"`, `relationship = "asd"`, `phone_number` diisi nomor aktif; kolom file dibiarkan kosong → dikirim sebagai ``).
4. Klik tombol **Save** (`button.btn.btn-primary[type=submit]`).
5. Amati hasil.

**Actual Result:**
- Setiap klik Save → modal menampilkan error merah di dekat kolom input; **tidak ada pickup person yang berhasil disimpan**.
- Console log menampilkan error: `Key (id)=(289) already exists.`, lalu `(290)`, `(291)`, … bertambah 1 setiap percobaan hingga `(328)` (40 percobaan dalam 11 detik).
- Network: `POST https://api.binabangsaschool.dev/api/v1/studentPickUpPersons` → **HTTP 400** dengan pesan error yang sama; body response berisi detail konflik primary key.

**Expected Result:**
- Save berhasil (HTTP 201/200), data pickup person baru muncul di daftar.
- ID baru di-generate otomatis oleh database (auto-increment) tanpa tabrakan dengan baris yang sudah ada.

---

## Root Cause Analysis

### Bug #1 — Sequence auto-increment tabel `student_pick_up_person` di belakang data aktual

Error `Key (id)=(289) already exists.` adalah pesan pelanggaran **unique constraint primary key** dari PostgreSQL. Pola khasnya: **nilai sequence (serial) untuk kolom `id` lebih kecil daripada `MAX(id)` baris yang sudah ada di tabel**, sehingga setiap INSERT mencoba ID yang sudah terpakai.

- **Tabel dibuat dengan `id SERIAL PRIMARY KEY`** — lihat migration `src/database/migrations/1716369166492-StudentPickUp.ts:23`:
  ```sql
  CREATE TABLE "student_pick_up_person" ("id" SERIAL NOT NULL, ..., CONSTRAINT "PK_ffc9bf3021a43c880a4606e4679" PRIMARY KEY ("id"))
  ```
- **Entity** memakai `@PrimaryGeneratedColumn()` (auto-increment) via `BaseEntity` — `src/common/base.entity.ts:17`:
  ```typescript
  @PrimaryGeneratedColumn()
  id: number;
  ```
  diwarisi oleh `StudentPickUpPerson` (`entities/student-pick-up-person.entity.ts:14`).
- **Service create()** tidak pernah mengisi `id` secara manual — ia membuat instance baru, set kolom bisnis, lalu `await spup.save()` — `src/modules/student-pick-up-person/student-pick-up-person.service.ts:122-168`. Artinya ID sepenuhnya bergantung pada sequence DB.
- **DTO create** juga tidak menerima field `id` (`dto/create-student-pick-up-person.dto.ts:12-57`) — request body dari Jam hanya berisi `full_name`, `relationship`, `is_primary`, `phone_number`, `file_id`, `file_url`. Tidak ada kesalahan di sisi payload.

**Mengapa N bertambah tiap percobaan:** sequence PostgreSQL **tidak ikut rollback** ketika INSERT gagal — nilai sequence sudah di-*advance* walau transaksi di-rollback. Jadi percobaan pertama memakai 289 (konflik), yang kedua 290 (konflik), dst., sampai nilai sequence melewati semua ID yang sudah terisi. Di tabel dev, baris dengan ID 289–328 (dan seterusnya) sudah ada — kemungkinan besar hasil **import/migrasi data (mis. dari legacy/smartbag lama, atau restore DB) yang menyisipkan ID eksplisit tanpa menyinkronkan sequence** (`setval`). Faktor pendukung: entity `BaseEntityWithDates` punya kolom `src_id`/`src_db`/`migrated_at` (`base.entity.ts:31-48`) — bukti bahwa data tabel ini memang diimpor dari sumber lain, dan import semacam itu adalah penyebab klasik desinkronisasi sequence.

### Faktor sekunder — UX: tombol Save tidak di-disable saat request in-flight

User mengklik Save ~40x dalam 11 detik tanpa feedback blokir. Ini memperparah dampak (banjir request 400 ke backend), walaupun bukan penyebab error.

---

## Bukti dari Jam (https://jam.dev/c/64c373a1-c8f2-4b06-b782-4f1a727b543b)

| Sumber | Temuan |
|--------|--------|
| **Video (t=0–11s)** | User membuka modal Add Pickup Person, mengisi `Full name` & `Relationship` = "asd", mengisi no. HP, lalu berulang kali menekan Save. Tiap Save memunculkan error merah di form; data tidak pernah tersimpan. |
| **Network** | `POST https://api.binabangsaschool.dev/api/v1/studentPickUpPersons` → **HTTP 400** berulang (46 event error; durasi ~69–236ms). Error message: `Key (id)=(289) already exists.` s/d `(328)`. Request body konsisten: `{"data":{"attributes":{"full_name":"asd","relationship":"asd","is_primary":false,"phone_number":"[disensor Jam]","file_id":,"file_url":{}}}}` — tanpa field `id`. |
| **Console** | 40 error bertipe `error` dengan pola `Key (id)=(N) already exists.` (N: 289→328, naik 1 per percobaan). Stack trace: `506.0875257e.chunk.js:1:2179` → `706.9f26bed9.chunk.js:1:6458` → `main.ea0c2210.js:2:335627` (frontend React). |
| **User events** | 40 klik pada `button.btn.btn-primary[type=submit]` berlabel **Save** antara t=3277ms–9711ms di halaman `/pickup`. |

---

## Affected Components

| Layer | File | Impact |
|-------|------|--------|
| Backend Service | `api_nest/src/modules/student-pick-up-person/student-pick-up-person.service.ts` (create, line 122-168) | Insert gagal karena ID dari sequence sudah dipakai; tidak ada penanganan/retry error duplikat key |
| Backend Entity | `api_nest/src/modules/student-pick-up-person/entities/student-pick-up-person.entity.ts` + `src/common/base.entity.ts` | `id` auto-increment (SERIAL) — tergantung sequence DB |
| Backend Migration | `api_nest/src/database/migrations/1716369166492-StudentPickUp.ts` (line 23) | Tabel `id SERIAL` — tidak ada sync sequence pasca-import |
| Database (dev) | Tabel `student_pick_up_person` (env `api.binabangsaschool.dev`) | Sequence `student_pick_up_person_id_seq` tertinggal dari `MAX(id)` (baris eksis ≥ 328) |
| Frontend | `student.smartbag.binabangsaschool.com` (chunk 506/706/main) | Save tidak di-disable saat in-flight → spam request; error 400 ditampilkan mentah di form |

---

## Proposed Solution Options

### Option A: Sinkronkan sequence tabel (Recommended — immediate fix)

Perbaikan satu kali di DB dev/staging (dan cek env lain), lalu verifikasi dengan save:

```sql
-- PostgreSQL: samakan sequence dengan MAX(id) tabel
SELECT setval(
  pg_get_serial_sequence('student_pick_up_person', 'id'),
  (SELECT COALESCE(MAX(id), 1) FROM student_pick_up_person)
);
```

Setelah ini, percobaan save berikutnya akan memakai `MAX(id)+1` dan berhasil. **Penting juga untuk cek tabel sejenis** yang ikut import (`student_pick_up`, `student_pick_up_setting`, `student_pick_up_request`) dan pindai semua tabel bermasalah dengan query pembanding `last_value` sequence vs `MAX(id)`.

### Option B: Cegah terulang di proses migrasi/import data

Tambah langkah sinkronisasi sequence otomatis setiap kali ada import data dengan ID eksplisit (seed/restore/script migrasi `src_id`). Contoh guard query untuk deteksi dini:

```sql
-- Deteksi semua tabel dengan desinkronisasi sequence
SELECT c.relname AS table_name,
       s.last_value,
       (SELECT MAX(id) FROM pg_class c2 JOIN pg_attribute a ON a.attrelid = c2.oid
        WHERE c2.relname = c.relname) AS max_id
FROM pg_sequences s
JOIN pg_class c ON c.relname = replace(s.sequencename, '_id_seq', '')
WHERE s.last_value < (SELECT MAX(id) FROM pg_class c2 JOIN pg_attribute a ON a.attrelid = c2.oid
                      WHERE c2.relname = replace(s.sequencename, '_id_seq', ''));
```

(Implementasi final bisa memakai function helper `sync_sequences()` yang dijalankan di akhir tiap job import/restore.)

### Option C: Perbaikan UX pencegahan spam (opsional, menyertai A/B)

Frontend: disable tombol Save selama request pending (guard double-submit), dan tampilkan pesan error yang bisa ditindaklanjuti (misal "Terjadi kesalahan server, coba lagi") alih-alih error mentah DB. Backend: mapping error duplikat key (kode `23505`) ke respons 4xx yang jelas + opsi retry.

---

## Notes

- Error terjadi di environment **dev** (`api.binabangsaschool.dev`) — perlu dicek apakah env staging/prod punya masalah sama (kemungkinan besar ya bila data juga hasil import).
- Root cause dipastikan dari pola error (ID naik monoton per percobaan = sequence tidak sinkron), bukan dari kode aplikasi — service/DTO/entity tidak mengandung bug pembuatan ID. Verifikasi final cukup dengan query `SELECT MAX(id) FROM student_pick_up_person;` vs `SELECT last_value FROM student_pick_up_person_id_seq;`.
- Perbaikan diharapkan pada environment dev dulu oleh tim yang mengelola database/API `api_nest`.
