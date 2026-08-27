# Smartbook Feature Deep-Dive (Bina Bangsa School)

**Sumber data:** crawl portal ASD (admin), teacher, dan principals legacy
**Tanggal:** 2026-08-26
**Metode:** grep lintas folder (getmenulist.json, endpoint_analysis.json, network_log.json, pages_meta.json, HTML hasil crawl) + live probe dengan session `bbs_mng`

---

## 1. Ringkasan

Smartbook adalah platform e-learning/kerja siswa dari vendor **heyhi.sg** yang terintegrasi ke portal BBS melalui:
1. **Menu di portal ASD & Teacher** — 5 item menu Smartbook + 4 item menu Leaps
2. **SSO** — `sso_heyhi.php` (teacher portal) dengan parameter base64
3. **Viewer member (paid)** — `ais/asd/smartbook/viewer.php` (portal ASD)
4. **Login log** — `ais/teachers/smartbook/login_log` (2MB tabel "BBS - Smartbook Attempt SSO")
5. **Dashboard analitik** — `clientreport.heyhi.sg/public/dashboard/...` (Metabase public)
6. **Leaps** — entry viewer terpisah (leaps_pri/leaps_asd)

---

## 2. Menu Smartbook (dari getmenulist.json)

### Portal ASD (getmenulist.json)
| ID | Menu | Link |
|----|------|------|
| 1163 | Teacher Stats - Smartbook | `https://clientreport.heyhi.sg/public/dashboard/8f589a1a-5693-4d87-926c-a2406b6d351d` |
| 1160 | Smartbook User Login | `../teachers/smartbook/login_log` |
| 1164 | Worksheet Stats - Smartbook | `https://clientreport.heyhi.sg/public/dashboard/a596cec7-ef8c-4e4e-9a76-77dc468e4419` |
| 1165 | Subject Stats - Smartbook | `https://clientreport.heyhi.sg/public/dashboard/da204a56-968f-45a0-b326-1f6964a59b81` |
| 1166 | Smartbook Member | `https://zone.binabangsaschool.com/ais/asd/smartbook/viewer.php` |

### Portal Teacher (getmenulist.json)
| ID | Menu | Link |
|----|------|------|
| 10160 | Smartbook User Login | `../teachers/smartbook/login_log/` |
| 10170 | Teacher Stats - Smartbook | `https://clientreport.heyhi.sg/public/dashboard/8f589a1a-...` |
| 10180 | Worksheet Stats - Smartbook | `https://clientreport.heyhi.sg/public/dashboard/a596cec7-...` |
| 10190 | Subject Stats - Smartbook | `https://clientreport.heyhi.sg/public/dashboard/da204a56-...` |
| 10200 | Smartbook Member | `https://zone.binabangsaschool.com/ais/asd/smartbook/viewer.php` |

### Menu Leaps (ASD)
| ID | Menu | Link |
|----|------|------|
| 621 / 611 | Leaps Entry | `../teachers/leaps_asd/`, `../teachers/leaps_pri/` |
| 612 / 622 | Leaps Report | `../teachers/choose_campus_asd_leaps.php` |
| 623 / 613 | Viewer Leaps | `../teachers/leaps_asd/link_ais_viewer_leaps.php` |

### Menu Leaps (Teacher)
| ID | Menu | Link |
|----|------|------|
| 591 | Leaps | `../teachers/leaps/link_ais.php` |
| 593 | Viewer Leaps | `../teachers/leaps/link_ais_viewer_leaps.php` |

---

## 3. SSO ke heyhi.sg (Smartbook)

Ditemukan di hasil analisis endpoint & metadata halaman (`endpoint_analysis.json` & `pages_meta.json`):

```
https://teachers.binabangsaschool.com/sso_heyhi.php?role_user=1&auth=dGVhY2hlcjIwMjE4MQ==&reg=NA==&token=8e3a591226caad9a66fb948c9c6a348a
```

**Decode parameter:**
| Param | Value | Decode |
|-------|-------|--------|
| `role_user` | `1` | role guru |
| `auth` | `dGVhY2hlcjIwMjE4MQ==` | **base64 username** → `teacher202181` |
| `reg` | `NA==` | **base64** → `4` (region/reg) |
| `token` | `8e3a591226caad9a66fb948c9c6a348a` | MD5 hash (32 hex) — kemungkinan MD5(username + salt/reg) |

**Hasil crawl:** halaman `sso_heyhi.php` di-crawl saat session teacher habis → menampilkan `"Access Denied, Username invalid!"` (71 byte) — artinya endpoint ini **memvalidasi token** dan menolak jika token tidak cocok dengan session.

**Kesimpulan:** SSO Smartbook bekerja dengan mengirim username (base64) + region (base64) + token MD5. Token kemungkinan di-generate server-side per user (MD5 dari kombinasi username/reg/salt). Tanpa token valid, akses ditolak.

---

## 4. Smartbook Member Viewer (Paid Viewer)

**URL:** `https://zone.binabangsaschool.com/ais/asd/smartbook/viewer.php`
**Title:** `BBS Smartbook - Paid Viewer`
**Status:** ✅ bisa diakses dengan session `bbs_mng` (login ASD)

### Filter (dropdown)
| Field | Options |
|-------|---------|
| `ay_option` (Academicyear) | `27` (2026/2027), `26` (2025/2026), `25` (2024/2025) |
| `campus_option` (Campus) | `-1` (All), `2` (KJ-S), `4` (PIK-S), `6` (BDG-S), `8` (SMG-S), `10` (MLG-S), `14` (BPN-S) |
| `cohort_option` (Cohort) | `-1` (All), `7` Sec1Acc, `8` Sec1Exp, `9` Sec2Acc, `10` Sec2Exp, `11` Sec3Acc, `12` Sec3Exp, `14` Sec4Exp, `15` JCB, `16` JC1, `17` JC2 |
| `status` (Enroll Status) | `-1` (All), `1` Enrolled, `3` Paid, `4` Not Paid, `2` None |

### Data statistik yang ditampilkan (per subject)
```
EL   Clear 828 /901 (91.9)  Paid: 729 /901 (80.91)
MATH Clear 1034/1131 (91.42) Paid: 947 /1131 (83.73)
SCI  Clear 474 /494 (95.95)  Paid: 448 /494 (90.69)
PHY  ...
```
→ Menampilkan jumlah siswa **enrolled/clear** vs **paid** per mata pelajaran Smartbook (EL, MATH, SCI, PHY, dll).

### Tabel detail
| # | Campus | Student | Class | Enrolled |
|---|--------|---------|-------|----------|

### Endpoint terkait (ditemukan di HTML)
- `https://zone.binabangsaschool.com/library/html2pdf/examples/smartbook_paid_report.php?` — **export laporan paid ke PDF**
- `update_payment.php` — update status payment (data: student id)

---

## 5. Smartbook Login Log ("Smartbook Attempt SSO")

**URL:** `https://zone.binabangsaschool.com/ais/teachers/smartbook/login_log`
**Status:** ✅ 200, **2.07MB** — tabel besar berisi log setiap percobaan SSO ke Smartbook.

Title: `BBS - Smartbook Attempt SSO`
→ Mencatat seluruh percobaan login/SSO user ke platform Smartbook (audit trail).

---

## 6. Dashboard Analitik heyhi.sg (Metabase public)

3 dashboard public di `clientreport.heyhi.sg`:
1. **Teacher Stats** — `public/dashboard/8f589a1a-5693-4d87-926c-a2406b6d351d`
2. **Worksheet Stats** — `public/dashboard/a596cec7-ef8c-4e4e-9a76-77dc468e4419`
3. **Subject Stats** — `public/dashboard/da204a56-968f-45a0-b326-1f6964a59b81`

**Catatan:** `clientreport.heyhi.sg` **timeout saat diakses** dari jaringan ini (handshake SSL gagal) — kemungkinan diblokir/region-locked. Status aksesibilitas publik belum terkonfirmasi.

---

## 7. Leaps (module terkait Smartbook)

Leaps adalah modul **Learning Engagement / Assessment Progress** yang terpisah namun berdekatan dengan Smartbook (link di menu berurutan).

### Alur akses (network log)
```
GET  https://teachers.binabangsaschool.com/leaps_pri/index.php
GET  https://teachers.binabangsaschool.com/leaps_pri/leaps.php
POST https://teachers.binabangsaschool.com/leaps_pri/view/leaps/view_data.php  (xhr - ambil data)
```
→ `link_ais.php?campus=0` (zone) redirect ke `leaps_pri/leaps.php` (teachers portal).

### Endpoint
| File | Fungsi |
|------|--------|
| `leaps_pri/link_ais.php` | Entry point (link dari ASD ke teacher portal) |
| `leaps_pri/link_ais_viewer_leaps.php` | Viewer leaps |
| `leap_levels.php` | Level leaps |
| `leap_levels _service.php` | Service leaps |
| `view/leaps/view_data.php` | **POST xhr** ambil data leaps |
| `viewer_leaps.php` (teacher) | Viewer leaps student, form action `services/view_leaps_data.php` |

### Viewer Leaps (teacher portal, `_viewer_leaps.php.html`)
- Form: `services/view_leaps_data.php`
- Field: dropdown `Choose Class` (contoh "SAMPLE CLASS")
- Script: jquery.autocomplete.min.js

---

## 8. Token Endpoint (gettoken_nya)

**URL:** `https://zone.binabangsaschool.com/ais_new/index.php/tickets/gettoken_nya?tkn=<md5>&utp=<md5>`
**Status:** ✅ 200 dengan session

| Input | Output |
|-------|--------|
| Tanpa param | `Token Invalid! 1` (16 byte) |
| `tkn` + `utp` (pasangan salah) | `Token Invalid! 3` |
| `tkn=605ff764c617d3cd28dbbdd72be8f9a2&utp=e4da3b7fbbce2345d7772b0674a318d5` | **36KB halaman "Tickets"** (dashboard tickets) |
| `tkn=605ff764c617d3cd28dbbdd72be8f9a2&utp=a87ff679a2f3e71d9181a67b7542122c` | `Token Invalid!` |

**Kesimpulan:** `gettoken_nya` memvalidasi pasangan `tkn` (token) + `utp` (user token hash). Hanya pasangan yang cocok yang membuka halaman "Tickets" (36KB). Token `605ff764c617d3cd28dbbdd72be8f9a2` adalah MD5 — kemungkinan MD5 dari user id.

**Catatan:** `utp=e4da3b7fbbce2345d7772b0674a318d5` = MD5("5") (ditemukan saat crawl), `utp=a87ff679a2f3e71d9181a67b7542122c` = MD5("7"). Jadi `utp` = MD5 dari angka (id user/role).

---

## 9. Ringkasan Endpoint Smartbook

| Endpoint | Metode | Fungsi | Akses |
|----------|--------|--------|-------|
| `/ais/asd/smartbook/viewer.php` | GET/POST | Paid viewer (enroll vs paid per subject) | ✅ session ASD |
| `/ais/teachers/smartbook/login_log` | GET | Log percobaan SSO (2MB) | ✅ session ASD |
| `teachers.binabangsaschool.com/sso_heyhi.php` | GET | SSO ke heyhi.sg (auth=base64, reg=base64, token=MD5) | 🔒 butuh token valid |
| `clientreport.heyhi.sg/public/dashboard/*` | GET | Dashboard analitik (Metabase) | ⚠️ timeout dari sini |
| `/library/html2pdf/examples/smartbook_paid_report.php` | GET | Export laporan paid → PDF | ✅ session |
| `update_payment.php` | POST | Update status payment | (belum diprobe) |
| `/ais_new/index.php/tickets/gettoken_nya` | GET | Validasi tkn+utp → halaman Tickets | ✅ session + pasangan tkn/utp valid |
| `leaps_pri/leaps.php` + `view/leaps/view_data.php` | GET/POST | Viewer Leaps (learning progress) | ✅ session |

---

## 10. Security Notes

1. **SSO token MD5** — token `8e3a5912...` adalah MD5 plain (bukan HMAC), bisa di-brute jika salt lemah; `auth`/`reg` hanya base64 (mudah dibaca).
2. **gettoken_nya** — `utp` adalah MD5 dari angka sederhana (MD5("5"), MD5("7")), menunjukkan skema token lemah.
3. **Login log 2MB** — endpoint `login_log` mengekspos seluruh riwayat percobaan SSO user (potensi info leak) tanpa pagination terlihat.
4. **Viewer member** — menampilkan status pembayaran siswa per subject (data sensitif komersial), hanya dilindungi session PHPSESSID.
