# Notes — Smartbook Integration (Full Manage heyhi.sg)

## Ringkasan Analisis

| Portal legacy | Temuan |
|---------------|--------|
| Zone ASD (portal admin) | Menu 1160/1163-1166 + Leaps 611/612/613/621/622/623; `smartbook/viewer.php` = Paid Viewer (18KB, title "BBS Smartbook - Paid Viewer"); `smartbook/login_log` 2MB (title "BBS - Smartbook Attempt SSO"); `gettoken_nya` validasi tkn+utp; `update_payment.php`; `smartbook_paid_report.php` |
| Teacher portal | Menu 10160/10170-10200 + Leaps 591/593; SSO `sso_heyhi.php?role_user=1&auth=<b64>&reg=<b64>&token=<MD5>`; menu "SSO Smartbook" & "Smartbook Survey" (badge new); "SSO Kotakode" (platform lain); `link_ais2.php?token=<md5>&id=<userid>` → form_param base64 (user_id=21046, campus=4 PIK-S, user_type=2); `viewer_leaps.php` form `services/view_leaps_data.php`; `update_leap_cat.php` |
| Principals portal | Leaps flow: `link_ais.php` → `leaps_pri/leaps.php` → `view/leaps/view_data.php` (POST xhr); `leap_levels.php?iquwk=1&asndj=Leadership`, `iquwk=5&asndj=Achievements`, `iquwk=10&asndj=Service`; `gettoken_nya` di home |

Dokumen detail: `reference/SMARTBOOK_FEATURES.md`.

## Komponen yang Harus Diimplementasikan (Full Manage)

1. **SSO ke heyhi.sg** — generate URL `sso_heyhi.php` (role_user, auth, reg, token), redirect, validasi, audit ke `smartbook_sso_log`. Token MD5 legacy → **HMAC** (security fix).
2. **Member Viewer** — filter (AY/campus/cohort/status), statistik enrolled vs paid per subject, tabel per student.
3. **SSO Log** — audit trail percobaan SSO (paginasi, filter campus & tanggal).
4. **Leaps Management** — viewer per kategori (Leadership/Achievements/Service) + update kategori (`update_leap_cat`).
5. **Dashboard heyhi** — embed/redirect 3 dashboard public (Teacher/Worksheet/Subject Stats).
6. **Ticket/Token** — validasi `gettoken_nya` (tkn+utp) → halaman Tickets.
7. **Export PDF & Update Payment** — laporan paid (PDF) + update status payment (optimistic lock).

## Keputusan yang Perlu Dikonfirmasi

1. **Sumber data enrollment**: apakah `smartbook_enrollment` di-sync dari heyhi.sg via job berkala (modul `external-service-integration`) atau di-input manual? — mempengaruhi desain upsert (EC-03).
2. **Skena token SSO**: legacy MD5 plain (`auth`/`reg` base64) — apakah diganti HMAC di sistem baru? (direkomendasikan, lihat Security Notes). Perlu konfirmasi kompatibilitas dengan heyhi.sg.
3. **Update payment**: cukup PATCH `/smartbook/payment/:id`, atau perlu integrasi modul `billing`/`payment` yang sudah ada di api_nest?
4. **Dashboard heyhi.sg** (`clientreport.heyhi.sg/public/dashboard/*`) timeout dari jaringan lokal — perlu cek dari jaringan yang bisa akses apakah dashboard benar-benar public (tanpa auth) atau perlu embed token.
5. **SSO Kotakode** — SSO ke platform lain (token string panjang, display:none). Perlu konfirmasi apakah ini juga harus diimplementasikan atau diabaikan.

## Referensi Legacy

- SSO: `teachers.binabangsaschool.com/sso_heyhi.php?role_user=1&auth=<b64>&reg=<b64>&token=<md5>` — auth decode `teacher202181`, reg `4`
- Member Viewer: `ais/asd/smartbook/viewer.php` — title "BBS Smartbook - Paid Viewer"; filter ay(27/26/25), campus(-1,2,4,6,8,10,14), cohort(-1,7-17), status(-1,1,3,4,2)
- Login Log: `ais/teachers/smartbook/login_log` — title "BBS - Smartbook Attempt SSO", kolom: #, Campus, User, 19 Aug2026 ... 26 Aug2026
- Leaps: `leaps_pri/leaps.php`, `view/leaps/view_data.php`, `leap_levels.php?iquwk&asndj`, `viewer_leaps.php`, `services/view_leaps_data.php`, `update_leap_cat.php`
- Export/Update: `library/html2pdf/examples/smartbook_paid_report.php`, `update_payment.php`
- Token: `ais_new/index.php/tickets/gettoken_nya?tkn=<md5>&utp=<md5>` — `utp` = MD5 angka sederhana

## Open Question

- Apakah perlu menampilkan **harga per subject** (data komersial) di viewer, atau hanya status paid?
- Apakah login log perlu export ke CSV/Excel (selain paginasi)?
