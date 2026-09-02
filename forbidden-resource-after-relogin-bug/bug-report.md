# Bug Report: Error "Forbidden Resource" pada Aksi POST/PUT/DELETE — Sembuh Setelah Relogin

## Metadata
- **Reported by:** Keluhan berulang user (via Mahdy Arief)
- **Date:** 2026-09-02
- **Codebase:** `smartbag` (`D:\Work\BBS\requirement\smartbag`)
- **Severity:** High (mengganggu semua user, terjadi pada aksi mutasi)
- **Module:** Auth / Authorization (CASL Permission) — lintas modul
- **Environment:** Teacher Portal (teacher.smartbag.binabangsaschool.com)

## Deskripsi Bug
User sering mengeluh: saat melakukan **aksi** (POST/PUT/PATCH/DELETE) di aplikasi, muncul error **"Forbidden resource"** dan user dilempar ke halaman **/login**. Setelah **login ulang**, aksi yang sama berhasil dilakukan — masalah sembuh sementara.

## Steps to Reproduce
1. Login sebagai teacher/admin di browser A
2. (Di belakang layar) Login kembali di browser/device B dengan akun yang sama, **atau** token di-refresh dari session lain
3. Kembali ke browser A, lakukan aksi mutasi (contoh: simpan form, update data, delete)
4. Muncul error **"Forbidden resource"** (403) dan browser A di-redirect ke `/login`
5. Login ulang di browser A → aksi berhasil

## Expected Behavior
Session yang sudah login seharusnya tetap valid; aksi mutasi tidak boleh gagal dengan 403 hanya karena token di-generate ulang di tempat lain.

## Actual Behavior
Aksi mutasi gagal dengan 403 `Forbidden resource`, user di-redirect ke `/login`. Setelah login ulang (token baru disimpan di DB dan browser), aksi berhasil — hingga token kembali tidak sinkron.

## Root Cause Analysis

### 1. Backend: Token tunggal per user — setiap login/refresh MENIMPA token di DB
**File:** `api_nest/src/modules/auth/auth.service.ts`

Sistem menyimpan **hanya satu `accessToken` per user** di tabel user/teacher/admin. Setiap kali login atau refresh token, nilai baru **menimpa** nilai lama:

```typescript
// auth.service.ts:320 (login teacher) — dan line 400, 501, 553, 607, 705 untuk flow lain
user.accessToken = token.accessToken;  // token lama HANGUS
```

Akibatnya: login di device B membuat token device A menjadi **invalid** (tidak cocok dengan DB).

### 2. Backend: Middleware membandingkan token header dengan token DB secara eksak
**File:** `api_nest/src/common/middleware/convert-token-to-user.middleware.ts:51-53`

```typescript
const accessToken = req.headers.authorization.split(' ')[1];
if (accessToken === undefined || userFromDb.accessToken !== accessToken) {
  req.user = null;   // ← token mismatch → user di-null-kan
  next();
  return;
}
```

Ketika token di header (device A) ≠ token di DB (hasil login device B / refresh), `req.user` menjadi `null` → request dianggap tidak punya user.

### 3. Backend: PermissionsGuard (CASL) gagal → ForbiddenException default
**File:** `api_nest/src/modules/casl/permission.guard.ts` + `api_nest/src/modules/casl/casl-ability.factory.ts`

`PermissionsGuard.canActivate()` membangun ability via `createForUser(req.user)`. Ketika permission yang dibutuhkan (`@CheckPermissions(...)`) tidak ada di ability → guard mengembalikan `false` → NestJS melempar `ForbiddenException` default dengan message **"Forbidden resource"** (status 403).

### 4. Backend: ForbiddenErrorFilter me-redirect ke /login untuk method mutasi
**File:** `api_nest/src/filters/forbidden-error.filter.ts:15-24`

```typescript
if (
  exception instanceof HttpException &&
  exception.getStatus() == 403 &&
  exception.message === 'Forbidden resource' &&
  (method === 'POST' || method === 'PUT' || method === 'DELETE' || method === 'PATCH')
) {
  res.redirect('/login');   // ← user dilempar ke login
}
```

Inilah yang menjelaskan gejala: **aksi** (POST/PUT/DELETE/PATCH) yang gagal → langsung redirect ke `/login`. GET tidak terpengaruh (tidak di-redirect), sehingga user hanya mengeluh "saat melakukan aksi".

### 5. Frontend: refresh token menyimpan token baru tapi race dengan DB
**File:** `bbs/client-teacher/src/actions/makeApiRequest.js:158-221`

Frontend memanggil `/auth/refresh` dan menyimpan `accessToken` baru ke `localStorage`, lalu `window.location.reload()`. Jika di saat yang sama session lain melakukan login/refresh, token DB bisa saja sudah berganti lagi → token di localStorage tetap mismatch → 403 berulang.

## Kesimpulan Akar Masalah
**Race condition / invalidation session antar-device**: desain single-token-per-user (`auth.service.ts` menimpa `accessToken` di DB setiap login/refresh) dikombinasikan dengan middleware `convert-token-to-user` yang melakukan perbandingan eksak token header vs DB, menghasilkan session lama hangus secara mendadak. `ForbiddenErrorFilter` lalu mengubah kegagalan 403 pada method mutasi menjadi redirect ke `/login` — itulah kenapa user "dipaksa relogin" dan "sembuh setelah relogin" (karena login ulang menulis token baru yang cocok dengan DB).

## Saran Fix

### Option A (Recommended — Backend: stop redirect ke /login pada 403)
**File:** `api_nest/src/filters/forbidden-error.filter.ts`

Hapus redirect ke `/login` untuk 403. 403 adalah *authorization* error (user valid tapi tidak punya hak), bukan *authentication* error. Redirect hanya layak untuk 401 (token tidak valid/expired). User akan melihat pesan error yang jelas di UI, dan tidak ada session yang "hilang" mendadak.

```typescript
// Hapus blok res.redirect('/login') — biarkan exception diproses normal oleh exception filter lain
```

### Option B (Backend: multi-token / jangan timpa token lama)
**File:** `api_nest/src/modules/auth/auth.service.ts`

Jangan menyimpan satu `accessToken` per user. Opsi:
- Simpan **list token aktif** (mis. kolom JSON array atau tabel terpisah `user_access_tokens`), sehingga login di device lain tidak membatalkan session yang ada.
- **Atau** lepaskan validasi eksak `userFromDb.accessToken !== accessToken` di middleware, cukup validasi JWT signature/expiry (`AuthHelper.decode` / `JwtService.verify`).

### Option C (Backend: jaga konsistensi di PermissionsGuard)
**File:** `api_nest/src/modules/casl/permission.guard.ts` + `casl-ability.factory.ts`

Tambah guard di `createForUser` untuk menangani `req.user === null` dengan jelas (401 vs 403) dan pastikan ability dibangun dari data yang selalu fresh (tidak bergantung pada state token).

## Files Terkait
| File | Path | Peran |
|---|---|---|
| Exception Filter | `smartbag/api_nest/src/filters/forbidden-error.filter.ts` | Redirect ke /login saat 403 'Forbidden resource' pada mutasi |
| Token Middleware | `smartbag/api_nest/src/common/middleware/convert-token-to-user.middleware.ts` | Bandingkan token header vs token DB; null-kan user jika mismatch |
| Auth Service | `smartbag/api_nest/src/modules/auth/auth.service.ts` | Generate & timpa accessToken di DB setiap login/refresh |
| CASL Guard | `smartbag/api_nest/src/modules/casl/permission.guard.ts` | Guard permission → false → ForbiddenException |
| CASL Factory | `smartbag/api_nest/src/modules/casl/casl-ability.factory.ts` | Bangun ability dari UserRoleExtension + teacher roles |
| Frontend API | `smartbag/bbs/client-teacher/src/actions/makeApiRequest.js` | Refresh token & redirect ke /login saat gagal |

## Catatan untuk Engineer
- **Prioritas tinggi** karena memengaruhi semua user dan semua aksi mutasi.
- Untuk reproduksi cepat di lokal: login di dua browser dengan akun sama → aksi mutasi di browser pertama pasti 403 + redirect login.
- Verifikasi juga apakah `ForbiddenErrorFilter` terdaftar sebagai global filter di `main.ts` — pastikan fix tidak merusak penanganan 401 yang benar.
