# Schema — Lesson Plan

> Status: DRAFT — mengikuti konvensi `api_nest` (TypeORM 0.3.10, PostgreSQL, entity extends `BaseEntityWithDates`).
> Base entity `src/common/base.entity` menyediakan `id`, `createdAt`, `updatedAt`, `deletedAt` (soft delete) secara implisit — tidak perlu didefinisikan ulang.

## Entitas 1: `LessonPlan` (table `lesson_plan`)

```ts
// src/modules/lesson-plan/entities/lesson-plan.entity.ts
@Entity({ orderBy: { createdAt: 'DESC' } })
export class LessonPlan extends BaseEntityWithDates {
  @Index()
  @Column({ name: 'teacher_id' })
  teacherId: number;

  @ManyToOne(() => Employee)
  @JoinColumn({ name: 'teacher_id' })
  teacher: Relation<Employee>;

  @Index()
  @Column({ name: 'class_subject_id' })
  classSubjectId: number;

  @ManyToOne(() => HomeroomSubjectTeacher) // atau entity class-subject yang berlaku
  @JoinColumn({ name: 'class_subject_id' })
  classSubject: Relation<HomeroomSubjectTeacher>;

  @Index()
  @Column({ name: 'academic_year_id' })
  academicYearId: number;

  @ManyToOne(() => AcademicYear)
  @JoinColumn({ name: 'academic_year_id' })
  academicYear: Relation<AcademicYear>;

  @Column({ type: 'int' })
  term: number; // 1-4

  @Column({ type: 'int' })
  week: number; // 1-40

  @Column({ type: 'varchar', length: 255 })
  topic: string;

  @Index()
  @Column({ name: 'source_lesson_plan_id', type: 'int', nullable: true })
  sourceLessonPlanId: number | null; // tracking hasil copy

  @ManyToOne(() => LessonPlan, { nullable: true })
  @JoinColumn({ name: 'source_lesson_plan_id' })
  sourceLessonPlan: Relation<LessonPlan>;

  @Column({
    type: 'enum',
    enum: StatusTypeEnum,
    default: StatusTypeEnum.ACTIVE,
  })
  activeStatus: StatusTypeEnum;

  @OneToMany(() => LessonPlanDetail, (d) => d.lessonPlan, { cascade: true })
  details: Relation<LessonPlanDetail[]>;

  @OneToMany(() => LessonPlanComment, (c) => c.lessonPlan)
  comments: Relation<LessonPlanComment[]>;
}
```

### Kolom (SQL)

| Column | Type | Null | Default | Notes |
|--------|------|------|---------|-------|
| id | int PK | no | autoincrement | dari BaseEntityWithDates |
| teacher_id | int FK → employee.id | no | | index |
| class_subject_id | int FK | no | | index |
| academic_year_id | int FK → academic_year.id | no | | index |
| term | int | no | | 1-4 |
| week | int | no | | 1-40 |
| topic | varchar(255) | no | | |
| source_lesson_plan_id | int FK → lesson_plan.id | yes | null | index |
| active_status | enum(ACTIVE, INACTIVE) | no | ACTIVE | |
| created_at / updated_at / deleted_at | timestamptz | | | dari BaseEntityWithDates |

### Constraints & Index

- **UNIQUE** `uk_lesson_plan_teacher_class_ay_term_week` pada `(teacher_id, class_subject_id, academic_year_id, term, week)` — aturan bisnis #1.
- **INDEX** `idx_lesson_plan_library` pada `(academic_year_id, class_subject_id)` — query library.
- **INDEX** `idx_lesson_plan_teacher_ay` pada `(teacher_id, academic_year_id)` — query list guru sendiri.

## Entitas 2: `LessonPlanDetail` (table `lesson_plan_detail`)

```ts
@Entity()
export class LessonPlanDetail extends BaseEntityWithDates {
  @Index({ unique: true })
  @Column({ name: 'lesson_plan_id' })
  lessonPlanId: number;

  @OneToOne(() => LessonPlan, (lp) => lp.details, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'lesson_plan_id' })
  lessonPlan: Relation<LessonPlan>;

  @Column({ name: 'main_objectives', type: 'text', nullable: true })
  mainObjectives: string | null;

  @Column({ name: 'higher_order_objectives', type: 'text', nullable: true })
  higherOrderObjectives: string | null;

  @Column({ type: 'text', nullable: true })
  pedagogy: string | null; // serialized JSON array, contoh: '["lecture","Group Discussion"]'

  @Column({ name: 'material_resources', type: 'text', nullable: true })
  materialResources: string | null; // serialized JSON array

  @Column({ type: 'text', nullable: true })
  activities: string | null;

  @Column({ name: 'assessment_before', type: 'text', nullable: true })
  assessmentBefore: string | null; // serialized JSON array

  @Column({ name: 'assessment_during', type: 'text', nullable: true })
  assessmentDuring: string | null;

  @Column({ name: 'assessment_after', type: 'text', nullable: true })
  assessmentAfter: string | null;

  @Column({ type: 'text', nullable: true })
  assignment: string | null;

  @Column({ type: 'text', nullable: true })
  reflection: string | null;
}
```

### Kolom (SQL)

| Column | Type | Null | Notes |
|--------|------|------|-------|
| id | int PK | no | |
| lesson_plan_id | int FK → lesson_plan.id | no | UNIQUE, CASCADE delete |
| main_objectives | text | yes | |
| higher_order_objectives | text | yes | |
| pedagogy | text | yes | JSON array string |
| material_resources | text | yes | JSON array string |
| activities | text | yes | |
| assessment_before | text | yes | JSON array string |
| assessment_during | text | yes | JSON array string |
| assessment_after | text | yes | JSON array string |
| assignment | text | yes | |
| reflection | text | yes | |

## Entitas 3: `LessonPlanComment` (table `lesson_plan_comment`)

```ts
export enum LessonPlanCommentTypeEnum {
  HOD = 'HOD',
  PRINCIPAL = 'PRINCIPAL',
}

@Entity()
export class LessonPlanComment extends BaseEntityWithDates {
  @Index()
  @Column({ name: 'lesson_plan_id' })
  lessonPlanId: number;

  @ManyToOne(() => LessonPlan, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'lesson_plan_id' })
  lessonPlan: Relation<LessonPlan>;

  @Index()
  @Column({ name: 'commenter_id' })
  commenterId: number;

  @ManyToOne(() => Employee)
  @JoinColumn({ name: 'commenter_id' })
  commenter: Relation<Employee>;

  @Column({ name: 'comment_type', type: 'enum', enum: LessonPlanCommentTypeEnum })
  commentType: LessonPlanCommentTypeEnum;

  @Column({ type: 'text' })
  comment: string;
}
```

### Kolom (SQL)

| Column | Type | Null | Notes |
|--------|------|------|-------|
| id | int PK | no | |
| lesson_plan_id | int FK → lesson_plan.id | no | index, CASCADE |
| commenter_id | int FK → employee.id | no | index |
| comment_type | enum(HOD, PRINCIPAL) | no | |
| comment | text | no | |
| created_at | timestamptz | | dari Base |

## Migrations

- Generate: `npm run migration:generate --name=create-lesson-plan` (di `api_nest`, pakai `migration-source.ts`).
- File migration ditaruh di `src/database/migrations/`.
- Tidak ada migrasi data (greenfield — tabel baru, tidak ada data legacy).

## Catatan Desain

1. **JSON array disimpan sebagai TEXT** (bukan `jsonb`) — konsisten dengan gaya entity existing yang sederhana; serializer/deserializer di service layer. Alternatif `jsonb` bisa dipertimbangkan di review (catat di `notes.md`).
2. **Soft delete** — `activeStatus` (StatusTypeEnum) dipakai seperti modul `lesson`; `deletedAt` dari `BaseEntityWithDates` juga tersedia jika ingin full soft delete.
3. **Relasi `LessonPlanDetail` OneToOne** — karena 1 lesson plan = 1 set detail (bukan banyak). CASCADE delete otomatis.
4. **Enum comment type** — string enum `'HOD' | 'PRINCIPAL'`, disimpan sebagai PostgreSQL enum.
