# Notes — Teacher Card (Kartu Guru)

## Ringkasan Analisis

| Portal legacy | Temuan |
|---------------|--------|
| Teacher portal | Menu id **31** — mengarah ke `recruitment_new/slick_app.php` di modul Staff & HR / Kepegawaian (section 9). |
| Portal ASD | Varian `slick_app_asd.php` — kemungkinan menampilkan kartu guru dengan akses lebih luas (semua staff). |
| Principals portal | Tidak disebut secara eksplisit; kemungkinan mengakses via portal ASD atau melihat staff per campus-nya. |

## Keterbatasan Data

**Tidak ada HTML dump, probe, atau screenshot** yang tersedia untuk fitur ini. Analisis hanya didasarkan pada:
1. Menu id 31 → `recruitment_new/slick_app.php` (teacher portal).
2. Varian `slick_app_asd.php` (portal ASD).
3. Nama modul yang sudah ada di `api_nest` (employee, employee-identity, employee-position, dll).
4. Konsep umum "kartu guru" (compact profile card).

Akibatnya, **hampir semua field dan layout kartu bersifat ESTIMASI/PROPOSAL** — jangan diperlakukan sebagai fakta.

## Asumsi yang Perlu Diverifikasi

### Layout & Field Kartu

| Asumsi | Keterangan |
|--------|------------|
| [A1] Kartu menampilkan foto staff | Diasumsikan dari konsep kartu identitas — foto adalah elemen utama. |
| [A2] Field: nama, NIP, NIK, gender, posisi, campus, kontak | Field standar profile card; field aktual legacy belum dikonfirmasi. |
| [A3] Format kartu (vertical/horizontal, modal/halaman penuh) | Layout aktual tidak diketahui — tidak diinventarisasi karena tidak ada HTML dump. |
| [A4] Ada aksi print/export | Diasumsikan dari fungsi umum kartu identitas yang perlu dicetak. |
| [A5] `employee_identity` punya kolom `nip`, `nik`, `photo_url` | Nama kolom aktual bisa berbeda (mis. `employee_number`, `photo_path`, `identity_number`). |

### Portal & Akses

| Asumsi | Keterangan |
|--------|------------|
| [A6] Teacher dapat melihat kartu sendiri (self-view) | Diasumsikan dari sifat "kartu guru" — perlu dikonfirmasi apakah teacher hanya melihat dirinya sendiri. |
| [A7] Admin/Principal dapat melihat kartu semua staff | Diasumsikan dari peran admin — perlu dikonfirmasi batasan per campus. |
| [A8] Dual portal mirroring (Admin + Teacher) | Mengikuti arsitektur smartbag dual portal. |

### Data & Teknis

| Asumsi | Keterangan |
|--------|------------|
| [A9] `employee_position` punya penanda posisi utama (`is_primary`) | Nama kolom aktual perlu diverifikasi; bisa `is_main` atau tanpa penanda. |
| [A10] `employee_identity` relasi 1:1 dengan employee | Bisa jadi 1:N (multiple identity records) — lihat EC-06. |
| [A11] Foto disimpan di object storage dengan URL | Bisa jadi foto disimpan sebagai BLOB di database atau path lokal. |
| [A12] Kolom `birth_place`, `join_date`, `position_type` ada | Kolom ini hanya proposal di schema.md — belum tentu ada di entity aktual. |

## Saran Probe Ulang

1. **Probe `recruitment_new/slick_app.php` di portal ASD** (dan varian `slick_app_asd.php`) — jika masih bisa diakses, ambil HTML dump untuk melihat layout & field aktual kartu.
2. **Baca entity definitions di `api_nest`**:
   - `src/modules/employee/entities/employee.entity.ts`
   - `src/modules/employee-identity/entities/employee-identity.entity.ts`
   - `src/modules/employee-position/entities/employee-position.entity.ts`
   untuk memastikan nama kolom aktual (nip, nik, photo, is_primary, dll).
3. **Cek struktur modul recruitment di legacy** — pastikan konteks akses kartu (apakah hanya dari appraisal/recruitment, atau juga dari menu staff biasa).
4. **Wawancara stakeholder** — tanya: "Data apa yang wajib muncul di kartu guru? Perlu foto? Perlu print/PDF?".

## Relasi ke Fitur Lain

Fitur Teacher Card terkait dengan fitur lain yang juga menampilkan data employee:

| Fitur | Relasi |
|-------|--------|
| `form-leave` | Menampilkan data employee pada form cuti — bisa memakai komponen kartu ringkas. |
| `appraisal-new` / `appraisal-summary` | Gateway staff database & report — menampilkan data guru per campus. |
| `teacher-cv` | CV guru yang lebih detail — teacher card bisa menjadi ringkasan dari CV. |
| `training-staff` | Data staff peserta pelatihan — kartu untuk identifikasi peserta. |
| `employee` (modul inti) | Sumber data employee — kartu guru adalah consumer data employee. |

## Catatan Desain

- **Read-only:** Fitur murni read-only — tidak ada endpoint POST/PUT/PATCH/DELETE.
- **No new tables:** Semua data sudah ada di `employee`, `employee-identity`, `employee-position` — tidak perlu tabel/migrasi baru.
- **Caching:** Card payload dapat di-cache (Redis/in-memory, TTL ±1 jam) karena data jarang berubah.
- **Dual portal:** Admin Portal (lintas campus) + Teacher Portal (self-view / scope campus Principal).
- **Proposal field:** Semua field di `api-contract.md` dan `schema.md` bersifat proposal dan wajib diverifikasi sebelum implementasi.