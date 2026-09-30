---
feature: Wiki Catalog Stale Flags Update (bbs-wiki)
slug: wiki-catalog-stale-flags
status: draft
author: System Analyst (dari temuan AI Operate Test TC-019)
date: 2026-09-30
target_release: TBD
---

# Wiki Catalog Stale Flags Update

## Overview

Pembaruan dokumentasi `bbs-wiki` untuk 3 temuan dari TC-019 (verifikasi
`open-questions.md` terhadap codebase & produksi, 2026-09-30): kontradiksi yang
dicatat wiki sudah tidak akurat dengan code hari ini. Wiki adalah "product
catalog" yang jadi referensi AI operate — dokumen basi berisiko menghasilkan
keputusan operasi yang salah.

## Problem / Motivation

| # | Flag wiki | Fakta terverifikasi 2026-09-30 | Halaman wiki terdampak |
|---|---|---|---|
| 1 | OTP UI toggle bypass ada di admin & teacher `Login.jsx` | `OTP_ENABLED` **tidak ada lagi** di admin `Login.jsx` (hanya di nest `device-security.service.ts`); tidak ada di `.env.example` | `access.md`, `systems.md`, `open-questions.md` |
| 2 | `teacher-leave` tidak ada sebagai module api_nest | Folder **`teacher-leave` ada** sekarang (muncul via git pull terbaru) | `other-modules.md` |
| 3 | `GET /ccaRegistrations` default pageSize 5 | Response tanpa param memuat 6 baris (count 6, meta.pageCount null) — tidak terbukti 5 lagi | `cca.md` |

Wiki `README.md` sendiri menetapkan: "This wiki is the product catalog... source
of truth for feature math: the linked note, then the live code path named in that
note." Ketika live code berubah, catalog wajib di-update — kalau tidak, AI dan
engineer baru akan mengikuti aturan yang sudah mati.

## Scope

### In Scope
- Edit 4 halaman wiki di atas (edit dokumen, bukan code).
- Tandai tiap flag di `open-questions.md` sebagai resolved/updated dengan tanggal
  dan bukti (file:line, hasil probe API).
- Cek silang teacher `Login.jsx` untuk flag OTP sebelum menulis (bukti admin saja
  belum cukup).

### Out of Scope
- Menulis ulang halaman wiki lain yang belum diverifikasi.
- Perubahan code di api_nest/bbs.
- Menambah halaman wiki baru.

## User Stories

### As an AI operate agent
I want the wiki catalog to match live code
So that test expectations and operate decisions follow the current contract.

### As a new engineer
I want `open-questions.md` to only list unresolved contradictions
So that I don't chase issues that are already fixed.

## Acceptance Criteria

- [ ] `access.md` + `systems.md`: bagian OTP toggle di-update (admin Login.jsx
      tidak lagi memuat toggle; status teacher diverifikasi lalu dicatat).
- [ ] `other-modules.md`: catatan "teacher-leave tidak ada" dihapus/direvisi +
      tanggal verifikasi.
- [ ] `cca.md`: klaim default pageSize dikoreksi/di-qualify (dengan bukti probe).
- [ ] `open-questions.md`: 3 flag diberi status resolved/updated + link bukti
      (`operate-smartbag/test-cases/TC-019-wiki-contract-flags/result.md`).
- [ ] Semua edit mencantumkan tanggal verifikasi (2026-09-30).

## UI / UX Changes

Tidak ada (dokumentasi saja).

### Affected Portals
- [ ] Admin (client/)
- [ ] Student (client-student/)
- [ ] Teacher (client-teacher/)

## API Changes

Tidak ada. (Probe read-only yang dipakai sebagai bukti:

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/ccaRegistrations` | Bukti default pagination |

)

## Database Changes

Tidak ada.

## Business Rules / Validation

1. Setiap klaim wiki yang direvisi harus punya bukti code (path:line) atau probe
   API ber-tanggal.
2. Flag di `open-questions.md` hanya boleh ditutup bila kedua sisi (wiki + code)
   sudah dicek ulang — bukan dari satu sumber saja.
3. Kaidah AGENTS.md operate berlaku: code menang untuk perilaku API; wiki menang
   untuk konteks produk — jadi update perilaku, bukan menghapus konteks.

## Error Handling

Tidak relevan (dokumentasi).

## Dependencies

- Repo `bbs-wiki` (git pull terbaru sebelum edit).
- Bukti dari `operate-smartbag/test-cases/TC-019-wiki-contract-flags/result.md`.
- Codebase `smartbag/api_nest` sebagai sumber verifikasi.
