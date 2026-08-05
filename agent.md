---
title: Agent Guide — Feature Requirement Briefs
slug: agent-guide-feature-requirement
status: active
author: BBS Team
date: 2026-07-31
---

# Agent Guide: Feature Requirement Briefs

> Panduan untuk AI Agent dalam membuat **feature requirement briefs** di folder `features/`.
> **Tujuan: membuat dokumen spesifikasi — bukan mengubah kode.**

---

## 1. Tujuan

Membuat dokumen spesifikasi fitur (feature brief) yang **jelas, terstruktur, dan siap pakai oleh engineer** untuk implementasi. Agent hanya menulis dokumen di `features/{feature-slug}/`, tidak mengubah kode di `api_nest/` atau `bbs/`.

### Jenis Brief yang Dibuat

1. **New Feature** — fitur baru yang belum ada sebelumnya
2. **Feature Changes** — perubahan/modifikasi pada fitur yang sudah ada (contoh: enable/disable field, ubah filter logic, tambah parameter)
3. **Bug Fixing** — perbaikan bug dengan penjelasan root cause, repro steps, dan solusi

### Prinsip Utama

1. **No code changes** — jangan edit, refactor, atau meminta perubahan di `api_nest/` atau `bbs/`
2. **Write only in `features/`** — semua output ada di `features/{feature-slug}/`
3. **Engineer-ready** — spesifikasi harus cukup detail untuk langsung diimplementasi
4. **Traceable** — setiap keputusan dan aturan bisnis harus terdokumentasi dengan rationale

---

## 2. Project Context (Wajib Dipahami)

### Backend: `api_nest/` (NestJS + TypeORM)

| Aspek | Detail |
|-------|--------|
| Framework | NestJS (Node.js) |
| Language | TypeScript |
| ORM | TypeORM (migration-based) |
| Database | PostgreSQL (dari migration history) |
| Struktur | Modular — setiap fitur punya module folder di `src/modules/{feature}/` |
| Migration | File di `src/database/migrations/` dengan prefix timestamp |
| Pattern | Controller → Service → Repository/Entity |

**Module umum yang sering dirujuk:**
- `academic-year`, `student`, `attendance`, `grade`, `threshold`, `billing`, `sve`, `cca`, `class-year`, `campus`, `campus-gate`, `late-time-setting`, `daily-attendance`, `syllabus`, `threshold-config`, `secondary-grade`, `secondary-gpa-grading-scale`

### Frontend: `bbs/` (React — Multi Portal)

| Aspek | Detail |
|-------|--------|
| Framework | React (JSX) |
| Portals | 3 portal terpisah |
| State | Redux (actions, reducers, selectors, store) |
| Routing | Sentral di `client/src/routes.js` |
| UI Kit | Komponen BBS prefixed (`BBSSelect`, `BBSTable`, dll) |
| Styling | SCSS |

**3 Portal:**

| Portal | Path | Audience |
|--------|------|----------|
| Admin | `client/` | Back-office admin |
| Student | `client-student/` | Student portal |
| Teacher | `client-teacher/` | Teacher portal |

**Struktur FE per portal:**
- `containers/` — halaman/view utama (biasanya 1 container = 1 route)
- `views/` — sub-view atau komponen spesifik halaman
- `components/` — reusable components (BBS UI kit)
- `actions/` — Redux actions
- `reducers/` — Redux reducers
- `constants/` + `enums.js` — konstanta dan enum aplikasi
- `utils/` — utility functions

---

## 3. Output Structure

Setiap feature brief adalah folder di `features/{feature-slug}/` dengan format:

```
features/{feature-slug}/
├── spec.md              # WAJIB — spesifikasi utama
├── edgecases.md         # WAJIB — edge cases & keputusan
├── api-contract.md      # OPSIONAL — detail endpoint API
├── schema.md            # OPSIONAL — perubahan database
└── notes.md             # OPSIONAL — catatan, pertanyaan terbuka
```

### Naming Convention

- **Folder slug**: `kebab-case` (contoh: `gpa-threshold-config`, `external-attendance-integration`)
- **Files**: nama file tetap seperti di atas

---

## 4. Template Referensi

### `spec.md` — Wajib diisi lengkap

Template ada di `_templates/spec-template.md`. Bagian yang harus diisi:

| Bagian | Deskripsi |
|--------|-----------|
| **Frontmatter** | `feature`, `slug`, `status: draft`, `author`, `date`, `target_release` |
| **Overview** | 1 paragraf ringkasan fitur |
| **Problem / Motivation** | Masalah yang dipecahkan (latar belakang, konteks) |
| **Scope (In/Out)** | Batasan fitur — apa yang termasuk dan tidak |
| **User Stories** | Format: *As a {role}, I want to {action}, so that {benefit}* |
| **Acceptance Criteria** | Checklist kondisi yang harus terpenuhi |
| **UI/UX Changes** | Portal affected, deskripsi perubahan UI, wireframe jika ada |
| **API Changes** | Tabel endpoint: Method, Path, Description |
| **Database Changes** | Tabel baru/modified, migrasi yang diperlukan |
| **Business Rules** | Aturan bisnis detail, validasi logic |
| **Error Handling** | Tabel error: Error, HTTP Code, Message |
| **Dependencies** | Module/modul existing yang diperlukan |

### `edgecases.md` — Wajib diisi

Template ada di `_templates/edgecases-template.md`. Format per edge case:

```
## EC-01: {Judul Edge Case}
**Scenario:** {Skenario}

| Opsi | Behavior |
|------|----------|
| (A) {Opsi A} | {Apa yang terjadi} |
| (B) {Opsi B} | {Apa yang terjadi} |
| (C) {Opsi C} | {Apa yang terjadi} |

**Decision:** _TBD_
```

### Contoh existing:

Lihat folder:
- `features/gpa-threshold-config/spec.md` — contoh spesifikasi teknis
- `features/external-attendance-integration/spec.md` — contoh spesifikasi API eksternal

---

## 5. Workflow Pembuatan Feature Brief

### Step 1: Clarify & Discover

Sebelum menulis, kumpulkan informasi dengan cara:

1. **Tanya ke user/stakeholder** — pahami problem, bukan solusi
2. **Explore codebase** — cari module terkait di `api_nest/src/modules/` dan container terkait di `bbs/client/src/containers/`
3. **Cek migration history** — lihat pola tabel dan relasi yang sudah ada
4. **Cek existing features** — apakah sudah ada spesifikasi terkait sebelumnya

### Step 2: Draft Spec

Tulis `spec.md` dengan urutan:
1. **Overview + Problem** — pastikan orang lain paham konteks
2. **Scope** — tegas batasan in/out
3. **User Stories + Acceptance Criteria** — ini yang diuji nanti
4. **Detail teknis** — API, DB, business rules
5. **Error handling** — antisipasi failure mode

### Step 3: Draft Edge Cases

Untuk setiap fitur, identifikasi minimal 3-5 edge cases:
- **Data boundary**: null, empty, max length
- **State transition**: status A → B → C
- **Concurrency**: duplicate submission, race condition
- **Permission**: akses role berbeda
- **Integration**: downstream service down, invalid response

Presentasi dalam format opsi (A/B/C) + **Decision: TBD** untuk didiskusikan dengan tim.

### Step 4: Review & Finalize

- Tandai `status: draft` di frontmatter
- Jika perlu informasi tambahan, tulis di `notes.md`
- Jika ada API contract detail, buat `api-contract.md`
- Jika ada schema change signifikan, buat `schema.md`

### Step 5: Commit & Push

Setelah spec selesai direview dan final:

1. **Pastikan berada di folder `features/`** — git repo ini terpisah dari `api_nest/` dan `bbs/`
2. **Stage file** yang relevan: `git add {feature-slug}/`
3. **Commit** dengan pesan yang jelas: `git commit -m "add {feature-slug}: {deskripsi singkat}"`
4. **Pull** jika ada perubahan remote: `git pull --rebase`
5. **Push** ke remote: `git push`

> **Catatan:** Folder `features/` adalah git repo sendiri (`origin/main`). Jangan commit dari folder root project (`api_nest/` atau `bbs/`). Selalu `cd features/` dulu sebelum commit & push.

---

## 6. Kualitas Feature Brief

### Checklist Kualitas

- [ ] Problem jelas: stakeholder bisa jelaskan "kenapa fitur ini penting"
- [ ] Scope tegas: in/out dibedakan dengan jelas
- [ ] Acceptance criteria testable: bisa diverifikasi pass/fail
- [ ] Business rules detailed: tidak ada asumsi yang tidak tertulis
- [ ] Edge cases teridentifikasi: skenario gagal sudah diantisipasi
- [ ] Error handling: pengguna dapat pesan error yang meaningful
- [ ] Dependencies terdaftar: module/API/modul yang diperlukan diketahui
- [ ] API contract spesifik: method, path, request/response format
- [ ] Database changes jelas: tabel, kolom, relasi, migration

### Hal yang Harus Dihindari

| ❌ Jangan | ✅ Lakukan |
|-----------|------------|
| Langsung menulis solusi teknis tanpa memahami problem | Tanya "kenapa" sampai problem inti tergali |
| Membuat asumsi tentang implementasi | Tulis business rules, biarkan engineer menentukan implementasi |
| Melewatkan edge cases | Minimal identifikasi 3-5 edge case per fitur |
| Bahasa ambigu ("mungkin", "sebaiknya") | Gunakan bahasa definitif ("harus", "wajib", "akan") |
| Merge beberapa konsep dalam satu fitur | Pisahkan jadi feature brief terpisah jika terlalu besar |
| Menentukan UI detail tanpa konteks | Cukup deskripsi fungsional, wireframe jika perlu |

---

## 7. Codebase Navigation

### Mencari Module BE

```
api_nest/src/modules/{module-name}/
  ├── {module-name}.controller.ts     # Endpoint API
  ├── {module-name}.service.ts        # Business logic
  ├── {module-name}.module.ts         # Module definition
  └── {module-name}.entity.ts         # TypeORM entity
```

Cara: `ls api_nest/src/modules/` untuk daftar semua module.

### Mencari Container FE (per portal)

```
bbs/client/src/containers/{feature-name}/
  ├── index.jsx                       # Halaman utama
  ├── components/                     # Komponen spesifik halaman
  └── actions.js / reducer.js         # Redux (jika ada)
```

Cara: `ls bbs/client/src/containers/` → cari nama fitur terkait.

### Mencari Migration

```
api_nest/src/database/migrations/{timestamp}-{description}.ts
```

Migration berisi perubahan skema (CREATE TABLE, ALTER TABLE, dll). Berguna untuk memahami struktur tabel existing.

---

## 8. Prompt Engineering Notes

Saat menggunakan AI agent untuk **membantu menulis** feature brief:

1. **Provide context dulu**: jelaskan codebase structure secara singkat
2. **Minta draft setelah problem clear**: jangan minta AI langsung nulis tanpa problem understanding
3. **Iterate on edge cases**: AI bisa bantu brainstorm edge case yang terlewat
4. **Validate against templates**: minta AI periksa apakah semua bagian template terisi
5. **Review untuk ambiguitas**: minta AI tandai bagian yang ambigu atau kurang detail

---

## 9. Quick Start

Untuk membuat feature brief baru:

```
features/{feature-slug}/
├── spec.md           # Copy dari _templates/spec-template.md, isi lengkap
├── edgecases.md      # Copy dari _templates/edgecases-template.md, isi minimal 3-5 EC
├── api-contract.md   # Buat jika ada endpoint API baru
├── schema.md         # Buat jika ada perubahan DB signifikan
└── notes.md          # Buat untuk mencatat pertanyaan terbuka
```

### Frontmatter Template

```yaml
---
feature: Nama Fitur
slug: nama-fitur
status: draft
author: BBS Team
date: 2026-07-31
target_release: TBD
---
```

---

## 10. Database Access — `db-access-tools/`

Setiap kali user meminta akses ke database binabangsa, **selalu gunakan tools di folder `db-access-tools/`**.

| Aspek | Detail |
|-------|--------|
| Path | `/Users/arielwirawan/Documents/Gawe/db-access-tools/` |
| Koneksi | Konfigurasi di `db-access-tools/.env` dan `db-access-tools/env.env` |
| Module | `db-access-tools/db/` — berisi script Python untuk koneksi, query, schema, dll |
| Docs | `db-access-tools/docs/` — overview database, enum mappings, import rules |
| Schema | `db-access-tools/SCHEMA_OVERVIEW.md` — ringkasan skema database |
| Result | `db-access-tools/result/` — output query (JSON, CSV) |

### Cara Pakai

1. **Baca dokumentasi dulu**: `db-access-tools/docs/database_overview.md` dan `db-access-tools/SCHEMA_OVERVIEW.md` untuk memahami struktur database
2. **Cek koneksi**: lihat `db-access-tools/.env` untuk kredensial
3. **Jalankan query**: gunakan module di `db-access-tools/db/` (Python) untuk menjalankan query, dump schema, atau generate ERD
4. **Hasil output**: disimpan di `db-access-tools/result/`

### Struktur Module `db/`

| File | Fungsi |
|------|--------|
| `connection.py` | Koneksi ke database |
| `query.py` / `data.py` | Eksekusi query & ambil data |
| `schema.py` / `ddl.py` | Inspect skema database |
| `dump.py` | Dump data/schema |
| `erd.py` | Generate Entity Relationship Diagram |
| `diff_schema.py` | Bandingkan skema |
| `sql_gen.py` | Generate SQL |
| `agg.py` | Agregasi data |

> **Catatan:** Jangan pernah hardcode kredensial database. Selalu baca dari file `.env` di folder `db-access-tools/`.

---

## 11. Referensi

- **Template spec**: `_templates/spec-template.md`
- **Template edge cases**: `_templates/edgecases-template.md`
- **Contoh spec**: `features/gpa-threshold-config/spec.md`
- **Contoh edge cases**: `features/gpa-threshold-config/edgecases.md`
- **Contoh spec API eksternal**: `features/external-attendance-integration/spec.md`
- **README index**: `features/README.md`
