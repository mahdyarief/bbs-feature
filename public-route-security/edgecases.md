# Edge Cases — Public Route Security

## EC-01: Mekanisme kredensial integrasi attendance
**Scenario:** Gate hardware kini kirim `token` di body; mau diganti apa?

| Opsi | Behavior |
|------|----------|
| (A) Pindah token ke env + rotasi berkala | Perubahan minimal; secret tetap statis antar rotasi |
| (B) HMAC signature + timestamp (secret env) | Tahan replay; butuh update firmware/konfigurasi device |

**Decision:** _TBD_ (rekomendasi (A) segera karena murah, (B) sebagai target).

---

## EC-02: Route `sync*` mana yang benar-benar dipakai cron produksi?
**Scenario:** ±18 route sync @Public — sebagian mungkin dipanggil scheduler
eksternal (cron-job.org / k8s cron) dengan cara tertentu; menambah password
guard bisa mem-break scheduler.

| Opsi | Behavior |
|------|----------|
| (A) Inventaris cron dulu (devops), lalu guard semuanya | Aman penuh; butuh koordinasi |
| (B) Guard yang jelas tidak dipakai cron; sisanya diberi token juga | Bertahap, risiko lupa sisanya |

**Decision:** _TBD_ (rekomendasi (A) dengan deadline; sementara itu
kandidat temuan tetap tercatat).

---

## EC-03: `campuses/:id` dan `files` GET publik
**Scenario:** Data master campus & metadata file terbaca tanpa token.

| Opsi | Behavior |
|------|----------|
| (A) Kunci (hapus @Public) | Aman; cek dulu form publik/landing yang mungkin memakai |
| (B) Biarkan, dokumentasikan | Data tidak sensitif; katalog publik memang terbuka |

**Decision:** _TBD_

---

## EC-04: Rate limit untuk endpoint publik
**Scenario:** Endpoint publik read (`academicYears`, `leadgens`, dsb.) bisa
di-scrape tanpa batas.

| Opsi | Behavior |
|------|----------|
| (A) Tambah rate limit global (throttler guard) | Melindungi scrape & abuse; perlu uji beban |
| (B) Biarkan (sudah ada rate limit impersonate saja) | Risiko scrape data master |

**Decision:** _TBD_ (terkait TC-038: rate limit POST sudah ada di path tertentu).
