---
status: DRAFT
feature: National Report Card
slug: national-report
---

# API Contract — National Report Card

## Konvensi

- **Base path:** `/api/v1/national-reports`
- **Format response:** `{ data: { ... } }` untuk sukses, `{ errors: [...] }` untuk error
- **Auth:** Semua endpoint memerlukan session/authentication (JWT/session)
- **Permission:** SUBJECT_MODULES `national-report`

---

## Endpoints

### GET /api/v1/class-in-year

Daftar class per Academic Year. Endpoint ini merupakan prasyarat — digunakan untuk mendapatkan `classId` yang akan dipakai di endpoint National Report.

**Query Parameters:**

| Parameter | Tipe | Required | Deskripsi |
|-----------|------|----------|-----------|
| `academicYearId` | integer | Yes | ID academic year (contoh: 25 = 2024/2025) |
| `campusId` | integer | No | Filter campus (default: semua campus sesuai role) |

**Response 200:**

```json
{
  "data": [
    {
      "id": 2005,
      "academicYearId": 25,
      "academicYear": "2024/2025",
      "campusId": 2,
      "campusName": "KJ-S",
      "level": "Secondary 3 Accelerated",
      "code": "Sec3Acc",
      "name": "Secondary 3 Taylor",
      "homeroomTeacher": "Brian Thomas Houston",
      "studentCount": 24
    },
    {
      "id": 2011,
      "academicYearId": 25,
      "academicYear": "2024/2025",
      "campusId": 2,
      "campusName": "KJ-S",
      "level": "Junior College 2",
      "code": "JC2",
      "name": "Junior College 2 Mendel",
      "homeroomTeacher": "Erika Ester",
      "studentCount": 21
    }
  ],
  "meta": {
    "total": 12,
    "page": 1,
    "pageSize": 50
  }
}
```

---

### GET /api/v1/national-reports/{classId}/pdf

Generate dan download PDF Nasional Report Card (SKL + Transkrip) untuk satu class.

**Path Parameters:**

| Parameter | Tipe | Required | Deskripsi |
|-----------|------|----------|-----------|
| `classId` | integer | Yes | ID class (contoh: 2005 untuk Secondary 3 Taylor) |

**Query Parameters:**

| Parameter | Tipe | Required | Deskripsi |
|-----------|------|----------|-----------|
| `version` | string | No | `new` (default) atau `old` untuk varian format lama |

**Response 200:** File PDF binary (`application/pdf`).

**Response Headers:**
```
Content-Type: application/pdf
Content-Disposition: attachment; filename="national-report-card-class-2005-ay-2024-2025.pdf"
```

**Response 404 (class not found):**
```json
{
  "errors": [
    {
      "code": "CLASS_NOT_FOUND",
      "message": "Class with id 9999 not found"
    }
  ]
}
```

**Response 422 (no grade data):**
```json
{
  "errors": [
    {
      "code": "NO_GRADE_DATA",
      "message": "No grade data available for class 2005 in academic year 2024/2025"
    }
  ]
}
```

---

## Struktur PDF Output

Setiap PDF Nasional Report Card berisi halaman per siswa secara berurutan:

### Halaman 1: Surat Keterangan Lulus (SKL)

| Field | Deskripsi | Contoh |
|-------|-----------|--------|
| Nomor SKL | Format: `{nomor}/CJCE/{campus}/{tahun}` | 002/CJCE/KJ/2025 |
| Nama Lengkap | Nama siswa | Audrey Tiara Gozali |
| Tempat, Tanggal Lahir | TTL siswa | Jakarta, 21 November 2008 |
| NISN | Nomor Induk Siswa Nasional | 0085608038 |
| Satuan Pendidikan | Nama sekolah | SMA Bina Bangsa School Kebon Jeruk |
| NPSN | Nomor Pokok Sekolah Nasional | 20104408 |
| Tanggal Kelulusan | Tanggal lulus | 4 Mei 2026 |
| Dinyatakan | Status kelulusan | LULUS |
| Nilai | 3 mapel: Pendidikan Agama, PPKN, Bahasa Indonesia | 90.00, 90.00, 90.67 |
| Rata-rata | Rata-rata nilai | 90.22 |
| Kepala Sekolah | Nama + gelar | Richard, M.Fin, MBS |

### Halaman 2: Transkrip Nilai

| Field | Deskripsi | Contoh |
|-------|-----------|--------|
| Nomor Transkrip | Format: `{nomor}/TN/SMA/{campus}/{tahun}` | 002/TN/SMA/KJ/2025 |
| Nama Lengkap | Nama siswa | Audrey Tiara Gozali |
| TTL | Tempat tanggal lahir | Jakarta, 21 November 2008 |
| NISN | NISN | 0085608038 |
| Nomor Ijazah | Nomor ijazah | 133202500008747 |
| Tanggal Kelulusan | Tanggal lulus | 4 Mei 2026 |
| Peminatan | Jurusan/peminatan | (IPA/IPS) |
| Nilai (13 mapel) | Agama, PPKN, B.Indo, B.Ing, B.Mandarin, Matematika, Biologi, Kimia, Komputer, Fisika, Keterampilan Umum, Penjas, Seni & Prakarya | 90.00, 90.00, 90.67, ... |
| Rata-rata | Rata-rata 13 mapel | 86.50 |
| Kepala Sekolah | Tanda tangan | Richard, M.Fin, MBS |

---

## Mapping Legacy Field → API Field

### SKL

| Legacy (PHP) | API Field | Keterangan |
|-------------|-----------|------------|
| `nomor_skl` | `certificateNumber` | Nomor SKL |
| `nama` | `studentName` | Nama siswa |
| `ttl` | `birthPlace`, `birthDate` | Tempat, tanggal lahir |
| `nisn` | `nisn` | NISN |
| `satuan_pendidikan` | `schoolName` | Satuan pendidikan |
| `npsn` | `npsn` | NPSN |
| `tanggal_kelulusan` | `graduationDate` | Tanggal kelulusan |
| `dinyatakan` | `status` | Status (LULUS) |
| `nilai` | `subjects[]` | Array mapel + nilai |
| `rata_rata` | `average` | Rata-rata nilai |

### Transkrip

| Legacy (PHP) | API Field | Keterangan |
|-------------|-----------|------------|
| `nomor_transkrip` | `transcriptNumber` | Nomor transkrip |
| `nomor_ijazah` | `diplomaNumber` | Nomor ijazah |
| `peminatan` | `major` | Peminatan/jurusan |
| `nilai` | `subjects[]` | Array 13 mapel + nilai |
| `rata_rata` | `average` | Rata-rata |