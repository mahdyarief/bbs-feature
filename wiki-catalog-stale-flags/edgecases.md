# Edge Cases — Wiki Catalog Stale Flags Update

> **Status: PENDING** — jawab langsung di file ini, pilih opsi atau tulis custom.

## EC-01: OTP toggle masih ada di teacher `Login.jsx` (belum dicek saat TC-019)
**Scenario:** Bukti TC-019 hanya dari admin `Login.jsx`. Teacher portal punya file serupa yang bisa saja masih memuat toggle.

| Opsi | Behavior |
|------|----------|
| (A) Cek dulu, tulis status per-portal | `access.md` mencatat: admin = dihapus, teacher = masih ada / tidak — akurat per portal |
| (B) Langsung tulis "toggle sudah dihapus" | Berisiko salah bila teacher masih punya |

**Decision:** _TBD_ (rekomendasi: A)

---

## EC-02: `OTP_ENABLED` di nest masih dipakai `device-security.service.ts`
**Scenario:** Toggle UI hilang, tapi switch env di API tetap ada dan tetap menonaktifkan OTP bila tidak diset.

| Opsi | Behavior |
|------|----------|
| (A) Update wiki tanpa menyentuh env | Dokumentasi mencatat switch env tetap ada; hanya klaim "UI toggle" yang dikoreksi |
| (B) Sekalian usulkan hardening env | Keluar dari scope dokumentasi; buat spec terpisah |

**Decision:** _TBD_ (rekomendasi: A)

---

## EC-03: Default pageSize CCA tidak conclusive (data hanya 6 baris)
**Scenario:** Probe menunjukkan 6 baris tanpa param — tidak bisa membedakan "default 5 tidak berlaku lagi" vs "semua data cuma 6".

| Opsi | Behavior |
|------|----------|
| (A) Qualify klaim di wiki | Tulis "default tidak lagi terverifikasi 5 (probe 2026-09-30: 6 baris, count 6); verifikasi ulang saat data > 6" |
| (B) Tambah data test lalu probe ulang | Butuh write di production — tidak layak untuk klarifikasi dokumen |

**Decision:** _TBD_ (rekomendasi: A)

---

## EC-04: Wiki sudah berubah di remote saat edit dilakukan (git pull mengubah halaman terkait)
**Scenario:** Saat edit `access.md`, remote berubah (orang lain meng-update OTP page).

| Opsi | Behavior |
|------|----------|
| (A) Pull --rebase lalu review ulang flag | Pastikan flag masih relevan sebelum commit |
| (B) Force push | ❌ Jangan |

**Decision:** _TBD_ (rekomendasi: A — standar workflow)

---

## EC-05: AI operate bergantung pada klaim wiki lama di file lain (tidak hanya 4 halaman ini)
**Scenario:** `_shared/api.md` operate-smartbag masih menulis "jangan kirim header bbs-web-token" dst — sebagian klaim wiki lain mungkin juga basi tapi belum diverifikasi.

| Opsi | Behavior |
|------|----------|
| (A) Scope ketat: hanya 3 flag terverifikasi | Tidak menyebar perubahan tanpa bukti; flag lain ditunggu TC berikutnya |
| (B) Audit wiki menyeluruh | Bagus tapi besar; buat spec/fase terpisah |

**Decision:** _TBD_ (rekomendasi: A; audit menyeluruh bisa jadi TC-020+)
