# API Contract — Form Leave Workflow (Approval & Teacher Leave Management)

> Status: APPROVED — diselaraskan langsung dengan controller `api_nest/src/modules/teacher-leave/leave.controller.ts` (NestJS 10, `@Controller({ version: '1', path: 'leaves' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.LEAVE`.
> Base path: global prefix `api` → URL lengkap `/api/v1/leaves`.

---

## Ringkasan Endpoint

| Method | Endpoint | Deskripsi | Permission | Akses Role |
|--------|----------|-----------|------------|------------|
| `GET` | `/v1/leaves` | List pengajuan cuti guru (paginated, filter, ordering Canceled terbawah) | `READ` | Principal (Teacher Portal), Admin (Admin Portal - View Only), Guru (milik sendiri) |
| `GET` | `/v1/leaves/:id` | Detail permohonan cuti (termasuk status, lampiran, komentar reviewer) | `READ` | Principal, Admin, Guru pemohon |
| `PATCH` | `/v1/leaves/:id/status` | Update status permohonan (`Approved Paid/Unpaid`, `Declined`, `Canceled`) & komentar review | `UPDATE` | Principal / Vice Principal |
| `POST` | `/v1/leaves` | Create form permohonan cuti baru (trigger email ke Principal) | `CREATE` | Guru (Teacher Portal) |
| `DELETE` | `/v1/leaves/:id` | Soft delete cuti guru (hanya draft/pending sebelum diproses) | `DELETE` | Guru pemohon |

---

## 1. GET /v1/leaves — List Form Leave (dengan Custom Ordering Canceled di Terbawah)

Mengambil daftar permohonan cuti. Pada Teacher Portal (menu `Form Leave Teacher`), Principal melihat pengajuan cuti seluruh guru di unit/campus-nya.  
**Aturan Ordering Khusus:** Seluruh record dengan `leaveStatus = 'CANCELED'` selalu berada di posisi paling bawah, terlepas dari tanggal pembuatannya.

### Query Parameters:
- `page`: int (default 1)
- `limit`: int (default 10)
- `campusId`: int (opsional)
- `teacherId`: int (opsional)
- `leaveType`: int (opsional: 1=SICK, 2=MATERNITY, 3=PATERNITY, 4=UNPAID, 15=OTHER)
- `leaveStatus`: string (opsional: `PENDING`, `APPROVED_UNPAID`, `APPROVED_PAID`, `DECLINED`, `CANCELED`)
- `year`: int (opsional, contoh `2026`)

### Response 200 OK:

```json
{
  "data": [
    {
      "id": 520,
      "employeeId": 21046,
      "campusId": 3,
      "dateFrom": "2026-09-10",
      "dateTo": "2026-09-12",
      "position": "Teacher",
      "department": "Science Department",
      "leaveType": 1,
      "reason": "High fever and medical rest advised by doctor",
      "attachmentFileId": "6c2cfb4d-1a2b-4c3d-8e9f-0a1b2c3d4e5f",
      "activeStatus": "1",
      "leaveStatus": "PENDING",
      "reviewerComment": null,
      "statusChangedBy": null,
      "statusChangedAt": null,
      "createdAt": "2026-09-04T08:30:00.000Z",
      "updatedAt": "2026-09-04T08:30:00.000Z",
      "employee": {
        "id": 21046,
        "fullName": "Alexzandra Thenu",
        "email": "alexzandra.thenu@binabangsaschool.com"
      },
      "attachmentFile": {
        "id": "6c2cfb4d-1a2b-4c3d-8e9f-0a1b2c3d4e5f",
        "name": "Surat Keterangan Dokter.pdf",
        "url": "https://storage.binabangsaschool.com/files/surat-dokter-520.pdf"
      }
    },
    {
      "id": 515,
      "employeeId": 20411,
      "campusId": 3,
      "dateFrom": "2026-09-01",
      "dateTo": "2026-09-01",
      "position": "Teacher",
      "department": "Primary",
      "leaveType": 1,
      "reason": "Gastric acid symptoms",
      "attachmentFileId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
      "activeStatus": "1",
      "leaveStatus": "APPROVED_PAID",
      "reviewerComment": "Approved. Medical certificate verified.",
      "statusChangedBy": 502,
      "statusChangedAt": "2026-09-01T09:15:00.000Z",
      "createdAt": "2026-09-01T07:10:00.000Z",
      "updatedAt": "2026-09-01T09:15:00.000Z",
      "employee": {
        "id": 20411,
        "fullName": "Sarah Widyaningtyas",
        "email": "sarah.widyaningtyas@binabangsaschool.com"
      },
      "statusChangedByUser": {
        "id": 502,
        "fullName": "Linawati Lauw"
      }
    },
    {
      "id": 498,
      "employeeId": 19820,
      "campusId": 3,
      "dateFrom": "2026-08-20",
      "dateTo": "2026-08-21",
      "position": "Teacher",
      "department": "Aesthetic",
      "leaveType": 4,
      "reason": "Personal urgent family matter",
      "attachmentFileId": null,
      "activeStatus": "1",
      "leaveStatus": "CANCELED",
      "reviewerComment": "Canceled upon teacher request due to rescheduled event.",
      "statusChangedBy": 502,
      "statusChangedAt": "2026-08-20T10:00:00.000Z",
      "createdAt": "2026-08-19T14:00:00.000Z",
      "updatedAt": "2026-08-20T10:00:00.000Z",
      "employee": {
        "id": 19820,
        "fullName": "Budi Santoso",
        "email": "budi.santoso@binabangsaschool.com"
      },
      "statusChangedByUser": {
        "id": 502,
        "fullName": "Linawati Lauw"
      }
    }
  ],
  "count": 45,
  "meta": {
    "page": 1,
    "limit": 10,
    "itemCount": 45,
    "pageCount": 5,
    "hasPreviousPage": false,
    "hasNextPage": true
  }
}
```

---

## 2. PATCH /v1/leaves/:id/status — Update Status & Reviewer Comment

Dipanggil saat Principal mengubah status cuti melalui dropdown pada menu `Form Leave Teacher` di Teacher Portal.  
Otomatis mencatat `statusChangedBy` dari ID reviewer login, memperbarui `statusChangedAt`, dan memicu job antrean email ke guru pemohon.

### Request Body (`UpdateTeacherLeaveStatusDto`):

```json
{
  "leaveStatus": "APPROVED_PAID",
  "comment": "Approved as paid sick leave. Get well soon."
}
```

| Field | Type | Required | Deskripsi |
|-------|------|----------|-----------|
| `leaveStatus` | enum string | Ya | Pilihan: `PENDING`, `APPROVED_UNPAID`, `APPROVED_PAID`, `DECLINED`, `CANCELED` |
| `comment` | string | Kondisional | Catatan review Principal. **Wajib** diisi jika `leaveStatus` adalah `DECLINED` atau `CANCELED`; opsional saat `APPROVED`. |

### Response 200 OK:

```json
{
  "data": {
    "id": 520,
    "employeeId": 21046,
    "campusId": 3,
    "dateFrom": "2026-09-10",
    "dateTo": "2026-09-12",
    "leaveType": 1,
    "leaveStatus": "APPROVED_PAID",
    "reviewerComment": "Approved as paid sick leave. Get well soon.",
    "statusChangedBy": 502,
    "statusChangedAt": "2026-09-04T09:00:00.000Z",
    "activeStatus": "1",
    "updatedAt": "2026-09-04T09:00:00.000Z"
  }
}
```

### Validasi & Error:
- `400 Bad Request`: `Comment is required when declining or canceling a leave request` (bila status `DECLINED`/`CANCELED` tapi comment kosong).
- `403 Forbidden`: `Only Principal or Vice Principal can update leave status` (bila user login bukan Principal/VP).
- `404 Not Found`: `Teacher leave not found or inactive` (bila ID tidak ditemukan atau `activeStatus = 0`).

---

## 3. Integrasi Email Notification Service (Bull Queue: `SEND_EMAIL`)

### Trigger 1: Pengiriman Baru (`POST /v1/leaves`)
- **Sender:** Sistem AIS (`noreply@binabangsaschool.com`) atas nama Guru Pemohon.
- **Recipient:** Alamat email Principal / VP kampus terkait (`principal.email`).
- **Subject:** `[AIS Form Leave] New Leave Request Submitted - {Employee Name}`
- **Payload Data Queue:**
  ```json
  {
    "process": "LEAVE_SUBMITTED_NOTIFICATION",
    "recipient": "principal.pik@binabangsaschool.com",
    "context": {
      "leaveId": 520,
      "employeeName": "Alexzandra Thenu",
      "campusName": "PIK-P",
      "dateFrom": "2026-09-10",
      "dateTo": "2026-09-12",
      "leaveType": "Sick Leave",
      "reason": "High fever and medical rest advised by doctor",
      "hasAttachment": true
    }
  }
  ```

### Trigger 2: Perubahan Status (`PATCH /v1/leaves/:id/status`)
- **Sender:** Sistem AIS (`noreply@binabangsaschool.com`).
- **Recipient:** Email guru pemohon (`employee.email`).
- **Subject:** `[AIS Form Leave] Your Leave Request Status: {New Status}`
- **Payload Data Queue:**
  ```json
  {
    "process": "LEAVE_STATUS_CHANGED_NOTIFICATION",
    "recipient": "alexzandra.thenu@binabangsaschool.com",
    "context": {
      "leaveId": 520,
      "employeeName": "Alexzandra Thenu",
      "newStatus": "Approve ( paid leave )",
      "reviewerName": "Linawati Lauw",
      "reviewerComment": "Approved as paid sick leave. Get well soon.",
      "dateFrom": "2026-09-10",
      "dateTo": "2026-09-12"
    }
  }
  ```
