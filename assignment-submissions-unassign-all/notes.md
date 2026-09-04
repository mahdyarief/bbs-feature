# Notes — Unassign All Submissions

## Open Questions / Yang Perlu Diklarifikasi

1. **Route-level authorization (EC-03).** Apakah module `student-assignment` sudah punya permission guard (`@CheckPermissions` / role-teacher guard) yang bisa dipakai, atau perlu dibuat baru? Saat ini rute `deleteStudentAssignment` (single) tidak punya guard — apakah konvensi ini yang ingin dipertahankan?
2. **Soft-delete recovery.** Apakah ada mekanisme "restore" untuk `StudentAssignment` yang sudah di-soft-remove? Jika tidak, bulk unassign efektif permanen (hanya bisa dibuat ulang via assign ulang). Perlu dipastikan ini sesuai ekspektasi teacher.
3. **Toast / feedback message.** Pesan sukses apa yang diinginkan? Saran: "All N submissions for this assignment have been unassigned." (dengan `count` dari response).
4. **Re-distribute flow.** Setelah bulk unassign, teacher biasanya akan assign ulang ke class yang benar. Apakah perlu langsung mengarahkan ke halaman assign, atau cukup kembali ke list kosong? (Out of scope — hanya catatan UX.)
5. **Pagination vs bulk.** Endpoint bulk menghapus SELURU submission untuk assignment (lintas halaman, lintas classroom filter). Ini berbeda dari filter `classroomId` di UI. Perlu dipastikan teacher paham bahwa "Unassign All" = seluruh class, bukan hanya yang terlihat di halaman ini.

## Asumsi

- `StudentAssignmentService.deleteAllByAssignment(assignmentId)` sudah ada dan teruji (lihat `smartbag/api_nest/src/modules/student-assignment/student-assignment.service.ts:488`).
- Frontend `AssignmentDetailsSubmissions.jsx` sudah punya `isOwnChapter` gate, `studentAssignmentsApi?.refresh()`, `bbsConfirm`, dan `BBSButton` — tinggal tambah handler + tombol.
- `fromApi.deleteStudentAssignment(id)` sudah ada sebagai pola yang bisa ditiru untuk action bulk.

## Referensi File (tidak diubah)

| File | Catatan |
|------|---------|
| `smartbag/api_nest/src/modules/student-assignment/student-assignment.service.ts` | `deleteAllByAssignment()` sudah ada (line ~488). |
| `smartbag/api_nest/src/modules/student-assignment/student-assignment.controller.ts` | Perlu tambah route `DELETE /assignment/:assignmentId`. |
| `smartbag/bbs/client-teacher/src/actions/fromApi.js` | Perlu tambah `deleteAllStudentAssignmentsByAssignment()`. |
| `smartbag/bbs/client-teacher/src/views/lessonBuilder/assignments/AssignmentDetailsSubmissions.jsx` | Perlu tambah tombol + `handleUnassignAll()`. |
| `smartbag/bbs/client-teacher/src/actions/makeApiRequest.js` | Sumber caveat redux (empty `data: []` MERGE tidak drop stale rows). |
