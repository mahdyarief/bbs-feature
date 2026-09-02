# Schema — Form Leave Workflow (Teacher Leave Status)

> Status: DRAFT — **EXTEND** schema fase 1 (`form-leave/schema.md`); tabel `teacher_leave` ditambah kolom status approval via ALTER TABLE.
> Fase 1 sudah memiliki entitas `TeacherLeave` dengan kolom: `teacherId`, `campusId`, `dateFrom`, `dateTo`, `position`, `department`, `leaveType`, `reason`, `attachmentFileId`, `activeStatus`.
> Fase 2 menambahkan kolom berikut tanpa mengubah kolom existing.

## Modified Entity: `TeacherLeave` (table `teacher_leave`)

### Enum Baru: `TeacherLeaveStatusEnum`

```ts
// src/modules/teacher-leave/entities/teacher-leave-status.enum.ts
export enum TeacherLeaveStatusEnum {
  PENDING = 'PENDING',
  APPROVED_BY_ADMIN = 'APPROVED_BY_ADMIN',
  APPROVED_BY_PRINCIPAL = 'APPROVED_BY_PRINCIPAL',
  REJECTED = 'REJECTED',
}
```

### Kolom Tambahan (ALTER TABLE)

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| `leave_status` | enum(PENDING, APPROVED_BY_ADMIN, APPROVED_BY_PRINCIPAL, REJECTED) | no | `PENDING` | Default untuk record baru dan existing (backfill) |
| `admin_comment` | text | yes | null | Komentar Admin (tipe '1' analog `commentsleave`) |
| `principal_comment` | text | yes | null | Komentar Principal (tipe '2' analog `comments_principal`) |
| `status_changed_by` | int FK → employee.id | yes | null | User id yang terakhir mengubah status |
| `status_changed_at` | timestamptz | yes | null | Timestamp terakhir status diubah |

### Entity TypeScript (extend fase 1)

```ts
// src/modules/teacher-leave/entities/teacher-leave.entity.ts
// — TAMBAHKAN properti berikut ke entity yang sudah ada dari fase 1 —

@Column({
  name: 'leave_status',
  type: 'enum',
  enum: TeacherLeaveStatusEnum,
  default: TeacherLeaveStatusEnum.PENDING,
})
leaveStatus: TeacherLeaveStatusEnum;

@Column({ name: 'admin_comment', type: 'text', nullable: true })
adminComment: string | null;

@Column({ name: 'principal_comment', type: 'text', nullable: true })
principalComment: string | null;

@Index()
@Column({ name: 'status_changed_by', type: 'int', nullable: true })
statusChangedBy: number | null;

@ManyToOne(() => Employee, { nullable: true })
@JoinColumn({ name: 'status_changed_by' })
statusChangedByUser: Relation<Employee>;

@Column({ name: 'status_changed_at', type: 'timestamptz', nullable: true })
statusChangedAt: string | null;
```

### Constraints & Index (tambahan)

- **INDEX** `idx_teacher_leave_status` pada `(leave_status)` — filter/pencarian by status.
- **INDEX** `idx_teacher_leave_status_changed_by` pada `(status_changed_by)` — audit/reporting.

### Diagram Entity (lengkap dengan fase 1)

```
teacher_leave
├── id (PK, autoincrement)                    ← fase 1
├── teacher_id (FK → employee.id, indexed)    ← fase 1
├── campus_id (FK → campus.id, indexed)       ← fase 1
├── date_from (date)                          ← fase 1
├── date_to (date)                            ← fase 1
├── position (varchar 100)                    ← fase 1
├── department (varchar 100, nullable)        ← fase 1
├── leave_type (enum)                         ← fase 1
├── reason (text)                             ← fase 1
├── attachment_file_id (FK → file.id, nullable, indexed) ← fase 1
├── active_status (enum ACTIVE/INACTIVE)      ← fase 1
├── leave_status (enum, default PENDING)      ← fase 2 [NEW]
├── admin_comment (text, nullable)            ← fase 2 [NEW]
├── principal_comment (text, nullable)        ← fase 2 [NEW]
├── status_changed_by (FK → employee.id, nullable, indexed) ← fase 2 [NEW]
├── status_changed_at (timestamptz, nullable) ← fase 2 [NEW]
├── created_at (timestamptz)                  ← BaseEntityWithDates
├── updated_at (timestamptz)                  ← BaseEntityWithDates
└── deleted_at (timestamptz, nullable)        ← BaseEntityWithDates
```

## Migrations

- Generate: `npm run migration:generate --name=add-teacher-leave-status` (di `api_nest`, pakai `migration-source.ts`).
- File migration ditaruh di `src/database/migrations/`.
- Migrasi data: `UPDATE teacher_leave SET leave_status = 'PENDING' WHERE leave_status IS NULL;` (backfill untuk record existing dari fase 1).

## Catatan Desain

1. **ALTER TABLE vs new table** — kolom ditambahkan ke tabel `teacher_leave` existing karena relasi 1:1 (satu leave punya satu status). Tidak perlu tabel status terpisah di fase 2.
2. **Status default PENDING** — record baru dari fase 1 (create) otomatis mendapat `PENDING` tanpa perlu ubah kode create. Record existing fase 1 juga backfill ke `PENDING`.
3. **Komentar terpisah per role** — `adminComment` dan `principalComment` dipisahkan untuk mempertahankan jejak siapa berkata apa (mengikuti pola legacy `commentsleave` vs `comments_principal`).
4. **Tidak ada history table** — hanya `statusChangedBy/At` terakhir. Jika diperlukan riwayat transisi, itu enhancement dengan tabel `teacher_leave_status_log` terpisah.
5. **Relasi status_changed_by** — opsional (nullable) karena record existing dari fase 1 belum punya `statusChangedBy`.