# Schema — Lesson Plan

> Status: APPROVED — diselaraskan langsung dengan entitas eksisting di `smartbag/api_nest/src/modules/lesson-plan/entities/` dan form legacy `create2.php?id=103581` (TypeORM 0.3.10, PostgreSQL, entity extends `BaseEntityWithDates`).
> Base entity `src/common/base.entity` menyediakan `id`, `createdAt`, `updatedAt`, `deletedAt` (soft delete).

## Overview Entitas Modul Lesson Plan

```
LessonPlan (header metadata, FK SubjectYear, AcademicYear, Employee)
├── LessonPlanDetail (1:1 body, objectives, serialized JSON arrays/strings)
├── LessonPlanMaterialFile[] (1:N attachment multi-file PPT, PDF, VIDEO terhubung ke File UUID)
└── LessonPlanComment[] (1:N catatan review HOD / Principal)
```

---

## Entitas 1: `LessonPlan` (table `lesson_plan`)

Entitas header informasi rencana pembelajaran per guru, kelas, tahun akademik, term, dan week.
Sesuai implementasi eksisting di `api_nest/src/modules/lesson-plan/entities/lesson-plan.entity.ts`.

```ts
// src/modules/lesson-plan/entities/lesson-plan.entity.ts
import { BaseEntityWithDates } from '../../../common/base.entity';
import { AcademicYear } from '../../academic-year/entities/academic-year.entity';
import { Employee } from '../../employee/entities/employee.entity';
import { SubjectYear } from '../../subject-year/entities/subject-year.entity';
import { StatusTypeEnum } from '../../../types/enums';
import {
  Column,
  Entity,
  Index,
  JoinColumn,
  ManyToOne,
  OneToMany,
  OneToOne,
  Relation,
} from 'typeorm';
import { LessonPlanComment } from './lesson-plan-comment.entity';
import { LessonPlanDetail } from './lesson-plan-detail.entity';

@Entity({
  orderBy: {
    createdAt: 'DESC',
  },
})
@Index(
  'uk_lesson_plan_teacher_class_ay_term_week',
  ['teacherId', 'classSubjectId', 'academicYearId', 'term', 'week'],
  {
    unique: true,
    where: `"active_status" = '1' AND "deleted_at" IS NULL`,
  },
)
@Index('idx_lesson_plan_library', ['academicYearId', 'classSubjectId'])
@Index('idx_lesson_plan_teacher_ay', ['teacherId', 'academicYearId'])
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

  @ManyToOne(() => SubjectYear)
  @JoinColumn({ name: 'class_subject_id' })
  classSubject: Relation<SubjectYear>;

  @Index()
  @Column({ name: 'academic_year_id' })
  academicYearId: number;

  @ManyToOne(() => AcademicYear)
  @JoinColumn({ name: 'academic_year_id' })
  academicYear: Relation<AcademicYear>;

  @Column({ type: 'int' })
  term: number; // 1 - 4

  @Column({ type: 'int' })
  week: number; // Tervalidasi terhadap AcademicYearWeek per term

  @Column({ type: 'varchar', length: 255 })
  topic: string;

  @Index()
  @Column({ name: 'source_lesson_plan_id', type: 'int', nullable: true })
  sourceLessonPlanId: number | null; // tracking asal copy

  @ManyToOne(() => LessonPlan, { nullable: true })
  @JoinColumn({ name: 'source_lesson_plan_id' })
  sourceLessonPlan: Relation<LessonPlan>;

  @Column({
    name: 'active_status',
    type: 'enum',
    enum: StatusTypeEnum,
    default: StatusTypeEnum.ACTIVE,
  })
  activeStatus: StatusTypeEnum;

  @OneToOne(() => LessonPlanDetail, (detail) => detail.lessonPlan, {
    cascade: true,
  })
  detail: Relation<LessonPlanDetail>;

  @OneToMany(() => LessonPlanMaterialFile, (f) => f.lessonPlan, { cascade: true })
  materialFiles: Relation<LessonPlanMaterialFile[]>;

  @OneToMany(() => LessonPlanComment, (comment) => comment.lessonPlan)
  comments: Relation<LessonPlanComment[]>;
}
```

### Kolom `lesson_plan`

| Column | Type | Null | Default | Deskripsi |
|--------|------|------|---------|-----------|
| `id` | int PK | no | autoincrement | ID unik lesson plan |
| `teacher_id` | int FK → employee.id | no | | ID guru pembuat / pengampu |
| `class_subject_id` | int FK → subject_year.id | no | | Relasi assignment kelas + subject (`SubjectYear`) |
| `academic_year_id` | int FK → academic_year.id | no | | Tahun akademik |
| `term` | int | no | | Nilai term (1, 2, 3, 4) |
| `week` | int | no | | Nilai minggu (disinkronkan dengan `AcademicYearWeek`) |
| `topic` | varchar(255) | no | | Topik rencana pembelajaran |
| `source_lesson_plan_id` | int FK → lesson_plan.id | yes | null | ID lesson plan sumber jika merupakan hasil copy |
| `active_status` | enum(1, 0) / enum(ACTIVE, INACTIVE) | no | ACTIVE | Status record aktif |
| `created_at` / `updated_at` / `deleted_at` | timestamptz | | | Timestamps audit dari `BaseEntityWithDates` |

---

## Entitas 2: `LessonPlanDetail` (table `lesson_plan_detail`)

Menyimpan rincian konten form `create2.php`. Field checkbox disimpan sebagai serialized JSON array string (tipe `text`) di database, kemudian di-parse/serialize secara konsisten oleh `LessonPlanService`.

```ts
// src/modules/lesson-plan/entities/lesson-plan-detail.entity.ts
import { BaseEntityWithDates } from '../../../common/base.entity';
import {
  Column,
  Entity,
  Index,
  JoinColumn,
  OneToOne,
  Relation,
} from 'typeorm';
import { LessonPlan } from './lesson-plan.entity';

@Entity()
export class LessonPlanDetail extends BaseEntityWithDates {
  @Index({ unique: true })
  @Column({ name: 'lesson_plan_id' })
  lessonPlanId: number;

  @OneToOne(() => LessonPlan, (lessonPlan) => lessonPlan.detail, {
    onDelete: 'CASCADE',
  })
  @JoinColumn({ name: 'lesson_plan_id' })
  lessonPlan: Relation<LessonPlan>;

  @Column({ name: 'main_objectives', type: 'text', nullable: true })
  mainObjectives: string | null;

  @Column({ name: 'higher_order_objectives', type: 'text', nullable: true })
  higherOrderObjectives: string | null;

  /**
   * Checkbox Pedagogy:
   * Serialized JSON array of strings:
   * ["lecture", "Group Discussion", "Problem based learning", "Kagan Cooperative Learning", "Discovery learning", "Case Studies", "Peer teaching", "Others: <custom_text>"]
   */
  @Column({ type: 'text', nullable: true })
  pedagogy: string | null;

  /**
   * Checkbox Material / Resources:
   * Serialized JSON array of strings:
   * ["Power Point", "Pdf", "Video", "videolink: https://...", "HBL Resources", "Teachers demo", "Models ( Hands on material )", "Practical Activities", "website: https://...", "others: <custom_text>"]
   */
  @Column({ name: 'material_resources', type: 'text', nullable: true })
  materialResources: string | null;

  /**
   * Activities (Textarea detail aktivitas & alokasi waktu pembelajaran)
   */
  @Column({ type: 'text', nullable: true })
  activities: string | null;

  /**
   * Assessment Before Lesson:
   * Serialized JSON array: ["Short Quiz", "Questioning"]
   */
  @Column({ name: 'assessment_before', type: 'text', nullable: true })
  assessmentBefore: string | null;

  /**
   * Assessment During Lesson:
   * Serialized JSON array: ["Observation", "Questioning", "Discussion", "Peer / self assessment", "Individual whiteboard"]
   */
  @Column({ name: 'assessment_during', type: 'text', nullable: true })
  assessmentDuring: string | null;

  /**
   * Assessment After Lesson:
   * Serialized JSON array: ["Short quiz", "Games", "Discussion", "Peer / self assessment", "Test", "Project"]
   */
  @Column({ name: 'assessment_after', type: 'text', nullable: true })
  assessmentAfter: string | null;

  /**
   * Assignment Checkboxes & Notes:
   * Serialized JSON string atau structured text mencakup Classwork, Homework, Lab report, Project.
   */
  @Column({ type: 'text', nullable: true })
  assignment: string | null;

  /**
   * Teacher Reflection (Textarea 3 pertanyaan pemandu)
   */
  @Column({ type: 'text', nullable: true })
  reflection: string | null;
}
```

### Kolom `lesson_plan_detail`

| Column | Type | Null | Deskripsi |
|--------|------|------|-----------|
| `id` | int PK | no | Primary key autoincrement |
| `lesson_plan_id` | int FK → lesson_plan.id | no | Relasi 1:1, UNIQUE, CASCADE delete |
| `main_objectives` | text | yes | Sasaran utama dari SOW (disalin manual) |
| `higher_order_objectives` | text | yes | Sasaran Higher Order Thinking Skills |
| `pedagogy` | text (serialized JSON array) | yes | Pilihan 7 strategi pedagogi terstandarisasi BBS + Others |
| `material_resources` | text (serialized JSON array) | yes | Pilihan material/resources (PPT, PDF, Video, HBL, Demo, Models, Practical, Website, Others) |
| `activities` | text | yes | Detail rencana aktivitas pembelajaran dan alokasi durasi |
| `assessment_before` | text (serialized JSON array) | yes | Checklist asesmen sebelum pembelajaran |
| `assessment_during` | text (serialized JSON array) | yes | Checklist asesmen saat pembelajaran berlangsung |
| `assessment_after` | text (serialized JSON array) | yes | Checklist asesmen setelah pembelajaran selesai |
| `assignment` | text (serialized JSON array/obj) | yes | Detail 4 jenis penugasan (Classwork, Homework, Lab Report, Project) |
| `reflection` | text | yes | Refleksi guru setelah sesi pengajaran |

---

## Entitas 3: `LessonPlanMaterialFile` (table `lesson_plan_material_file`)

Menyimpan attachment multi-file materi (PPT, PDF, Video) per lesson plan yang terhubung ke tabel generic `File` (UUID).
File disimpan dan di-rename secara otomatis dengan standar penamaan:  
`[CATEGORY] - [Topic] - Term [Term] - Week [Week] - [Counter].[ext]`

```ts
// src/modules/lesson-plan/entities/lesson-plan-material-file.entity.ts
import { Column, Entity, Index, JoinColumn, ManyToOne, Relation } from 'typeorm';
import { BaseEntityWithDates } from '../../../common/base.entity';
import { File } from '../../file/entities/file.entity';
import { LessonPlan } from './lesson-plan.entity';

export enum LessonPlanMaterialCategoryEnum {
  PPT = 'PPT',
  PDF = 'PDF',
  VIDEO = 'VIDEO',
}

@Entity({ name: 'lesson_plan_material_file' })
export class LessonPlanMaterialFile extends BaseEntityWithDates {
  @Index()
  @Column({ name: 'lesson_plan_id', type: 'int' })
  lessonPlanId: number;

  @ManyToOne(() => LessonPlan, (lp) => lp.materialFiles, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'lesson_plan_id' })
  lessonPlan: Relation<LessonPlan>;

  @Column({
    type: 'enum',
    enum: LessonPlanMaterialCategoryEnum,
  })
  category: LessonPlanMaterialCategoryEnum; // PPT, PDF, VIDEO

  @Index()
  @Column({ name: 'file_id', type: 'uuid' })
  fileId: string;

  @ManyToOne(() => File, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'file_id' })
  file: Relation<File>;

  @Column({ name: 'file_name', type: 'varchar', length: 255 })
  fileName: string; // Misal: PPT - Chemical Reactions - Term 1 - Week 3 - 01.pptx

  @Column({ name: 'file_url', type: 'text' })
  fileUrl: string;

  @Column({ name: 'counter_number', type: 'int', default: 1 })
  counterNumber: number;
}
```

### Kolom `lesson_plan_material_file`

| Column | Type | Null | Default | Deskripsi |
|--------|------|------|---------|-----------|
| `id` | int PK | no | autoincrement | Primary key |
| `lesson_plan_id` | int FK → lesson_plan.id | no | | ID lesson plan induk, CASCADE delete |
| `category` | enum(PPT, PDF, VIDEO) | no | | Kategori material file |
| `file_id` | uuid FK → file.id | no | | Relasi ke UUID tabel File |
| `file_name` | varchar(255) | no | | Nama file dengan penamaan baku |
| `file_url` | text | no | | URL file pada storage |
| `counter_number` | int | no | 1 | Nomor urut counter per kategori material |

---

## Entitas 4: `LessonPlanComment` (table `lesson_plan_comment`)

Menyimpan catatan review dan umpan balik dari HOD (Head of Department) dan Principal.

```ts
// src/modules/lesson-plan/entities/lesson-plan-comment.entity.ts
import { BaseEntityWithDates } from '../../../common/base.entity';
import { Employee } from '../../employee/entities/employee.entity';
import { LessonPlanCommentTypeEnum } from '../../../types/enums';
import {
  Column,
  Entity,
  Index,
  JoinColumn,
  ManyToOne,
  Relation,
} from 'typeorm';
import { LessonPlan } from './lesson-plan.entity';

@Entity()
export class LessonPlanComment extends BaseEntityWithDates {
  @Index()
  @Column({ name: 'lesson_plan_id' })
  lessonPlanId: number;

  @ManyToOne(() => LessonPlan, (lessonPlan) => lessonPlan.comments, {
    onDelete: 'CASCADE',
  })
  @JoinColumn({ name: 'lesson_plan_id' })
  lessonPlan: Relation<LessonPlan>;

  @Index()
  @Column({ name: 'commenter_id' })
  commenterId: number;

  @ManyToOne(() => Employee)
  @JoinColumn({ name: 'commenter_id' })
  commenter: Relation<Employee>;

  @Column({
    name: 'comment_type',
    type: 'enum',
    enum: LessonPlanCommentTypeEnum,
  })
  commentType: LessonPlanCommentTypeEnum; // HOD atau PRINCIPAL

  @Column({ type: 'text' })
  comment: string;
}
```

### Kolom `lesson_plan_comment`

| Column | Type | Null | Deskripsi |
|--------|------|------|-----------|
| `id` | int PK | no | Primary key autoincrement |
| `lesson_plan_id` | int FK → lesson_plan.id | no | ID lesson plan induk, CASCADE delete |
| `commenter_id` | int FK → employee.id | no | ID pegawai pemberi komentar review (HOD atau Principal) |
| `comment_type` | enum(HOD, PRINCIPAL) | no | Tipe peran pemberi review |
| `comment` | text | no | Isi teks catatan / feedback |

---

## Indeks & Constraint Database

1. **Unique Constraint:**
   - `uk_lesson_plan_teacher_class_ay_term_week` pada `lesson_plan` untuk kolom `(teacher_id, class_subject_id, academic_year_id, term, week)` dengan klausa kondisi `"active_status" = '1' AND "deleted_at" IS NULL`.
   - `unique` pada `lesson_plan_detail(lesson_plan_id)`.
2. **Performance Indexes:**
   - `idx_lesson_plan_library` pada `(academic_year_id, class_subject_id)` untuk mempercepat pencarian di Lesson Plan Library.
   - `idx_lesson_plan_teacher_ay` pada `(teacher_id, academic_year_id)` untuk list mingguan per guru.
   - `idx_lesson_plan_material_file_lp_cat` pada `(lesson_plan_id, category)` pada `lesson_plan_material_file`.

