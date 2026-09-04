# API Contract — Unassign All Submissions (Bulk Remove StudentAssignments by Assignment)

> Status: DRAFT — mengikuti konvensi `api_nest` (NestJS 10, `@Controller({ version: '1' })`, response wrapper `{ data }`).
> Semua endpoint memerlukan JWT Bearer token (global `JwtAuthGuard`).
> **Base path:** global prefix `api` → URL lengkap `/api/v1/studentAssignments/...`.
> **Portal:** Teacher Portal (`client-teacher`).

## Konvensi Response

- Sukses (single / bulk): `{ "data": [...], "count": number }` — untuk bulk delete, `data` dikembalikan `[]` dan `count` = jumlah row yang di-soft-remove.
- Error: `HttpException` standar NestJS.

## Endpoint: Bulk Unassign All Submissions for an Assignment

### `DELETE /v1/studentAssignments/assignment/:assignmentId`

Soft-delete **semua** `StudentAssignment` yang terikat pada `assignmentId` tersebut. Tidak menghapus baris `assignment` itu sendiri, tidak menghapus `student`, tidak menyentuh `grade`/`answer` (karena row `StudentAssignment` ikut ter-soft-delete, data grade/answer di dalamnya ikut "tertutup").

**Path params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `assignmentId` | int | yes | ID assignment yang ingin di-unassign seluruh submission-nya. Divalidasi `ParseIntPipe`. |

**Request body:** none (empty / `null`).

**Response 200 (sukses):**

```json
{
  "data": [],
  "count": 28
}
```

`count` = jumlah `StudentAssignment` yang di-soft-remove. Jika tidak ada baris yang cocok, `count: 0` dengan status tetap 200.

**Response 404 (assignment tidak ditemukan):**

```json
{
  "statusCode": 404,
  "message": "NoAssignmentFoundError"
}
```

**Response 403 (jika route-level teacher guard ditambahkan):**

```json
{
  "statusCode": 403,
  "message": "ForbiddenAccessError"
}
```

## Implementasi Backend (referensi)

- **Service sudah ada:** `StudentAssignmentService.deleteAllByAssignment(assignmentId): Promise<number>`
  - Cek assignment ada (`NoAssignmentFoundError` jika tidak).
  - `StudentAssignment.find({ where: { assignment: { id } } })`.
  - Jika `length === 0` → return `0` (tidak error).
  - `StudentAssignment.softRemove(rows)` → return `rows.length`.
- **Yang kurang:** route di `StudentAssignmentController`. Tambahkan:

  ```ts
  @Delete('/assignment/:assignmentId')
  async deleteAllByAssignment(
    @Param('assignmentId', ParseIntPipe) assignmentId: number,
  ) {
    const count = await this.studentAssignmentService.deleteAllByAssignment(assignmentId);
    return { data: [], count };
  }
  ```

  Harus didaftarkan **sebelum** `@Delete(':id')` agar tidak shadowing.

## ⚠️ Frontend Redux Caveat (penting)

Alur action di `bbs/client-teacher/src/actions/makeApiRequest.js` (via `makeApiRequestThunk`):

- REDUX dispatch tipe derived dari `body.data` / `body.included`.
- Bulk delete yang mengembalikan `data: []` akan memicu `studentAssignment/merge` dengan map kosong → **reducer TIDAK menghapus baris yang sudah ada di store**.
- **Konsekuensi:** UI tetap menampilkan baris lama (stale) setelah bulk delete berhasil, kecuali list di-refresh secara eksplisit.
- **Solusi wajib:** setelah dispatch bulk action, panggil `studentAssignmentsApi?.refresh()` (atau re-dispatch `getStudentAssignments` untuk assignment ini) agar daftar bersih.

## Payload Frontend (action layer)

Tambahkan di `bbs/client-teacher/src/actions/fromApi.js`, meniru `deleteStudentAssignment`:

```js
deleteAllStudentAssignmentsByAssignment(assignmentId) {
  return makeApiRequestThunk(
    HTTP_METHODS.DELETE,
    `/studentAssignments/assignment/${assignmentId}`,
    null,
    ACTION_TYPES.MERGE
  );
}
```

Di view `AssignmentDetailsSubmissions.jsx`:

```js
function handleUnassignAll() {
  bbsConfirm({
    message:
      "Are you sure you want to unassign ALL students from this assignment? " +
      "This will remove every student submission for this assignment.",
    onConfirm: () => {
      dispatch(fromApi.deleteAllStudentAssignmentsByAssignment(assignmentId));
      // wajib: refresh agar redux store bersih dari stale rows
      studentAssignmentsApi?.refresh();
    }
  });
}
```
