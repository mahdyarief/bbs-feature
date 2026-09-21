---
title: Menu "No Submission" Lesson Plan tampil di Teacher Portal — seharusnya hanya Admin (HOD/Principal/Admin)
status: open
severity: major
product: BBS LMS
portal: Teacher
author: System Analyst
date: 2026-09-21
jam: ~
---

# Lesson Plan "No Submission" muncul di Teacher Portal padahal seharusnya hanya Admin

## Summary

Pada **Teacher Portal**, menu **Lesson Plan → No Submission** (`/lesson-plan/no-submission`) tampil di sidebar dan halamannya dapat diakses. Menu/halaman ini **seharusnya hanya ada di Admin Portal**, karena data no-submission hanya boleh dilihat oleh **HOD, Principal, dan Admin**. Teacher reguler tidak seharusnya melihat menu ini, apalagi membuka halaman dan datanya.

**Ekspektasi:** Item nav "No Submission" tidak tampil untuk teacher reguler di Teacher Portal; akses langsung via URL `/lesson-plan/no-submission` diblokir (redirect / 403) untuk non-HOD/Principal/Admin.

---

## Test Identity / Akun Akses

| Field | Nilai |
|-------|-------|
| Reporter / Tester | System Analyst |
| Email (Jam account) | — |
| Jam author ID | — |
| User ID (dari console log, mis. `selfUser`) | — |
| Portal URL | https://teacher.smartbag.binabangsaschool.com/lesson-plan/no-submission |
| Environment API | teacher.smartbag.binabangsaschool.com |
| Data konteks (akun teacher reguler non-HOD/Principal) | — |
| Browser / OS | — |

---

## Steps to Reproduce

1. Login ke **Teacher Portal** dengan akun **teacher reguler** (bukan HOD/Principal/Admin).
2. Buka sidebar menu **Lesson Plan**.
3. Perhatikan item **"No Submission"** → klik, atau akses langsung URL `/lesson-plan/no-submission`.

**Actual Result:** Menu "No Submission" tampil di sidebar dan halaman terbuka beserta data no-submission.

**Expected Result:** Menu tidak tampil untuk teacher reguler; akses langsung via URL ditolak. Hanya HOD/Principal/Admin yang boleh melihat (dan di portal Admin saja sesuai requirement).

---

## Affected Files / Root Cause Analysis

- `bbs/client-teacher/src/containers/_nav.jsx` (line ~298-305): item nav `"No Submission"` → `/lesson-plan/no-submission` **tanpa flag gating**, berbeda dengan item "Lesson Plan Viewer" (line ~306-314) yang memakai `requirePrincipalOrHod: true`.
- `bbs/client-teacher/src/routes.js` (line ~337-339): route `/lesson-plan/no-submission` → `LessonPlanNoSubmission` juga **tanpa guard role**.
- `bbs/client-teacher/src/views/lessonPlan/LessonPlanNoSubmission.jsx` (line ~22, ~70): halaman langsung memanggil `fromApi.getLessonPlanNoSubmission` tanpa cek `usePrincipalOrHod`.
- Gating utility yang sudah tersedia namun tidak dipakai di sini: `bbs/client-teacher/src/hooks/usePrincipalOrHod.js` (menyediakan `isPrincipalOrHod`, `isPrincipalOrVp`, `isHOD`).
- Bandingkan dengan Admin Portal: `bbs/client/src/containers/_nav.jsx` (line ~971-972) — di admin menu ini memang disediakan.

Root cause: item nav dan route di Teacher Portal tidak diberi pembatasan role (principal/HOD), sehingga semua teacher melihat dan dapat mengakses halaman tersebut.

---

## Impact

- Teacher reguler melihat menu yang bukan untuknya → kebingungan & keluhan.
- Potensi kebocoran informasi: data kelalaian pengumpulan lesson plan rekan sejawat terekspos ke teacher biasa.
- Inkonsistensi permission antara FE dan BE.

---

## Saran Solusi

1. Di `client-teacher/src/containers/_nav.jsx`: tambahkan `requirePrincipalOrHod: true` pada item nav "No Submission" (mengikuti pola "Lesson Plan Viewer").
2. Di `client-teacher/src/routes.js` / wrapper route: guard route `/lesson-plan/no-submission` dengan `isPrincipalOrHod` (redirect ke `/lesson-plan` atau 403 jika bukan HOD/Principal).
3. Opsional (defense in depth): di `LessonPlanNoSubmission.jsx`, cek `usePrincipalOrHod()` sebelum fetch; konfirmasi juga apakah BE endpoint `/lesson-plans/no-submission` sudah memvalidasi role (HOD/Principal/Admin) — jika belum, tambahkan assert di API.
4. Sesuaikan dengan requirement: jika memang **hanya Admin Portal** yang menyediakan menu ini, alternatifnya adalah **menghapus nav + route** dari Teacher Portal sepenuhnya.

---

## Acceptance Criteria

- [ ] Teacher reguler tidak melihat menu "No Submission" di sidebar Teacher Portal.
- [ ] Akses langsung `/lesson-plan/no-submission` oleh teacher reguler diblokir (redirect/403), termasuk saat refresh.
- [ ] HOD & Principal (jika diputuskan tetap ada di Teacher Portal) masih bisa mengakses menu & halaman.
- [ ] BE endpoint `/lesson-plans/no-submission` menolak request dari non-HOD/Principal/Admin (jika belum).
- [ ] Tidak ada regresi pada menu Lesson Plan lain (Library, File Library, Viewer).
