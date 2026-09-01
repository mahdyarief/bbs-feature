# Feature Brief: Empty State Informatif di Halaman Detail BMT Files

## Metadata
- **Requested by:** Mahdy Arief (via Jam.dev)
- **Date:** 2026-09-01
- **Module:** BMT (Biannual/Mid-Term) Files Download
- **Halaman:** `/bmt-files/:sveExamTypeId` (detail per subject, contoh `/bmt-files/571`)
- **Environment:** Teacher Portal (teacher.smartbag.binabangsaschool.com)

## Latar Belakang
Di halaman **detail BMT Files** (`/bmt-files/:id`, misal `/bmt-files/571`), setiap baris paper type hanya menampilkan file ketika status dokumen sudah `APPROVED_FBP`. Ketika **semua file pada subject tersebut kosong** (belum ada dokumen berstatus `APPROVED_FBP`), halaman hanya menampilkan tabel kosong — header "Paper Type / File Name / Last Updated By" dengan baris-baris paper type yang tidak ada file-nya, tanpa penjelasan apa pun. User tidak tahu kenapa file belum muncul dan menganggapnya sebagai bug (lihat Jam https://jam.dev/c/eb14ca3e-3b5a-4165-91ae-5469c1815713).

## Tujuan
Menampilkan **empty state informatif** di halaman detail `/bmt-files/:id` ketika semua file kosong, menjelaskan alasan file belum tersedia sehingga user paham ini kondisi normal (belum waktunya), bukan error.

## Behavior Requirement
Ketika **semua file pada subject detail kosong** (tidak ada satupun `sveDocument` berstatus `APPROVED_FBP` yang dapat ditampilkan), tampilkan pesan yang menginformasikan bahwa file belum tersedia karena salah satu dari dua kondisi berikut:

1. **Status file belum menjadi FBP (Final Board Paper / Approved FBP)** — file akan muncul ketika status dokumen sudah `APPROVED_FBP`
2. **Subject belum masuk ke dalam range week exam** — file baru muncul ketika tanggal hari ini sudah berada di dalam jangkauan exam week (berdasarkan set -1 week / `substractBy: 7` di backend)

File akan muncul ketika: status file menjadi **FBP** **DAN** sudah masuk ke dalam range week exam (mulai 1 minggu sebelum week exam berlangsung).

## Konteks Teknis (untuk engineer)

### Lokasi Perubahan
**File:** `bbs/client-teacher/src/views/bmt-file-download/BmtFileDownloadDetail.jsx`

Konteks kode saat ini:
- Setiap paper type di-render melalui map `filteredSvePaperTypeResources` (line 205-265)
- File hanya ditampilkan ketika `!isEmpty(sveDocument) && sveDocument?.status === "APPROVED_FBP"` (line 230-231 dan 252-253)
- Jika tidak ada satupun dokumen berstatus `APPROVED_FBP`, semua baris hanya berisi nama paper type tanpa file/link/uploader → tampak seperti tabel kosong

**Kriteria "semua file kosong":** tidak ada satupun `sveDocumentResources` yang berstatus `APPROVED_FBP` (atau `sveDocumentResources` kosong).

Titik penambahan empty state: setelah map `filteredSvePaperTypeResources` (setelah line 265), tambahkan kondisi render empty state ketika tidak ada file yang bisa ditampilkan. Layout tetap konsisten dengan empty state lain di project (icon `sveEmptyState.svg`, centered).

### Status FBP
- Status enum backend: `SveDocumentStatusEnum.APPROVED_FBP` — `api_nest/src/types/enums/sve-document-status-type.ts:7`
- Frontend enum: `SVE_STATUS_ENUM.APPROVED_FBP = "Approved FBP"` — `bbs/client-teacher/src/enums.js:177`
- Pengecekan status di detail page: `BmtFileDownloadDetail.jsx:231,253` — `sveDocument?.status === "APPROVED_FBP"`

### Week Exam Range (set -1 week)
- Backend `findBmtOnly` menggunakan `substractBy: 7` (7 hari = 1 minggu) untuk menentukan week yang aktif:
  - `api_nest/src/modules/sve-exam-type/sve-exam-type.service.ts:524-541`
  - Filter: `today >= (week.startDate - 7 days) AND today <= week.endDate`
- Frontend mengirim `substractBy: 7` pada API call list `getBMTSveExamTypesByUser`:
  - `bbs/client-teacher/src/views/bmt-file-download/BmtFileDownload.jsx:83`
- Detail page menampilkan info week via `academicYearWeek` (line 143-145: `Term X Week Y`) — saat belum set, tampil "Week is not set yet"

## Suggested Copy (draft pesan)

**Title:** `BMT Files Not Available`

**Body:**
> Files are not yet available because the document status has not reached **Approved FBP (Final Board Paper)**, or the subject has not entered the exam week range yet.
>
> Files will appear when the document status becomes **Approved FBP** and the current date is within the exam week window (starting from 1 week before the exam week).

## Acceptance Criteria
1. Ketika halaman `/bmt-files/:id` (detail subject) tidak memiliki satupun file berstatus `APPROVED_FBP`, tampilkan title + penjelasan di atas (bukan hanya tabel/baris kosong)
2. Teks menjelaskan dua kemungkinan penyebab: status belum FBP **atau** subject belum masuk range week exam
3. Teks menjelaskan syarat file muncul: status jadi FBP **dan** sudah masuk range week exam (set -1 week)
4. Empty state hanya muncul ketika semua file kosong; jika minimal ada satu file `APPROVED_FBP`, tabel tetap ditampilkan seperti biasa
5. Layout empty state konsisten dengan halaman lain (icon `sveEmptyState.svg`, centered)
6. Tidak ada perubahan behavior API / backend — murni perubahan UI di `BmtFileDownloadDetail.jsx`

## Files Terkait
| File | Path | Peran |
|---|---|---|
| Halaman Detail BMT Files | `bbs/client-teacher/src/views/bmt-file-download/BmtFileDownloadDetail.jsx` | Frontend — empty state UI (lokasi perubahan) |
| Halaman List BMT Files | `bbs/client-teacher/src/views/bmt-file-download/BmtFileDownload.jsx` | Frontend — referensi pola empty state |
| Status Enum (frontend) | `bbs/client-teacher/src/enums.js` | Referensi status FBP |
| Status Enum (backend) | `api_nest/src/types/enums/sve-document-status-type.ts` | Referensi status FBP |
| Backend week range | `api_nest/src/modules/sve-exam-type/sve-exam-type.service.ts` | Logika `substractBy: 7` |

## Catatan
- Perubahan ini murni UI/UX (hanya menambah konten empty state di `BmtFileDownloadDetail.jsx`), tidak menyentuh API, reducer, atau backend.
- Kondisi empty state dapat ditentukan dari `sveDocumentResources` — tidak ada dokumen berstatus `APPROVED_FBP`.
