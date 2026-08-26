# Schema — Form Leave (Teacher Leave)

> Status: DRAFT — mengikuti konvensi `api_nest` (TypeORM 0.3.10, PostgreSQL, entity extends `BaseEntityWithDates`).
> Base entity `src/common/base.entity` menyediakan `id`, `createdAt`, `updatedAt`, `deletedAt` (soft delete) secara implisit — tidak perlu didefinisikan ulang.

## Entitas 1: `TeacherLeave` (table `teacher_leave`)

```ts
// src/modules/teacher-leave/entities/teacher-leave.entity.ts
export enum TeacherLeaveTypeEnum {
  SICK_L = 'SICK_L',
  MATERNITY_L = 'MATERNITY_L',
  PATERNITY_L = 'PATERNITY_L',
  UNPAID_L = 'UNPAID_L',
  OTHER = 'OTHER',
}

@Entity({ orderBy: { createdAt: 'DESC' } })
export class TeacherLeave extends BaseEntityWithDates {
  @Index()
  @Column({ name: 'teacher_id' })
  teacherId: number;

  @ManyToOne(() => Employee)
  @JoinColumn({ name: 'teacher_id' })
  teacher: Relation<Employee>;

  @Index()
  @Column({ name: 'campus_id' })
  campusId: number;

  @ManyToOne(() => Campus)
  @JoinColumn({ name: 'campus_id' })
  campus: Relation<Campus>;

  @Column({ name: 'date_from', type: 'date' })
  dateFrom: string; // YYYY-MM-DD

  @Column({ name: 'date_to', type: 'date' })
  dateTo: string; // YYYY-MM-DD

  @Column({ type: 'varchar', length: 100 })
  position: string;

  @Column({ type: 'varchar', length: 100, nullable: true })
  department: string | null;

  @Column({ name: 'leave_type', type: 'enum', enum: TeacherLeaveTypeEnum })
  leaveType: TeacherLeaveTypeEnum;

  @Column({ type: 'text' })
  reason: string;

  @Index()
  @Column({ name: 'attachment_file_id', type: 'int', nullable: true })
  attachmentFileId: number | null;

  @ManyToOne(() => File, { nullable: true })
  @JoinColumn({ name: 'attachment_file_id' })
  attachmentFile: Relation<File>;

  @Column({
    type: 'enum',
    enum: StatusTypeEnum,
    default: StatusTypeEnum.ACTIVE,
  })
  activeStatus: StatusTypeEnum;
}
```

### Kolom (SQL)

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | dari BaseEntityWithDates |
| teacher_id | int FK → employee.id | no | | index |
| campus_id | int FK → campus.id | no | | index |
| date_from | date | no | | YYYY-MM-DD |
| date_to | date | no | | YYYY-MM-DD |
| position | varchar(100) | no | | default "Teacher" di UI |
| department | varchar(100) | yes | null | |
| leave_type | enum(SICK_L, MATERNITY_L, PATERNITY_L, UNPAID_L, OTHER) | no | | mapping legacy id 1/3/4/5/6 |
| reason | text | no | | |
| attachment_file_id | int FK → file.id | yes | null | index; PDF |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | |
| created_at / updated_at / deleted_at | timestamptz | | | dari BaseEntityWithDates |

### Constraints & Index

- **INDEX** `idx_teacher_leave_teacher_created` pada `(teacher_id, created_at)` — query list milik guru.
- **INDEX** `idx_teacher_leave_campus_created` pada `(campus_id, created_at)` — query reporting per campus (future).
- **INDEX** `idx_teacher_leave_date_range` pada `(teacher_id, date_from, date_to)` — cek overlap (edge case).

## Migrations

- Generate: `npm run migration:generate --name=create-teacher-leave` (di `api_nest`, pakai `migration-source.ts`).
- File migration ditaruh di `src/database/migrations/`.
- Tidak ada migrasi data (greenfield — tabel baru, tidak ada data legacy).

## Catatan Desain

1. **Relasi `attachment_file_id` → `File`** — memakai modul `file` existing (`FileEntityTypeEnum.ATTACHMENT_FILE`, `file-entity-type.ts:12`); hanya menyimpan id, download via `GET /v1/files/download`.
2. **Soft delete** — `activeStatus` (StatusTypeEnum) dipakai seperti modul `lesson`; `deletedAt` dari `BaseEntityWithDates` juga tersedia jika ingin full soft delete.
3. **Enum leave type** — string enum `'SICK_L' | 'MATERNITY_L' | 'PATERNITY_L' | 'UNPAID_L' | 'OTHER'`, disimpan sebagai PostgreSQL enum; mapping dari legacy `leavetype_id` (1/3/4/5/6) dilakukan di service saat migrasi data (jika ada).
4. **Tidak ada unique constraint kombinasi** — guru bisa punya banyak pengajuan leave; overlap dicek di service (edge case EC-01).
