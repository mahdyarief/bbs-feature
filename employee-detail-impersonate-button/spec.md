---
feature: Employee Detail Impersonate Button
slug: employee-detail-impersonate-button
status: draft
author: System Analyst
date: 2026-10-15
target_release: TBD
---

# Employee Detail Impersonate Button

## Overview

Menambahkan tombol **Impersonate** di halaman Employee Detail (`/employees/:employeeId`), di header kanan atas bersebelahan dengan tombol **Edit** yang sudah ada. Tombol ini memakai ulang alur impersonate yang sudah ada di Employee List (modal + API yang sama), sehingga admin tidak perlu kembali ke list untuk melakukan impersonate terhadap employee yang sedang dilihat detailnya.

Referensi visual: `C:\Users\ihsan\OneDrive\Documents\Work\arsip tiket\impersonate on detail\image (47).png` — kotak merah menandai posisi tombol di header detail, di sebelah kiri tombol Edit.

## Problem / Motivation

Saat ini tombol Impersonate hanya ada di **Employee List** (`Employees.jsx` → kolom aksi per baris). Alur kerja admin yang wajar adalah: cari employee di list → klik untuk buka detail → verifikasi data (posisi, status, identitas) → impersonate. Karena tombol tidak ada di halaman detail, admin harus **kembali ke list** dan mencari ulang baris employee tersebut untuk impersonate — tidak efisien dan rawan salah baris.

## Scope

### In Scope

- Tambah tombol **Impersonate** di header Employee Detail (`EmployeeDetail.jsx` → `FINAHeader`), di sebelah kiri tombol **Edit**.
- Tombol membuka `ImpersonationModal` yang sama seperti di list (cek portal via `POST /auth/impersonate/check`, login via `POST /auth/impersonate/login`).
- Permission gate: `BBSCan I=READ a=EMPLOYEE` (sama seperti tombol di list).
- Disabled state: `disabled` ketika `employee.activeStatus !== "ACTIVE"` (sama seperti di list).
- Loading state: tombol tidak ditampilkan / disabled selama data employee masih loading (lihat EC-03).

### Out of Scope

- Perubahan API impersonate (endpoint, payload, response) — tidak ada.
- Perubahan `ImpersonationModal` itu sendiri — pakai ulang apa adanya.
- Perubahan permission/role impersonate di backend.
- Tombol impersonate di halaman lain (Organization Detail, Employee Edit, dsb).
- Perbaikan bug `impersonate-portal-mismatch-500` (HTTP 500 vs 400) — ditangani di tiket terpisah.

## User Stories

### As a Super Admin / HR Admin

I want to klik tombol Impersonate langsung dari halaman Employee Detail
So that saya tidak perlu kembali ke list untuk impersonate setelah memverifikasi data employee.

### As a Support Staff

I want to memastikan employee yang saya impersonate adalah orang yang benar (sudah saya cek detailnya)
So that mengurangi risiko salah impersonate akibat salah klik baris di list.

## Acceptance Criteria

- [ ] Di halaman `/employees/:employeeId`, header kanan atas menampilkan tombol **Impersonate** di sebelah kiri tombol **Edit**, dengan ikon + label yang konsisten dengan tombol di list (`HiOutlineUserCircle` + teks "Impersonate").
- [ ] Klik tombol Impersonate membuka `ImpersonationModal` dengan `targetEmployee` = employee yang sedang dilihat (id, username/emailBbs, fullName, employeeTypes, roles).
- [ ] Alur di dalam modal identik dengan dari list: cek available portals → pilih portal (ADMIN / TEACHER) → login + redirect ke portal tujuan.
- [ ] Tombol hanya tampil untuk user dengan permission `READ EMPLOYEE` (dibungkus `BBSCan`, sama seperti di list).
- [ ] Tombol Edit tetap tampil dengan permission-nya sendiri (`UPDATE EMPLOYEE`) — kedua tombol independen; user dengan READ tapi tanpa UPDATE tetap melihat Impersonate tanpa Edit, dan sebaliknya.
- [ ] Tombol Impersonate dalam keadaan **disabled** ketika `employee.activeStatus !== "ACTIVE"`.
- [ ] Tidak ada regresi: tombol Edit, QR code, dan semua section detail tetap berfungsi seperti sebelumnya.

## UI / UX Changes

Halaman Employee Detail, header (`FINAHeader`, `EmployeeDetail.jsx:412-437`):

```
Sebelum:
[Nama Employee]                    [Edit]

Sesudah:
[Nama Employee]          [Impersonate] [Edit]
```

- Tombol Impersonate: style ghost/outline primary dengan ikon `HiOutlineUserCircle` (sama seperti di list) agar konsisten.
- Posisi: di dalam container kanan header, sebelum (sebelah kiri) tombol Edit.
- Referensi screenshot: `arsip tiket/impersonate on detail/image (47).png`.

### Affected Portals

- [x] Admin (client/)
- [ ] Student (client-student/)
- [ ] Teacher (client-teacher/)

## API Changes

Tidak ada perubahan API. Menggunakan endpoint yang sudah ada:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/auth/impersonate/check` | Cek available portals (dipanggil dari `ImpersonationModal`) |
| POST | `/api/v1/auth/impersonate/login` | Login sebagai target employee per portalType |

## Database Changes

Tidak ada.

### New Tables

- —

### Modified Tables

- —

### Migrations

- —

## Business Rules / Validation

1. Visibility tombol diatur oleh CASL: `BBSCan I=ACTION_MODULES.READ a=SUBJECT_MODULES.EMPLOYEE` (identik dengan `Employees.jsx:242`).
2. Disabled ketika `employee?.activeStatus !== "ACTIVE"` (identik dengan `Employees.jsx:248`).
3. `targetEmployee` yang diteruskan ke modal dibentuk dari data employee di detail: `{ id, username: employee.username || employee.emailBbs, fullName, employeeTypes, roles }` — sama seperti `handleImpersonate` di `Employees.jsx:101-111`. Jika field tidak tersedia di response detail (mis. `roles`), fallback ke `[]` dan biarkan modal tetap berfungsi (portal check tidak bergantung padanya).
4. Tidak ada validasi tambahan di sisi detail page; validasi portal dilakukan server-side via `/check` + `/login` seperti alur existing.

## Error Handling

| Error | HTTP Code | Message |
|-------|-----------|---------|
| Employee tidak punya portal apa pun | 200 (check, `availablePortals: []`) | Modal menampilkan "This employee does not have access to any portals" (behavior existing, tidak berubah) |
| Employee non-ACTIVE | — (client-side) | Tombol disabled; tidak ada request yang dikirim |

## Dependencies

- `src/components/ImpersonationModal.jsx` — dipakai ulang tanpa modifikasi.
- `src/actions/api/auth.js` (`checkImpersonation`, `impersonateLogin`) — tidak berubah.
- Reducer `impersonationPortals` & `targetEmployees` (`reducers/root.js:293-294`) — tidak berubah.
- File yang diubah (estimasi engineer): hanya `client/src/views/employees/list/EmployeeDetail.jsx` (tambah state modal + tombol + render modal).
- Tiket terkait (bukan dependensi): `features/impersonate-portal-mismatch-500/bug-report.md`.
