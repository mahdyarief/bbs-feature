# Edge Cases — Employee Detail Impersonate Button

> **Status: REVIEW** — keputusan untuk EC-01, EC-02, EC-03, EC-05 sudah dikonfirmasi user (2026-10-15). EC-04 opsional untuk review engineer.

## EC-01: Employee tidak punya akses portal apa pun
**Scenario:** Employee ACTIVE tetapi `POST /auth/impersonate/check` mengembalikan `availablePortals: []` (mis. karyawan non-pendidik tanpa role ADMIN/TEACHER).

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan modal tetap | Modal terbuka, menampilkan pesan "This employee does not have access to any portals" (behavior `ImpersonationModal` existing, line 49-51) |
| (B) Sembunyikan tombol | Tombol disembunyikan jika employee tidak punya portal — butuh data tambahan di halaman detail (tidak ada) |
| (C) Tombol disabled dengan tooltip | Perlu cek portal sebelum render — menambah request tambahan di halaman detail |

**Decision:** (A) — memakai behavior modal yang sudah ada, tanpa request tambahan. Konsisten dengan employee list.

---

## EC-02: Employee non-ACTIVE (INACTIVE / deleted)
**Scenario:** Employee yang dibuka detailnya berstatus `INACTIVE` (atau dilihat dari tab deleted di list).

| Opsi | Behavior |
|------|----------|
| (A) Tombol disabled | `disabled` ketika `employee.activeStatus !== "ACTIVE"` — identik dengan employee list |
| (B) Tombol selalu aktif | Modal tetap bisa dibuka; kegagalan diserahkan ke error handling server |

**Decision:** (A) — dikonfirmasi user ("Sama seperti list"). Tidak ada request yang dikirim untuk employee non-ACTIVE.

---

## EC-03: Data employee masih loading saat halaman detail dibuka
**Scenario:** Halaman detail baru dibuka, `employee` (dari `useResourceMapper`) masih `undefined` karena `getEmployee` belum selesai. Tombol impersonate butuh `employee.id` / `employee.fullName`.

| Opsi | Behavior |
|------|----------|
| (A) Tombol disabled sampai data siap | Tombol dirender tapi `disabled` selama `!employee`; aktif setelah data loaded |
| (B) Tombol tidak dirender sampai data siap | Tombol muncul setelah `employee` ada — header berubah layout (Edit juga mengikuti pola ini: tombol Edit tampil meski data kosong, hanya routing-nya butuh id) |
| (C) Tombol dirender & bisa diklik dengan data parsial | Risiko modal terbuka dengan `targetEmployee` tidak lengkap (id undefined) |

**Decision:** (A) — tombol dirender dalam `BBSCan` dan diberi `disabled={!employee || employee.activeStatus !== "ACTIVE"}`. Paling aman: tidak ada interaksi sebelum data siap, layout header stabil. (Catatan: tombol Edit di kode existing tidak di-disable saat loading — konsistensi visual dijaga dengan hanya men-disable tombol baru.)

---

## EC-04: Field `roles` / `employeeTypes` tidak tersedia di response `getEmployee`
**Scenario:** Response detail employee tidak menyertakan `roles` atau `employeeTypes` (beda dengan response list yang memuat keduanya).

| Opsi | Behavior |
|------|----------|
| (A) Fallback ke `[]` | `targetEmployee` dibentuk dengan `roles: employee.roles || []` — modal tetap berfungsi karena portal check hanya butuh `id` |
| (B) Tambah relation di `getEmployee` | Ubah `relationsObj` pada `EmployeeDetail.jsx:45-55` agar response memuat roles — perubahan lebih besar, di luar scope UI |

**Decision:** (A) — fallback ke `[]`; `ImpersonationModal` (line 251-253) hanya menampilkan roles sebagai info, tidak dipakai untuk validasi. Jika saat implementasi ternyata `employeeTypes`/`roles` benar-benar dibutuhkan oleh modal (lihat `targetEmployeeData` di modal), engineer boleh memilih (B) dan mencatatnya di notes.

---

## EC-05: User dengan READ EMPLOYEE tapi tanpa UPDATE EMPLOYEE
**Scenario:** Role admin yang hanya punya permission read employee (mis. operator/auditor) membuka detail. Tombol Edit (permission UPDATE) tidak tampil untuk user ini.

| Opsi | Behavior |
|------|----------|
| (A) Impersonate tampil, Edit tidak | Kedua tombol independen, masing-masing dibungkus `BBSCan` sendiri — impersonate (READ) tetap tersedia |
| (B) Impersonate ikut disembunyikan | Impersonate juga dibungkus UPDATE — bertentangan dengan keputusan permission READ |

**Decision:** (A) — dikonfirmasi user (permission READ EMPLOYEE). Impersonate dan Edit independen: user READ-only melihat Impersonate tanpa Edit; user UPDATE tanpa READ melihat Edit tanpa Impersonate (kasus terakhir tidak terjadi di praktik karena READ hampir selalu menyertai UPDATE, tetapi secara implementasi kedua gate tidak saling tergantung).
