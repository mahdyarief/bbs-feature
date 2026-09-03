# API Contract — Form Leave Workflow (Status Approval)

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> **Endpoint baru:** `PATCH /v1/leaves/:id/status` — ditambahkan ke controller `leave.controller.ts` yang sudah ada dari fase 1.
> **Response extend:** `GET /v1/leaves` dan `GET /v1/leaves/:id` dari fase 1 sekarang menyertakan field status approval.
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.LEAVE`.

## Enum Leave Status

| Enum value | Deskripsi |
|------------|-----------|
| `PENDING` | Default — belum di-review |
| `APPROVED_BY_ADMIN` | Disetujui oleh Admin (tahap 1) |
| `APPROVED_BY_PRINCIPAL` | Disetujui oleh Principal (final) |
| `REJECTED` | Ditolak (oleh Admin atau Principal) |

## State Machine (Transisi Valid)

```
PENDING  ──→ APPROVED_BY_ADMIN
PENDING  ──→ REJECTED
APPROVED_BY_ADMIN ──→ APPROVED_BY_PRINCIPAL
APPROVED_BY_ADMIN ──→ REJECTED
```

Semua transisi lain → 400 "Invalid status transition".

## Endpoint Detail

### 1. PATCH /v1/leaves/:id/status — Ubah status + komentar

**Body (UpdateTeacherLeaveStatusDto):**

```json
{
  "leaveStatus": "APPROVED_BY_ADMIN",
  "comment": "Approved. Please coordinate with substitute teacher."
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `leaveStatus` | enum | Ya | `PENDING | APPROVED_BY_ADMIN | APPROVED_BY_PRINCIPAL | REJECTED` |
| `comment` | string | Tidak (wajib saat REJECTED) | Komentar reviewer |

**Catatan:**
- `statusChangedBy` diisi otomatis dari `req.user.id` — TIDAK ada di body.
- `statusChangedAt` diisi otomatis timestamp server — TIDAK ada di body.
- `adminComment` vs `principalComment` ditentukan oleh role reviewer (Admin → `adminComment`, Principal → `principalComment`). Jika role tidak bisa ditentukan dari `req.user`, gunakan fallback: simpan di `adminComment`.
- Validasi transisi: service mengecek state machine — transisi tidak valid → 400.
- Validasi reject: jika `leaveStatus = REJECTED` dan `comment` kosong/null → 400.
- Record `activeStatus = INACTIVE` → 404.

**Response 200:**

```json
{
  "data": {
    "id": 101,
    "leaveStatus": "APPROVED_BY_ADMIN",
    "adminComment": "Approved. Please coordinate with substitute teacher.",
    "principalComment": null,
    "statusChangedBy": 586,
    "statusChangedAt": "2026-09-02T14:30:00.000Z",
    "dateFrom": "2026-09-01",
    "dateTo": "2026-09-02",
    "position": "Teacher",
    "department": "Science",
    "leaveType": "SICK",
    "reason": "Flu, need rest",
    "attachmentFileId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "employeeId": 21046,
    "campusId": 4,
    "activeStatus": "ACTIVE",
    "createdAt": "2026-08-26T10:00:00.000Z",
    "updatedAt": "2026-09-02T14:30:00.000Z"
  }
}
```

> **Catatan:** `leaveType` di request menggunakan string key (mis. `"SICK"`) yang di-transform ke numeric enum oleh `@Transform(({ value }) => LeaveTypeEnum[value])`. Response menampilkan nilai numeric (1=SICK, 2=MATERNITY, 3=PATERNITY, 4=UNPAID, 15=OTHER). `employeeId` menggantikan `teacherId` — nama guru dapat diakses melalui relasi `employee` (entity `Employee`). `attachmentFileId` bertipe uuid (string).

### 2. GET /v1/leaves — List (extend fase 1)

Response fase 1 ditambah field status approval:

```json
{
  "data": [
    {
      "id": 101,
      "dateFrom": "2026-09-01",
      "dateTo": "2026-09-02",
      "position": "Teacher",
      "department": "Science",
      "leaveType": "SICK",
      "reason": "Flu, need rest",
      "attachmentFileId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "attachment": { "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890", "url": "/api/v1/files/download?id=a1b2c3d4-e5f6-7890-abcd-ef1234567890" },
      "employeeId": 21046,
      "campusId": 4,
      "activeStatus": "ACTIVE",
      "leaveStatus": "APPROVED_BY_ADMIN",
      "adminComment": "Approved.",
      "principalComment": null,
      "statusChangedBy": 586,
      "statusChangedAt": "2026-09-02T14:30:00.000Z",
      "createdAt": "2026-08-26T10:00:00.000Z",
      "updatedAt": "2026-09-02T14:30:00.000Z"
    }
  ],
  "count": 3,
  "meta": {
    "page": 1,
    "pageSize": 10,
    "itemCount": 3,
    "pageCount": 1,
    "hasPreviousPage": false,
    "hasNextPage": false
  }
}
```

### 3. GET /v1/leaves/:id — Detail (extend fase 1)

Response fase 1 ditambah field status approval (sama seperti di atas, `leaveStatus`, `adminComment`, `principalComment`, `statusChangedBy`, `statusChangedAt`).

## Referensi Implementasi

- State machine pattern: validasi transisi via map object `{ [from: string]: string[] }` — contoh:
  ```ts
  const VALID_TRANSITIONS: Record<LeaveStatusEnum, LeaveStatusEnum[]> = {
    [LeaveStatusEnum.PENDING]: [LeaveStatusEnum.APPROVED_BY_ADMIN, LeaveStatusEnum.REJECTED],
    [LeaveStatusEnum.APPROVED_BY_ADMIN]: [LeaveStatusEnum.APPROVED_BY_PRINCIPAL, LeaveStatusEnum.REJECTED],
    [LeaveStatusEnum.APPROVED_BY_PRINCIPAL]: [],
    [LeaveStatusEnum.REJECTED]: [],
  };
  ```
- Controller pattern: `src/modules/teacher-leave/leave.controller.ts` (file existing yang di-extend) — `@Patch(':id/status')`, `@CheckPermissions([{ action: ACLTypeEnum.UPDATE, subject: ModulesTypeEnum.LEAVE }])`, `@Req() req: Request`, `ParseIntPipe`.
- File upload: tidak ada perubahan — tetap menggunakan endpoint `POST /v1/files` dari fase 1.