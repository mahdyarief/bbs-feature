# Audit — `throw new Error` di Service yang Dipanggil Controller Publik

> Tanggal: 2026-09-30. Metode: grep `throw new Error(` di `api_nest/src/modules/**/*.ts`
> (37 lokasi, 12 modul + 1 spec), lalu klasifikasi manual: user error (→400) vs
> failure internal (500 sah) vs background job.
> Motivasi: bug `impersonate-portal-mismatch-500` (TC-038) — pola yang sama
> dicari di seluruh codebase.

## Prinsip klasifikasi

| Klasifikasi | Arti | Status seharusnya |
|---|---|---|
| **A — User error** | Input pemanggil salah/tidak ada (resource tidak ditemukan, body salah) | → **400/404** via `BadRequestException`/`NotFoundException` |
| **B — State conflict** | Data valid tapi state tidak mengizinkan (gradebook locked, token expired) | → **400/409** via `BadRequestException`/`ConflictException` |
| **C — Internal/infra** | Integrasi pihak ketiga/SES/DB gagal | → 500 sah (biarkan, mungkin tambah konteks) |
| **D — Background job** | Dipanggil dari processor/queue, bukan HTTP request | → 500 tidak terlihat client; prioritas rendah |

## Temuan per Modul (37 lokasi)

### 🔴 Prioritas 1 — `student-transcript` (9 lokasi) — semua A, semua route publik

Controller hanya punya route POST generate + GET grades (dipakai UI admin).
Semua error di bawah terjadi karena input/state pemanggil → client menerima 500:

| Line | Pesan | Klasifikasi | Seharusnya |
|---|---|---|---|
| service:50 | Master Level "Primary 6" not found | A (config missing, tapi dipicu request) | 500 sah — env config, TETAPI jarang; biarkan |
| service:69, 647 | Student not found | **A** | `NotFoundException` |
| service:74, 652 | Student does not have a current class year | **A** | `BadRequestException` |
| service:78 | (transcript eligibility) | **B** | `BadRequestException` |
| service:524 | Student grade not found | **A** | `NotFoundException` |
| service:571 | No Grade 6 students found... | **A** (generate bulk tanpa kandidat) | `BadRequestException` |
| controller:100 | Invalid request structure. Expected grades array. | **A** — ini persis pola validation, di CONTROLLER | `BadRequestException` |
| service:1604 (cambridge) | Gradebook is locked | **B** | `ConflictException` |

### 🔴 Prioritas 1 — `student-cambridge-grade` (10 lokasi) — A/B, dipakai gradebook UI

| Line | Pesan | Klasifikasi |
|---|---|---|
| 1603, 1925, 2947 | Subject year not found | A → 404 |
| 1609 | Class year not found | A → 404 |
| 1612 | Subject not found | A → 404 |
| 1604 | **Gradebook is locked** | B → 409 (lock memang fitur SVE/gradebook — user butuh tahu kenapa ditolak) |
| 1633, 2954 | No effective syllabus found | B → 400 |
| 1671 | (conditional) | B → 400 |

### 🟡 Prioritas 2 — `auth` (3 lokasi tersisa setelah fix impersonate)

| Line | Pesan | Klasifikasi |
|---|---|---|
| 1184 | Token is required for cross portal access | A → 400 |
| 1219 | Employee not found for the provided token | A → 401/404 |
| 1223 | Employee account is not active | B → 403 |

(Cross-portal access dipakai fitur switch-portal — jalur user nyata.)

### 🟡 Prioritas 2 — `cca-registration-rules` (3 lokasi)

Line 49, 62, 94: "CCA Registration Rules not found" — A → 404.
Dipanggil dari CRUD admin CCA registration rules (wiki `cca.md` admin routes).

### 🟢 Prioritas 3 — sisanya

| Modul:line | Pesan | Klasifikasi |
|---|---|---|
| class-year/transfer-student-batch:727 | Subject year N not found | A (batch transfer) → kumpulkan jadi report, bukan throw pertama |
| class-year/transfer-student-batch:1086 | Transfer failed during ... | B → 400 dengan konteks |
| daily-attendance:287 | classYearId is required | A → 400 |
| security-deposit:32 | No security deposit found for student | A → 404 |
| billing-report:346 | Token is expired! | B → 401 |
| invoice:364, 621 | Educa8 access token / integrasi | C → 500 sah |
| mailer sve-document.processor:124 | SES email failed | D → job |
| external-service-integration:74, 81 | (commented) | n/a |
| audit-log spec:64 | test | n/a |

## Ringkasan Eksekutif

| Prioritas | Jumlah lokasi | Modul | Alasan |
|---|---|---|---|
| 🔴 1 | 16 | student-transcript (8), student-cambridge-grade (8) | Route publik UI grading; user akan sering memicu 500 saat input/state salah |
| 🟡 2 | 6 | auth (3), cca-registration-rules (3) | Jalur user nyata (cross-portal, CRUD rules) |
| 🟢 3 | 8 | transfer batch, daily-attendance, security-deposit, billing-report | Dipakai lebih jarang |
| ✅ Sah 500 | 4 | invoice (2), mailer processor (1), auth env config (1) | Internal/infra |

## Rekomendasi Perbaikan

1. **Satu PR per modul prioritas 1** (`student-transcript`, `student-cambridge-grade`):
   ganti `throw new Error` → `NotFoundException` / `BadRequestException` /
   `ConflictException` sesuai tabel. Perubahan mekanis, risiko rendah, tanpa
   perubahan perilaku sukses.
2. **Search-and-replace terpandu** untuk prioritas 2–3 dengan tabel di atas sebagai
   acuan klasifikasi.
3. **Pencegahan berulang**: tambahkan lint rule / code review checklist
   "service yang melempar error user-facing harus pakai HttpException subclass"
   (pola Option C di bug report `impersonate-portal-mismatch-500`).
4. Setelah fix, jalankan ulang TC-038 + smoke GET/POST negatif pada modul terkait
   (test case operate siap dipakai ulang).

## Bukti Pendukung

- Grep penuh 2026-09-30: 37 lokasi `throw new Error(` (daftar lengkap di atas).
- Bug rujukan: `features/impersonate-portal-mismatch-500/bug-report.md`
  (auth.service.ts:1059 — satu-satunya yang sudah dilaporkan terpisah; biarkan
  fix-nya menyatu dengan PR audit ini).
