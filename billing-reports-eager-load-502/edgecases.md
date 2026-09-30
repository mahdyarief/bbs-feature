# Edge Cases — Billing Reports 502

## EC-01: Strategi refactor pre-query
**Scenario:** Pre-query kini eager-load seluruh entitas `billings` per
MasterProduct hanya untuk mendapat daftar id.

| Opsi | Behavior |
|------|----------|
| (A) EXISTS subquery di QB utama | Satu query; planner yang optimasi; paling bersih |
| (B) Pre-query id-only + chunk `IN (...)` | Perubahan minimal; tetap ada round-trip tambahan |
| (C) Join langsung ke QB utama dengan GROUP BY | Satu query; bentuk respons per-product harus dijaga |

**Decision:** _TBD_ (rekomendasi (A)).

---

## EC-02: Dataset global tetap berat meski eager-load dihilangkan
**Scenario:** Branch permitted punya puluhan ribu masterProduct aktif berbilling.

| Opsi | Behavior |
|------|----------|
| (A) Wajibkan filter (campusId/branchId/periode) via DTO `@IsNumber()` non-optional | Tidak ada query lintas-campus lagi; 400 bila tanpa filter |
| (B) Biarkan global + index | Mengandalkan skala data; risiko 502 kembali saat data tumbuh |

**Decision:** _TBD_ (rekomendasi (A) — sejajarkan dengan rekomendasi sistemik
"list endpoint wajib filter domain" di COVERAGE.md).

---

## EC-03: Pattern-guide untuk keluarga list-endpoint
**Scenario:** Tiga bug sejenis kini terdokumentasi (`/students` no-filter,
`/topics` eager-load, `billingReports` keduanya) dengan 3 varian penyebab.

| Opsi | Behavior |
|------|----------|
| (A) Buat satu spec lintas ("List Endpoint Performance Guide") | Aturan wajib: filter domain non-optional, eager minimal, pageSize bounded; 3 endpoint jadi contoh kasus |
| (B) Fix per endpoint tanpa guide | Cepat, tapi endpoint baru bisa mengulangi pola |

**Decision:** _TBD_ (rekomendasi (A); bisa menggantikan 3 brief terpisah jadi
checklist implementasi per endpoint).

---

## EC-04: Route download Excel ikut lambat
**Scenario:** Modul ini punya route download (Workbook exceljs) yang mungkin
memakai pola query sama.

| Opsi | Behavior |
|------|----------|
| (A) Audit sekalian di brief ini | Satu PR, risiko scope lebih besar |
| (B) Terpisah setelah list sehat | Isolasi risiko; download butuh streaming agar memory aman |

**Decision:** _TBD_
