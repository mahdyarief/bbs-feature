# Edge Cases — Obj-Params Invalid 500

## EC-01: Whitelist kunci vs try/catch saja
**Scenario:** Kunci `orderObj`/`relationsObj` tidak divalidasi sampai masuk
TypeORM.

| Opsi | Behavior |
|------|----------|
| (A) Whitelist dari entity metadata | Error jadi 400 presisi ("unknown column"); butuh akses metadata per entity |
| (B) Try/catch global → 400 generik | Murah dan cepat; pesan tidak presisi ("invalid orderObj") |

**Decision:** _TBD_ (rekomendasi (B) dulu, (A) menyusul bila perlu pesan presisi).

---

## EC-02: Frontend yang memakai obj-params
**Scenario:** FE mungkin selama ini mengirim obj-params "sebenarnya error tapi
diam saja" (mis. relationsObj nama salah dan diabaikan 200) — setelah fix jadi
400 dan UI bisa rusak.

| Opsi | Behavior |
|------|----------|
| (A) Audit pemanggilan FE (grep client/ untuk relationsObj/orderObj/wheresObj) | Temukan pemakaian buruk sebelum mengubah perilaku |
| (B) Luncurkan fix langsung | 400 baru bisa menampakkan bug FE yang selama ini tersamar |

**Decision:** _TBD_ (rekomendasi (A), bersama fix sortBy yang satu keluarga).

---

## EC-03: `wheresObj` operator injection
**Scenario:** Isi operator bebas: `{"id":{"gte":0}}` sah, tapi operator asing
(mis. `{"id":{"$regex":"..."}}`) diteruskan ke TypeORM.

| Opsi | Behavior |
|------|----------|
| (A) Whitelist operator (contains, in, gte, lte, eq, like, dsb.) | Aman penuh; enumerasi eksplisit |
| (B) Pass-through (status quo) | Fleksibel; risiko error 500 / perilaku tak terduga |

**Decision:** _TBD_ (rekomendasi (A) menyusul; untuk sekarang try/catch 400).

---

## EC-04: Kompabilitas dengan `breakthrough` flag
**Scenario:** `breakthrough=true` (skip campus permit) ada di DTO global yang
sama; fix ini jangan sampai mengubah perilakunya.

| Opsi | Behavior |
|------|----------|
| (A) Pisahkan PR: error-handling saja | Scope bersih; breakthrough dievaluasi terpisah |
| (B) Sekalian guard breakthrough per role | Satu PR dua topik — review lebih berat |

**Decision:** _TBD_ (rekomendasi (A)).
