---
feature: National Report Card
slug: national-report
---

# Schema — National Report Card

## Ringkasan

Tidak ada tabel baru yang diusulkan — National Report Card adalah **view/generator** yang membaca data dari tabel yang sudah ada di `api_nest`. Namun ada beberapa catatan penting terkait data nilai yang perlu dipastikan ketersediaannya.

---

## 1. Existing Entities yang Dibaca

### class_room

| Column | Tipe | Keterangan |
|--------|------|------------|
| id | integer (PK) | Class ID (contoh: 2005) |
| name | varchar | Nama class (contoh: "Secondary 3 Taylor") |
| code | varchar | Kode class (contoh: "Sec3Acc") |
| level | varchar | Level pendidikan (contoh: "Secondary 3 Accelerated") |
| campus_id | integer (FK) | Relasi ke campus |
| academic_year_id | integer (FK) | Relasi ke academic_year |
| active_status | enum | ACTIVE / INACTIVE |

Sumber: `src/modules/class-room/entities/class-room.entity.ts` (perlu verifikasi nama entity)

### student

| Column | Tipe | Keterangan |
|--------|------|------------|
| id | integer (PK) | Student ID |
| name | varchar | Nama lengkap siswa |
| nisn | varchar | Nomor Induk Siswa Nasional |
| birth_place | varchar | Tempat lahir |
| birth_date | date | Tanggal lahir |
| active_status | enum | ACTIVE / INACTIVE |

### student_class / class_student

Tabel join yang menghubungkan student ke class_room per academic year.

| Column | Tipe | Keterangan |
|--------|------|------------|
| student_id | integer (FK) | Relasi ke student |
| class_room_id | integer (FK) | Relasi ke class_room |
| academic_year_id | integer (FK) | Relasi ke academic_year |
| active_status | enum | ACTIVE / INACTIVE |

### academic_year

| Column | Tipe | Keterangan |
|--------|------|------------|
| id | integer (PK) | AY ID (contoh: 25 = 2024/2025) |
| name | varchar | Nama (contoh: "2024/2025") |
| start_year | integer | Tahun mulai |
| end_year | integer | Tahun selesai |

---

## 2. Data Nilai — Perlu Verifikasi

Data nilai SKL dan Transkrip diambil dari kombinasi sumber per level siswa. Berikut daftar yang perlu dipastikan ada di smartbag (saat ini berupa gap yang perlu dikonfirmasi dengan modul grading):

| Data | Sumber | Level | Status di api_nest |
|------|--------|-------|--------------------|
| GP Value | `secondary-gpa-grading-scale` | Semua | ✅ Ada |
| Subject Unit | `secondary-gpa-subject-unit` | Semua | ✅ Ada |
| AS PUM (Pure Unit Mark) | Cambridge grade | JC2 | ⚠️ Perlu verifikasi |
| SA1 (Semester 1) | Nilai semester | Semua | ⚠️ Perlu verifikasi |
| FYA (Final Year Assessment) | Nilai akhir tahun | Sec 4 / lainnya | ⚠️ Perlu verifikasi |
| IGCSE PUM | Cambridge grade | Sec 4 | ⚠️ Perlu verifikasi |
| FYA June | Nilai akhir tahun | Selain JC2 & Sec 4 | ⚠️ Perlu verifikasi |
| A PUM | Cambridge grade | JC2 | ⚠️ Perlu verifikasi |

### Formula Final Mark (dari bbs-notes)

```
Jika level = JC2:
    Final Mark = max(AS PUM, SA1) untuk academic, SA1 untuk non-academic
               = max(A PUM, FYA) untuk level di JC2
Jika level = SEC 4:
    Final Mark = max(FYA, IGCSE PUM)
Selain JC2 & SEC 4:
    Final Mark = FYA June
Untuk Sec 4 & JC2 secara umum:
    Final Mark = GP Value × Unit
```

---

## 3. Proses Generate PDF

### Alur

```
Input: classId (integer)
1. Load class_room by id
2. Load academic_year dari class_room
3. Load semua student di class tersebut (via class_student)
4. Untuk setiap student:
   a. Tentukan level (JC2, Sec 4, atau lainnya)
   b. Ambil nilai final sesuai logika level
   c. Komposisi SKL: 3 mapel (Pendidikan Agama, PPKN, Bahasa Indonesia) + rata-rata
   d. Komposisi Transkrip: 13 mapel + rata-rata
5. Generate PDF menggunakan library PDF (puppeteer, html2pdf, atau wkhtmltopdf)
6. Return PDF file
```

### Library PDF

Legacy menggunakan `html2pdf` (PHP library). Untuk smartbag (NestJS), rekomendasi:
- **Puppeteer** (headless Chrome) — render HTML ke PDF, paling akurat
- **PDFKit** / **jsPDF** — generate PDF langsung di Node.js
- **LibreOffice** — konversi DOCX/HTML ke PDF via CLI

---

## 4. Migrasi

Tidak ada migrasi database baru untuk fitur ini. Data sudah ada di tabel yang sudah ada. Namun jika data PUM/FYA/SA1 belum tersimpan, migrasi untuk tabel nilai baru mungkin diperlukan di modul grading terpisah.