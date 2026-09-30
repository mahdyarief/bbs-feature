# Edge Cases — Transfer Audit Pagination Fix

## EC-01: Sekalian migrasi ke `/api/v1/transfer-audit`?
**Scenario:** Modul ini live di `/api/` tanpa `/v1` (`@Controller('transfer-audit')`
non-version) — satu-satunya route bisnis non-version selain `health`. Frontend yang
selalu menambah `/v1` mendapat 404.

| Opsi | Behavior |
|------|----------|
| (A) Fix pagination saja, path tetap `/api/` | Risiko semantik: konsumen lama tidak break; FE harus tahu path khusus ini |
| (B) Sekalian daftarkan `version: '1'` | Konsisten dengan mayoritas; tapi path lama `/api/transfer-audit` hilang → breaking change bagi pemanggil non-version |

**Decision:** _TBD_ (grep dulu pemanggil di `client/` — bila FE memang selalu pakai
`/v1`, opsi (B) justru memperbaiki bug tersembunyi; kalau ada pemanggil `/api/`,
pertahankan dual: `version: ['1'], path: 'transfer-audit'` tidak persis sama —
pertimbangkan `@Controller({ version: '1', path: 'transfer-audit' })` +
koordinasi konsumen).

---

## EC-02: Isi body `/statistics` yang diharapkan
**Scenario:** Tidak ada kontrak tertulis apa yang seharusnya keluar dari
`getStatistics()`.

| Opsi | Behavior |
|------|----------|
| (A) Agregat minimum | `{total, success, failed, lastTransferAt}` — cukup untuk dashboard dasar |
| (B) Agregat per campus + per periode | Lebih kaya tapi butuh spesifikasi & kemungkinan kolom tambahan |

**Decision:** _TBD_ (rekomendasi mulai dari (A)).

---

## EC-03: Request dengan `page=0` atau `limit=0`
**Scenario:** Setelah koersi eksplisit, pemanggil mengirim `?page=0&limit=0`.

| Opsi | Behavior |
|------|----------|
| (A) Clamp ke minimum 1 | `page=max(1,page)`, `limit=max(1,min(limit,100))` — tanpa error |
| (B) Tolak 400 | Ketat tapi menuntut FE rapi |

**Decision:** _TBD_ (rekomendasi (A) clamp, konsisten toleransi API lain).

---

## EC-04: Tabel audit kosong di environment baru
**Scenario:** `transfer_audit_log` belum pernah terisi (fresh env) — request
pertama tanpa query.

| Opsi | Behavior |
|------|----------|
| (A) Return 200 `{audits:[], total:0, page:1}` | Kosong = valid, bukan error |
| (B) 404 | Menyesatkan — resource ada, datanya kosong |

**Decision:** _TBD_ (rekomendasi (A) — dan inilah perilaku setelah fix).
