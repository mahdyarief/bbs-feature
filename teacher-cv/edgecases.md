# Edge Cases — Teacher CV

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.
> Setelah semua edge cases di-decide, update spec.md jika ada perubahan behavior.

## EC-01: Data pendidikan kosong
**Scenario:** Staff tidak memiliki data riwayat pendidikan sama sekali (belum diinput di modul HR, atau sumber data pendidikan belum tersedia di api_nest).

| Opsi | Behavior |
|------|----------|
| (A) Tampilkan seksi dengan placeholder | Seksi Pendidikan tetap ditampilkan dengan label, tapi isi "—" (konsisten dengan placeholder field kosong). Dokumen tetap lengkap secara struktur. |
| (B) Sembunyikan seksi penuh | Seksi Pendidikan dihilangkan seluruhnya dari dokumen CV jika array-nya kosong. Dokumen lebih ringkas, tapi pembaca tidak tahu bahwa data memang tidak ada. |
| (C) Tampilkan pesan "Data belum tersedia" | Seksi ditampilkan dengan teks penjelasan bahwa data belum tersedia / belum diinput. Lebih informatif untuk keperluan audit. |

**Decision:** _TBD_ — rekomendasi awal: (A) placeholder "—" untuk field kosong, dan seksi dengan array kosong tetap ditampilkan header-nya agar struktur CV konsisten; jika seluruh seksi kosong, pertimbangkan (C) di header dokumen.

---

## EC-02: Pengalaman kerja banyak / halaman panjang → print multi-page
**Scenario:** Staff memiliki banyak entri pengalaman kerja / pelatihan sehingga dokumen CV melebihi satu halaman A4.

| Opsi | Behavior |
|------|----------|
| (A) Print multi-page alami | Dokumen memakai CSS `@media print` + `page-break-inside: avoid` per entri; halaman terpotong rapi di batas seksi. Pengguna mencetak beberapa halaman via browser print dialog. |
| (B) Batasi jumlah entri per halaman | Paginasi manual: mis. 10 entri per halaman dengan header berulang — lebih terkontrol tapi kompleks dan tidak standar untuk dokumen CV. |
| (C) Export PDF server-side | Backend men-generate PDF dengan pagination otomatis (library PDF). Menjamin hasil cetak konsisten lintas browser, tapi menambah dependensi backend. |

**Decision:** _TBD_ — rekomendasi awal: (A) `page-break-inside: avoid` per seksi/entri untuk print via browser; (C) dapat dipertimbangkan sebagai enhancement bila hasil print browser tidak konsisten.

---

## EC-03: Staff nonaktif
**Scenario:** Staff berstatus INACTIVE (nonaktif / sudah keluar), namun admin ingin melihat atau mencetak CV-nya (arsip).

| Opsi | Behavior |
|------|----------|
| (A) Admin/principal tetap bisa melihat | Staff INACTIVE tetap muncul di dropdown/search admin dan CV-nya tetap dapat dilihat/dicetak; ada penanda status nonaktif di header CV. |
| (B) Hanya admin (portal ASD) yang bisa | Staff INACTIVE hanya bisa diakses oleh Admin Portal; Principal teacher portal tidak bisa (scope campus mungkin sudah berubah). |
| (C) Sembunyikan staff nonaktif | Staff INACTIVE tidak muncul di daftar; CV hanya untuk staff aktif. Kurang mendukung kebutuhan arsip. |

**Decision:** _TBD_ — rekomendasi awal: (A) admin/principal tetap bisa melihat dengan penanda status nonaktif; perlu konfirmasi kebijakan arsip HR.

---

## EC-04: NIK / NIP kosong
**Scenario:** Data NIK (nomor KTP) atau NIP (nomor induk pegawai) tidak terisi di `employee-identity` / `employee`.

| Opsi | Behavior |
|------|----------|
| (A) Placeholder "—" | Field NIK/NIP ditampilkan "—" di header CV; dokumen tetap dirender normal. |
| (B) Sembunyikan field | Field yang kosong tidak ditampilkan sama sekali; header lebih bersih tapi struktur dokumen bervariasi antar staff. |
| (C) Tampilkan peringatan | CV dirender dengan peringatan (mis. badge "data NIP belum lengkap") — membantu admin mendeteksi data bermasalah. |

**Decision:** _TBD_ — rekomendasi awal: (A) placeholder "—" untuk konsistensi struktur; (C) bisa ditambahkan sebagai badge kecil untuk admin.

---

## EC-05: Akses lintas campus admin
**Scenario:** Admin (portal ASD) ingin melihat CV staff yang berada di campus berbeda dari campus default `req.user.campusId`, atau Principal ingin melihat staff di luar campus-nya.

| Opsi | Behavior |
|------|----------|
| (A) Admin lintas campus penuh | Admin Portal dapat mengirim `campusId` mana pun dan melihat CV staff di campus mana pun (permission HR). Principal teacher portal tetap dibatasi campus-nya. |
| (B) Batasi per campus | Setiap pengguna hanya bisa melihat staff dalam campus-nya; admin ASD harus memiliki relasi campus tertentu. Kurang fleksibel untuk HR pusat. |
| (C) Whitelist campus | Admin hanya bisa akses campus yang diberikan via permission list; transparan namun perlu setup tambahan. |

**Decision:** _TBD_ — rekomendasi awal: (A) admin lintas campus dengan scope via `campusId`; Principal dibatasi `req.user.campusId` (403 bila memaksa campus lain).

---

## EC-06: Pendidikan/perguruan tidak terstruktur di legacy (free text) → bagaimana di-smartbag
**Scenario:** Di legacy, kolom pendidikan/institusi disimpan sebagai free text (tidak ada master data perguruan), sehingga bisa bervariasi (mis. "UNJ", "Univ. Negeri Jakarta", "Universitas Negeri Jakarta").

| Opsi | Behavior |
|------|----------|
| (A) Tetap free text di smartbag | Simpan `institution` sebagai varchar bebas — tidak ada master data; konsisten dengan legacy, namun data kotor (variasi nama) tidak terselesaikan. |
| (B) Master data perguruan | Buat tabel master `institution` (id, name, alias[]) dan pilih via dropdown — data rapi dan bisa laporan; perlu seeding & migrasi data, di luar scope awal. |
| (C) Normalisasi bertahap | Tetap free text di tabel opsional `teacher_education`, tapi tambahkan kolom `institution_normalized` / mapping alias untuk laporan — kompromi antara (A) dan (B). |

**Decision:** _TBD_ — rekomendasi awal: (A) untuk rilis awal (mengikuti legacy), dengan catatan bahwa master data perguruan (B/C) dapat dijadikan enhancement bersama modul HR.
