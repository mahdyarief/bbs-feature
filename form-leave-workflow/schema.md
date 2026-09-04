# Schema — Form Leave Workflow (Approval & Teacher Leave Management)

> Status: APPROVED — diselaraskan langsung dengan entitas eksisting di `api_nest/src/modules/teacher-leave/entities/leave.entity.ts`, alur Principal Portal (`ais_legacy/principals_tool`), dan modul mailer Bull Queue (`api_nest/src/modules/mailer/`).
> Base entity `src/common/base.entity` menyediakan `id`, `createdAt`, `updatedAt`, `deletedAt` (soft delete).

## Modified Entity: `Leave` (table `leave`)

Entitas utama permohonan izin/cuti guru. Pada fase workflow ini, kolom status, komentar review, dan audit trail peninjau ditambahkan ke tabel `leave`.

### Enum Status: `LeaveStatusEnum`

Disimpan di `src/types/enums/leave-status.ts` (mengikuti konvensi penempatan enum smartbag).  
Nilai enum merefleksikan dropdown status pada `ais_legacy/principals_tool/leaves/approval_iframe.html`:
- `0` → `PENDING` (Pending)
- `1` → `APPROVED_UNPAID` (Approve unpaid leave)
- `2` → `APPROVED_PAID` (Approve paid leave)
- `3` → `DECLINED` (Decline)
- `6` → `CANCELED` (Cancel Request)

```ts
// src/types/enums/leave-status.ts
export enum LeaveStatusEnum {
  PENDING = 'PENDING',
  APPROVED_UNPAID = 'APPROVED_UNPAID',
  APPROVED_PAID = 'APPROVED_PAID',
  DECLINED = 'DECLINED',
  CANCELED = 'CANCELED',
}
```

### Kolom Tabel `leave` (Schema Update)

| Column | Type | Null | Default | Deskripsi |
|--------|------|------|---------|-----------|
| `id` | int PK | no | autoincrement | Primary key |
| `employee_id` | int FK → employee.id | no | | ID guru pemohon (indexed) |
| `campus_id` | int FK → campus.id | no | | ID campus sekolah (indexed) |
| `date_from` | date | no | | Tanggal mulai cuti |
| `date_to` | date | no | | Tanggal selesai cuti |
| `position` | varchar(100) | no | | Posisi/jabatan guru |
| `department` | varchar(100) | yes | null | Departemen akademik guru |
| `leave_type` | enum(LeaveTypeEnum) | no | | Jenis cuti (SICK=1, MATERNITY=2, PATERNITY=3, UNPAID=4, OTHER=15) |
| `reason` | text | no | | Alasan pengajuan cuti |
| `attachment_file_id` | uuid FK → file.id | yes | null | ID UUID dokumen lampiran (surat dokter/bukti) |
| `active_status` | enum(StatusTypeEnum) | no | ACTIVE (1) | Status aktif record (tidak diubah saat CANCELED) |
| `leave_status` | enum(LeaveStatusEnum) | no | PENDING | Status persetujuan cuti (`PENDING`, `APPROVED_UNPAID`, `APPROVED_PAID`, `DECLINED`, `CANCELED`) |
| `reviewer_comment` | text | yes | null | Catatan feedback/komentar review dari Principal |
| `status_changed_by` | int FK → employee.id | yes | null | ID pegawai (Principal) yang memperbarui status |
| `status_changed_at` | timestamptz | yes | null | Waktu saat status diperbarui |
| `created_at` / `updated_at` / `deleted_at` | timestamptz | | | Timestamps audit dari `BaseEntityWithDates` |

### Definisi Entity TypeScript (`leave.entity.ts`)

```ts
// src/modules/teacher-leave/entities/leave.entity.ts
import { LeaveTypeEnum } from 'src/types/enums/leave-type';
import { LeaveStatusEnum } from 'src/types/enums/leave-status';
import {
  Column,
  Entity,
  Index,
  JoinColumn,
  ManyToOne,
  Relation,
} from 'typeorm';
import { BaseEntityWithDates } from '../../../common/base.entity';
import { StatusTypeEnum } from '../../../types/enums';
import { Campus } from '../../campus/entities/campus.entity';
import { Employee } from '../../employee/entities/employee.entity';
import { File } from '../../file/entities/file.entity';

@Entity({
  orderBy: {
    createdAt: 'DESC',
  },
})
export class Leave extends BaseEntityWithDates {
  @Index()
  @Column({ name: 'employee_id' })
  employeeId: number;

  @ManyToOne(() => Employee, (employee) => employee.leaves)
  @JoinColumn({ name: 'employee_id' })
  employee: Relation<Employee>;

  @Index()
  @Column({ name: 'campus_id' })
  campusId: number;

  @ManyToOne(() => Campus, (campus) => campus.leaves)
  @JoinColumn({ name: 'campus_id' })
  campus: Relation<Campus>;

  @Column({ name: 'date_from', type: 'date' })
  dateFrom: string;

  @Column({ name: 'date_to', type: 'date' })
  dateTo: string;

  @Column({ type: 'varchar', length: 100 })
  position: string;

  @Column({ type: 'varchar', length: 100, nullable: true })
  department: string | null;

  @Column({ name: 'leave_type', type: 'enum', enum: LeaveTypeEnum })
  leaveType: LeaveTypeEnum;

  @Column({ type: 'text' })
  reason: string;

  @Index()
  @Column({ name: 'attachment_file_id', type: 'uuid', nullable: true })
  attachmentFileId: string | null;

  @ManyToOne(() => File, { nullable: true })
  @JoinColumn({ name: 'attachment_file_id' })
  attachmentFile: Relation<File>;

  @Column({
    name: 'active_status',
    type: 'enum',
    enum: StatusTypeEnum,
    default: StatusTypeEnum.ACTIVE,
  })
  activeStatus: StatusTypeEnum;

  @Index()
  @Column({
    name: 'leave_status',
    type: 'enum',
    enum: LeaveStatusEnum,
    default: LeaveStatusEnum.PENDING,
  })
  leaveStatus: LeaveStatusEnum;

  @Column({ name: 'reviewer_comment', type: 'text', nullable: true })
  reviewerComment: string | null;

  @Index()
  @Column({ name: 'status_changed_by', type: 'int', nullable: true })
  statusChangedBy: number | null;

  @ManyToOne(() => Employee, { nullable: true })
  @JoinColumn({ name: 'status_changed_by' })
  statusChangedByUser: Relation<Employee>;

  @Column({ name: 'status_changed_at', type: 'timestamptz', nullable: true })
  statusChangedAt: string | null;
}
```

---

## Aturan Query Ordering Khusus (Record Canceled Paling Bawah)

Sesuai requirement bisnis: record yang dibatalkan (`CANCELED`) **tidak dihapus** dari database, melainkan tetap tersimpan dan selalu berada pada **urutan paling bawah (order paling bawah)** dalam daftar tampilan dan response endpoint `findAll`.

### Implementasi QueryBuilder pada `LeaveService`:

```ts
const queryBuilder = Leave.createQueryBuilder('leave')
  .leftJoinAndSelect('leave.employee', 'employee')
  .leftJoinAndSelect('leave.attachmentFile', 'attachmentFile')
  .leftJoinAndSelect('leave.statusChangedByUser', 'statusChangedByUser')
  .where('leave.activeStatus = :activeStatus', { activeStatus: StatusTypeEnum.ACTIVE })
  .orderBy(`CASE WHEN leave.leave_status = '${LeaveStatusEnum.CANCELED}' THEN 1 ELSE 0 END`, 'ASC')
  .addOrderBy('leave.createdAt', 'DESC');
```

---

## Integrasi Email Notification Schema & Types

Terhubung langsung dengan Bull Queue processor di `api_nest/src/modules/mailer/`:

1. **Queue Name:** `SEND_EMAIL`
2. **Process Types Baru:**
   - `LEAVE_SUBMITTED_NOTIFICATION`:
     - Recipient: Email Principal/VP unit bersangkutan (`employee.email`)
     - Subject: `[AIS Leave Notification] New Teacher Leave Request - {Teacher Name}`
     - Body Context: Nama guru, unit/campus, tanggal cuti, jenis cuti, alasan.
   - `LEAVE_STATUS_CHANGED_NOTIFICATION`:
     - Recipient: Email guru pemohon (`employee.email`)
     - Subject: `[AIS Leave Notification] Teacher Leave Request Status Updated - {Status}`
     - Body Context: Nama guru, tanggal cuti, jenis cuti, status baru (`Approved Paid / Unpaid / Declined / Canceled`), catatan reviewer/Principal.

---

## Migrations

- Generate: `npm run migration:generate --name=add-leave-status-workflow`
- Data backfill: `UPDATE leave SET leave_status = 'PENDING' WHERE leave_status IS NULL;`
