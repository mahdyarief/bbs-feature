---
title: Draft mapping target_branch untuk 57 employee Sec/JC (draft, butuh koreksi)
status: draft
author: System Analyst
date: 2026-10-13
derived_from: data-report.md
---

# Draft `target_branch` — 57 employee (employment & branch kosong)

**Status: DRAFT — bukan daftar eksekusi.** Kolom `target_branch` diturunkan dari
**token terakhir domain email**, lalu diverifikasi terhadap pola email → branch
dari employee yang SUDAH punya branch (query agregat, n ≥ 3 per kombinasi).

## 1. Sumber & metode

1. Token = segmen terakhir `split_part(email_bbs,'@',1)` setelah `.`
   (mis. `abigael.piks` → `piks`).
2. Verifikasi pola dari `employee JOIN employee_branches_branch` (46 kombinasi, n ≥ 3).
3. Cek alternatif `employee_position.branch_id` → **NULL untuk semua 57 target**,
   jadi token email adalah satu-satunya petunjuk.
4. `branch.id` dari DB live (16 baris, termasuk branch TEST).

### Pola token → branch (hasil verifikasi)

| Token | branch_id | branch | Keyakinan |
|---|---|---|---|
| `pik`, `piks`, `pikp`, `pikps`, `music.piks` | **2** | Pantai Indah Kapuk | tinggi (n ratusan) |
| `kj`, `kjs`, `kjp`, `kjps` | **1** | Kebon Jeruk | tinggi |
| `bdg`, `bdgs`, `bdgp`, `bdgps` | **3** | Bandung | tinggi |
| `smg`, `smgs`, `smgp`, `smgps` | **4** | Semarang | tinggi |
| `mlg`, `mlgs`, `mlgp`, `mlgps` | **5** | Malang | tinggi |
| `bpn`, `bpns`, `bpnp`, `bpnps` | **6** | Balikpapan | tinggi |

### ⚠️ Kecualangan yang terdeteksi (alasan draft ini WAJIB dikoreksi)

Pola token **tidak unik** — ada email dengan token sama tetapi branch berbeda:

- `kjs` → ada 3 email masuk branch **8 (PIK TEST)**
- `bdg` → ada 9 email masuk branch **13 (BDG Test)**
- `bpn` → ada 4 email masuk branch **12 (BPN Test)**
- `pikp` → ada 7 email masuk branch **16 (Induction)**
- `music` → ambigu: muncul di branch 1 (n=42) dan 2 (n=28)

Jadi mapping di bawah adalah **default production**, bisa saja salah untuk karyawan
TEST/Induction. Konfirmasi per-orang untuk yang ragu.

## 2. Rekap target (setelah re-check 2026-10-13)

- Dari 57: **1 sudah punya branch** (28382 Abigael, branch 2 — hasil edit via Jam tadi;
  report lama masih menyebut kosong → report sudah setengah stale).
- `employee_position.branch_id` NULL untuk semua 57 → tidak ada fallback otoritatif.
- Sisa belum berubah: 56.

## 3. Draft per-branch (49 aktif + 8 non-aktif ditandai ⏸)

### branch 2 — Pantai Indah Kapuk (24)

| employee_id | Nama | token | active | catatan |
|---|---|---|---|---|
| 28382 | Abigael Montemayor Manzano | piks | 1 | ✅ sudah branch 2 (via Jam) |
| 27630 | Aloysius Gonzaga Adi Suryo Nugroho | piks | 1 | |
| 27631 | Christine Joy Jaurigue David | piks | 1 | |
| 28757 | Dameita Sumbayak | piks | 1 | |
| 27632 | Damien Martin Joseph Hutchman | piks | 1 | |
| 27633 | Desy Carolina | piks | 1 | |
| 28756 | Erick Heriyanto | piks | 1 | |
| 28759 | Georgina Bailey Mills | piks | 1 | |
| 28760 | Godlas Harianja | piks | 1 | |
| 27634 | Jeffrey Johan | piks | 1 | |
| 27635 | Jia Xiaoyu | piks | 2 | ⏸ non-aktif |
| 27636 | Joseph Roy Gordon | piks | 2 | ⏸ non-aktif |
| 27495 | Julie Ann Basconcillo | pik | 1 | satu-satunya JC-only |
| 28758 | Laxman Singh Rawat | piks | 1 | |
| 27637 | Louis Wirja | piks | 2 | ⏸ non-aktif |
| 27638 | Mark Joseph Canumay Diaz | piks | 1 | |
| 27639 | Qin Zhiting | piks | 1 | |
| 27640 | Rivoldy Christian Taroreh | piks | 1 | |
| 27641 | Simran Kishore Ramchandani | piks | 1 | |
| 27642 | Song Sisi | piks | 2 | ⏸ non-aktif |
| 27643 | Xu Shuo | piks | 2 | ⏸ non-aktif |
| 27644 | Yu Li Jin | piks | 1 | |
| 28794 | Zhang Yaning | piks | 1 | |
| 29171 | Vania Jolie | music.piks → piks | 1 | token terakhir `piks` |

### branch 5 — Malang (9)

| employee_id | Nama | token | active |
|---|---|---|---|
| 27614 | Andri Setyo Subagyo | mlgs | 2 ⏸ |
| 27618 | Casielyn Miraflores Capistrano | mlgs | 1 |
| 27615 | Daniel David Widiya | mlgs | 1 |
| 28771 | Dusmant Bastarayi | mlgs | 1 |
| 28850 | Edna Mamawag Saga | mlgs | 1 |
| 28770 | Godofredo Dilla Degsi Jr. | mlgs | 1 |
| 26983 | John Joseph Estillore | mlgs | 2 ⏸ |
| 28772 | Mark Dave Gagal Rarugal | mlgs | 1 |
| 27304 | Orlene Abrazado Gagui | mlgs | 1 |

### branch 4 — Semarang (6)

| employee_id | Nama | token | active |
|---|---|---|---|
| 27177 | Chandra Seto Agus | smgs | 1 |
| 28484 | Fransisca Agustin Suryaningtyas | smgs | 1 |
| 28776 | Huang Jingran | smgs | 1 |
| 28735 | Jellie Ann Moraza Tiu | smgs | 1 |
| 28407 | Joko Puji Widodo | smgs | 1 |
| 28775 | Qin Beibei | smgs | 1 |

### branch 3 — Bandung (8)

| employee_id | Nama | token | active | catatan |
|---|---|---|---|---|
| 29130 | Dwi Kurniati Halim | bdg | 1 | |
| 27693 | Evert Octavianus Amanupunjo | bdgs | 1 | |
| 29036 | Feby Dharmastuti | bdg | 1 | |
| 26925 | Haikal | bdgs.migdup | 1 | ⚠️ token ambigu (`migdup`), bdgs → Bandung |
| 27023 | Heriawan | bdgs | 1 | |
| 28767 | Hu Jingqiu | bdgs | 1 | |
| 27083 | Vita Fradiantika | bdgs | 1 | |
| 27703 | Yogeeta Rajesh Bhambani | bdgs | 1 | |

### branch 1 — Kebon Jeruk (4)

| employee_id | Nama | token | active |
|---|---|---|---|
| 27698 | Gary Lee Turnmire | kjs | 2 ⏸ |
| 28745 | Spencer James McGill | kjs | 1 |
| 28747 | Tetyana Dyedkova | kjs | 1 |
| 28744 | Wayan Jana Eko Putra | kjs | 1 |

### branch 6 — Balikpapan (1)

| employee_id | Nama | token | active |
|---|---|---|---|
| 28180 | Li Xin Ran | bpns | 1 |

### ❓ Ambiguous — TIDAK boleh dieksekusi tanpa konfirmasi (2)

| employee_id | Nama | Alasan |
|---|---|---|
| 27620 | Guindo Renegades Nada Dewa Min Syah | email `Guindo@` — tanpa token branch sama sekali |
| 28060 | Richard Herawaty | token `migdup` — bukan kode branch manapun (email `richard.migdup@`) |

### ⏭ Skip: non-aktif (8, termasuk 3 artifak)

27614, 27698, 26983, 27636, 27637, 27642, 27643 + artifak migrasi
**781, 22170, 24678** (`migrated.unknown.*`) → rekomendasi: kecualikan dari batch,
verifikasi terpisah (lihat data-report.md Option C).

## 4. Rekap eksekusi yang diusulkan

| Batch | Jumlah | Isi |
|---|---|---|
| Aktif + yakin | 44 | branch 2 (20 aktif), 5 (7), 4 (6), 3 (8), 1 (3), 6 (1) |
| Aktif + ambigu | 2 | 27620, 28060 — tanya dulu |
| Non-aktif | 7 | tahan |
| Artifak migrasi | 3 | skip, verifikasi terpisah |
| Sudah beres | 1 | 28382 (branch 2 via Jam) |
| **Total** | **57** | |

*(Hitung batch aktif-yakin: 57 − 2 ambigu − 8 non-aktif − 1 sudah = 46; dikurangi 28382
yang sudah beres → 45. Koreksi angka final setelah konfirmasi.)*

## 5. Sebelum eksekusi — masih perlu

1. **Koreksi kamu** atas draft di atas (terutama 2 ambigu + kecurigaan branch TEST).
2. **Refresh token `.secret`** — token sekarang expired (API return 401/403 kosong).
3. **GET sample 1–2 employee** → cek `titleType`/`campusIds` existing, preview payload PUT.
4. Konfirmasi **enum `employee_types`**: resep Jam menghasilkan `["TEACHER"]` /
   DB pakai `{3}` (2088 row) — mapping API↔DB belum diverifikasi.
5. Putuskan perlakuan **8 non-aktif + 3 artifak**.
