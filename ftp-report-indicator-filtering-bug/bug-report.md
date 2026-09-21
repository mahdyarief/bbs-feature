---
name: ftp-report-indicator-filtering-bug
description: FTP Report hanya menampilkan 8 dari 12 indicator Character/Trait/Values karena filtering tidak terlihat
type: bug

---

# Bug Report: FTP Report Character/Trait/Values Indicator Tidak Lengkap

## Informasi Umum

- **Severity:** major
- **Module:** FTP Report / EBadge
- **Environment:** Production (academic_year 2026/2027)
- **Term:** 1

## Deskripsi Masalah

Pada FTP Report / EBadge score / Ebadge grade, indicator Character/Trait/Values untuk siswa Gabriel Hasan hanya menampilkan 8 indicator dari total 12 indicator yang seharusnya ada. Sementara itu, siswa Ethan Gabriel Koo menampilkan semua 12 indicator.

## Root Cause

Berdasarkan investigasi database dan codebase smartbag (`api_nest`), ditemukan beberapa temuan kunci:

### 1. Data di Database Lengkap (12 indicator)
Kedua siswa memiliki 12 indicator di tabel `student_ftp_conduct`:

| ftp_report_table_id | Gabriel Hasan (rows with badges) | Ethan Gabriel Koo (rows with badges) |
|---|---|---|
| 1 | 4 | 5 |
| 2 | 5 | 4 |
| 3 | 5 | 5 |
| 4 | 3 | 3 |
| 5 | 4 | 4 |
| **6** | **1** | **5** |
| **7** | **2** | **5** |
| **8** | **3** | **4** |
| 9 | 4 | 4 |
| **10** | **2** | **4** |
| 11 | 5 | 5 |
| 12 | 5 | 5 |

### 2. Template Tidak Ada Filtering
File `api_nest/src/templates/student-reports/student-ftp-report.hbs` hanya melakukan iterasi sederhana:
```handlebars
{{#each studentFtps}}
  <tr>
    <td>{{inc @index}}</td>
    <td>{{{character}}}</td>
    <td>{{{description}}}</td>
    <td>{{{conduct}}}</td>
  </tr>
{{/each}}
```

Template ini **tidak memiliki filtering** berdasarkan jumlah badges atau completeness data.

### 3. Tabel ftp_report_table memiliki kolom isTermXActive yang Tidak Dipakai
Entity `FtpReportTable` (`api_nest/src/modules/ftp-report-table/entities/ftp-report-table.entity.ts`) memiliki kolom:
- `is_term1_active`, `is_term2_active`, `is_term3_active`, `is_term4_active`

Namun kolom-kolom ini **tidak digunakan** untuk filtering di `ftp-report.helper.ts`.

### 4. Kemungkinan Penyebab Filtering
Karena template tidak filter, filtering kemungkinan terjadi di:
- **Service layer** (`student-ftp-conduct.service.ts` atau `student-ftp-conduct-v2.service.ts`)
- **Controller** yang memanggil service
- **Logic di `ftp-report.helper.ts`** sebelum mengirim data ke template

Indicator yang "hilang" dari Gabriel Hasan (id 6, 7, 8, 10) memiliki pola:
- Sangat sedikit rows dengan badges (1-3 rows dari 5 rows total)
- Mungkin ada threshold minimum data yang tidak terpenuhi

## Evidence

### Database Query Hasil
```sql
-- Gabriel Hasan: indicator 6 hanya 1 row dengan badges
SELECT ftp_report_table_id, COUNT(*) as total_rows, 
       COUNT(CASE WHEN badges > 0 THEN 1 END) as rows_with_badges
FROM student_ftp_conduct 
WHERE student_report_id = 441 AND term = 1 
GROUP BY ftp_report_table_id;

-- Hasil indicator 6: total_rows=5, rows_with_badges=1
```

### Perbandingan Visual
- **Gabriel Hasan**: 8 indicator muncul di report print
- **Ethan Gabriel Koo**: 12 indicator muncul di report print

## Actual Result
FTP Report untuk Gabriel Hasan hanya menampilkan 8 indicator Character/Trait/Values di term 1, padahal di database tercatat 12 indicator. 4 indicator yang tidak muncul: Diligence (6), Persistence (7), Self-Awareness (8), Task-on-Time (10).

## Expected Result
Semua 12 indicator Character/Trait/Values harus muncul di FTP Report untuk semua siswa, terlepas dari jumlah badges/rows yang dimiliki. Jika ada indicator yang memang tidak diisi, seharusnya tetap muncul dengan status "Not Rated" atau kosong, bukan dihilangkan sama sekali.

## Files yang Terkait
- `smartbag/api_nest/src/templates/student-reports/student-ftp-report.hbs`
- `smartbag/api_nest/src/helpers/reports/ftp-report.helper.ts`
- `smartbag/api_nest/src/modules/student-ftp-conduct/entities/student-ftp-conduct.entity.ts`
- `smartbag/api_nest/src/modules/ftp-report-table/entities/ftp-report-table.entity.ts`
- `smartbag/api_nest/src/modules/student-ftp-conduct/v2/student-ftp-conduct-v2.service.ts`

## Saran Perbaikan
1. **Tidak filter indicator berdasarkan completeness data** - Semua indicator harus muncul di report
2. **Jika ada alasan business untuk filtering**, tambahkan dokumentasi dan pastikan konsisten
3. **Tambahkan logging** untuk indicator yang di-filter keluar untuk keperluan debugging
4. **Pertimbangkan menggunakan `is_term1_active`** dari `ftp_report_table` sebagai kontrol tampilan yang eksplisit
