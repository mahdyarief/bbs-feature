# API Contract — Lesson Plan

> Status: APPROVED — diselaraskan langsung dengan implementasi backend `api_nest` (`src/modules/lesson-plan/`, NestJS 10, `@Controller({ version: '1', path: 'lesson-plans' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`). Permission di-enforce via `@CheckPermissions` dengan subject `ModulesTypeEnum.LESSON_PLAN`.
> **Base path:** global prefix `api` → URL lengkap `/api/v1/lesson-plans`.
> **Dual portal:** endpoint yang sama dikonsumsi dua portal — Teacher Portal (`client-teacher`) dan Admin Portal (`client`).

---

## Ringkasan Endpoint

| Method | Endpoint | Deskripsi | Permission |
|--------|----------|-----------|------------|
| `GET` | `/v1/lesson-plans` | List lesson plan milik guru (paginated + filter AY, class, term, week) | `READ` |
| `POST` | `/v1/lesson-plans` | Create lesson plan baru (header + detail form) | `CREATE` |
| `GET` | `/v1/lesson-plans/:id` | Detail lengkap lesson plan (termasuk detail, material files, dan review comments) | `READ` |
| `PUT` | `/v1/lesson-plans/:id` | Update seluruh isi lesson plan (header & detail form) | `UPDATE` |
| `PATCH` | `/v1/lesson-plans/:id/header` | Update cepat header saja (topic, term, week, classSubjectId) | `UPDATE` |
| `DELETE` | `/v1/lesson-plans/:id` | Soft delete lesson plan (ubah activeStatus ke 0) | `DELETE` |
| `POST` | `/v1/lesson-plans/:id/files` | Multi-file upload materi pembelajaran (PPT, PDF, VIDEO) dengan auto-renaming | `UPDATE` |
| `DELETE` | `/v1/lesson-plans/:id/files/:fileId` | Hapus file lampiran materi pembelajaran | `DELETE` |
| `POST` | `/v1/lesson-plans/:id/copy` | Copy lesson plan ke target AY dan Class (`SubjectYear`) | `CREATE` |
| `GET` | `/v1/lesson-plans/library` | Lesson Plan Library (semua guru per campus, filter AY, subject, classroom, term, week) | `READ` |
| `GET` | `/v1/lesson-plans/library/classrooms` | Dropdown classroom filter untuk library | `READ` |
| `GET` | `/v1/lesson-plans/library/subjects` | Dropdown subject filter untuk library | `READ` |
| `GET` | `/v1/lesson-plans/viewer` | Sub-menu Lesson Plan Viewer khusus HOD & Principal (list rencana ajar guru lintas kelas/campus) | `READ` |
| `POST` | `/v1/lesson-plans/:id/comments` | Tambah / update catatan review komentar HOD atau Principal | `CREATE` / `UPDATE` |
| `GET` | `/v1/lesson-plans/no-submission` | Monitoring daftar guru yang belum submit lesson plan per term & week | `READ` |

---

## 1. POST /v1/lesson-plans — Create Lesson Plan

Menyimpan rencana pembelajaran baru beserta seluruh komponen form `create2.php`.

### Request Body (`CreateLessonPlanDto`):

```json
{
  "academicYearId": 27,
  "classSubjectId": 38103,
  "term": 1,
  "week": 3,
  "topic": "Chemical Reactions & Rates",
  "teacherId": 21046,
  "mainObjectives": "Students will understand reaction rates and collision theory as outlined in SOW Unit 4.",
  "higherOrderObjectives": "Evaluate experimental data to deduce the rate law and activation energy.",
  "pedagogy": [
    "lecture",
    "Group Discussion",
    "Problem based learning",
    "Kagan Cooperative Learning"
  ],
  "materialResources": [
    "Power Point",
    "Pdf",
    "Video",
    "videolink: https://www.youtube.com/watch?v=example123",
    "Teachers demo",
    "Models ( Hands on material )",
    "website: https://chemguide.co.uk/physical/basicrates.html",
    "others: Lab glassware: 50ml burettes and HCl 0.1M"
  ],
  "activities": "<p>1. Starter (10 min): Short quiz on previous stoichiometry topic.<br>2. Main (50 min): Lab demonstration of magnesium ribbon in HCl.<br>3. Plenary (20 min): Group discussion and exit ticket.</p>",
  "assessmentBefore": [
    "Short Quiz",
    "Questioning"
  ],
  "assessmentDuring": [
    "Observation",
    "Questioning",
    "Discussion",
    "Peer / self assessment",
    "Individual whiteboard"
  ],
  "assessmentAfter": [
    "Short quiz",
    "Games",
    "Discussion",
    "Peer / self assessment",
    "Test"
  ],
  "assignment": "{\"classwork\":\"Worksheet 4.1 Questions 1 to 5\",\"homework\":\"Read textbook pages 102-108 and answer summary review\",\"labReport\":\"Full formal lab report on Reaction Kinetics\",\"project\":\"\"}",
  "reflection": "Strategy was engaging; students grasped collision theory well. Next session will incorporate more time for student calculations."
}
```

### Response 201 Created:

```json
{
  "data": {
    "id": 103581,
    "academicYearId": 27,
    "classSubjectId": 38103,
    "teacherId": 21046,
    "term": 1,
    "week": 3,
    "topic": "Chemical Reactions & Rates",
    "sourceLessonPlanId": null,
    "activeStatus": "1",
    "createdAt": "2026-09-04T08:30:00.000Z",
    "updatedAt": "2026-09-04T08:30:00.000Z",
    "teacherName": "Herlina Susanti",
    "className": "Sec 3-1",
    "subjectName": "Chemistry",
    "levelName": "Secondary 3",
    "academicYearLabel": "2026/2027",
    "detail": {
      "id": 501,
      "lessonPlanId": 103581,
      "mainObjectives": "Students will understand reaction rates and collision theory as outlined in SOW Unit 4.",
      "higherOrderObjectives": "Evaluate experimental data to deduce the rate law and activation energy.",
      "pedagogy": [
        "lecture",
        "Group Discussion",
        "Problem based learning",
        "Kagan Cooperative Learning"
      ],
      "materialResources": [
        "Power Point",
        "Pdf",
        "Video",
        "videolink: https://www.youtube.com/watch?v=example123",
        "Teachers demo",
        "Models ( Hands on material )",
        "website: https://chemguide.co.uk/physical/basicrates.html",
        "others: Lab glassware: 50ml burettes and HCl 0.1M"
      ],
      "activities": "<p>1. Starter (10 min)...</p>",
      "assessmentBefore": [
        "Short Quiz",
        "Questioning"
      ],
      "assessmentDuring": [
        "Observation",
        "Questioning",
        "Discussion",
        "Peer / self assessment",
        "Individual whiteboard"
      ],
      "assessmentAfter": [
        "Short quiz",
        "Games",
        "Discussion",
        "Peer / self assessment",
        "Test"
      ],
      "assignment": "{\"classwork\":\"Worksheet 4.1 Questions 1 to 5\",\"homework\":\"Read textbook pages 102-108 and answer summary review\",\"labReport\":\"Full formal lab report on Reaction Kinetics\",\"project\":\"\"}",
      "reflection": "Strategy was engaging..."
    }
  }
}
```

---

## 2. GET /v1/lesson-plans/:id — Get Detail

Mengambil detail penuh lesson plan untuk form viewer / edit, termasuk detail konten dan riwayat komentar review dari HOD dan Principal.

### Response 200 OK:

```json
{
  "data": {
    "id": 103581,
    "topic": "Chemical Reactions & Rates",
    "term": 1,
    "week": 3,
    "teacherId": 21046,
    "teacherName": "Herlina Susanti",
    "classSubjectId": 38103,
    "className": "Sec 3-1",
    "subjectName": "Chemistry",
    "levelName": "Secondary 3",
    "academicYearId": 27,
    "academicYearLabel": "2026/2027",
    "activeStatus": "1",
    "createdAt": "2026-09-04T08:30:00.000Z",
    "updatedAt": "2026-09-04T08:30:00.000Z",
    "detail": {
      "id": 501,
      "mainObjectives": "Students will understand reaction rates...",
      "higherOrderObjectives": "Evaluate experimental data...",
      "pedagogy": ["lecture", "Group Discussion"],
      "materialResources": ["Power Point", "Pdf", "Video", "Teachers demo"],
      "activities": "...",
      "assessmentBefore": ["Short Quiz", "Questioning"],
      "assessmentDuring": ["Observation", "Questioning"],
      "assessmentAfter": ["Short quiz", "Discussion"],
      "assignment": "{\"classwork\":\"Exercise 1\",\"homework\":\"\",\"labReport\":\"\",\"project\":\"\"}",
      "reflection": "..."
    },
    "materialFiles": [
      {
        "id": 12,
        "category": "PPT",
        "fileId": "6c2cfb4d-1a2b-4c3d-8e9f-0a1b2c3d4e5f",
        "fileName": "PPT - Chemical Reactions - Term 1 - Week 3 - 01.pptx",
        "fileUrl": "https://storage.binabangsaschool.com/files/PPT - Chemical Reactions - Term 1 - Week 3 - 01.pptx",
        "counterNumber": 1
      },
      {
        "id": 13,
        "category": "PDF",
        "fileId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
        "fileName": "PDF - Chemical Reactions - Term 1 - Week 3 - 01.pdf",
        "fileUrl": "https://storage.binabangsaschool.com/files/PDF - Chemical Reactions - Term 1 - Week 3 - 01.pdf",
        "counterNumber": 1
      }
    ],
    "comments": [
      {
        "id": 7,
        "commentType": "HOD",
        "commenterId": 401,
        "commenter": {
          "id": 401,
          "fullName": "Stewart James Spiessens"
        },
        "comment": "Well structured lab plan. Ensure safety goggles are emphasized.",
        "createdAt": "2026-09-04T10:00:00.000Z"
      },
      {
        "id": 8,
        "commentType": "PRINCIPAL",
        "commenterId": 502,
        "commenter": {
          "id": 502,
          "fullName": "Linawati Lauw"
        },
        "comment": "Good integration of cooperative learning strategies.",
        "createdAt": "2026-09-04T11:15:00.000Z"
      }
    ]
  }
}
```

---

## 3. PATCH /v1/lesson-plans/:id/header — Update Header Only

Mereplikasi interaksi tombol `edithead` dan `saveupdate` di `create2.php` (update cepat topik/term/week/kelas tanpa harus mengirim ulang seluruh detail body).

### Request Body (`UpdateLessonPlanHeaderDto`):

```json
{
  "topic": "Chemical Reactions & Catalysts Updated",
  "term": 1,
  "week": 3,
  "classSubjectId": 38103
}
```

### Response 200 OK:

```json
{
  "data": {
    "id": 103581,
    "topic": "Chemical Reactions & Catalysts Updated",
    "term": 1,
    "week": 3,
    "classSubjectId": 38103,
    "updatedAt": "2026-09-04T09:15:00.000Z"
  }
}
```

---

## 4. POST /v1/lesson-plans/:id/files — Upload Material Files (Multi-File with Auto-Rename)

Mengunggah file materi pembelajaran (Power Point, PDF, atau Video) dengan dukungan multiple file upload.
Setiap file yang berhasil diunggah akan otomatis di-rename pada backend storage dengan format:
`[CATEGORY] - [Topic] - Term [Term] - Week [Week] - [Counter].[ext]`

- **Content-Type:** `multipart/form-data`
- **Form fields:**
  - `category`: string (`PPT`, `PDF`, atau `VIDEO`)
  - `files`: multiple binary files (Max 20MB per file)
    * Format PPT: `.ppt`, `.pptx`, `.key`
    * Format PDF: `.pdf`
    * Format Video: `.mp4`, `.3gp`, `.avi`, `.flv`

### Response 201 Created:

```json
{
  "data": [
    {
      "id": 12,
      "lessonPlanId": 103581,
      "category": "PPT",
      "fileId": "6c2cfb4d-1a2b-4c3d-8e9f-0a1b2c3d4e5f",
      "fileName": "PPT - Chemical Reactions - Term 1 - Week 3 - 01.pptx",
      "fileUrl": "https://storage.binabangsaschool.com/files/PPT - Chemical Reactions - Term 1 - Week 3 - 01.pptx",
      "counterNumber": 1
    },
    {
      "id": 13,
      "lessonPlanId": 103581,
      "category": "PPT",
      "fileId": "7d3efa4e-2b3c-5d4e-9f0a-1b2c3d4e5f6a",
      "fileName": "PPT - Chemical Reactions - Term 1 - Week 3 - 02.pptx",
      "fileUrl": "https://storage.binabangsaschool.com/files/PPT - Chemical Reactions - Term 1 - Week 3 - 02.pptx",
      "counterNumber": 2
    }
  ]
}
```

---

## 5. DELETE /v1/lesson-plans/:id/files/:fileId — Remove Material File

Menghapus file lampiran materi pembelajaran yang terhubung ke lesson plan.

### Response 200 OK:

```json
{
  "data": {
    "success": true,
    "message": "Lesson plan material file deleted successfully"
  }
}
```

---

## 6. GET /v1/lesson-plans/viewer — Sub-Menu Lesson Plan Viewer (HOD & Principal Only)

Endpoint khusus untuk sub-menu Lesson Plan Viewer pada Teacher Portal (`client-teacher`), hanya dapat diakses oleh user ber-role **HOD** atau **Principal** (`usePrincipalOrHod === true`). Menampilkan daftar rencana ajar seluruh guru untuk departemen/campus terkait.

- **Query Parameters:**
  - `academicYearId`: int (opsional, default AY aktif)
  - `term`: int (opsional)
  - `week`: int (opsional)
  - `subjectId`: int (opsional)
  - `teacherId`: int (opsional)
  - `page`: int (default 1)
  - `limit`: int (default 10)

### Response 200 OK:

```json
{
  "data": [
    {
      "id": 103581,
      "topic": "Chemical Reactions & Rates",
      "term": 1,
      "week": 3,
      "teacherId": 21046,
      "teacherName": "Herlina Susanti",
      "className": "Sec 3-1",
      "subjectName": "Chemistry",
      "academicYearLabel": "2026/2027",
      "hasComments": true,
      "commentsCount": 2,
      "createdAt": "2026-09-04T08:30:00.000Z"
    }
  ],
  "meta": {
    "total": 45,
    "page": 1,
    "limit": 10,
    "totalPages": 5
  }
}
```

---

## 7. POST /v1/lesson-plans/:id/comments — Add / Update Review Comment

HOD atau Principal memberikan catatan review pada rencana pembelajaran guru (alur komentar review terintegrasi seperti form comment approval di Principal Portal).

### Request Body (`CreateLessonPlanCommentDto`):

```json
{
  "commentType": "HOD",
  "comment": "Please specify the rubrics used for evaluating the lab report."
}
```

### Response 201 Created:

```json
{
  "data": {
    "id": 9,
    "lessonPlanId": 103581,
    "commentType": "HOD",
    "commenterId": 401,
    "comment": "Please specify the rubrics used for evaluating the lab report.",
    "createdAt": "2026-09-04T12:00:00.000Z"
  }
}
```

---

## 8. POST /v1/lesson-plans/:id/copy — Copy Lesson Plan

Menduplikasi seluruh isi rencana pembelajaran ke target Academic Year dan Class Subject (`SubjectYear`) baru.

### Request Body (`CopyLessonPlanDto`):

```json
{
  "targetAcademicYearId": 28,
  "targetClassSubjectId": 38204
}
```

### Response 201 Created:

Mengembalikan data objek `LessonPlan` baru yang terbentuk dengan `sourceLessonPlanId` menunjuk ke lesson plan asal.
Komentar review dan refleksi guru lama tidak ikut disalin ke target baru.

